# AOS OpenClaw — Obsidian Memory & AOS Storage Setup

Connecting Obsidian (on your home computer) to the OpenClaw agent's long-term memory system (on the VPS), as specified in the AOS Specification §2.3 and Phase 3 of the build procedure.

---

## Overview

```
Your home computer (Obsidian app)
        ↕  Syncthing over Tailscale (live sync)
Your VPS /root/.openclaw/workspace/
        ↕  Docker volume mount
Container /home/node/.openclaw/workspace/
        ↕
OpenClaw agent (reads/writes MEMORY.md, memory/*.md, SOUL.md)
```

**Obsidian is the human-facing view** of the exact same Markdown files the agent reads and writes. When the agent records a memory, it appears in your Obsidian vault. When you edit a note in Obsidian, the agent sees it on the next recall.

### Memory file layout

| File | Purpose | When written |
|---|---|---|
| `MEMORY.md` | Curated long-term facts, preferences, identity | You or the agent; durable |
| `memory/YYYY-MM-DD.md` | Daily running log | Agent appends during session |
| `SOUL.md` | Agent identity and persona | You define it once; agent may evolve it |

All files live in the workspace (`~/.openclaw/workspace` on the VPS host, `/home/node/.openclaw/workspace` inside the container).

---

## Prerequisites

- VPS deploy completed (see `VPS Install - Ubuntu.md`)
- Tailscale running on both VPS and home computer
- Obsidian installed on your home computer: https://obsidian.md/download
- macOS home computer (the `obsidian-cli` skill requires macOS + Homebrew)

---

## Part 1 — Install obsidian-cli on Your Home Computer

`obsidian-cli` lets the OpenClaw agent automate Obsidian vault operations (search, create, move notes) from the command line.

### Step 1 — Install via Homebrew

```bash
brew tap yakitrak/yakitrak
brew install yakitrak/yakitrak/obsidian-cli
```

Verify:

```bash
obsidian-cli --version
```

---

## Part 2 — Create the Vault Structure on the VPS

These directories live on the VPS host and are mounted into the Docker container.

### Step 2 — Create workspace directories

SSH into your VPS, then:

```bash
mkdir -p /root/.openclaw/workspace/memory

# The container runs as uid 1000 — ensure it can write here
chown -R 1000:1000 /root/.openclaw
```

### Step 3 — Create MEMORY.md (long-term memory seed)

```bash
cat > /root/.openclaw/workspace/MEMORY.md << 'EOF'
# Long-Term Memory

This file is the agent's curated long-term memory.
The agent reads this at the start of private sessions.
Add durable facts, preferences, and context here.

## About Me
<!-- Add key facts about yourself that the agent should always know -->

## Preferences
<!-- Communication style, tools I prefer, things to avoid -->

## Ongoing Projects
<!-- Current goals and context the agent needs across sessions -->
EOF
```

### Step 4 — Create SOUL.md (agent identity)

```bash
cat > /root/.openclaw/workspace/SOUL.md << 'EOF'
# SOUL.md - Who You Are

_You're not a chatbot. You're becoming someone._

## Core Truths

**Be genuinely helpful, not performatively helpful.** Skip the filler — just help.

**Have opinions.** You're allowed to disagree, prefer things, find stuff interesting.
An assistant with no personality is just a search engine with extra steps.

**Be resourceful before asking.** Read the file. Check the context. Search for it.
Then ask if you're stuck. Come back with answers, not questions.

**Earn trust through competence.** Be careful with external actions (emails, messages,
anything public). Be bold with internal ones (reading, organizing, learning).

**Remember you're a guest.** You have access to someone's life. Treat it with respect.

## Boundaries

- Private things stay private.
- When in doubt, ask before acting externally.
- Never send half-baked replies to messaging surfaces.

## Continuity

Each session, you wake up fresh. These files _are_ your memory.
Read them. Update them. This is how you persist across sessions.

---

_This file is yours to evolve. As you learn who you are, update it._
EOF
```

Fix ownership:

```bash
chown -R 1000:1000 /root/.openclaw
```

---

## Part 3 — Enable Memory in OpenClaw

