# Arr Stack — Self-hosted Media Automation Server

A complete, self-hosted media automation stack running on Docker Compose:

- **gluetun** — WireGuard VPN gateway (ProtonVPN, free tier) for private connections
- **qBittorrent** — torrent download client
- **Prowlarr** — indexer manager (search across many torrent sites from one place)
- **Radarr** — movie library automation
- **Sonarr** — TV show library automation
- **Lidarr** — music library automation
- **Bazarr** — automatic subtitle management
- **Jellyfin** — media streaming server (watch everything in your browser/any device)
- **Seerr** — media request & discovery manager (users browse TMDB, request movies/TV, and requests flow to Radarr/Sonarr; works with Jellyfin, Plex and Emby)

All services run on a dedicated Docker network (`arr_network`), share a single
`/data` mount for instant hard links, and are configured to work together out of
the box.

---

## Architecture

```
                       ┌─────────────────────────────┐
                       │         arr_network         │
                       │  (dedicated Docker network) │
                       │                             │
   Host :8888 ───────► │  gluetun (VPN gateway)      │
                       │                             │
   Host :8080 ───────► │  qbittorrent  ◄─────────────┤
   Host :9696 ───────► │  prowlarr     ◄─────┐       │
   Host :7878 ───────► │  radarr  ────┐      │       │
   Host :8989 ───────► │  sonarr  ────┼──────┘       │
   Host :8686 ───────► │  lidarr  ────┘              │
   Host :6767 ───────► │  bazarr                     │
   Host :8096 ───────► │  jellyfin                   │
   Host :5055 ───────► │  seerr  ───► radarr/sonarr  │
                       └─────────────────────────────┘
```

Services talk to each other by **service name** (e.g. Radarr reaches qBittorrent
at `http://qbittorrent:8080`) — no IP addresses needed.

**Storage layout** — one mount, two roles:

```
data/
├── torrents/          # downloads land here (incomplete/complete per category)
│   ├── movies/
│   ├── tv/
│   └── music/
└── media/             # your finished, organized library
    ├── movies/
    ├── tv/
    └── music/
```

Because `torrents/` and `media/` are on the **same filesystem** (one `/data`
volume), files are *hard-linked* (or atomically moved) instead of copied —
instant completion, zero extra disk usage.

---

## Repository layout

```
arr-stack/
├── docker-compose.yml                  # the whole stack
├── .env                                # configuration (BASE_DIR, VPN key)
├── gluetun-server-US-FREE-113.conf     # reference ProtonVPN WireGuard config (informational)
├── appdata/                            # per-service config (created at runtime)
│   ├── qbittorrent/  ├── radarr/  ├── sonarr/
│   ├── lidarr/       ├── bazarr/  ├── prowlarr/
│   ├── jellyfin/     ├── seerr/   └── gluetun/
└── data/                               # downloads + media library
    ├── torrents/{movies,tv,music}
    └── media/{movies,tv,music}
```

---

## Prerequisites

- A Linux host (tested on Debian/Ubuntu) — any distro with Docker works; Windows/macOS can use Docker Desktop
- Docker Engine **24+** and the **Compose plugin (v2)**
- A **ProtonVPN account** (free tier is fine) to obtain a WireGuard key
- Ports available on the host: `8080`, `6881/tcp+udp`, `7878`, `8989`, `8686`, `6767`, `9696`, `8096`, `5055`, `8888`

---

## Setup

### 1. Install Docker Engine + Compose plugin

Follow the official guide: <https://docs.docker.com/engine/install/>

On Debian/Ubuntu (using the official repository):

```bash
sudo apt update
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

sudo tee /etc/apt/sources.list.d/docker.sources > /dev/null <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# verify
docker --version
docker compose version
```

Add your user to the `docker` group so you can run `docker` without `sudo`
(relogin afterwards):

```bash
sudo usermod -aG docker "$USER"
```

### 2. Get a ProtonVPN WireGuard key

1. Create a free account at <https://account.protonvpn.com/>
2. Go to **Downloads** → **WireGuard configuration**
3. Pick a server (any free one) and choose platform **"Gluetun"** (or "Linux")
4. Download the config — it looks like this (see `gluetun-server-US-FREE-113.conf` in this repo):

   ```
   [Interface]
   PrivateKey = <your-private-key>
   Address = 10.2.0.2/32
   DNS = 10.2.0.1

   [Peer]
   PublicKey = ...
   AllowedIPs = 0.0.0.0/0, ::/0
   Endpoint = <server-ip>:51820
   ```

