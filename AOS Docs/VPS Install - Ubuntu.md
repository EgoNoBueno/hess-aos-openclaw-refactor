# AOS OpenClaw — Ubuntu VPS Install Procedure

Deploying **this fork** of OpenClaw (the Hess AOS refactor) to a Ubuntu VPS using Docker.  
This builds and runs the gateway from the local source tree rather than pulling from npm.

---

## Prerequisites

| Requirement | Details |
|---|---|
| VPS OS | Ubuntu 22.04 LTS or 24.04 LTS (64-bit) |
| RAM | 2 GB minimum (4 GB recommended for Docker builds) |
| Disk | 20 GB minimum |
| Access | SSH with root or sudo |
| Local machine | Git, SSH client |

---

## What you need before you start

- SSH access to your VPS (`ssh root@YOUR_VPS_IP` or a sudo user)
- A GitHub account with access to this repo (or the code already on the VPS)
- API key(s) for whichever AI provider you plan to use (Anthropic, OpenAI, etc.)
- Bot credentials for any channels you want to connect (Telegram token, Discord token, etc.)

---

## Step 1 — Connect to the VPS

```bash
ssh root@YOUR_VPS_IP
```

> If you provisioned a sudo user instead of root, substitute your username and prefix relevant commands with `sudo`.

---

## Step 2 — Update the system and install base tools

```bash
apt-get update && apt-get upgrade -y
apt-get install -y git curl ca-certificates unzip
```

---

## Step 3 — Install Tailscale on the VPS

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

Authenticate and join your tailnet:

```bash
tailscale up
```

A browser URL will be printed — open it on any machine to approve the VPS. Once approved, get its Tailscale IP:

```bash
tailscale ip -4
# Example output: 100.x.x.x
```

Note this IP — you will use it to connect from your home computer.

---

## Step 4 — Install Docker

Use the official Docker install script:

```bash
curl -fsSL https://get.docker.com | sh
```

Verify:

```bash
docker --version
docker compose version
```

Expected output (versions may differ):

```
Docker version 26.x.x, build ...
Docker Compose version v2.x.x
```

> **If you are running as a non-root user** add yourself to the docker group and re-login:
> ```bash
> usermod -aG docker $USER
> newgrp docker
> ```

---

## Step 5 — Clone this repository onto the VPS

```bash
git clone https://github.com/openclaw/openclaw.git
cd openclaw
```

> If this is a private fork, use the appropriate repo URL and ensure your SSH key or personal access token is configured.

---

## Step 6 — Create persistent state directories

Docker containers are ephemeral; all long-lived state lives on the host.

```bash
mkdir -p /root/.openclaw/workspace

# The container runs as uid 1000 (the 'node' user in the base image).
chown -R 1000:1000 /root/.openclaw
```

---

## Step 7 — Configure environment variables

Create a `.env` file in the repo root.  
**Do not commit this file.**

```bash
cat > .env << 'EOF'
# Image name (used by docker compose)
OPENCLAW_IMAGE=openclaw:local

# Gateway auth token — generate a strong random value
OPENCLAW_GATEWAY_TOKEN=change-me-now

# 'lan' lets Docker expose the port to the host; UFW restricts it to Tailscale only (see Step 9)
OPENCLAW_GATEWAY_BIND=lan
OPENCLAW_GATEWAY_PORT=18789
OPENCLAW_BRIDGE_PORT=18790

# Host paths that will be volume-mounted into the container
OPENCLAW_CONFIG_DIR=/root/.openclaw
OPENCLAW_WORKSPACE_DIR=/root/.openclaw/workspace

# Required if you use Gmail via gog
GOG_KEYRING_PASSWORD=change-me-now
EOF
```

Generate strong secrets to replace the placeholders:

```bash
openssl rand -hex 32   # run once for OPENCLAW_GATEWAY_TOKEN
openssl rand -hex 32   # run again for GOG_KEYRING_PASSWORD
```

Edit `.env` and paste the generated values.

---

## Step 8 — Review the docker-compose.yml

The repo ships a `docker-compose.yml` at the root.  
For a production VPS deployment, confirm it matches this profile (edit as needed):

```yaml
services:
  openclaw-gateway:
    image: ${OPENCLAW_IMAGE:-openclaw:local}
    build: .
    restart: unless-stopped
    environment:
      HOME: /home/node
      NODE_ENV: production
      TERM: xterm-256color
      OPENCLAW_GATEWAY_TOKEN: ${OPENCLAW_GATEWAY_TOKEN}
      GOG_KEYRING_PASSWORD: ${GOG_KEYRING_PASSWORD}
    volumes:
      - ${OPENCLAW_CONFIG_DIR}:/home/node/.openclaw
      - ${OPENCLAW_WORKSPACE_DIR}:/home/node/.openclaw/workspace
    ports:
      # Bind to all host interfaces so Tailscale can route traffic in.
      # UFW (Step 9) restricts port 18789 to the tailscale0 interface only.
      - "${OPENCLAW_GATEWAY_PORT:-18789}:18789"
    command:
      [
        "node", "openclaw.mjs", "gateway",
        "--bind", "${OPENCLAW_GATEWAY_BIND:-loopback}",
        "--port", "18789",
        "--allow-unconfigured"
      ]
```

