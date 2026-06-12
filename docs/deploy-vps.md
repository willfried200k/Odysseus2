# Manifeste de Déploiement Odysseus pour VPS

Ce guide regroupe les instructions pour déployer Odysseus sur un serveur virtuel (VPS) de deux manières : avec **Coolify** (méthode recommandée et automatisée), ou **manuellement** de façon autonome.

---

## 1. Déploiement via Coolify v4 (Recommandé)

Coolify simplifie considérablement la gestion du serveur, des certificats SSL (HTTPS) et des mises à jour. Voici la configuration exacte pour qu'Odysseus fonctionne parfaitement :

### A. Création du projet
1. Dans Coolify, créez une nouvelle ressource de type **Docker Compose** basée sur votre dépôt GitHub.
2. Choisissez la branche `main`.

### B. Configuration du Domaine et du Port (Crucial)
Par défaut, Coolify tente de router le trafic web vers le port 80. Comme Odysseus tourne sur le port 7000, vous devez **spécifier ce port directement dans le nom de domaine**.
* Allez dans l'onglet **Configuration** de votre service `odysseus`.
* Dans le champ **Domains**, entrez votre domaine suivi de `:7000`. Par exemple :
  👉 `https://odysseus.votre-domaine.com:7000`
*(Coolify retirera le `:7000` pour les visiteurs extérieurs mais saura qu'il doit envoyer le trafic interne sur la bonne porte).*

### C. Variables d'Environnement
Allez dans l'onglet **Environment Variables** et ajoutez cette liste (c'est indispensable pour le réseau, la sécurité et le premier compte admin) :

```env
# 1. Configuration réseau
ALLOWED_ORIGINS=https://odysseus.votre-domaine.com
APP_BIND=0.0.0.0
APP_PORT=7000

# 2. Sécurité
AUTH_ENABLED=true
SECURE_COOKIES=true
LOCALHOST_BYPASS=false

# 3. Contournement d'un bug de Coolify avec SearXNG
SEARXNG_SECRET=b29a4d6e8f1c3a5b7d9e2f4c6a8b0d2e4f6c8a0b2d4e6f8c0a2b4d6e8f0c2a4

# 4. Premier administrateur (personnalisez ces valeurs)
ODYSSEUS_ADMIN_USER=admin
ODYSSEUS_ADMIN_PASSWORD=MonMotDePasseSecret123
```

### D. Lancement
1. Sauvegardez tout.
2. Cliquez sur **Deploy** (ou Redeploy).
3. Patientez 1 à 2 minutes, puis rendez-vous sur votre domaine. Connectez-vous avec l'utilisateur et le mot de passe que vous avez choisis dans les variables d'environnement !

---

## 2. Déploiement Manuel sur VPS Vierge (Sans Coolify)

Ce guide regroupe toutes les commandes nécessaires pour déployer Odysseus de façon totalement autonome en ligne de commande.

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