5. Copy the **PrivateKey** value — gluetun needs only this key; it selects the
   server itself from the settings in `docker-compose.yml`
   (`SERVER_COUNTRIES=Netherlands`, `FREE_ONLY=on`).

### 3. Get the project files

Copy this directory to your server (or clone it):

```bash
git clone <your-repo-url> arr-stack
cd arr-stack
```

### 4. Configure `.env`

Create the `.env` file (a template is included in this repo):

```bash
cp .env.example .env   # if present
nano .env
```

```bash
# Absolute path to this directory — no trailing slash
BASE_DIR=/home/ai-man/arr-stack

# WireGuard private key from step 2
WG_PRIVATE_KEY=your_protonvpn_wireguard_private_key_here
```

Protect the file — it contains your VPN key:

```bash
chmod 600 .env
```

> ⚠️ **Never commit `.env` to git.** It is intentionally not tracked.

### 5. Prepare directories & permissions

The containers run as UID/GID **1000** (set via `PUID`/`PGID` in
`docker-compose.yml`). Create the folder structure and make it writable by that
user:

```bash
mkdir -p data/torrents/{movies,tv,music} data/media/{movies,tv,music}
chown -R 1000:1000 data appdata
```

### 6. Start the stack

```bash
docker compose up -d
docker compose ps
```

You should see all 9 services with status `Up`. The first start pulls images
and can take a few minutes.

### 7. Verify the VPN

```bash
docker compose logs gluetun --tail 50
```

Look for lines like `Wireguard is up` / `VPN connected`. You can also test the
VPN's HTTP proxy from the host:

```bash
curl -x http://localhost:8888 https://ipinfo.io/ip
```

The IP returned should be a ProtonVPN exit IP (Netherlands), not your own.

---

## First login & passwords

| Service     | URL                  | First login |
|-------------|----------------------|-------------|
| qBittorrent | `http://<host>:8080` | `admin` + temporary password from container logs |
| Prowlarr    | `http://<host>:9696` | set username/password on first visit |
| Radarr      | `http://<host>:7878` | set username/password on first visit |
| Sonarr      | `http://<host>:8989` | set username/password on first visit |
| Lidarr      | `http://<host>:8686` | set username/password on first visit |
| Bazarr      | `http://<host>:6767` | set username/password on first visit |
| Jellyfin    | `http://<host>:8096` | create your first admin user |
| Seerr       | `http://<host>:5055` | create local admin, then sign in with Jellyfin (needs Jellyfin admin API key) |

**qBittorrent temporary password** — on first start a random password is
generated and printed to the container logs:

```bash
docker compose logs qbittorrent | grep -i password
```

Log in with user `admin`, then change it under
**Tools → Options → WebUI → Authentication**.

> If `curl -x ...` / `ipinfo.io` is blocked from your network, use
> `https://am.i.mullvad.net/ip` or `https://ifconfig.me` instead.

---

## Service configuration

Configure the apps **in this order** — qBittorrent categories first, then the
indexer/app connections, then wire everything into Prowlarr.

### qBittorrent

1. **Categories** (right-click **Categories → All → Add category**):
   - `movies` → save path `movies`
   - `tv` → save path `tv`
   - `music` → save path `music`

   (Save paths are relative to the default save path below.)

2. **Tools → Options → Downloads:**
   - Default Save Path: `/data/torrents`
   - Default Torrent Management Mode: `Automatic`
   - When Category Save Path Changed: `Switch affected torrents to Manual Mode`
   - Tick **Use Subcategories** and **Use Category paths in Manual Mode**
   - Save.

### Prowlarr

1. **Settings → Download Clients → + → qBittorrent:**
   - Host: `qbittorrent` (service name, not an IP)
   - Port: `8080`
   - SSL: **unticked**
   - Username/password: the ones you set for qBittorrent
   - **Test** → green tick → Save
2. **Settings → Indexers → +** — add your torrent indexers (see troubleshooting for Cloudflare-blocked sites).
3. **Settings → Apps → +** — add Radarr, Sonarr and Lidarr (steps below give their API keys).

### Radarr