Key points:
- `restart: unless-stopped` keeps the gateway alive after reboots.
- The port binds to all host interfaces so Tailscale can route traffic in; UFW (Step 9) restricts it to the `tailscale0` interface only.
- `--allow-unconfigured` lets the gateway start before a model is configured; remove it once you have configured a provider.

---

## Step 9 — Firewall: restrict port 18789 to Tailscale only

This is the key step that replaces the SSH tunnel. The gateway port is open on the host but only reachable via Tailscale:

```bash
ufw allow OpenSSH
ufw allow in on tailscale0 to any port 18789 proto tcp   # allow from tailnet only
ufw deny 18789                                             # block from public internet
ufw enable
```

Verify:

```bash
ufw status
```

Expected output includes:
```
18789                      ALLOW IN    on tailscale0
18789                      DENY
OpenSSH                    ALLOW IN    Anywhere
```

---

## Step 10 — Build the Docker image

This step compiles TypeScript, bundles the UI, and produces the `dist/` output inside the image.  
It takes 5–15 minutes on first run.

```bash
docker compose build
```

> **OOM on small VPS?**  
> The Dockerfile already limits Node's heap during `pnpm install` (`NODE_OPTIONS=--max-old-space-size=2048`).  
> If the build still gets killed (exit 137), either upgrade to a 4 GB RAM droplet or add 2 GB swap:
> ```bash
> fallocate -l 2G /swapfile
> chmod 600 /swapfile
> mkswap /swapfile
> swapon /swapfile
> echo '/swapfile none swap sw 0 0' >> /etc/fstab
> ```
> Then re-run `docker compose build`.

---

## Step 11 — Launch the gateway

```bash
docker compose up -d openclaw-gateway
```

Check it started:

```bash
docker compose logs -f openclaw-gateway
```

Look for a line like:

```
[gateway] listening on ws://127.0.0.1:18789
```

Press `Ctrl+C` to stop following logs (the gateway keeps running in the background).

---

## Step 12 — Access the Control UI via Tailscale

No SSH tunnel needed. On your **home computer** (which must also be on the same Tailscale tailnet), open a browser directly to the VPS Tailscale IP:

```
http://100.x.x.x:18789/
```

Replace `100.x.x.x` with your VPS Tailscale IP from Step 3. Paste your `OPENCLAW_GATEWAY_TOKEN` when prompted.

> **Tailscale not up yet on your home computer?**  
> Install Tailscale on your home machine too: https://tailscale.com/download  
> Both devices must be logged into the same tailnet account.

---

## Step 13 — Configure your AI provider

> **Deployment phases:**
> - **Phase 1 (now):** Single cloud provider — OpenAI. Get the system working first.
> - **Phase 2:** Add local Ollama on your home computer (free inference, via Tailscale).
> - **Phase 3:** Multi-model swarm — Architect (Claude/GPT-4o) + Worker (Haiku/DeepSeek) + Critic (local Ollama). See `AOS Docs/Obsidian Memory Setup.md` for the embedding side of Phase 2+.

Inside the Control UI (or via `openclaw config set` after exec-ing into the container):

```bash
# Phase 1: Set OpenAI API key (start here)
docker compose exec openclaw-gateway node openclaw.mjs config set openai.apiKey YOUR_OPENAI_KEY

# Set default model
docker compose exec openclaw-gateway node openclaw.mjs config set model.default gpt-4o-mini

# Set gateway mode
docker compose exec openclaw-gateway node openclaw.mjs config set gateway.mode local
```

Alternative providers (swap in as needed):

```bash
# Anthropic (Claude)
docker compose exec openclaw-gateway node openclaw.mjs config set anthropic.apiKey YOUR_ANTHROPIC_KEY
docker compose exec openclaw-gateway node openclaw.mjs config set model.default claude-opus-4-5
```

---

## Step 14 — Verify persistence

Confirm state is written to the host volume (not lost on restarts):

```bash
ls -la /root/.openclaw/
```

You should see `openclaw.json`, `credentials/`, `sessions/`, and `workspace/`.

Test a restart:

```bash
docker compose restart openclaw-gateway
docker compose logs -f openclaw-gateway
```

The gateway should come back and your configuration should be intact.

