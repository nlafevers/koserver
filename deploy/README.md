# Deployment Sample Files

Copy-paste-ready configuration referenced by [`../deployment-guide.md`](../deployment-guide.md).
Read the guide first — these files need your real paths, hostnames, and domains before they will
work.

| File | Purpose |
| :--- | :--- |
| `Caddyfile` | Caddy reverse proxy + automatic HTTPS for both apps (Docker service-name targets, with native `localhost` alternatives commented inline). |
| `docker-compose.yml` | Combined KOPDS + KOSYNC + Caddy stack. Only Caddy publishes ports; the apps stay on the internal network. |
| `systemd/kopds.service` | systemd unit for the native KOPDS binary (dedicated user, hardened). |
| `systemd/kosync.service` | systemd unit for the native KOSYNC binary (dedicated user, hardened). |

KOPDS and KOSYNC are independent — deploy one or both. See the guide for the native (recommended),
Docker, and free-tier GCP walkthroughs.