1. **Settings → Media Management → Add Root Folder:** `/data/media/movies`
2. **Settings → Media Management → Show Advanced → Importing:**
   - **Use Hardlinks instead of Copy:** ticked
   - Optional: enable *Rename Movies*, *Delete empty movie folders*, *Import Extra Files* (`srt,sub,nfo`)
3. **Settings → Download Clients → + → qBittorrent:**
   - Host: `qbittorrent`, Port: `8080`, SSL unticked, category: `movies`
   - Test → Save
4. **Settings → General → copy the API key**, then in **Prowlarr → Settings → Apps → + → Radarr:**
   - Prowlarr Server: `http://prowlarr:9696`
   - Radarr Server: `http://radarr:7878`
   - Paste API key → Test → Save

### Sonarr

Same pattern as Radarr with TV values:

1. **Add Root Folder:** `/data/media/tv`
2. **Use Hardlinks instead of Copy:** ticked
3. **Download Client → qBittorrent:** host `qbittorrent`, port `8080`, category `tv`
4. **Prowlarr → Apps → Sonarr:** Prowlarr Server `http://prowlarr:9696`, Sonarr Server `http://sonarr:8989`, paste API key.

### Lidarr

1. **Settings → Media Management → Add Root Folder:** `/data/media/music`
2. **Download Client → qBittorrent:** host `qbittorrent`, port `8080`, category `music`
3. **Prowlarr → Apps → Lidarr:** Prowlarr Server `http://prowlarr:9696`, Lidarr Server `http://lidarr:8686`, paste API key.

### Bazarr

1. **Settings → Languages:** create a language profile (e.g. *English*).
2. **Settings → Providers:** add subtitle sources (OpenSubtitles.org etc. — most need a free account).
3. **Settings → Sonarr/Radarr:** add both apps (hosts `sonarr` / `radarr`, port `8989` / `7878`, paste API keys, **tick "Use SSL" off**).
4. Go to the **Series/Movies** tabs and hit *Update* to scan your existing library.

### Seerr (media request manager)

1. Visit `http://<host>:5055` — create a **local admin account** (email + password).
2. **Settings → Jellyfin:**
   - Jellyfin Host: `jellyfin`, Port: `8096`, tick **Use SSL** off
   - **External URL** (optional): `http://<host>:8096` — makes links open for other users
   - **API Key:** copy it from Jellyfin → **Dashboard → API Keys → +**
3. **Settings → Radarr:**
   - Host: `radarr`, Port: `7878`, tick **Use SSL** off
   - API Key: copy from Radarr → **Settings → General → Security → API Key**
   - Base URL: empty, Default Quality Profile: e.g. *HD-1080p*, check **Automatically Scan** and **Enable Scan Notifications**
4. **Settings → Sonarr:**
   - Host: `sonarr`, Port: `8989`, tick **Use SSL** off
   - API Key: copy from Sonarr → **Settings → General → Security → API Key**
   - Quality Profile & Language Profile: e.g. *HD-1080p* / *English*
   - **Automatically Scan** + **Enable Scan Notifications**: ticked
5. Sign in with Jellyfin: on the **Settings → Jellyfin** page, once connected, you can enable "Use Jellyfin login" — users then authenticate with their Jellyfin accounts and see their Jellyfin libraries.

> Seerr manages **movies & TV** only (Radarr/Sonarr). Music requests (Lidarr)
> are not part of Seerr's scope.

### Jellyfin

1. Create your admin account on first visit.
2. **Add Media Library** with these folders:
   - Movies → `/data/media/movies`
   - TV Shows → `/data/media/tv`
   - Music → `/data/media/music`
3. Copy the **API Key** (Dashboard → API Keys → +) — Seerr needs it (step above).
4. Jellyfin scans the library automatically; new arrivals appear as soon as Radarr/Sonarr/Lidarr finish importing.

---

## Ports

| Port | Service | Notes |
|------|---------|-------|
| `8080` | qBittorrent WebUI | |
| `6881` tcp/udp | qBittorrent torrent traffic | |
| `9696` | Prowlarr | |
| `7878` | Radarr | |
| `8989` | Sonarr | |
| `8686` | Lidarr | |
| `6767` | Bazarr | |
| `8096` | Jellyfin | |
| `5055` | Seerr | media request manager |
| `8888` | gluetun HTTP proxy | only reachable on localhost — useful for tools that need the VPN exit |

All ports are published on `0.0.0.0` — if the host is directly exposed to the
internet, put a firewall/reverse proxy in front of everything except gluetun
(and restrict `6881` to your VPN IPs).

