# Downloader — Déploiement conteneurisé

Déploiement Docker Compose pour **Downloader**, une plateforme self-hosted de gestion de médias (Symfony & Python, image Docker unique) qui gère les téléchargements vidéo/torrents via Alldebrid, un pipeline musical complet de Spotify à la bibliothèque, la découverte TMDB/Stremio, les automatisations NAS et une passerelle Jellyfin/Nuvio.

Ce dépôt ne contient que les fichiers de déploiement. Le code source et la documentation complète se trouvent dans le dépôt de l'application.

| Ressource | Lien |
| --------- | ---- |
| Code source & documentation de l'application | https://github.com/Gautardos/downloader |
| Déploiement conteneurisé (ce dépôt) | https://github.com/Gautardos/docker-downloader |
| Image Docker | `gautardos/downloader-app` |

> [!TIP]
> Pour les fonctionnalités, les tableaux de configuration et les captures d'écran, consultez le [README de l'application](https://github.com/Gautardos/downloader/blob/main/README.fr.md). Un guide intégré est aussi disponible sur `/automation/help`.

---

## ✨ Contenu

- **Vidéos & torrents** — magnets, fichiers `.torrent`, liens directs, recherche SQLite de torrents, regroupement en packs et renommage assisté par IA.
- **Discover** — recherche TMDB, addons Stremio/Nuvio comme sources de streams, import de collections Nuvio et débridage en un clic.
- **Musique** — recherche Spotify, téléchargements Zotify, Music Library unifiée, éditeur de tags, récupération/audit des lyrics, lecteur audio.
- **Automations** — index de la bibliothèque NAS, résolution TMDB, renommage Jellyfin, synchronisation Scrob/MDBList, profil qualité et téléchargements automatiques des films/épisodes manquants.
- **Jellyfin / Nuvio** — API compatible Jellyfin en lecture seule et direct-play sous `/jellyfin`.
- **Système** — workers de téléchargement et d'automation, tracker temps réel, historique, authentification/CSRF, fournisseur IA OpenRouter/xAI.

---

## 📸 Captures d'écran

| Music Library | Automations — Library Browser |
| ------------- | ----------------------------- |
| ![Music Library](https://raw.githubusercontent.com/Gautardos/downloader/main/screenshots/v2.7/01-music-library-overview.png) | ![Library Browser](https://raw.githubusercontent.com/Gautardos/downloader/main/screenshots/v2.6/04-automations-library-folders.png) |

| Discover | Automations — Downloads |
| -------- | ----------------------- |
| ![Collections Discover](https://raw.githubusercontent.com/Gautardos/downloader/main/screenshots/nouveautes/05-discover-collections.png) | ![Téléchargements automation](https://raw.githubusercontent.com/Gautardos/downloader/main/screenshots/v2.6/07-automations-downloads.png) |

---

## 🐳 Installation (Linux)

### Prérequis

- **Docker** et **Docker Compose v2**.
- Un compte **Alldebrid** pour les fonctionnalités vidéo/torrents.
- Un compte développeur **Spotify** et une session Spotify pour la musique.
- Optionnel : clé **TMDB** (Discover), clé **OpenRouter/xAI** (IA), identifiants **Scrob/MDBList** (Automations), un client **Jellyfin/Nuvio**.

> [!IMPORTANT]
> Le fichier Compose utilise `network_mode: host`. Sous Docker Desktop pour Windows, cela nécessite **Docker Desktop ≥ 4.34** avec des conteneurs Linux (voir [Windows](#-windows) ci-dessous).

### 1. Cloner ce dépôt

```bash
git clone https://github.com/Gautardos/docker-downloader.git ~/downloader-app
cd ~/downloader-app
```

### 2. Créer les répertoires de données

```bash
mkdir -p var/{db,home,storage}
mkdir -p downloads music nas
```

- `var/db/` — bases SQLite (recherche torrents, index d'automation).
- `var/home/` — home persistant (credentials Spotify, archive Zotify).
- `var/storage/` — configuration, file d'attente, historique, logs et caches.
- `downloads/`, `music/`, `nas/` — points de montage médias. Adaptez-les à vos disques ou partages NFS/SMB.

### 3. Configurer `.env`

```bash
cp .env.example .env
```

```env
IMAGE_TAG=latest
PORT=8000
DATA_DIR=./var
DOWNLOADS_DIR=./downloads
MUSIC_DIR=./music
NAS_DIR=./nas
```

Utilisez une version explicite (par exemple `IMAGE_TAG=2.7.0`) pour un déploiement reproductible. Ce sont des chemins **hôte** montés vers des chemins conteneur fixes.

> [!NOTE]
> L'image sert le HTTP sur le port **8000**. Conservez `PORT=8000` sauf si vous personnalisez l'image ; avec `network_mode: host`, l'application est accessible sur ce port.

### 4. Créer le secret applicatif

Les secrets ne doivent jamais être stockés dans le fichier Compose ni dans `.env`. Créez `var/.env.local` :

```bash
printf 'APP_ENV=prod\nAPP_DEBUG=0\nAPP_SECRET=%s\n' "$(openssl rand -hex 32)" > var/.env.local
```

Il est monté en lecture seule sur `/var/www/html/.env.local`.

### 5. Démarrer le conteneur

```bash
docker compose up -d
```

L'application est accessible à l'adresse **`http://<votre-ip>:8000`**.

---

## 📄 Référence du fichier Compose

```yaml
services:
  downloader:
    image: gautardos/downloader-app:${IMAGE_TAG:-latest}
    container_name: downloader-container
    network_mode: host
    environment:
      - PORT=${PORT:-8000}
    volumes:
      - ${DATA_DIR:-./var}/.env.local:/var/www/html/.env.local:ro
      - ${DATA_DIR:-./var}/db:/var/www/html/var/db
      - ${DATA_DIR:-./var}/home:/var/www/html/var/home
      - ${DATA_DIR:-./var}/storage:/var/www/html/var/storage
      - ${DOWNLOADS_DIR:-./downloads}:/downloads
      - ${MUSIC_DIR:-./music}:/music
      - ${NAS_DIR:-./nas}:/nas
    restart: unless-stopped
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```

---

## 🪟 Windows

> [!IMPORTANT]
> **`network_mode: host` nécessite Docker Desktop ≥ 4.34** (sorti en octobre 2024).
> Il était disponible en **bêta** dans les versions 4.29 – 4.33 et devait être activé manuellement.
> Il fonctionne uniquement avec les **conteneurs Linux** (le mode par défaut de Docker Desktop).

### 1. Vérifier la version de Docker Desktop

Ouvrez Docker Desktop → **Settings → About** et vérifiez que la version est **4.34 ou ultérieure**.

Si vous utilisez une version plus ancienne, deux options :

- **Mettez à jour Docker Desktop** depuis [docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop/).
- Ou n'utilisez pas `network_mode: host` et passez au mapping de ports (voir *Alternative sans host networking* ci-dessous).

### 2. Activer le host networking (Docker Desktop 4.29 – 4.33 uniquement)

Si vous êtes sur une version bêta (4.29 – 4.33), activez la fonction manuellement :

1. Dans Docker Desktop, allez dans **Settings → Resources → Network**.
2. Cochez **"Enable host networking"**.
3. Cliquez sur **Apply & restart**.

> [!NOTE]
> À partir de la version **4.34+**, la fonction est activée par défaut — aucune étape supplémentaire nécessaire.

### 3. Cloner et créer les répertoires

Ouvrez **PowerShell** :

```powershell
git clone https://github.com/Gautardos/docker-downloader.git "$HOME\downloader-app"
cd "$HOME\downloader-app"

New-Item -ItemType Directory -Force -Path `
  "$HOME\downloader-app\var\db", "$HOME\downloader-app\var\home", "$HOME\downloader-app\var\storage" | Out-Null

# Adaptez la lettre de lecteur à votre configuration
New-Item -ItemType Directory -Force -Path "D:\downloads", "D:\music", "D:\nas" | Out-Null
```

> [!TIP]
> Vous pouvez placer ces répertoires sur n'importe quel lecteur. Les chemins ci-dessus sont des exemples — adaptez-les à votre configuration.

### 4. Configurer `.env`

Copiez `.env.example` vers `.env` et utilisez des chemins Windows avec des barres obliques :

```powershell
Copy-Item .env.example .env
```

```env
IMAGE_TAG=latest
PORT=8000
DATA_DIR=C:/Users/<VotreUtilisateur>/downloader-app/var
DOWNLOADS_DIR=D:/downloads
MUSIC_DIR=D:/music
NAS_DIR=D:/nas
```

### 5. Créer le secret applicatif

```powershell
"APP_ENV=prod`nAPP_DEBUG=0`nAPP_SECRET=$([guid]::NewGuid().ToString('N') + [guid]::NewGuid().ToString('N'))`n" |
  Set-Content -NoNewline -Encoding ascii var\.env.local
```

### 6. Démarrer le conteneur

```powershell
cd "$HOME\downloader-app"
docker compose up -d
```

L'application est accessible à l'adresse **`http://localhost:8000`**.

> [!WARNING]
> Assurez-vous que les répertoires existent **avant** de démarrer le conteneur, sinon Docker les créera en tant que répertoires appartenant à root et les permissions pourraient être incorrectes.

### Alternative sans host networking

Si votre version de Docker Desktop ne prend pas en charge `network_mode: host`, remplacez-le par un mapping de ports explicite :

```yaml
services:
  downloader:
    image: gautardos/downloader-app:latest
    container_name: downloader-container
    # network_mode: host          ← supprimer ou commenter
    ports:
      - "8000:8000"               # ← utiliser le mapping de ports
    environment:
      - PORT=8000
    # ... reste de la configuration inchangé
```

> [!CAUTION]
> Avec un mapping de ports, seul le port mappé est accessible. Si l'application utilise des ports internes supplémentaires (par ex. pour `librespot-auth`), ces fonctionnalités peuvent ne pas fonctionner correctement sans `network_mode: host`.

---

## 🔐 Première connexion

| Champ        | Valeur  |
| ------------ | ------- |
| Utilisateur  | `admin` |
| Mot de passe | `admin` |

> [!TIP]
> Modifiez-les immédiatement dans **Settings → App Security**.

---

## 📂 Volumes

```
Hôte                                    → Conteneur
────────────────────────────────────────────────────────────────
<DATA_DIR>/db                           → /var/www/html/var/db         (bases SQLite)
<DATA_DIR>/home                         → /var/www/html/var/home       (credentials Spotify, archive)
<DATA_DIR>/storage                      → /var/www/html/var/storage    (config, file, historique, logs)
<DOWNLOADS_DIR>                         → /downloads                   (sortie torrent/vidéo)
<MUSIC_DIR>                             → /music                       (téléchargements musique)
<NAS_DIR>                               → /nas                         (bibliothèque NAS, lecture/écriture)
<DATA_DIR>/.env.local                   → /var/www/html/.env.local     (APP_SECRET, lecture seule)
```

> [!WARNING]
> Les chemins configurés dans l'application (Settings et Automations) sont des chemins **internes au conteneur**. Si vous montez un répertoire hôte sur `/music`, réglez **Music Download Root** sur `/music` — jamais le chemin hôte.

---

## 🏗️ Architecture

Le conteneur exécute quatre services gérés par **Supervisor** :

| Service | Rôle |
| ------- | ---- |
| **Apache** | Sert l'application Symfony sur le port 8000 |
| **Redis** | Stocke les sessions PHP (évite la contention de verrous de session) |
| **Download Worker** | Traite séquentiellement la file de téléchargement (`app:download-worker`) |
| **Automation Worker** | Exécute les scans planifiés, synchronisations de listes et intents de releases (`app:automation-worker`) |

---

## ⚙️ Configuration

Les réglages se font dans l'interface web via **Settings** (`/config`), sauf :

- **TMDB et addons** — configurés dans le tiroir Discover.
- **Automations** — configurés dans **Automations → Configuration** (stocké dans `var/db/automation.db`).
- **Secrets d'exécution** — `var/.env.local` (`APP_SECRET`, `APP_ENV`, `APP_DEBUG`).

Consultez le [README de l'application](https://github.com/Gautardos/downloader/blob/main/README.fr.md) pour la référence complète (clés API, chemins, musique, Jellyfin, configuration Spotify, configuration Discover).

---

## 🔧 Maintenance

### Consulter les logs

```bash
docker exec -it downloader-container cat /var/log/apache2_error.log
docker exec -it downloader-container cat /var/www/html/var/storage/logs/worker.log
docker exec -it downloader-container cat /var/www/html/var/storage/logs/automation_worker.log
docker exec -it downloader-container cat /var/www/html/var/storage/logs/worker_error.log
docker exec -it downloader-container cat /var/www/html/var/storage/logs/automation_worker_error.log
docker exec -it downloader-container cat /var/log/supervisord.log
```

Les logs applicatifs sont dans `var/storage/logs/` et tournent à **5 Mo par fichier avec 5 sauvegardes**.

### Redémarrer les services

```bash
# Tout le conteneur
docker compose restart

# Uniquement le download worker
docker exec -it downloader-container supervisorctl restart worker

# Uniquement l'automation worker
docker exec -it downloader-container supervisorctl restart automation-worker
```

### Mise à jour

```bash
docker compose pull
docker compose up -d
```

Pour épingler une version, définissez `IMAGE_TAG=2.7.0` dans `.env` avant le pull. Aucune migration manuelle n'est nécessaire.

### Sauvegarde & restauration

Sauvegardez `var/storage/`, `var/db/`, `var/home/` et `var/.env.local`. Le panneau **Settings → Backup & Restore** permet aussi d'exporter/importer la configuration sous forme de ZIP.

---

## 🔗 Dépôts liés

- **Application :** https://github.com/Gautardos/downloader
- **Déploiement conteneurisé :** https://github.com/Gautardos/docker-downloader
- **Release notes :** [2.6](https://github.com/Gautardos/downloader/blob/main/RELEASE_NOTES_2.6.md) · [2.7](https://github.com/Gautardos/downloader/blob/main/RELEASE_NOTES_2.7.md)