---

## Optional: Tailscale Serve (HTTPS + identity headers)

If you want HTTPS and the ability to skip entering a token in the browser (using Tailscale identity auth instead), you can run Tailscale Serve on the host alongside the Docker gateway:

```bash
# On the VPS host (not inside Docker)
tailscale serve --bg 18789
```

Then access via the MagicDNS HTTPS URL (shown in `tailscale cert --domains` output or your Tailscale admin panel):

```
https://<machine-name>.<tailnet-name>.ts.net/
```

For this to work without a token, also set in `openclaw.json`:

```json5
{
  gateway: {
    bind: "loopback",
    tailscale: { mode: "serve" },
    auth: { allowTailscale: true }
  }
}
```

And change `OPENCLAW_GATEWAY_BIND=loopback` in your `.env` (revert it from `lan`), and change the docker-compose port back to `"127.0.0.1:18789:18789"`.  
This is more complex — the direct `tailnet` IP + UFW approach in the main steps is simpler for most setups.

---

## Optional: Bake extra binaries into the image

If your AOS workflows require external binaries (e.g., `gog` for Gmail, `wacli` for WhatsApp), bake them into the image at build time.  
**Do not install them at runtime** — they will not survive a container restart.

Create a `Dockerfile.aos` extending the base:

```dockerfile
FROM openclaw:local

USER root

# Gmail CLI
RUN curl -L https://github.com/steipete/gog/releases/latest/download/gog_Linux_x86_64.tar.gz \
  | tar -xz -C /usr/local/bin && chmod +x /usr/local/bin/gog

# WhatsApp CLI
RUN curl -L https://github.com/steipete/wacli/releases/latest/download/wacli_Linux_x86_64.tar.gz \
  | tar -xz -C /usr/local/bin && chmod +x /usr/local/bin/wacli

USER node
```

Build and tag it:

```bash
docker build -f Dockerfile.aos -t openclaw:aos .
```

Update `OPENCLAW_IMAGE=openclaw:aos` in `.env`, then:

```bash
docker compose up -d openclaw-gateway
```

---

## Optional: Chromium / browser automation

The Dockerfile supports baking Playwright Chromium in at build time (avoids the 60–90s install on each container start):

```bash
docker compose build --build-arg OPENCLAW_INSTALL_BROWSER=1
```

Adds ~300 MB to the image.

---

## Firewall hardening (already done in Step 9)

The setup restricts port 18789 to `tailscale0` only:

```bash
ufw allow OpenSSH
ufw allow in on tailscale0 to any port 18789 proto tcp
ufw deny 18789
ufw enable
```

Port 18789 is unreachable from the public internet. Only devices on your tailnet can reach it.

---

## Ongoing maintenance

### Pull code updates and rebuild

```bash
git pull --rebase origin main
docker compose build
docker compose up -d openclaw-gateway
```

### View live logs

```bash
docker compose logs -f openclaw-gateway
```

### Stop the gateway

```bash
docker compose stop openclaw-gateway
```

### Full teardown (preserves state volumes)

```bash
docker compose down
```

> State in `/root/.openclaw` is never touched by `docker compose down`. Your config and sessions survive.

---

## Persistence summary

| What | Where on host | Survives restart? |
|---|---|---|
| Gateway config (`openclaw.json`) | `/root/.openclaw/` | Yes |
| Model auth / API keys | `/root/.openclaw/credentials/` | Yes |
| Agent sessions | `/root/.openclaw/sessions/` | Yes |
| Workspace files | `/root/.openclaw/workspace/` | Yes |
| WhatsApp session | `/root/.openclaw/` | Yes |
| Installed npm packages | Container image | Rebuild required |
| Extra binaries (gog, wacli…) | Container image | Rebuild required |

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `docker compose build` killed (exit 137) | OOM during `pnpm install` | Add 2 GB swap (see Step 10) or upgrade VPS RAM |
| Gateway starts then immediately exits | Missing `.env` values | Check `docker compose logs openclaw-gateway` for missing env vars |
| `http://100.x.x.x:18789/` unreachable | Tailscale not connected | Run `tailscale status` on both machines; both must be on the same tailnet |
| Config is lost after restart | Volume not mounted | Confirm `OPENCLAW_CONFIG_DIR` in `.env` and that `chown 1000:1000` was run |
| `permission denied` on `/root/.openclaw` | Wrong ownership | `chown -R 1000:1000 /root/.openclaw` |
| Port 18789 reachable from public internet | UFW rule missing or wrong | Re-run the UFW commands in Step 9; confirm with `ufw status` |
| Build fails on `pnpm ui:build` | Missing UI deps | The Dockerfile handles this; if running outside Docker, run `pnpm ui:install` first |