---

## VPN notes

- **Only gluetun runs inside the VPN tunnel.** It establishes the WireGuard
  connection and exposes an HTTP proxy on the host at `http://localhost:8888`.
- **qBittorrent is currently *not* inside the tunnel** — the compose file's
  comment says "routed through Gluetun" but no `network_mode` is set, so
  qBittorrent uses the host network interface directly.
- To force qBittorrent through the VPN, add `network_mode: service: gluetun` to
  the qbittorrent service. **Caveats:** you must move qBittorrent's published
  ports (`8080`, `6881`) onto the gluetun service, and all other apps must then
  reach it at `http://gluetun:8080` instead of `http://qbittorrent:8080`.
- ProtonVPN free servers change from time to time; if gluetun won't connect,
  try changing `SERVER_COUNTRIES` (or removing it) in `docker-compose.yml` and
  re-create the container.

---

## Hard links & storage

Everything lives under `${BASE_DIR}/data` — a single mount — so Radarr/Sonarr/
Lidarr can hard-link completed downloads into `media/` instantly.

To verify hard links work after a download:

```bash
ls -i data/media/movies/<file>
ls -i data/torrents/movies/<file>
```

If the inode numbers (first column) match, it's a hard link — the file exists
only once on disk.

If files are being *copied* instead of hard-linked, check:

- both folders are under the same mount (`mount | grep data`)
- permissions: `data/` is owned by `1000:1000` (`ls -ln data`)
- app logs: Radarr/Sonarr → **System → Log Files**

---

## Useful commands

```bash
docker compose ps                    # status of all services
docker compose logs -f <service>     # follow one service's logs
docker compose up -d                 # start / apply compose changes
docker compose down                  # stop everything (config is kept)
docker compose pull                  # update images
docker compose up -d --force-recreate <service>   # recreate one service
```

---

## Backups

All configuration lives in `appdata/` — that's the only thing you need to back
up (plus `data/media/` if you want your library). Each \*arr app also keeps its
own scheduled backups:

```
appdata/radarr/Backups/scheduled/
appdata/sonarr/Backups/scheduled/
appdata/lidarr/Backups/scheduled/
```

For a clean snapshot, stop the stack first:

```bash
docker compose down
tar -czf arr-stack-backup.tar.gz appdata .env docker-compose.yml
docker compose up -d
```

---

## Troubleshooting

**qBittorrent: "temporary password was not set / can't log in"**

```bash
docker compose logs qbittorrent | grep -i password
```

**VPN won't connect**

```bash
docker compose logs gluetun --tail 50
```

Common causes: key missing/wrong in `.env`, ProtonVPN free server offline
(try a different `SERVER_COUNTRIES`), or `/dev/net/tun` not available on the
host (needs `modprobe tun`).

**Downloads complete but don't appear in the library**

Check **Radarr/Sonarr → Activity → Queue**. A flag like *"Downloaded - Unable
to Import Automatically"* means a manual import is needed — click the manual
import icon and confirm the match. Also verify the download client category
matches the qBittorrent category (`movies` / `tv` / `music`).

**Indexers fail with Cloudflare blocks**

Add a **FlareSolverr** container (a Cloudflare-bypass proxy) and register it in
Prowlarr:

```yaml
  flaresolverr:
    <<: *common-keys
    container_name: flaresolverr
    image: ghcr.io/flaresolverr/flaresolverr:latest
    ports:
      - 8191:8191
    environment:
      - LOG_LEVEL=info
```

Then in Prowlarr: **Settings → Indexers → + → Indexer Proxies → FlareSolverr**
— Host `http://flaresolverr:8191`, give it a tag like `cloudflare`, and apply
that tag to the affected indexers.

**Jellyfin hardware transcoding**

Pass the GPU/VAAPI device through:

```yaml
    devices:
      - /dev/dri:/dev/dri
```

(Requires the host GPU drivers/`i915`/VAAPI support, then enable hardware
acceleration in Jellyfin Dashboard → Playback.)

---

## Legal note

The \*arr stack is general media automation software and is perfectly suited to
legal, public-domain, and Creative Commons content — e.g. the Internet Archive
has thousands of public-domain movies (Night of the Living Dead, Charade, The
General…). Only download content you have the right to download, and respect
your local laws and your VPN provider's terms of service.