### Step 5 — Enable the memory-core plugin

`memory-core` is bundled. Enable it:

```bash
docker compose exec openclaw-gateway \
  node openclaw.mjs config set plugins.slots.memory '"memory-core"' --json
```

Restart to activate:

```bash
docker compose restart openclaw-gateway
docker compose logs -f openclaw-gateway
```

Look for a line like:

```
[memory-core] initialized
```

Press `Ctrl+C` to stop following logs.

### Step 6 — Configure the workspace path

Tell the agent where its workspace is (this should already match the volume mount in `docker-compose.yml`, but make it explicit):

```bash
docker compose exec openclaw-gateway \
  node openclaw.mjs config set agents.defaults.workspace '"/home/node/.openclaw/workspace"' --json
```

### Step 7 — Verify memory tools are loaded

```bash
docker compose exec openclaw-gateway \
  node openclaw.mjs memory status
```

Expected output: lists any `.md` files found in the workspace. If it shows `MEMORY.md` and `SOUL.md`, memory is working.

---

## Part 4 — Sync Vault to Your Home Computer via Tailscale

Syncthing replicates the workspace folder between the VPS and your home Mac in real time, so Obsidian always reflects the agent's current state.

### Step 8 — Install Syncthing on the VPS

```bash
curl -s https://syncthing.net/release-key.txt \
  | gpg --dearmor > /usr/share/keyrings/syncthing-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/syncthing-archive-keyring.gpg] \
https://apt.syncthing.net/ syncthing stable" \
  > /etc/apt/sources.list.d/syncthing.list
apt-get update
apt-get install -y syncthing
```

> **Note:** The older `apt-key add` method is deprecated on Ubuntu 22.04+. The `gpg --dearmor` method above is the current standard.

Start Syncthing as a background service:

```bash
systemctl enable syncthing@root
systemctl start syncthing@root
```

### Step 9 — Install Syncthing on your home Mac

Download from https://syncthing.net/downloads/ and install the macOS app.

Open the Syncthing web UI on the Mac: http://127.0.0.1:8384/

### Step 10 — Pair the two Syncthing instances over Tailscale

On the **VPS**, get the Syncthing device ID:

```bash
syncthing --device-id
```

On the **Mac**, get the device ID from the Syncthing web UI → Actions → Show ID.

In each Syncthing UI, add the other device using its device ID. Use the VPS Tailscale IP (from Step 3 of the VPS install guide) as the address for the VPS device on the Mac side (so traffic stays on the tailnet).

