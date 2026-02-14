# Downloader App

A self-hosted media management platform built with Symfony & Python, packaged as a single Docker image. It handles video/torrent downloads via Alldebrid and provides a complete music pipeline — from Spotify search to tagged, organized files in your library.

Project repo : https://github.com/Gautardos/downloader

---

## ✨ Features

### 🎬 Videos & Torrents
- **Magnet & `.torrent` upload** with Alldebrid cloud debridding.
- **Built-in search engine** — local CSV database for searching torrents by title and auto-filling magnets.
- **Magnet preview** — inspect file list, sizes and Alldebrid status before committing.
- **Pack grouping** — automatic detection of series/albums with recursive folder creation.
- **Batch folder creation** — one-click creation of all missing directories during bulk import.
- **AI-assisted renaming** — Grok suggests clean, normalized file names (✨ button, hidden when no Grok key).

### 🎵 Music (Music Explorer)
- **Spotify search** — search tracks/albums directly from the UI using the Spotify Web API.
- **High-quality downloads** — powered by CLI tools (zotify/spotdl) with configurable quality up to `very_high`.
- **Full ID3 tag editor** — Artist, Album, Title, Year, Genre, Track Number.
- **Lyrics** — automatic fetching of synced LRC lyrics (LRCLib) and plain-text lyrics (Genius).
- **AI genre classification** — automatic genre tagging via Grok with a fully customizable expert prompt.
- **Regex genre mapping** — alternative rule-based genre classification with configurable JSON patterns.
- **Library automation** — move, rename, and organize files into `Artist/Artist - Album - Track - Title.ext`.
- **Spotify authentication** — generate Spotify credentials directly from Settings via the built-in `librespot-auth` binary.

### 🛠️ System
- **Download queue** — sequential processing via a background worker (Supervisor-managed).
- **Real-time tracker** — live download progress with polling and browser notifications.
- **Full history** — detailed action logs with retention limit.
- **Authentication** — simple login system with configurable credentials (default: `admin` / `admin`).
- **Adaptive UI** — navigation items are hidden automatically when their dependencies are not configured.
- **JSON storage** — no SQL database needed; all configuration, history, and queue are stored as JSON files.

---

## 🐳 Installation (Docker)

The application is distributed as a pre-built Docker image.

### Prerequisites
- **Docker** and **Docker Compose** (v2) installed on your host.
- An **Alldebrid** account (for video/torrent features).
- A **Spotify** developer account (for music features).
- Optionally, a **Grok** API key (for AI renaming and genre detection).

---

### 🐧 Linux

#### 1. Create the project directory

```bash
mkdir -p ~/downloader-app/var/home
mkdir -p ~/downloader-app/var/storage
```

- `var/home/` — persistent home directory for the application (Spotify credentials, archive logs).
- `var/storage/` — persistent application data (configuration, download queue, history).

#### 2. Modify `docker-compose.yml`

Adapt the volume paths and environment variables to your setup (see the **Configuration** section below).

#### 3. Start the container

```bash
cd ~/downloader-app
docker compose up -d
```

The application is now accessible at **`http://<your-ip>:8000`**.

---

### 🪟 Windows

> [!IMPORTANT]
> **`network_mode: host` requires Docker Desktop ≥ 4.34** (released October 2024).
> It was available as a **beta** in versions 4.29 – 4.33 and had to be enabled manually.
> It works only with **Linux containers** (the default mode in Docker Desktop).

#### 1. Verify your Docker Desktop version

Open Docker Desktop → **Settings → About** and check that the version is **4.34 or later**.

If you are running an older version, either:
- **Update Docker Desktop** from [docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop/).
- Or skip `network_mode: host` and use port mapping instead (see *Alternative without host networking* below).

#### 2. Enable host networking (Docker Desktop 4.29 – 4.33 only)

If you are on a beta version (4.29 – 4.33), you must enable the feature manually:

1. In Docker Desktop, go to **Settings → Resources → Network**.
2. Check **"Enable host networking"**.
3. Click **Apply & restart**.

> [!NOTE]
> From version **4.34+** the feature is enabled by default — no extra step needed.

#### 3. Create the project directory

Open **PowerShell** and run:

```powershell
# Create the project directory on your Windows drive
mkdir -p "$HOME\downloader-app\var\home"
mkdir -p "$HOME\downloader-app\var\storage"

# Create directories for media output (adapt the drive letter to your setup)
mkdir -p "D:\downloads"
mkdir -p "D:\music"
```

