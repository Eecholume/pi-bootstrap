# elminster-stack

Web stack for Elminster (Raspberry Pi 5, 16GB) — Phase 2 of Project Elminster.

Phase 1 gave us Ollama + 15 models + a terminal CLI.
Phase 2 gives Mary a browser she can open on her phone and just chat.

## Architecture

```
Mary's phone/laptop
  → https://chat.lab.hoens.fun     → Caddy → Open WebUI (:8080) → Ollama (:11434)

Abe's browser
  → https://dockge.lab.hoens.fun   → Caddy → Dockge (:5001)
```

All services run on Elminster (`10.0.0.70`). All traffic is LAN-only. Caddy terminates TLS using a wildcard cert for `*.lab.hoens.fun` via Cloudflare DNS-01 ACME.

## Components

| Service | Image | Purpose |
|---|---|---|
| Docker Engine | docker-ce (apt) | Container runtime |
| Caddy | Custom build w/ cloudflare plugin | Reverse proxy + TLS |
| Dockge | `louislam/dockge:1` | Docker compose manager |
| Open WebUI | `ghcr.io/open-webui/open-webui:main` | Chat UI for Ollama |

## Prerequisites

- Elminster with Ollama running (Phase 1 complete)
- Cloudflare API token with `Zone:DNS:Edit` on `hoens.fun`
- Internet connection (pulls Docker images)

## Deploy

```bash
git clone https://github.com/Eecholume/elminster-stack.git
cd elminster-stack
sudo bash deploy.sh
```

The script will prompt for the Cloudflare API token. Have it ready.

## What It Does

1. Installs Docker CE from the official repo (with trixie→bookworm fallback)
2. Prompts for Cloudflare API token, stores it at `/etc/caddy/.env` (mode 600)
3. Builds a custom Caddy image with the `caddy-dns/cloudflare` plugin
4. Deploys Caddy on host network for TLS on 443/80
5. Deploys Dockge on port 5001
6. Deploys Open WebUI on port 8080, connected to Ollama
7. Verifies everything is running

All compose stacks are installed to `/opt/stacks/` so Dockge can manage them.

## Post-Deploy

1. Add OPNsense Unbound host overrides:
   - `chat.lab.hoens.fun` → `10.0.0.70`
   - `dockge.lab.hoens.fun` → `10.0.0.70`
2. Open `https://dockge.lab.hoens.fun` — create Abe's admin account
3. Open `https://chat.lab.hoens.fun` — first user becomes admin (Abe), then create Mary's account
4. Lock registration: change `ENABLE_SIGNUP=true` → `false` in `/opt/stacks/open-webui/compose.yaml`, then restart

## Privacy

- `WEBUI_TELEMETRY=false` + `DO_NOT_TRACK=true`
- `ENABLE_COMMUNITY_SHARING=false`
- Docker CE has no telemetry
- Caddy has no telemetry
- Dockge has no telemetry
- Zero WAN exposure (OPNsense blocks inbound, Unbound split-horizon DNS)
- All data on local NVMe

## Rollback

```bash
# Remove Open WebUI
docker stop open-webui && docker rm open-webui
docker volume rm open-webui-data

# Remove Dockge
docker stop dockge && docker rm dockge
rm -rf /opt/dockge /opt/stacks

# Remove Caddy
docker stop caddy && docker rm caddy
rm -rf /etc/caddy

# Remove Docker entirely
sudo apt purge docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo rm -rf /var/lib/docker /var/lib/containerd
```

## License

MIT