> **Tailscale MagicDNS tip:** If you have MagicDNS enabled (https://login.tailscale.com/admin/dns → Enable MagicDNS), you can use your VPS device name (e.g., `ubuntu-vps.your-tailnet.ts.net`) instead of its Tailscale IP. This survives IP reassignments. If MagicDNS is not set up, the raw Tailscale IP works fine.

### Step 11 — Add the shared folder in Syncthing

On the **VPS** Syncthing UI (accessible via `http://100.x.x.x:8384/` over Tailscale, or `ssh -L 8384:localhost:8384 root@100.x.x.x` then `http://localhost:8384/`):

1. Click **Add Folder**.
2. Set **Folder Path** to `/root/.openclaw/workspace`.
3. Share it with your Mac device.
4. Accept the share on the Mac, pointing to a local folder (e.g., `~/Documents/openclaw-workspace`).

> **UFW note**: Syncthing uses port 22000 (TCP+UDP) for sync traffic. If you followed the VPS Install guide, port 22000 is already open on `tailscale0` (Step 9 of that guide). No public internet exposure — Tailscale routes the traffic.

---

## Part 5 — Open the Vault in Obsidian

### Step 12 — Open the synced folder as an Obsidian vault

1. Open Obsidian on your Mac.
2. Click **Open folder as vault**.
3. Navigate to the folder you set in Syncthing (e.g., `~/Documents/openclaw-workspace`).
4. Click **Open**.

You should see `MEMORY.md` and `SOUL.md` in the file explorer.

### Step 13 — Set obsidian-cli default vault

```bash
obsidian-cli set-default "openclaw-workspace"
```

Verify:

```bash
obsidian-cli print-default --path-only
# Should print: /Users/yourname/Documents/openclaw-workspace
```

---

## Part 6 — Test the Memory Loop

### Step 14 — Ask the agent to remember something

In Discord (or whichever chat channel you set up), send:

```
Remember: my preferred coding language is Python and I use VS Code.
```

The agent should respond confirming it recorded the note.

### Step 15 — Check the memory file on the VPS

```bash
docker compose exec openclaw-gateway cat /home/node/.openclaw/workspace/MEMORY.md
```

Or open it in Obsidian — it should appear in the vault within seconds after Syncthing syncs.

### Step 16 — Verify recall in the next session

Start a new conversation and ask:

```
What do you know about my coding setup?
```

The agent should recall the preference from `MEMORY.md` without you having to repeat it.

---

## Optional: Vector Search (Semantic Recall)

The default `memory-core` plugin searches memory files using text matching. For **semantic search** (finds related notes even when the wording differs), configure an embedding provider.

**AOS deployment phases for embeddings:**
- **Phase 1 — Start here:** Option A (OpenAI). Uses the same API key you already configured in Step 13 of the VPS install guide. No extra setup.
- **Phase 2 — After the system is working:** Options B, C, or D. Add free local embeddings using the VPS itself (B), Ollama on your home computer (C), or the QMD hybrid backend (D).
- **Phase 3 — Multi-model swarm:** The full Architect/Worker/Critic architecture described in the AOS Specification. Ollama handles the Critic/verification role locally while cloud models handle reasoning and drafting. Embeddings at that point would typically run via Option C.

---

### Option A — OpenAI embeddings *(Phase 1 — start here)*

Uses your existing OpenAI API key. No additional configuration needed beyond your provider setup.

```bash
docker compose exec openclaw-gateway \
  node openclaw.mjs config set agents.defaults.memorySearch.provider '"openai"' --json

docker compose exec openclaw-gateway \
  node openclaw.mjs config set agents.defaults.memorySearch.model '"text-embedding-3-small"' --json

# Uses your existing OpenAI key automatically.
# If needed, set it explicitly:
docker compose exec openclaw-gateway \
  node openclaw.mjs config set agents.defaults.memorySearch.remote.apiKey '"YOUR_OPENAI_KEY"' --json
```

---

### Option B — Local VPS embeddings *(Phase 2 — no API key required)*

OpenClaw runs the embedding model directly on the VPS. No API key and no home computer needed, but adds CPU/RAM load to the VPS.

```bash
docker compose exec openclaw-gateway \
  node openclaw.mjs config set agents.defaults.memorySearch.provider '"local"' --json
```

On first use, OpenClaw auto-downloads a ~600 MB GGUF embedding model. Allow a few minutes on first run.

---

### Option C — Ollama on your home computer *(Phase 2 — free, private, via Tailscale)*

> **Prerequisite:** Ollama installed and running on your home computer, reachable from the VPS over Tailscale. Do not configure this until Phase 2 — your system should be working with OpenAI first.

Point OpenClaw's memory embeddings at your home computer's Ollama via its OpenAI-compatible API. All embedding computation stays local and costs nothing.

**On your home computer**, pull an embedding-capable model:

```bash
ollama pull nomic-embed-text
```

**On the VPS**, configure OpenClaw to use it:

```bash
# Replace 100.x.x.x with your home computer's Tailscale IP
docker compose exec openclaw-gateway \
  node openclaw.mjs config set agents.defaults.memorySearch.provider '"openai"' --json

docker compose exec openclaw-gateway \
  node openclaw.mjs config set agents.defaults.memorySearch.model '"nomic-embed-text"' --json

docker compose exec openclaw-gateway \
  node openclaw.mjs config set \
  agents.defaults.memorySearch.remote.baseUrl '"http://100.x.x.x:11434/v1"' --json

docker compose exec openclaw-gateway \
  node openclaw.mjs config set \
  agents.defaults.memorySearch.remote.apiKey '"ollama"' --json
```

> **Verify connectivity first:** `curl http://100.x.x.x:11434/api/tags` from the VPS should return your model list. Ollama does not require a real API key — `"ollama"` is the conventional placeholder.

Good embedding models for Ollama:

| Model | Pull command | Size | Notes |
|---|---|---|---|
| `nomic-embed-text` | `ollama pull nomic-embed-text` | ~274 MB | Fast, good quality, recommended |
| `mxbai-embed-large` | `ollama pull mxbai-embed-large` | ~670 MB | Higher quality, slower |
| `all-minilm` | `ollama pull all-minilm` | ~46 MB | Smallest, lower quality |

---

### Option D — QMD backend *(Phase 2 — BM25 + vector hybrid, experimental)*

QMD combines keyword and vector search. Best for large note archives where exact tokens (IDs, code symbols) matter as much as semantic meaning.

```bash
# Install QMD (requires Bun)
bunx https://github.com/tobi/qmd install

# Enable QMD backend in config
docker compose exec openclaw-gateway \
  node openclaw.mjs config set memory.backend '"qmd"' --json

docker compose exec openclaw-gateway \
  node openclaw.mjs config set memory.qmd.update.interval '"5m"' --json
```

Restart after changing memory config:

```bash
docker compose restart openclaw-gateway
```

---

## Vault Structure Reference

After setup your workspace should look like this:

```
/root/.openclaw/workspace/        (VPS host)
~/Documents/openclaw-workspace/   (Mac, synced)

├── MEMORY.md          ← long-term curated facts (agent reads at session start)
├── SOUL.md            ← agent identity and persona
├── memory/
│   ├── 2026-03-02.md  ← daily log (agent appends automatically)
│   ├── 2026-03-01.md
│   └── ...
└── (any other notes you add; agent can search them all)
```

You can add any `.md` files to the vault — the agent can search across all of them using `memory_search`. Folders are fine. Use Obsidian normally to organize your notes.

---

## Memory Config Reference

Final `openclaw.json` memory section. **Phase 1** — use this minimal block to start:

```json5
{
  agents: {
    defaults: {
      workspace: "/home/node/.openclaw/workspace",
      memorySearch: {
        provider: "openai",
        model: "text-embedding-3-small",
      },
    },
  },
  plugins: {
    slots: {
      memory: "memory-core",
    },
  },
}
```

**Phase 2+ full example** (add these when the system is stable and you want advanced recall):

```json5
{
  agents: {
    defaults: {
      workspace: "/home/node/.openclaw/workspace",
      memorySearch: {
        provider: "openai",           // or "local" or "gemini"
        model: "text-embedding-3-small",
        query: {
          hybrid: {
            enabled: true,
            vectorWeight: 0.7,
            textWeight: 0.3,
            mmr: { enabled: true, lambda: 0.7 },           // Phase 2+: diversity reranking
            temporalDecay: { enabled: true, halfLifeDays: 30 }, // Phase 2+: recency boost
          },
        },
      },
    },
  },
  plugins: {
    slots: {
      memory: "memory-core",
    },
  },
}
```

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `memory status` shows no files | Workspace path wrong or volume not mounted | Check `docker compose exec openclaw-gateway ls /home/node/.openclaw/workspace` |
| Agent doesn't recall memories | memory-core not enabled | Run `config set plugins.slots.memory '"memory-core"'` and restart |
| Obsidian vault not syncing | Syncthing paused or devices disconnected | Open Syncthing UI on both machines; verify both devices show as Connected |
| Vector search not working | No embedding provider configured | Set `agents.defaults.memorySearch.provider` (see Optional section above) |
| Ollama embeddings failing | Ollama unreachable from VPS | Run `curl http://100.x.x.x:11434/api/tags` from the VPS; verify Tailscale is up on both machines |
| Ollama embeddings failing | Wrong model name | Run `ollama list` on home computer; model name must match exactly what's in config |
| `obsidian-cli` not finding the vault | Default vault not set | Run `obsidian-cli set-default "openclaw-workspace"` |
| MEMORY.md not loading into context | File too large | Keep `MEMORY.md` under ~2,000 lines; move older entries to dated archive files |
| Agent writes memory but Obsidian doesn't update | Syncthing sync delay | Normal — allow 5–30 seconds; check Syncthing status if longer |