> [!TIP]
> You can place these directories on any drive. The paths above are examples — adapt them to your setup.

#### 4. Write `docker-compose.yml`

Create a `docker-compose.yml` file inside `%USERPROFILE%\downloader-app\` with Windows-style volume paths:

```yaml
version: "3.8"

services:
  downloader:
    image: gautardos/downloader-app:latest
    container_name: downloader-container
    network_mode: host
    environment:
      - PORT=8000

    volumes:
      # Persistent app data (use your actual Windows paths)
      - C:\Users\<YourUser>\downloader-app\var\home:/var/www/html/var/home
      - C:\Users\<YourUser>\downloader-app\var\storage:/var/www/html/var/storage
      # Media directories
      - D:\downloads:/downloads
      - D:\music:/music

    restart: unless-stopped

    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```

> [!WARNING]
> **Volume paths on Windows** use the full path with the drive letter (`C:\Users\...`).
> Docker Desktop automatically converts them to the `/c/Users/...` format used by the Linux VM.
> Make sure the directories exist **before** starting the container, or Docker will create them as root-owned directories and permissions may be incorrect.

#### 5. Alternative without host networking (Docker Desktop < 4.29)

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
> With port mapping, only the mapped port is accessible. If the application uses additional ports internally (e.g., for `librespot-auth`), those features may not work correctly without `network_mode: host`.

#### 6. Start the container

```powershell
cd "$HOME\downloader-app"
docker compose up -d
```

The application is now accessible at **`http://localhost:8000`**.

---

### 🔐 First login

Open the application in your browser. The default credentials are:

| Field    | Value   |
| -------- | ------- |
| Username | `admin` |
| Password | `admin` |

> [!TIP]
> Change these immediately in **Settings → Admin User / Admin Password**.

---

## ⚙️ Configuration

All settings are managed through the web UI at **Settings** (`/config`). Below is a reference of every configurable option.

### 🔑 API Keys

| Setting | Purpose | Required for |
| ------- | ------- | ------------ |
| **Alldebrid API Key** | Debridding and downloading torrents/magnets | Videos & Torrents |
| **Grok API Key** | AI-assisted renaming and genre detection | AI features (optional) |
| **Grok Model** | Model to use (default: `grok-4-fast-non-reasoning`) | AI features |
| **Spotify Client ID** | Spotify Web API authentication | Music search |
| **Spotify Client Secret** | Spotify Web API authentication | Music search |
| **Genius API Token** | Plain-text lyrics fetching | Lyrics (optional) |
| **LRCLib Token** | Synced LRC lyrics fetching | Lyrics (optional) |

### 📂 Paths

| Setting | Default (Docker) | Description |
| ------- | ---------------- | ----------- |
| **Default Torrent Path** | `/downloads` | Where torrent/video files are saved |
| **Music Root Path** | `/music` | Temporary download directory for music |
| **Library Path** | *(empty)* | Final organized music library (e.g., `/nas/Music`) |
| **JSON Credentials Path** | `/var/www/html/var/home/.local/credentials.json` | Spotify auth credentials file |
| **Music Archive** | `/var/www/html/var/home/log/archive.txt` | Already-downloaded track log |
| **Venv Path** | `/opt/venv` | Python virtual environment (pre-configured in Docker) |
| **Music Binary** | `zotify` | CLI tool for music downloads |
| **Librespot Auth Binary** | `/var/www/html/var/librespot-auth/target/release/librespot-auth` | Path to the Spotify auth generator |

### 🎵 Music Settings

| Setting | Default | Description |
| ------- | ------- | ----------- |
| **Output Format** | `{artist} - {album} - {song_name}.{ext}` | File naming template |
| **Audio Format** | `mp3` | Output format (`mp3`, `ogg`, `flac`) |
| **Quality** | `very_high` | Audio quality level |
| **Skip Existing** | Off | Skip tracks already in archive |
| **Download Lyrics** | Off | Fetch and embed lyrics automatically |
| **Genre Tagging Mode** | `AI` | `AI` (Grok-based) or `Mapping` (regex-based) |

### 🔐 Admin

| Setting | Default | Description |
| ------- | ------- | ----------- |
| **Admin User** | `admin` | Login username |
| **Admin Password** | `admin` | Login password |
| **History Retention** | `500` | Maximum number of entries kept in history |

