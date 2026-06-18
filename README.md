# CESIZen — Guide d'installation sur la VM Azure

Ce guide couvre la configuration complète de la VM Azure jusqu'au lancement des environnements staging et production, depuis zéro.

---

## 1. La VM Azure

### Caractéristiques actuelles

| Paramètre | Valeur |
|-----------|--------|
| IP publique | `20.199.139.17` |
| Utilisateur | `azureuser` |
| OS | Ubuntu 22.04 LTS |
| Authentification | Clé `.pem` (dans `~/Downloads`) |

### Ports à ouvrir dans le NSG (Network Security Group)

Dans le portail Azure → Réseau → Règles de sécurité entrantes, ajouter :

| Priorité | Nom | Port | Protocole | Source | Description |
|----------|-----|------|-----------|--------|-------------|
| 300 | SSH | 22 | TCP | Votre IP (ou `*` en dernier recours) | Administration |
| 310 | HTTP | 80 | TCP | `*` | Prod (Traefik + redirection HTTPS) |
| 320 | HTTPS | 443 | TCP | `*` | Prod (TLS Let's Encrypt) |
| 330 | Staging | 8080 | TCP | `*` | Accès staging (protégé par BasicAuth) |
| 340 | Monitoring | 3001 | TCP | Votre IP | Uptime Kuma (protégé par son propre login) |

> **Conseil sécurité** : restreindre les ports 22 et 3001 à votre adresse IP plutôt qu'à `*`.

---

## 2. DNS — Label sur l'IP publique Azure

Le certificat Let's Encrypt en production nécessite un nom de domaine pointant vers la VM.

1. Dans le portail Azure, aller sur la ressource **Adresse IP publique** de la VM.
2. Cliquer sur **Configuration**.
3. Dans le champ **Étiquette du nom DNS**, saisir `cesizen`.
4. Sauvegarder → le FQDN généré sera :

```
cesizen.switzerlandnorth.cloudapp.azure.com
```

Ce domaine est celui utilisé dans la variable `DOMAIN` du `.env` de production.

---

## 3. Configuration SSH locale

Ajouter ceci dans `~/.ssh/config` sur votre machine :

```
Host cesizen
    HostName 20.199.139.17
    User azureuser
    IdentityFile ~/Downloads/<votre-clé>.pem
    IdentitiesOnly yes
```

Tester la connexion :

```bash
ssh cesizen
```

---

## 4. Sécurisation SSH sur la VM

Se connecter puis éditer la configuration SSH :

```bash
sudo nano /etc/ssh/sshd_config
```

Vérifier / modifier ces directives :

```
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
```

Redémarrer le service :

```bash
sudo systemctl restart ssh
```

---

## 5. Installation de Fail2ban

Fail2ban bloque automatiquement les IP qui échouent trop de tentatives SSH.

```bash
sudo apt update && sudo apt install -y fail2ban
```

Créer une configuration locale pour la jail SSH :

```bash
sudo nano /etc/fail2ban/jail.d/sshd.local
```

```ini
[sshd]
enabled  = true
port     = ssh
maxretry = 5
bantime  = 1h
findtime = 10m
```

Activer et démarrer :

```bash
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

Vérifier le statut :

```bash
sudo fail2ban-client status sshd
```

---

## 6. Installation de Docker sur la VM

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg

sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

Permettre à `azureuser` d'utiliser Docker sans `sudo` :

```bash
sudo usermod -aG docker azureuser
newgrp docker
```

---

## 7. Création des dossiers de déploiement

```bash
sudo mkdir -p /opt/cesizen/staging
sudo mkdir -p /opt/cesizen/prod
sudo chown -R azureuser:azureuser /opt/cesizen
```

---

## 8. Copie des fichiers Docker Compose

Depuis votre machine locale, dans le dossier `cesizen-cd/` :

**Staging :**

```bash
scp docker-compose.yml docker-compose.staging.yml cesizen:/opt/cesizen/staging/
```

**Production :**

```bash
scp docker-compose.yml docker-compose.prod.yml cesizen:/opt/cesizen/prod/
```

---

## 9. Création des fichiers `.env`

### 9.1 Staging — `/opt/cesizen/staging/.env`

Se connecter sur la VM puis :

```bash
nano /opt/cesizen/staging/.env
```

```dotenv
# Base de données
POSTGRES_USER=postgres
POSTGRES_PASSWORD=<mot-de-passe-fort>
POSTGRES_DB=cesiZen

# JWT
JWT_SECRET=<secret-long-aléatoire>
JWT_REFRESH_SECRET=<secret-long-aléatoire>
JWT_RESET_PASSWORD_SECRET=<secret-long-aléatoire>

# Email (Mailtrap)
MAILTRAP_USER=<identifiant-mailtrap>
MAILTRAP_PASS=<mot-de-passe-mailtrap>

# URLs (staging : l'IP suffit car pas de domaine)
FRONTEND_URL=http://20.199.139.17:8080
CORS_ALLOWED_ORIGINS=http://20.199.139.17:8080
API_URL=http://20.199.139.17:8080

# Staging
NODE_ENV=production
STAGING_PORT=8080
API_BASE_PATH=

# BasicAuth Traefik (générer avec : htpasswd -nb <user> <password>)
# Échapper les $ en $$ dans la valeur
STAGING_BASIC_AUTH_USERS=<user>:$$apr1$$...

# Seed auto au démarrage (true pour pré-remplir la BDD en staging)
AUTO_SEED=true

# Tag de l'image Docker à déployer
TAG=develop
```

> **Générer `STAGING_BASIC_AUTH_USERS`** (à faire sur votre machine, nécessite `apache2-utils`) :
> ```bash
> htpasswd -nb admin <mot-de-passe>
> # Puis doubler chaque $ dans la sortie avant de coller dans .env
> ```

### 9.2 Production — `/opt/cesizen/prod/.env`

```bash
nano /opt/cesizen/prod/.env
```

```dotenv
# Base de données
POSTGRES_USER=postgres
POSTGRES_PASSWORD=<mot-de-passe-fort-différent>
POSTGRES_DB=cesiZen

# JWT
JWT_SECRET=<secret-long-aléatoire>
JWT_REFRESH_SECRET=<secret-long-aléatoire>
JWT_RESET_PASSWORD_SECRET=<secret-long-aléatoire>

# Email (Mailtrap ou SMTP réel)
MAILTRAP_USER=<identifiant>
MAILTRAP_PASS=<mot-de-passe>

# URLs (domaine Azure)
DOMAIN=cesizen.switzerlandnorth.cloudapp.azure.com
FRONTEND_URL=https://cesizen.switzerlandnorth.cloudapp.azure.com
CORS_ALLOWED_ORIGINS=https://cesizen.switzerlandnorth.cloudapp.azure.com
API_URL=https://cesizen.switzerlandnorth.cloudapp.azure.com

# Let's Encrypt
ACME_EMAIL=<votre-email>

# Production
NODE_ENV=production
API_BASE_PATH=/api
AUTO_SEED=false

# Tag de l'image Docker à déployer
TAG=main
```

---

## 10. Premier lancement

### Staging

```bash
cd /opt/cesizen/staging
docker compose -f docker-compose.yml -f docker-compose.staging.yml up -d
```

### Production

```bash
cd /opt/cesizen/prod
docker compose -f docker-compose.yml -f docker-compose.prod.yml --profile production up -d
```

Vérifier que tous les conteneurs sont `Up` :

```bash
docker compose ps
```

---

## 11. Monitoring — Uptime Kuma

Uptime Kuma tourne en conteneur Docker standalone, indépendamment des stacks staging/prod.

### Lancement

```bash
docker run -d --restart=always -p 3001:3001 -v uptime-kuma:/app/data --name uptime-kuma louislam/uptime-kuma:2
```

Ouvrir `http://20.199.139.17:3001` et créer le compte admin au premier lancement.

### Configurer les notifications Discord

1. Dans Discord : **Paramètres du serveur → Intégrations → Webhooks → Nouveau webhook**, choisir le salon, copier l'URL.
2. Dans Uptime Kuma : **Settings → Notifications → Add Notification**
   - Type : **Discord**
   - Webhook URL : coller l'URL copiée
   - Cliquer **Test** pour vérifier, puis sauvegarder.

---

## Récapitulatif des URLs

| Environnement | URL |
|---------------|-----|
| Staging | `http://20.199.139.17:8080` (BasicAuth) |
| Production | `https://cesizen.switzerlandnorth.cloudapp.azure.com` |
| API Swagger (staging) | `http://20.199.139.17:8080/api-docs` (BasicAuth) |
| Monitoring | `http://20.199.139.17:3001` (login Uptime Kuma) |
