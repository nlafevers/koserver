# KOSERVER Deployment Guide

This guide walks you through deploying **KOPDS** (an OPDS book server for your Calibre library) and **KOSYNC** (a KOReader reading-progress sync server) on a single machine, behind a single HTTPS reverse proxy, so your KOReader devices can reach both securely from anywhere.

It is written for newcomers self-hosting on modest hardware — a Raspberry Pi, an old PC, or a free-tier cloud VM. The whole stack is Go-native and lightweight: the two app binaries plus [Caddy](https://caddyserver.com/) (a Go web server that handles HTTPS for you automatically).

> KOPDS and KOSYNC are completely independent. You can deploy just one, or both. Wherever a step is specific to one app, it is labelled.

Copy-paste-ready versions of every config file in this guide live in [`deploy/`](deploy/).

---

## 📖 Table of Contents

1.  [Overview](#-overview)
2.  [Prerequisites](#-prerequisites)
3.  [Part A — Native Install (recommended)](#-part-a--native-install-recommended)
4.  [Part B — Caddy Reverse Proxy & HTTPS](#-part-b--caddy-reverse-proxy--https)
5.  [Part C — Docker Alternative](#-part-c--docker-alternative)
6.  [Part D — Cloud: Free-Tier GCP VM](#-part-d--cloud-free-tier-gcp-vm)
7.  [Hardening & Operations](#-hardening--operations)
8.  [Connecting KOReader](#-connecting-koreader)
9.  [Troubleshooting](#-troubleshooting)

---

## 🗺 Overview

### What you are building

```
                           ┌───────────────────────────────────────────────────────┐
KOReader devices           │            Your server                                │
(phone, e-reader) ──HTTPS──┼──▶ Caddy ──┬──▶ KOPDS  (127.0.0.1:8080) ──▶ kopds.db ─┼─▶ Calibre library
                    :443   │  :80 :443  └──▶ KOSYNC (127.0.0.1:8081) ──▶ kosync.db │
                           └───────────────────────────────────────────────────────┘
        only 80/443 are open to the internet; 8080/8081 stay private
```

- **Caddy** is the only thing exposed to the internet (ports 80 and 443). It terminates HTTPS and forwards requests to the right app.
- **KOPDS** serves your Calibre library as an OPDS catalog and streams book files on download.
- **KOSYNC** stores reading progress so your devices stay in sync.
- The app ports (**8080**, **8081**) are reachable **only** from the server itself, never the internet. This is a security boundary the rest of the guide depends on — see [Why the app ports stay private](#why-the-app-ports-stay-private).

### Choose your path

| You want…                                                    | Follow |
| :---                                                         | :---   |
| The leanest setup for constrained hardware (**recommended**) | [Part A — Native](#-part-a--native-install-recommended) → [Part B — Caddy](#-part-b--caddy-reverse-proxy--https) |
| The most self-contained, hands-off setup                     | [Part C — Docker](#-part-c--docker-alternative) |
| To run it on a free cloud VM                                 | [Part D — GCP](#-part-d--cloud-free-tier-gcp-vm) (then A or C) |

**Why native is the default recommendation.** Docker adds ~250–300 MB of resident overhead. On a 1 GB cloud VM or an early Raspberry Pi that is a large fraction of your RAM. The native binaries plus Caddy idle at around 100 MB combined. If RAM is not scarce and you prefer container isolation, Part C is fully supported.

---

## 📋 Prerequisites

- **A Linux server** you can SSH into (Debian/Ubuntu and Raspberry Pi OS are assumed; commands use `apt` and `systemd`). Don't have one yet? [Part D](#-part-d--cloud-free-tier-gcp-vm) creates a free cloud VM.
- **A domain name** (or subdomain) you control, for automatic HTTPS. Two host records are ideal — e.g. `kopds.example.com` and `kosync.example.com`. No public domain? See [the DuckDNS option](#advanced-home-behind-nat-with-duckdns) for home setups behind a router.
- **(KOPDS only) A Calibre library** — a folder containing your books and a `metadata.db` file, reachable from the server (locally, or synced down with rclone — see [Part D](#kopds-the-calibre-library-on-a-cloud-vm)).
- Basic comfort with a terminal. Every command is spelled out.

---

## 🛠 Part A — Native Install (recommended)

This installs each app as a hardened `systemd` service running under its own unprivileged user. Do the steps for whichever app(s) you want; they are identical apart from names and ports.

### A1. Get the binaries

Download the latest release for your CPU architecture from the [KOPDS releases](https://github.com/nlafevers/kopds/releases) and [KOSYNC releases](https://github.com/nlafevers/kosync/releases) pages, or build from source (needs Go 1.25+; no C compiler required):

```bash
git clone --depth 1 https://github.com/nlafevers/kopds.git
git clone --depth 1 https://github.com/nlafevers/kosync.git
(cd kopds && go build -o kopds ./cmd/kopds)
(cd kosync && go build -o kosync ./cmd/kosync)
```

Place the binaries somewhere stable, e.g. `/opt/kopds/kopds` and `/opt/kosync/kosync`:

```bash
sudo mkdir -p /opt/kopds /opt/kosync
sudo cp kopds /opt/kopds/    # from each build directory
sudo cp kosync /opt/kosync/
```

### A2. Create users, directories, and permissions

Run each app as a dedicated system account with no login shell, and give it a data directory:

```bash
# KOPDS
sudo useradd -r -s /usr/sbin/nologin kopds
sudo mkdir -p /var/lib/kopds/cache
sudo chown -R kopds:kopds /var/lib/kopds /opt/kopds

# KOSYNC
sudo useradd -r -s /usr/sbin/nologin kosync
sudo mkdir -p /var/lib/kosync
sudo chown -R kosync:kosync /var/lib/kosync /opt/kosync
```

The apps themselves enforce strict permissions on the database (directory `0750`, file `0600`), so your reading data is not world-readable.

### A3. Configure

Both apps read their settings from environment variables (or a `config.yaml`). The full list lives in each app's README ([KOPDS](https://github.com/nlafevers/kopds#-configuration-reference), [KOSYNC](https://github.com/nlafevers/kosync#-configuration-reference)). For a reverse-proxied deployment the settings that matter most are:

- `KOPDS_BASE_URL` / **public HTTPS URL** — KOPDS builds absolute links into its OPDS feeds from this, so it **must** be the public address (e.g. `https://kopds.example.com`), not `localhost`.
- `KO*_TRUST_PROXY_HEADERS=true` — tells the app to read the real client IP from the `X-Forwarded-For` header that Caddy sets (used for rate limiting). Only safe behind a proxy; see [the security note](#why-the-app-ports-stay-private).
- `KO*_DATABASE_PATH` — an absolute path under the data directory you created.

You will set these in the systemd unit in step A5.

### A4. Create your first login

Both apps require authentication. Create a user with the built-in CLI **as the service user**, so the database file is owned correctly:

```bash
sudo -u kopds  env KOPDS_DATABASE_PATH=/var/lib/kopds/kopds.db   /opt/kopds/kopds  create-user admin
sudo -u kosync env KOSYNC_DATABASE_PATH=/var/lib/kosync/kosync.db /opt/kosync/kosync create-user admin
```

The CLI prompts for a password (hidden), creates and migrates the database on first run, and prints a single confirmation line. (For scripted setup, pipe the password in:
`echo 'pw' | sudo -u kopds /opt/kopds/kopds create-user admin --password-stdin`.)

> Set the same `*_DATABASE_PATH` here that the service uses, so the CLI and the server share one database file.

### A5. Install the systemd services

Copy the sample units from [`deploy/systemd/`](deploy/systemd/) to `/etc/systemd/system/`, editing the paths, hostnames, and `KOPDS_LIBRARY_PATH` to match your machine. The KOPDS unit:

```ini
[Unit]
Description=KOPDS OPDS Server
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=kopds
Group=kopds
ExecStart=/opt/kopds/kopds
Restart=on-failure
RestartSec=5

Environment=KOPDS_LIBRARY_PATH=/srv/calibre/library
Environment=KOPDS_DATABASE_PATH=/var/lib/kopds/kopds.db
Environment=KOPDS_IMAGE_CACHE_PATH=/var/lib/kopds/cache/images
Environment=KOPDS_BASE_URL=https://kopds.example.com
Environment=KOPDS_PORT=8080
Environment=KOPDS_LOG_LEVEL=info
Environment=KOPDS_TRUST_PROXY_HEADERS=true

# Hardening
NoNewPrivileges=true
ProtectSystem=full
ProtectHome=true
PrivateTmp=true
ReadWritePaths=/var/lib/kopds

[Install]
WantedBy=multi-user.target
```

The KOSYNC unit is the same shape with `kosync` names, `KOSYNC_*` variables, port `8081`, and `ReadWritePaths=/var/lib/kosync`.

> **Why `ReadWritePaths` matters:** the hardening directives make most of the filesystem read-only. SQLite writes its `-wal` and `-shm` companion files **in the database's directory**, so that directory must stay writable — that is exactly what `ReadWritePaths=` grants. Omit it and the database will fail to open.

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now kopds kosync
sudo systemctl status kopds kosync      # should be "active (running)"
journalctl -u kopds -f                  # follow the logs
```

A quick local check (before HTTPS is set up) confirms each app is listening:

```bash
curl -s http://127.0.0.1:8080/health    # KOPDS
curl -s http://127.0.0.1:8081/health    # KOSYNC
```

### A6. (KOPDS) Connecting your Calibre library

KOPDS reads `metadata.db` and streams book files from `KOPDS_LIBRARY_PATH`. If the library lives on the same machine, just point at the folder. If it lives elsewhere (a NAS, Nextcloud, or cloud storage), mount it with [rclone](https://rclone.org/) — see [Part D](#kopds-the-calibre-library-on-a-cloud-vm) for a full rclone mount example.

> **Keep the KOPDS index and cache on local disk** (`/var/lib/kopds`), never on the network/rclone mount. SQLite on a high-latency or flaky mount risks locking and corruption. KOPDS is designed to stream books across the network on demand while keeping its fast index local.

Now go to [Part B](#-part-b--caddy-reverse-proxy--https) to put HTTPS in front of everything.

---

## 🔒 Part B — Caddy Reverse Proxy & HTTPS

Both apps speak plain HTTP and use HTTP Basic Authentication, which sends credentials in clear text. A reverse proxy gives you **HTTPS** (encrypting everything) and lets one public address serve both apps. Caddy is the natural choice here: it is a single Go binary and obtains/renews Let's Encrypt certificates automatically.

### B1. Why subdomains (not subpaths)

Use a separate hostname per app — `kopds.example.com` and `kosync.example.com`. KOPDS embeds absolute URLs (built from `KOPDS_BASE_URL`) inside its OPDS feeds, and KOReader's sync client expects the sync server at the root of its configured address. Subdomains keep both simple; subpaths (`example.com/kopds`) require rewriting and are easy to get wrong.

### B2. Install Caddy

On Debian/Ubuntu/Raspberry Pi OS:

```bash
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update && sudo apt install -y caddy
```

This installs Caddy and a `caddy` systemd service that reads `/etc/caddy/Caddyfile`.

### B3. Write the Caddyfile

Edit `/etc/caddy/Caddyfile` to contain just:

```caddy
kopds.example.com {
	reverse_proxy localhost:8080
}

kosync.example.com {
	reverse_proxy localhost:8081
}
```

Reload Caddy:

```bash
sudo systemctl reload caddy
```

(The [`deploy/Caddyfile`](deploy/Caddyfile) sample uses Docker service names for Part C, with these `localhost` lines included as comments.)

### B4. Point DNS and confirm HTTPS

Create DNS **A records** for `kopds.example.com` and `kosync.example.com` pointing at your server's public IP. Once DNS resolves and ports 80/443 reach the server, Caddy fetches certificates on first request — no further action needed. Verify:

```bash
curl -sI https://kopds.example.com/health
curl -sI https://kosync.example.com/health
```

Make sure `KOPDS_BASE_URL` is the **https** URL (`https://kopds.example.com`) and restart KOPDS if you changed it (`sudo systemctl restart kopds`).

### Why the app ports stay private

`TRUST_PROXY_HEADERS=true` makes each app believe the `X-Forwarded-For` header. That is correct **only** when the app can be reached solely through Caddy — otherwise a client could connect to port 8080/8081 directly and spoof that header to defeat rate limiting and hide its IP.

Both apps listen on **all interfaces** (`:8080` / `:8081`), so you must block those ports from the internet:

- **Cloud (GCP, etc.):** inbound is default-deny — simply never open 8080/8081 in the firewall. Only allow 80, 443, and 22 (SSH). See [Part D](#d3-firewall--static-ip).
- **Home / self-managed:** use a host firewall (see [Hardening](#firewall)) to allow 8080/8081 only from localhost, or ensure your router never forwards them.

### Advanced: home behind NAT with DuckDNS

If you have no public domain and your home connection is behind a router/NAT (so Let's Encrypt can't reach port 80), use a free dynamic-DNS hostname from [DuckDNS](https://www.duckdns.org/) and let Caddy prove domain ownership over **DNS** instead of HTTP. This needs a Caddy build that includes the DuckDNS DNS plugin — the stock `apt` Caddy does **not** have it. Build a custom binary with [`xcaddy`](https://github.com/caddyserver/xcaddy):

```bash
xcaddy build --with github.com/caddy-dns/duckdns
```

Then use the DNS challenge in your Caddyfile (token from your DuckDNS account):

```caddy
kopds.example.duckdns.org {
	reverse_proxy localhost:8080
	tls {
		dns duckdns {env.DUCKDNS_TOKEN}
	}
}
```

This is an advanced path; if you have any real domain, the standard setup in B3–B4 is far simpler.

---

## 🐳 Part C — Docker Alternative

If you prefer containers, the combined stack in [`deploy/docker-compose.yml`](deploy/docker-compose.yml) runs KOPDS, KOSYNC, and Caddy together. Crucially, **only Caddy publishes ports** — the app containers are reachable solely on the internal Docker network, which keeps 8080/8081 off the internet automatically.

1. Install [Docker and Compose](https://docs.docker.com/get-docker/).
2. Copy `deploy/docker-compose.yml` and `deploy/Caddyfile` into a directory on your server.
3. In `docker-compose.yml`: set the Calibre library host path, both `KO*_BASE_URL`/hostnames. In `Caddyfile`: set your real hostnames (keep the `kopds:8080` / `kosync:8081` **service-name** targets — Caddy reaches the apps by container name).
4. Launch and create your first user:

```bash
docker compose up -d
docker exec -it kopds  ./kopds  create-user admin
docker exec -it kosync ./kosync create-user admin
```

> **CLI logging in Docker:** `docker exec` runs as a separate process — its output goes to your terminal, not to `docker logs`. CLI user-management commands will not appear in `docker logs` regardless of log settings. If you need a persistent record of these operations, add `KOPDS_LOG_PATH=/data/kopds.log` and `KOSYNC_LOG_PATH=/app/data/kosync.log` to the environment sections of docker-compose.yml; both data directories are already mounted volumes.

DNS and certificates work exactly as in [Part B](#b4-point-dns-and-confirm-https): point your A records at the host, and Caddy handles HTTPS on first request.

---

## ☁️ Part D — Cloud: Free-Tier GCP VM

Google Cloud's [Always Free](https://cloud.google.com/free/docs/free-cloud-features#compute) tier includes one small `e2-micro` VM, which is enough to run this stack. This section provisions the VM; the actual install is just [Part A](#-part-a--native-install-recommended) or [Part C](#-part-c--docker-alternative) run on it.

> Free-tier terms, eligible regions, and quotas change over time — **verify the current [Google Cloud Free Tier](https://cloud.google.com/free) terms** before relying on them. As of writing, the free `e2-micro` is limited to specific US regions (e.g. `us-west1`, `us-central1`, `us-east1`) and a modest monthly egress allowance.

### D1. The 1 GB reality check

An `e2-micro` has ~1 GB RAM. That shapes what fits:

- **KOSYNC** is featherweight (idles under ~15 MB). It is an ideal fit and leaves headroom to spare.
- **KOPDS** is heavier: it streams books, resizes covers, and — on a cloud VM — must reach your Calibre library over the network (rclone). The initial metadata scan of a large library is the peak-memory moment. Running both apps on 1 GB is doable but tight, so **a swap file is mandatory** (D4) and you should keep your library size sane and `KOPDS_LOG_LEVEL=info`.

### D2. Create the VM

Via the [Cloud Console](https://console.cloud.google.com/) (Compute Engine → Create instance) or the `gcloud` CLI.  If using the `gcloud` CLI, first you must install it on your machine.  Then run `gcloud auth login` and `gcloud init`.  Run `gcloud config list` to check your configuration.  Run `gcloud iam service-accounts list` to find your service account, which you will need below.  To find an image for your VM OS, run `gcloud compute images list --filter="name=ubuntu"`, replacing `ubuntu` with your preferred OS.  Before creating the VM you will also need to choose a region and zone.  Be aware, free-tier e2-micro VMs are available only in certain regions.  And then from within that region, you'll need to pick a zone.  Create the VM instance using the example below, but make sure to replace the <vm-name>, <project-name>, zone, <service-account>, <device-name>, image:
 
```bash
 gcloud compute instances create <vm-name> \
 --project=<project-name> \
 --zone=us-east1-b \
 --machine-type=e2-micro \
 --network-interface=network-tier=STANDARD,stack-type=IPV4_ONLY,subnet=default \
 --maintenance-policy=MIGRATE \
 --provisioning-model=STANDARD \
 --service-account=<service-account> \
 --scopes=\
https://www.googleapis.com/auth/devstorage.read_only,\
https://www.googleapis.com/auth/logging.write,\
https://www.googleapis.com/auth/monitoring.write,\
https://www.googleapis.com/auth/service.management.readonly,\
https://www.googleapis.com/auth/servicecontrol,\
https://www.googleapis.com/auth/trace.append \
 --tags=http-server,https-server \
 --create-disk=\
auto-delete=yes,\
boot=yes,\
device-name=<device-name>,\
image=projects/ubuntu-os-cloud/global/images/ubuntu-minimal-2604-resolute-amd64-v20260529,\
mode=rw,\
size=10,\
type=pd-standard \
 --no-shielded-secure-boot \
 --shielded-vtpm \
 --shielded-integrity-monitoring \
 --labels=goog-ec-src=vm_add-gcloud \
 --reservation-affinity=any
```

### D3. Firewall & static IP

- **Static IP (not free):** reserve and assign a static external IP so your DNS records stay valid across reboots (Console: VPC network → IP addresses → Reserve; or `gcloud compute addresses create`).  **A static IP is not included in the free-tier.**  You can use a dynamic IP, but you will have to update your DNS records every time you reboot the VM.
- **Firewall:** GCP inbound is **default-deny**. Add a rule allowing **only** TCP `80`, `443` (and `22` for SSH, usually already allowed). **Do not** open `8080`/`8081` — that is precisely the boundary from [Why the app ports stay private](#why-the-app-ports-stay-private).

```bash
gcloud compute firewall-rules create allow-http-https \
    --network=default \
    --direction=INGRESS \
    --priority=1000 \
    --action=ALLOW \
    --allow=tcp:80,tcp:443 \
    --source-ranges=0.0.0.0/0
```

### D4. Add swap (mandatory on 1 GB)

SSH in to your VM from the Console or `gcloud compute ssh <vm-name>`.

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile && sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

This prevents Out-of-Memory crashes during KOPDS's initial library scan.

### D5. DNS

Point your `kopds.*` and `kosync.*` A records at the VM's IP. With a public IP and ports 80/443 open, Caddy's automatic HTTPS just works — no DuckDNS workaround needed here.

### D6. Install the apps

Now follow **[Part A](#-part-a--native-install-recommended)** (recommended on 1 GB — lower overhead) or **[Part C](#-part-c--docker-alternative)**, then **[Part B](#-part-b--caddy-reverse-proxy--https)** for the proxy (Part A only; Part C bundles Caddy).

### KOPDS: the Calibre library on a cloud VM

Your books aren't on the VM, so mount them with rclone (Go-native, handles network blips, and can be throttled to spare a small VM). After [configuring an rclone remote](https://rclone.org/docs/) for your cloud storage, mount it read-only and run KOPDS against the mount, while keeping the KOPDS index/cache on the local disk:

```bash
rclone mount mystorage:calibre /srv/calibre/library \
  --read-only --allow-other --vfs-cache-mode minimal --bwlimit 8M --daemon
```

> **FUSE permissions (important):** if you create the mount as `root` but KOPDS runs as the `kopds` user, `kopds` cannot traverse the mount unless you pass `--allow-other` **and** uncomment `user_allow_other` in `/etc/fuse.conf`. Otherwise KOPDS gets "permission denied" reading the library. (Alternatively, run the mount as the same `kopds` user, in which case `--allow-other` isn't needed.)

For a permanent setup, wrap this in its own systemd service (ordered `Before=kopds.service`) so the mount is ready before KOPDS starts. Throttle with `--bwlimit` during the first scan so the metadata sync doesn't saturate a small VM's network or memory.

The sample unit is in [`deploy/systemd/kopds-library.service`](deploy/systemd/kopds-library.service). Configure rclone, install the unit, and enable it:

```bash
# Configure rclone as the kopds user — stores the config inside the kopds
# data directory so the system account can always find it.
sudo -u kopds env RCLONE_CONFIG=/var/lib/kopds/rclone.conf rclone config

# Create the mount point owned by the kopds user.
# Running the mount as kopds means --allow-other is not needed.
sudo mkdir -p /srv/calibre/library
sudo chown kopds:kopds /srv/calibre/library

# Install the unit, set your remote name, then enable it.
sudo cp deploy/systemd/kopds-library.service /etc/systemd/system/
sudo nano /etc/systemd/system/kopds-library.service   # replace mystorage:calibre
sudo systemctl daemon-reload
sudo systemctl enable --now kopds-library
sudo systemctl status kopds-library   # should show "active (running)"
```

`Before=kopds.service` enforces ordering only when both units activate together (e.g. at boot). To make `systemctl start kopds` always bring up the mount first when started manually, add these two lines to the `[Unit]` section of your `/etc/systemd/system/kopds.service`:

```ini
Wants=kopds-library.service
After=kopds-library.service
```

---

## 🛡 Hardening & Operations

### Firewall

On a self-managed/home box, use UFW to expose only what the public needs and keep the app ports local:

```bash
sudo ufw allow 22/tcp        # SSH
sudo ufw allow 80,443/tcp    # Caddy
sudo ufw enable
```

UFW's default is to deny other inbound traffic, so 8080/8081 are not reachable from outside. (On GCP, the cloud firewall in [D3](#d3-firewall--static-ip) does this job.)

### Backups

Both apps store everything in a single SQLite file, so backups are a file copy — but use SQLite's `.backup` so you never copy a half-written database:

```bash
# KOSYNC progress is irreplaceable — back it up:
sqlite3 /var/lib/kosync/kosync.db ".backup '/var/backups/kosync-$(date +%F).db'"
```

Put that in a daily `cron` job and copy the result off the machine. The **KOPDS** index is rebuildable from your Calibre library, so backing it up is optional; back up the Calibre library itself instead.

### Updates

- **Native:** download/build the new binary, replace it, `sudo systemctl restart kopds kosync`.
- **Docker:** `docker compose pull && docker compose up -d`.

### Rate limiting & fail2ban

Both apps rate-limit by client IP out of the box (`KO*_RATE_LIMIT_*`), which works correctly because `TRUST_PROXY_HEADERS=true` gives them the real IP from Caddy. For an extra layer, Caddy can rate-limit at the edge, and [`fail2ban`](https://github.com/fail2ban/fail2ban) can ban IPs that rack up 401s in your logs.

### Logs

By default each service logs to the journal — `journalctl -u kopds` / `-u kosync`. Set `KO*_JSON_LOG=true` if you ship logs to Loki/ELK. If you set `KO*_LOG_PATH` to a file, add a `logrotate` rule so it doesn't grow unbounded; otherwise rely on the journal, which rotates itself.

---

## 📱 Connecting KOReader

Once HTTPS is live, point your devices at the public URLs:

- **KOPDS (books):** OPDS catalog → Add catalog → `https://kopds.example.com/opds/v1.2/catalog`, with the username/password you created.
- **KOSYNC (progress):** Tools → Progress sync → Custom sync server → `https://kosync.example.com`, then Register/Login with your credentials.

Full client walkthroughs are in each app's README under **Usage with KOReader**.

---

## ❓ Troubleshooting

**Caddy returns 502 Bad Gateway.** The app isn't reachable from Caddy. Check the service is running (`systemctl status kopds`) and that the Caddyfile target matches how you deployed: `localhost:8080` for native, `kopds:8080` for Docker.

**Certificate won't issue.** Caddy needs inbound 80/443 and public DNS resolving to this server. Confirm your A record and firewall. Behind home NAT with no domain, use the [DuckDNS DNS challenge](#advanced-home-behind-nat-with-duckdns).

**KOPDS lists books but downloads 404.** The Calibre library mount dropped. KOPDS serves from its local index but streams files from the live library — remount it (check your rclone mount / `KOPDS_LIBRARY_PATH`).

**Database won't open / "readonly database" under systemd.** The data directory isn't writable by the sandbox. Make sure `ReadWritePaths=` lists it (SQLite needs to write `-wal`/`-shm` beside the `.db`).

**KOReader "Network Error" or 406.** Confirm the public URL is correct and HTTPS is valid. KOSYNC is strict about the `Accept: application/vnd.koreader.v1+json` header the client sends — a misconfigured proxy that rewrites headers can break it.

**Out-of-memory on a small VM.** Add swap ([D4](#d4-add-swap-mandatory-on-1-gb)) and lower load during the first KOPDS scan with rclone `--bwlimit`.

For deeper diagnosis, set `KO*_LOG_LEVEL=debug` and watch `journalctl -u <app> -f`. Each app's README has an app-specific troubleshooting section.