---

## 🎵 Spotify Setup

The music features require two independent Spotify configurations:

### 1. Spotify API Credentials (for metadata search)

1. Go to the [Spotify Developer Dashboard](https://developer.spotify.com/dashboard).
2. Log in and click **"Create app"**.
3. Fill in a name (e.g., `My Downloader`) and a redirect URI (e.g., `http://localhost:8000/callback`).
4. Save, then go to **Settings** on the app page.
5. Copy the **Client ID** and **Client Secret** into the application's Settings page.

### 2. Spotify Authentication (for music downloads)

The download engine needs a valid Spotify session stored as a `credentials.json` file.

**From the UI (recommended):**
1. Go to **Settings**.
2. Click the **⚡ Auth** button next to "JSON Credentials Path".
3. Open Spotify on your phone/desktop and select the device named `Speaker <timestamp>` as output (you need to have both devices on the same wifi).
4. The credentials file is generated, transformed, and stored automatically.

> [!NOTE]
> The ⚡ Auth button only appears when no credentials file exists at the configured path.

---

## 🏗️ Architecture

The Docker container runs three services managed by **Supervisor**:

| Service | Role |
| ------- | ---- |
| **Apache** | Serves the Symfony application on port 8000 |
| **Redis** | Stores PHP sessions (prevents session lock contention) |
| **Download Worker** | Background process that sequentially processes the download queue |

### Volume mapping explained

```
Host                                    → Container
────────────────────────────────────────────────────────────────
~/downloader-app/var/home               → /var/www/html/var/home        (Spotify creds, logs)
~/downloader-app/var/storage            → /var/www/html/var/storage     (config, queue, history)
/downloads                              → /downloads                    (torrent/video output)
/music                                  → /music                        (music output)
```

> [!WARNING]
> Paths configured in the Settings page refer to **container-internal** paths. If you mount `/music` on the host, set `Music Root Path` to `/music` in Settings — not the host path.

---

## 🔧 How Alldebrid Works

The application is **not** a BitTorrent client. It delegates P2P downloads to the [Alldebrid](https://alldebrid.com) cloud service:

1. **Submit** — magnet link or `.torrent` file uploaded via the dashboard.
2. **Cloud download** — Alldebrid fetches the content on its high-speed servers.
3. **Link extraction** — the API returns direct HTTP links for each file in the torrent once ready.
4. **Debridding** — each link is "unlocked" for full-speed direct download.
5. **Local transfer** — the background worker downloads files to your local storage directory.

---

## 📂 Project Structure

```
src/                → Symfony controllers and services
templates/          → Twig views (UI)
cli/                → Python scripts (download engine, lyrics, tag/rename/move)
public/             → Web entry point and static assets
var/storage/        → JSON data files (config, queue, history)
var/home/           → Persistent home (Spotify credentials, archive)
Dockerfile          → Container build definition
supervisord.conf    → Process manager configuration
```

### Volume mapping explained
```
Host                                    → Container
────────────────────────────────────────────────────────────────
~/nas                                   → /nas (Base storage path for torrent downloads)
```

### CLI Scripts

| Script | Purpose |
| ------ | ------- |
| `music_downloader.py` | Music download engine (wraps zotify/spotdl) |
| `lyrics_fetcher.py` | Fetches and embeds lyrics from LRCLib/Genius |
| `tag_rename_move.py` | Tags, renames, and moves files to the organized library |

---

## 🔧 Maintenance

### View logs

```bash
# Application logs (Apache)
docker exec -it downloader-container cat /var/log/apache2_error.log

# Worker logs
docker exec -it downloader-container cat /var/log/worker.log

# Supervisor logs
docker exec -it downloader-container cat /var/log/supervisord.log
```

### Restart services

```bash
# Restart the entire container
docker compose restart

# Restart only the download worker
docker exec -it downloader-container supervisorctl restart worker
```

### Update the application

```bash
docker compose pull
docker compose up -d
```

---

## 📝 Notes

- The application uses a file-based JSON storage system (`JsonStorage`) which avoids the need for any SQL database. All data persists in the `var/storage/` directory.
- Redis is used **only** for PHP session storage to prevent session lock contention during Ajax polling.
- The `librespot-auth` binary is compiled from Rust source during the Docker image build. It is included in the image and does not need to be installed separately.
