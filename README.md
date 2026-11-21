# Downloader — Containerized Deployment

Docker Compose deployment for **Downloader**, a self-hosted media management platform (Symfony & Python, single Docker image) that handles video/torrent downloads through Alldebrid, a full Spotify-to-library music pipeline, TMDB/Stremio discovery, NAS automations and a Jellyfin/Nuvio bridge.

This repository only contains the deployment files. The application source and full documentation live in the application repository.

| Resource | Link |
| -------- | ---- |
| Application source & documentation | https://github.com/Gautardos/downloader |
| Containerized deployment (this repository) | https://github.com/Gautardos/docker-downloader |
| Docker image | `gautardos/downloader-app` |

> [!TIP]
> For features, configuration tables and screenshots, see the [application README](https://github.com/Gautardos/downloader/blob/main/README.md). An in-app guide is also available at `/automation/help`.

---

## ✨ What's included

- **Videos & torrents** — magnets, `.torrent` files, direct links, SQLite torrent search, pack grouping and AI-assisted renaming.
- **Discover** — TMDB search, Stremio/Nuvio addons as stream sources, Nuvio collections import and one-click debrid.
- **Music** — Spotify search, Zotify downloads, unified Music Library, tag editor, lyrics fetch/audit, audio player.
- **Automations** — NAS library index, TMDB resolution, Jellyfin renaming, Scrob/MDBList sync, quality profile and automatic missing movies/episodes downloads.
- **Jellyfin / Nuvio** — read-only, direct-play Jellyfin-compatible API under `/jellyfin`.
- **System** — download and automation workers, real-time tracker, history, authentication/CSRF, OpenRouter/xAI AI provider.

---

## 📸 Screenshots

| Music Library | Automations — Library Browser |
| ------------- | ----------------------------- |
| ![Music Library](https://raw.githubusercontent.com/Gautardos/downloader/main/screenshots/v2.7/01-music-library-overview.png) | ![Library Browser](https://raw.githubusercontent.com/Gautardos/downloader/main/screenshots/v2.6/04-automations-library-folders.png) |

| Discover | Automations — Downloads |
| -------- | ----------------------- |
| ![Discover collections](https://raw.githubusercontent.com/Gautardos/downloader/main/screenshots/nouveautes/05-discover-collections.png) | ![Download automations](https://raw.githubusercontent.com/Gautardos/downloader/main/screenshots/v2.6/07-automations-downloads.png) |

---

## 🐳 Installation (Linux)

### Prerequisites

- **Docker** and **Docker Compose v2**.
- An **Alldebrid** account for video/torrent features.
- A **Spotify** developer account and a Spotify session for music.
- Optional: **TMDB** key (Discover), **OpenRouter/xAI** key (AI), **Scrob/MDBList** (Automations), a **Jellyfin/Nuvio** client.

> [!IMPORTANT]
> The Compose file uses `network_mode: host`. On Docker Desktop for Windows this requires **Docker Desktop ≥ 4.34** with Linux containers (see [Windows](#-windows) below).

### 1. Clone this repository

```bash
git clone https://github.com/Gautardos/docker-downloader.git ~/downloader-app
cd ~/downloader-app
```

### 2. Create the data directories

```bash
mkdir -p var/{db,home,storage}
mkdir -p downloads music nas
sudo chown -R 1000:1000 var downloads music nas
```

- `var/db/` — SQLite databases (torrent search, automation index).
- `var/home/` — persistent home (Spotify credentials, Zotify archive).
- `var/storage/` — configuration, queue, history, logs and caches.
- `downloads/`, `music/`, `nas/` — media mounts. Adapt them to your disks or NFS/SMB shares.

The application processes run as UID 1000 inside the container: data and media directories must be writable by that UID (on network shares, by the UID the share maps writes to).

### 3. Configure `.env`

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

Use an explicit version (for example `IMAGE_TAG=2.7.0`) for reproducible deployments. These are **host** paths mounted to fixed container paths.

> [!NOTE]
> The image serves HTTP on port **8000**. Keep `PORT=8000` unless you customize the image; with `network_mode: host`, the application is reachable on that port.

### 4. Create the application secret

Secrets must never be stored in the Compose file or in `.env`. Create this file before the first start: if it is missing, Docker creates a directory with that name instead. Create `var/.env.local`:

```bash
printf 'APP_ENV=prod\nAPP_DEBUG=0\nAPP_SECRET=%s\n' "$(openssl rand -hex 32)" > var/.env.local
chmod 600 var/.env.local
```

It is mounted read-only at `/var/www/html/.env.local`. The file must be owned by UID 1000.

### 5. Start the container

```bash
docker compose up -d
```

The application is available at **`http://<your-ip>:8000`**.

---

## 📄 Compose reference

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
> **`network_mode: host` requires Docker Desktop ≥ 4.34** (released October 2024).
> It was available as a **beta** in versions 4.29 – 4.33 and had to be enabled manually.
> It works only with **Linux containers** (the default mode in Docker Desktop).

### 1. Verify your Docker Desktop version

Open Docker Desktop → **Settings → About** and check that the version is **4.34 or later**.

If you are running an older version, either:

- **Update Docker Desktop** from [docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop/).
- Or skip `network_mode: host` and use port mapping instead (see *Alternative without host networking* below).

### 2. Enable host networking (Docker Desktop 4.29 – 4.33 only)

If you are on a beta version (4.29 – 4.33), enable the feature manually:

1. In Docker Desktop, go to **Settings → Resources → Network**.
2. Check **"Enable host networking"**.
3. Click **Apply & restart**.

> [!NOTE]
> From version **4.34+** the feature is enabled by default — no extra step needed.

### 3. Clone and create directories

Open **PowerShell**:

```powershell
git clone https://github.com/Gautardos/docker-downloader.git "$HOME\downloader-app"
cd "$HOME\downloader-app"

New-Item -ItemType Directory -Force -Path `
  "$HOME\downloader-app\var\db", "$HOME\downloader-app\var\home", "$HOME\downloader-app\var\storage" | Out-Null

# Adapt the drive letter to your setup
New-Item -ItemType Directory -Force -Path "D:\downloads", "D:\music", "D:\nas" | Out-Null
```

> [!TIP]
> You can place these directories on any drive. The paths above are examples — adapt them to your setup.

### 4. Configure `.env`

Copy `.env.example` to `.env` and use Windows paths with forward slashes:

```powershell
Copy-Item .env.example .env
```

```env
IMAGE_TAG=latest
PORT=8000
DATA_DIR=C:/Users/<YourUser>/downloader-app/var
DOWNLOADS_DIR=D:/downloads
MUSIC_DIR=D:/music
NAS_DIR=D:/nas
```

### 5. Create the application secret

```powershell
"APP_ENV=prod`nAPP_DEBUG=0`nAPP_SECRET=$([guid]::NewGuid().ToString('N') + [guid]::NewGuid().ToString('N'))`n" |
  Set-Content -NoNewline -Encoding ascii var\.env.local
```

### 6. Start the container

```powershell
cd "$HOME\downloader-app"
docker compose up -d
```

The application is available at **`http://localhost:8000`**.

> [!WARNING]
> Make sure the directories exist **before** starting the container, or Docker will create them as root-owned directories and permissions may be incorrect.

### Alternative without host networking

If your Docker Desktop version does not support `network_mode: host`, replace it with an explicit port mapping:

```yaml
services:
  downloader:
    image: gautardos/downloader-app:latest
    container_name: downloader-container
    # network_mode: host          ← remove or comment out
    ports:
      - "8000:8000"               # ← use port mapping instead
    environment:
      - PORT=8000
    # ... rest of the config unchanged
```

> [!CAUTION]
> With port mapping, only the mapped port is accessible. If the application uses additional ports internally (e.g. for `librespot-auth`), those features may not work correctly without `network_mode: host`.

---

## 🔐 First login

| Field    | Value   |
| -------- | ------- |
| Username | `admin` |
| Password | `admin` |

> [!TIP]
> Change these before exposing the application outside your LAN (it listens on all interfaces with network_mode: host). Put it behind a reverse proxy with HTTPS for remote access.

---

## 📂 Volumes

```
Host                                    → Container
────────────────────────────────────────────────────────────────
<DATA_DIR>/db                           → /var/www/html/var/db         (SQLite databases)
<DATA_DIR>/home                         → /var/www/html/var/home       (Spotify creds, archive)
<DATA_DIR>/storage                      → /var/www/html/var/storage    (config, queue, history, logs)
<DOWNLOADS_DIR>                         → /downloads                   (torrent/video output)
<MUSIC_DIR>                             → /music                       (music downloads)
<NAS_DIR>                               → /nas                         (NAS library, read/write)
<DATA_DIR>/.env.local                   → /var/www/html/.env.local     (APP_SECRET, read-only)
```

> [!WARNING]
> Paths configured in the app (Settings and Automations) refer to **container-internal** paths. If you mount a host directory at `/music`, set **Music Download Root** to `/music` — never the host path.

---

## 🏗️ Architecture

The container runs four services managed by **Supervisor**:

| Service | Role |
| ------- | ---- |
| **Apache** | Serves the Symfony application on port 8000 |
| **Redis** | Stores PHP sessions (prevents session lock contention) |
| **Download Worker** | Sequentially processes the download queue (`app:download-worker`) |
| **Automation Worker** | Runs scheduled scans, list syncs and release intents (`app:automation-worker`) |

---

## ⚙️ Configuration

Settings are managed in the web UI at **Settings** (`/config`), except:

- **TMDB and addons** — configured in the Discover drawer.
- **Automations** — configured in **Automations → Configuration** (stored in `var/db/automation.db`).
- **Runtime secrets** — `var/.env.local` (`APP_SECRET`, `APP_ENV`, `APP_DEBUG`).

See the [application README](https://github.com/Gautardos/downloader/blob/main/README.md) for the complete reference (API keys, paths, music, Jellyfin, Spotify setup, Discover setup).

---

## 🔧 Maintenance

### View logs

```bash
docker exec -it downloader-container cat /var/log/apache2_error.log
docker exec -it downloader-container cat /var/www/html/var/storage/logs/worker.log
docker exec -it downloader-container cat /var/www/html/var/storage/logs/automation_worker.log
docker exec -it downloader-container cat /var/www/html/var/storage/logs/worker_error.log
docker exec -it downloader-container cat /var/www/html/var/storage/logs/automation_worker_error.log
docker exec -it downloader-container cat /var/log/supervisord.log
```

Application logs live in `var/storage/logs/` and rotate at **5 MB per file with 5 backups**.

### Restart services

```bash
# Whole container
docker compose restart

# Download worker only
docker exec -it downloader-container supervisorctl restart worker

# Automation worker only
docker exec -it downloader-container supervisorctl restart automation-worker
```

### Update

```bash
docker compose pull
docker compose up -d
```

To pin a version, set `IMAGE_TAG=<version>` in `.env` before pulling. No manual migration is required between 2.x versions; coming from 1.x, see Upgrading from 1.x.

To roll back, set the previous `IMAGE_TAG` and run the same commands.

> [!NOTE]
> With the legacy docker-compose v1, run `docker-compose down` before `docker-compose up -d` after a pull.

### Upgrading from 1.x

Use the Compose file from docker-downloader; stop mounting `src/` and `templates/` (the code is in the image); create `var/.env.local` (see above); give existing data to UID 1000 (`sudo chown -R 1000:1000 <DATA_DIR>`).

### Backup & restore

Back up `var/storage/`, `var/db/`, `var/home/` and `var/.env.local`. The **Settings → Backup & Restore** panel can also export/import the configuration as a ZIP.

---

## 📝 Notes & Limits

- At startup the entrypoint enforces permissions on its data: `var/storage` 710, `var/storage/logs` 2750 (setgid), `*.log` 640, other files 600, `var/db` 700/600. To let an external log shipper read the logs, give `var/storage` and `var/storage/logs` a group it belongs to. Mounted media folders are never changed recursively: only their top level is adjusted when owned by root, and a refused change (network share) does not block startup.
- If the host already runs Redis on `127.0.0.1:6379` (host networking), the application uses it for sessions instead of the bundled one.

---

## 🔗 Related Repositories

- **Application:** https://github.com/Gautardos/downloader
- **Containerized deployment:** https://github.com/Gautardos/docker-downloader
- **Release notes:** [2.6](https://github.com/Gautardos/downloader/blob/main/RELEASE_NOTES_2.6.md) · [2.7](https://github.com/Gautardos/downloader/blob/main/RELEASE_NOTES_2.7.md)
