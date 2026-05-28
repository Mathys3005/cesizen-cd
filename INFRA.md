# Infrastructure CesiZen — Nginx, Certbot & Docker

## Table des matières
1. [Vue d'ensemble](#vue-densemble)
2. [Dockerfile du frontend](#dockerfile-du-frontend)
3. [Docker Compose](#docker-compose)
4. [Configuration nginx (reverse proxy)](#configuration-nginx-reverse-proxy)
5. [Certbot — certificat Let's Encrypt](#certbot--certificat-lets-encrypt)
6. [Variables d'environnement](#variables-denvironnement)
7. [Problèmes rencontrés et solutions](#problèmes-rencontrés-et-solutions)
8. [Schéma d'architecture](#schéma-darchitecture)

---

## Vue d'ensemble

Le projet tourne sur une VM Azure (Switzerland North) avec deux environnements Docker indépendants sur la même machine :

| Environnement | Dossier VM       | Tag image  | Ports exposés         |
|---------------|-----------------|------------|-----------------------|
| Staging       | `~/cesizen`      | `develop`  | API :3000, Web :5173  |
| Production    | `~/cesizen-prod` | `main`     | HTTP :80, HTTPS :443  |

En production, un nginx reverse proxy unifie tout sous un seul domaine HTTPS.

---

## Dockerfile du frontend

**Fichier :** `cesiZenWeb/Dockerfile`

Le frontend utilise un build **multi-stage** pour produire une image légère :

### Stage 1 — Build (`builder`)

```dockerfile
FROM node:24-alpine AS builder
WORKDIR /app

COPY package*.json ./
RUN npm ci          # install des dépendances (plus strict que npm install)

COPY . .

ARG VITE_API_BASE_URL
ENV VITE_API_BASE_URL=$VITE_API_BASE_URL

ARG VITE_IMAGES_BASE_URL
ENV VITE_IMAGES_BASE_URL=$VITE_IMAGES_BASE_URL

RUN npm run build
```

**Point critique :** Vite injecte `VITE_API_BASE_URL` dans le bundle JavaScript **au moment du build**, pas à l'exécution. Changer cette variable sans rebuilder l'image n'a aucun effet.

La valeur est passée via `--build-arg` dans le CI (GitHub Actions) depuis les **Variables d'environnement GitHub** (Settings → Environments → production → Variables).

Valeur correcte pour la production :
```
VITE_API_BASE_URL=https://cesizen.switzerlandnorth.cloudapp.azure.com/api
```

### Stage 2 — Serve (`nginx`)

```dockerfile
FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
```

L'image finale ne contient que nginx + les fichiers statiques compilés par Vite. Le `nginx.conf` embarqué dans l'image gère le routing SPA :

```nginx
server {
    listen 80;
    root /usr/share/nginx/html;

    location / {
        try_files $uri $uri/ /index.html;  # React Router gère le reste
    }

    location /assets/ {
        expires 1y;
        add_header Cache-Control "public, immutable";  # assets hashés par Vite
    }
}
```

---

## Docker Compose

**Fichier :** `cesizen-cd/docker-compose.yml`

### Service `db` — PostgreSQL

```yaml
db:
  image: postgres:16-alpine
  restart: unless-stopped
  env_file: .env            # POSTGRES_USER, POSTGRES_PASSWORD, POSTGRES_DB
  volumes:
    - pgdata:/var/lib/postgresql/data   # données persistantes
  networks:
    - backend                           # non exposé à l'extérieur
```

La base n'a pas de `ports:` — elle est uniquement accessible depuis le réseau interne `backend`.

### Service `api` — Backend NestJS

```yaml
api:
  image: ghcr.io/mathys3005/cesizenapi/cesizen-api:${TAG:-develop}
  ports:
    - "${API_PORT:-3000}:3000"    # staging: 3000, prod: 3001
  env_file: .env
  environment:
    DATABASE_URL: postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@db:5432/${POSTGRES_DB}
  depends_on:
    - db
  networks:
    - backend
```

`${TAG:-develop}` : utilise la variable `TAG` si définie, sinon `develop`. Le CI passe `TAG=main` pour la production.

### Service `web` — Frontend React

```yaml
web:
  image: ghcr.io/mathys3005/cesizenweb/cesizen-web:${TAG:-develop}
  ports:
    - "${FRONTEND_PORT:-5173}:80"   # staging: 5173, prod: 8080
  networks:
    - backend
```

En production `FRONTEND_PORT=8080` est défini dans le `.env` pour libérer le port 80 pour nginx.

### Service `nginx` — Reverse proxy (production uniquement)

```yaml
nginx:
  image: nginx:alpine
  profiles: ["production"]          # démarre SEULEMENT avec COMPOSE_PROFILES=production
  ports:
    - "80:80"
    - "443:443"
  volumes:
    - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro   # config montée directement
    - /etc/letsencrypt:/etc/letsencrypt:ro             # certificats Let's Encrypt
    - /var/www/certbot:/var/www/certbot:ro             # challenge ACME
  depends_on:
    - api
    - web
  networks:
    - backend
```

**Pourquoi monter directement en `conf.d/default.conf` ?**  
La configuration initiale utilisait `envsubst` pour injecter `$DOMAIN` via un entrypoint custom. En pratique, cette variable n'est pas utilisée dans la config Phase 1/2 (`server_name` est statique), et l'entrypoint écrivait un fichier vide à cause d'un problème de parsing YAML. Le montage direct est plus simple et fiable.

### Démarrage

```bash
# Staging (sans nginx)
docker compose up -d

# Production (avec nginx)
COMPOSE_PROFILES=production TAG=main docker compose up -d
```

---

## Configuration nginx (reverse proxy)

**Fichier :** `cesizen-cd/nginx.conf`

### Phase 1 — HTTP uniquement (avant certificat)

Utilisée pour que Certbot puisse valider le domaine via le challenge ACME `.well-known/acme-challenge/`.

### Phase 2 — HTTPS + redirect HTTP→HTTPS (configuration actuelle)

```nginx
# Bloc 1 : HTTP → redirige vers HTTPS, sauf pour le renouvellement Certbot
server {
    listen 80;
    server_name cesizen.switzerlandnorth.cloudapp.azure.com;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;      # challenge Let's Encrypt (renouvellement auto)
    }

    location / {
        return 301 https://$host$request_uri;   # redirect permanent vers HTTPS
    }
}

# Bloc 2 : HTTPS — routing vers les services internes
server {
    listen 443 ssl;
    server_name cesizen.switzerlandnorth.cloudapp.azure.com;

    ssl_certificate     /etc/letsencrypt/live/cesizen.../fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/cesizen.../privkey.pem;

    # /api/* → API (le préfixe /api est retiré avant d'envoyer à l'API)
    location /api/ {
        proxy_pass http://api:3000/;
    }

    # /api-docs → Swagger UI
    location /api-docs {
        proxy_pass http://api:3000/api-docs;
    }

    # /images/* → fichiers statiques servis par l'API
    location /images/ {
        proxy_pass http://api:3000/images/;
    }

    # /* → frontend React (catch-all)
    location / {
        proxy_pass http://web:80/;
    }
}
```

**Pourquoi `/api/` avec slash de fin dans `proxy_pass` ?**
`proxy_pass http://api:3000/` retire le préfixe `/api` de l'URL avant de la transmettre.
Exemple : `/api/articles` → `http://api:3000/articles` (pas `/api/articles`).

---

## Certbot — certificat Let's Encrypt

### Obtention du certificat (déjà fait)

```bash
sudo apt install certbot
sudo mkdir -p /var/www/certbot

sudo certbot certonly --webroot \
  -w /var/www/certbot \
  -d cesizen.switzerlandnorth.cloudapp.azure.com \
  --email megamaniac27@gmail.com \
  --agree-tos --non-interactive
```

Le certificat est stocké dans `/etc/letsencrypt/live/cesizen.switzerlandnorth.cloudapp.azure.com/` et monté en lecture seule dans le container nginx.

**Certificat actuel :**
- Expiration : 2026-08-25
- Type : ECDSA

### Renouvellement automatique

Certbot installe un timer systemd qui renouvelle automatiquement. Pour vérifier :

```bash
sudo certbot renew --dry-run
```

Après renouvellement, recharger nginx :

```bash
docker exec cesizen-prod-nginx-1 nginx -s reload
```

---

## Variables d'environnement

### `cesizen-cd/.env` (sur la VM, jamais commité)

| Variable          | Staging      | Production   | Description                    |
|-------------------|-------------|-------------|--------------------------------|
| `TAG`             | `develop`   | `main`      | Tag des images Docker          |
| `FRONTEND_PORT`   | `5173`      | `8080`      | Port hôte du frontend          |
| `API_PORT`        | `3000`      | `3001`      | Port hôte de l'API             |
| `POSTGRES_USER`   | —           | —           | Utilisateur PostgreSQL         |
| `POSTGRES_PASSWORD` | —         | —           | Mot de passe PostgreSQL        |
| `POSTGRES_DB`     | —           | —           | Nom de la base                 |

### Variables GitHub Actions (Settings → Environments)

| Variable              | Valeur production                                              |
|-----------------------|----------------------------------------------------------------|
| `VITE_API_BASE_URL`   | `https://cesizen.switzerlandnorth.cloudapp.azure.com/api`     |
| `VITE_IMAGES_BASE_URL`| `https://cesizen.switzerlandnorth.cloudapp.azure.com/images`  |

Ces variables sont injectées au `docker build` via `--build-arg` et baked dans le bundle JS.

---

## Problèmes rencontrés et solutions

### 1. `VITE_API_BASE_URL` sans préfixe `/api`
**Symptôme :** Le frontend appelait `/auth/login` au lieu de `/api/auth/login`.  
**Cause :** La variable ne contenait pas le suffixe `/api`.  
**Fix :** Mettre `https://domaine/api` dans les variables GitHub Actions et rebuilder l'image.

### 2. Nginx sans ports publiés
**Symptôme :** `docker port cesizen-prod-nginx-1` retournait vide, port 80 inaccessible.  
**Cause :** Le container nginx avait été créé quand le port 80 était déjà pris par `web` (`FRONTEND_PORT=80`).  
**Fix :** Changer `FRONTEND_PORT=8080` dans `.env`, puis `docker compose up -d --force-recreate nginx`.

### 3. `default.conf` vide dans nginx
**Symptôme :** Le template nginx.conf était imprimé dans les logs mais jamais écrit dans le container.  
**Cause :** L'entrypoint `envsubst` avec YAML `>` (folded scalar) ne gérait pas correctement la redirection shell `>`.  
**Fix :** Monter `nginx.conf` directement comme `/etc/nginx/conf.d/default.conf` sans entrypoint custom.

### 4. 502 après recréation de containers
**Symptôme :** nginx retournait 502 après le redémarrage d'un container upstream.  
**Cause :** nginx résout le DNS des upstreams au démarrage et cache les IPs. Quand un container redémarre et change d'IP, nginx garde l'ancienne.  
**Fix :** `docker exec cesizen-prod-nginx-1 nginx -s reload` pour forcer la re-résolution DNS.

---

## Schéma d'architecture

```
                         INTERNET
                            │
              ┌─────────────┴─────────────┐
              │                           │
           :80 (HTTP)                :443 (HTTPS)
              │                           │
              └─────────────┬─────────────┘
                            │
                    ┌───────▼────────┐
                    │  nginx:alpine  │  cesizen-prod-nginx-1
                    │  (prod only)   │  ports: 80, 443
                    └───────┬────────┘
                            │
            ┌───────────────┼────────────────┐
            │               │                │
     /api/* ▼        /images/* ▼      /* (catch-all) ▼
  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐
  │  api:3000    │  │  api:3000    │  │    web:80        │
  │  (NestJS)    │  │  /images/    │  │  (React + nginx) │
  │              │  └──────────────┘  │  SPA → index.html│
  │  cesizen-    │                    │                  │
  │  prod-api-1  │                    │  cesizen-        │
  │  :main       │                    │  prod-web-1      │
  └──────┬───────┘                    │  :main           │
         │                            └──────────────────┘
         │ DATABASE_URL
         ▼
  ┌──────────────┐
  │  db:5432     │
  │  (PostgreSQL)│
  │  cesizen-    │
  │  prod-db-1   │
  └──────────────┘


  Réseau Docker interne : cesizen-prod_backend (bridge)
  ┌─────────────────────────────────────────────────┐
  │  nginx  ←→  api  ←→  db                        │
  │         ←→  web                                 │
  └─────────────────────────────────────────────────┘

  Volumes :
  ┌─────────────────────────────────────────────────┐
  │  pgdata          → données PostgreSQL           │
  │  /etc/letsencrypt → certificats TLS (Certbot)  │
  │  /var/www/certbot → challenge ACME             │
  │  ./nginx.conf    → config reverse proxy        │
  └─────────────────────────────────────────────────┘

  Build frontend (CI GitHub Actions) :
  ┌─────────────────────────────────────────────────────────┐
  │  VITE_API_BASE_URL (GitHub Variable)                    │
  │       ↓ --build-arg                                     │
  │  Dockerfile → npm run build (Vite bake l'URL dans JS)  │
  │       ↓                                                 │
  │  Image ghcr.io/.../cesizen-web:main                    │
  └─────────────────────────────────────────────────────────┘
```
