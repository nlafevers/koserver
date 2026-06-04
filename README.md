# KOSERVER - Self-Hosted KOReader Backend

[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0)

KOSERVER is a documentation and deployment repository for two lightweight Go applications that together provide a complete self-hosted backend for [KOReader](https://koreader.rocks/) devices:

- **[KOPDS](https://github.com/nlafevers/kopds)** — an OPDS 1.2 server for browsing, searching, and downloading books from your Calibre library.
- **[KOSYNC](https://github.com/nlafevers/kosync)** — a reading-progress synchronization server that keeps your position in sync across all your KOReader devices.

Both apps are built in pure Go, ship as single binaries, and are designed to run comfortably on the most resource-constrained self-hosting hardware — a Raspberry Pi Zero, a free-tier cloud VM, or an old PC in a closet.

---

## 📖 Table of Contents

1. [Why KOSERVER?](#-why-koserver)
2. [Repository Layout](#-repository-layout)
3. [Quick Links](#-quick-links)
4. [License](#-license)

---

## 🚀 Why KOSERVER?

KOReader is a powerful open-source e-book reader, but its official sync and library infrastructure assumes an always-on internet connection and third-party services. KOSERVER exists for readers who want full ownership of their data:

- **Privacy-First:** Your reading habits, library, and sync data never leave your own hardware.
- **Featherweight:** The two binaries plus a Caddy reverse proxy idle at under 100 MB of RAM combined — less than Docker's overhead alone.
- **Zero Maintenance:** Both apps use a single SQLite file for storage. Backups are a one-line `sqlite3` command. Updates are a binary swap.
- **KOReader Native:** Both apps implement the exact protocols KOReader expects, so setup is a matter of pointing your device at your server's URL.

---

## 🗂 Repository Layout

```
koserver/
├── deployment-guide.md   # Full deployment walkthrough (native + Docker + GCP)
├── deploy/               # Copy-paste-ready config files
│   ├── Caddyfile         # Reverse proxy config (native + Docker variants)
│   ├── docker-compose.yml# Combined KOPDS + KOSYNC + Caddy stack
│   └── systemd/          # systemd service units for native installs
├── kopds/                # KOPDS source (separate git repository)
└── kosync/               # KOSYNC source (separate git repository)
```

> `kopds/` and `kosync/` are tracked in their own separate Git repositories. Code changes inside those directories must be committed there, not here.

---

## 🔗 Quick Links

| Resource | Link |
| :--- | :--- |
| **Deployment guide** (start here) | [deployment-guide.md](deployment-guide.md) |
| **KOPDS README** (OPDS server) | [kopds/README.md](kopds/README.md) |
| **KOSYNC README** (sync server) | [kosync/README.md](kosync/README.md) |
| **Sample configs** | [deploy/](deploy/) |
| **KOPDS releases** | https://github.com/nlafevers/kopds/releases |
| **KOSYNC releases** | https://github.com/nlafevers/kosync/releases |

---

## 📜 License

KOSERVER is released under the **GPL-3.0 License**. See the [LICENSE](LICENSE) file for details.
