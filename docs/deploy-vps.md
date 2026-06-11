# Manifeste de Déploiement Odysseus pour un nouveau VPS

Ce guide (ou manifeste) regroupe toutes les commandes nécessaires pour déployer Odysseus-MyAI sur un serveur virtuel (VPS) Linux vierge (Ubuntu/Debian), **sans utiliser Coolify**, de façon totalement autonome et sécurisée.

---

## 1. Prérequis système
Assurez-vous d'être connecté à votre VPS en SSH (`ssh root@ip-du-vps`).

Mettez à jour le système :
```bash
apt-get update && apt-get upgrade -y
```

## 2. Installation de Docker et Docker Compose
Si Docker n'est pas encore installé sur le serveur, exécutez le script d'installation officiel de Docker :

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh
```

## 3. Cloner le dépôt Odysseus
Récupérez le code source depuis votre dépôt GitHub configuré :

```bash
git clone https://github.com/willfried200k/Odysseus2.git /opt/odysseus
cd /opt/odysseus
```

## 4. Configuration (Variables d'environnement)
Odysseus utilise un fichier `.env` pour stocker vos préférences locales. Copiez le fichier d'exemple et personnalisez-le :

```bash
cp .env.example .env
```

Ouvrez le fichier `.env` avec un éditeur de texte (ex: `nano .env`) et assurez-vous d'avoir au minimum ces valeurs :

```env
# Sécurité de base
AUTH_ENABLED=true
LOCALHOST_BYPASS=false
SECURE_COOKIES=true

# Nom d'utilisateur administrateur de votre choix
ODYSSEUS_ADMIN_USER=admin

# (Optionnel) Pour autoriser votre domaine
ALLOWED_ORIGINS=https://votre-domaine.com
```

## 5. Exposer avec un nom de domaine via Caddy (Reverse Proxy)
Pour rendre l'application accessible sur Internet avec le HTTPS automatique (comme le faisait Coolify), nous allons ajouter un petit conteneur **Caddy** très léger en façade.

Créez un fichier appelé `docker-compose.override.yml` dans le dossier `/opt/odysseus` :
```bash
nano docker-compose.override.yml
```
Et collez ceci à l'intérieur (en remplaçant `votre-domaine.com` par votre vrai domaine pointant vers l'IP du VPS) :

```yaml
services:
  caddy:
    image: caddy:2-alpine
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - caddy_data:/data
      - caddy_config:/config
    command: caddy reverse-proxy --from https://votre-domaine.com --to http://odysseus:7000

volumes:
  caddy_data:
  caddy_config:
```

## 6. Lancement
Il ne vous reste plus qu'à démarrer toute la pile (Odysseus + SearXNG + ChromaDB + Caddy) en arrière-plan :

```bash
docker compose up -d --build
```

## 7. Première Connexion
1. Patientez 1 à 2 minutes le temps que tout démarre.
2. Allez lire les logs d'Odysseus pour récupérer le **mot de passe temporaire** de l'administrateur généré au premier démarrage :
   ```bash
   docker compose logs odysseus
   ```
3. Ouvrez votre navigateur sur `https://votre-domaine.com`.
4. Connectez-vous avec `admin` (ou votre `ODYSSEUS_ADMIN_USER`) et le mot de passe temporaire.
5. Allez dans les paramètres pour modifier ce mot de passe.

> [!TIP]
> **Maintenance** : Pour mettre à jour l'application à l'avenir, rendez-vous dans le dossier `/opt/odysseus` et tapez simplement :
> `git pull && docker compose up -d --build`
