# japan-plan-deploy

Deployment configuration for the Japan trip planner ([japan_plan](https://github.com/Br0bertinus/japan_plan)), served at https://japan.black-inc.dev.

This repo has no application code. It holds the Docker Compose file and the deploy workflow. The image is built in CI in `japan_plan` and pulled from GHCR on the server.

## Architecture

```
Internet ─► Caddy (owned by callsheet-deploy, ports 80/443, automatic TLS)
              ├─ $CALLSHEET_HOST         ─► callsheet ui container
              └─ japan.black-inc.dev     ─► japan-plan-ui container (this repo)
                                              nginx serving static Vite build
```

- `japan_plan` CI builds `ghcr.io/<owner>/japan-plan` (`:latest` and `:<sha>`), then sends a `deploy` `repository_dispatch` event to this repo.
- This repo's workflow SSHes into the VM, writes `.env`, and runs `scripts/deploy.sh` (fetch, `docker compose pull`, `up -d`, health check).
- The app is fully static: no API, no backend secrets.

## Decisions

### Separate deploy repo per app, mirroring callsheet
Same split as callsheet: app repo owns CI (build and push image), deploy repo owns CD (server config and secrets). Keeps deploy credentials out of the app repo and lets each app deploy independently.

### Reuse callsheet's Caddy instead of running a second proxy
The VM has one public IP and Caddy in `callsheet-deploy` already binds ports 80/443, so a second Caddy cannot start. Caddy there has a second site block for `japan.black-inc.dev` that proxies to `japan-plan-ui:80`.

Trade-off: routing for this app lives in `callsheet-deploy`, so adding or changing a hostname means editing that repo. If more apps are added, extract Caddy into its own shared proxy repo (the shared network below makes that a small move).

### Shared external Docker network `web`
Separate compose projects have separate default networks, so Caddy cannot reach this container by name. `callsheet-deploy` defines a network named `web` and attaches Caddy to it. This repo declares it `external: true` and attaches `japan-plan-ui`. Nothing is published to the host.

Consequence: **callsheet-deploy must have been deployed (creating `web`) before this stack can start.**

### Hostname is hardcoded in the Caddyfile
`callsheet-deploy` takes its hostname from a `CALLSHEET_HOST` secret. The japan hostname is not secret, so it is hardcoded in the Caddyfile to avoid another secret and `.env` entry.

### DNS
No change needed. The Cloudflare wildcard `*` A record (DNS only, grey cloud) already points `japan.black-inc.dev` at the VM. Do not enable the orange-cloud proxy, as it conflicts with Caddy's Let's Encrypt certificates.

## One-time setup

1. **callsheet-deploy**: merge the Caddy and network change and let it deploy (or run `bash scripts/deploy.sh` on the VM). This creates the `web` network and the `japan.black-inc.dev` route.
2. **GitHub**: create the `Br0bertinus/japan-plan-deploy` repo and push this directory to it.
3. **GitHub secrets**:

   | Repo | Secret | Value |
   |---|---|---|
   | `japan-plan-deploy` | `DEPLOY_HOST` | VM public IP (same as callsheet) |
   | `japan-plan-deploy` | `DEPLOY_USER` | `ubuntu` |
   | `japan-plan-deploy` | `DEPLOY_SSH_KEY` | Private key from `~/.ssh/callsheet_deploy` (or a new deploy key added to the VM's `authorized_keys`) |
   | `japan_plan` | `GH_PAT` | PAT (classic) with `repo` and `write:packages` (same as callsheet) |

4. **VM**: clone this repo next to callsheet-deploy:
   ```bash
   cd ~/projects
   git clone https://github.com/Br0bertinus/japan-plan-deploy.git
   ```
   The existing GHCR login on the VM (`read:packages`) also covers the new image.
5. **First image**: push to `main` in `japan_plan` so CI builds the image and triggers this repo's deploy. Running `deploy.sh` before an image exists fails on `docker compose pull`.
6. Verify: https://japan.black-inc.dev

## Troubleshooting

- **`network web declared as external, but could not be found`**: callsheet-deploy has not been deployed with the shared network yet (step 1).
- **502 from Caddy**: `japan-plan-ui` is not running or not on the `web` network. Check `docker compose ps` here and `docker network inspect web`.
- **`pull access denied`**: the VM's GHCR login is missing or the PAT lacks `read:packages`.
