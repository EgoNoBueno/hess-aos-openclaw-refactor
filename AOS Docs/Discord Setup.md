# AOS OpenClaw — Discord Setup Procedure

Connecting Discord to your OpenClaw gateway (running on the VPS).  
You will use the Discord app on your home computer to chat with your agent.  
The Discord bot connection itself runs from the VPS gateway.

---

## Overview

```
Your home computer (Discord app)
        ↕  Discord servers
Your VPS (OpenClaw gateway + Discord plugin)
```

The gateway on your VPS connects to Discord's API, receives your messages, and replies as a bot.

---

## Part 1 — Get a Discord Bot Token

### Step 1 — Create a Discord Application

1. Open https://discord.com/developers/applications in a browser on your home computer.
2. Log in with your Discord account.
3. Click **New Application** (top right).
4. Name it something like `OpenClaw` and click **Create**.

---

### Step 2 — Create a Bot

1. In the left sidebar, click **Bot**.
2. Click **Add Bot** → **Yes, do it!**.
3. Optionally give the bot a username (this is the name people see in Discord).

---

### Step 3 — Enable Required Intents

Still on the **Bot** page, scroll down to **Privileged Gateway Intents** and enable:

| Intent | Required? |
|---|---|
| Message Content Intent | **Required** — bot cannot read messages without this |
| Server Members Intent | Recommended — needed for user allowlists and name matching |
| Presence Intent | Optional |

Click **Save Changes** at the bottom.

---

### Step 4 — Copy Your Bot Token

1. Still on the **Bot** page, scroll back up.
2. Click **Reset Token** → **Yes, do it!**.

> Despite the name "Reset Token," this generates your first token — nothing is being deleted.

3. Copy the token that appears. **Save it somewhere safe** — you will only see it once.  
   Do not share it. Treat it like a password.

---

### Step 5 — Set OAuth2 Permissions and Add Bot to Your Server

1. In the left sidebar, click **OAuth2** → **URL Generator**.
2. Under **Scopes**, check:
   - `bot`
   - `applications.commands`
3. Under **Bot Permissions** (appears after checking `bot`), check:
   - View Channels
   - Send Messages
   - Read Message History
   - Embed Links
   - Attach Files
4. Copy the generated URL at the bottom.
5. Paste it into a new browser tab.
6. Select your Discord server from the dropdown and click **Continue** → **Authorize**.

Your bot now appears in your Discord server (offline until the gateway connects).

---

### Step 6 — Enable Developer Mode and Collect IDs

You need your **Server ID** and **User ID** for pairing and access control.

1. In the Discord app on your home computer, open **User Settings** (gear icon near your avatar).
2. Click **Advanced** → toggle on **Developer Mode**.
3. Close Settings.
4. Right-click your **server icon** in the left sidebar → **Copy Server ID**. Save it.
5. Right-click your **own avatar** anywhere in Discord → **Copy User ID**. Save it.

> You now have three things saved: **Bot Token**, **Server ID**, **User ID**.

---

### Step 7 — Allow Direct Messages from Server Members

For your bot to DM you (required for pairing):

1. Right-click your **server icon** → **Privacy Settings**.
2. Toggle on **Direct Messages**.

---

## Part 2 — Configure OpenClaw on the VPS

All commands below run on the VPS (SSH in first or use your terminal from Step 1 of the VPS install guide).

---

### Step 8 — Set Your Bot Token and Enable Discord

The Discord integration is built into this repo's Docker image — no separate install step is needed.  
Set the token via the OpenClaw CLI on the VPS — **do not paste it in chat**:

```bash
docker compose exec openclaw-gateway \
  node openclaw.mjs config set channels.discord.token '"YOUR_BOT_TOKEN"' --json

docker compose exec openclaw-gateway \
  node openclaw.mjs config set channels.discord.enabled true
```

Replace `YOUR_BOT_TOKEN` with the token you copied in Step 4.  
The inner double quotes (`'"..."'`) are required: the outer single quotes are shell quoting; the inner double quotes tell the config parser to treat the value as a JSON string.

Alternatively, set the token as an environment variable in your `.env` file in the repo root on the VPS:

```
DISCORD_BOT_TOKEN=YOUR_BOT_TOKEN
```

The env var is used as a fallback for the default account only. The `config set` method above is recommended.

> **Security note:** The `.env` file on the VPS is only readable by root. Never commit it to git.

---

### Step 9 — Restart the Gateway

After setting the token, restart so Discord connects:

```bash
docker compose restart openclaw-gateway
docker compose logs -f openclaw-gateway
```

Look for a line like:

```
[discord] connected as OpenClaw#1234
```

If you see it, the bot is online. Press `Ctrl+C` to stop following logs.

---

## Part 3 — Pair Your Discord Account

This links your Discord identity to OpenClaw so only you can control the agent.

---

### Step 10 — Send Your Agent a DM in Discord

1. Open Discord on your home computer.
2. Find the bot in your server's member list.
3. Click the bot to open a DM.
4. Send any message (e.g., `hello`).
5. The bot will respond with a **pairing code**.

---

### Step 11 — Approve the Pairing Code

On the VPS, approve the code:

```bash
docker compose exec openclaw-gateway node openclaw.mjs pairing list discord
docker compose exec openclaw-gateway node openclaw.mjs pairing approve discord <CODE>
```

Replace `<CODE>` with the code shown in the Discord DM.  
Pairing codes expire after 1 hour.

You can now chat with your agent via Discord DMs.

---

## Part 4 — Set Up Guild (Server) Channels (Recommended)

This lets your agent respond in specific channels on your Discord server, not just DMs.

---

### Step 12 — Add Your Server to the Allowlist

```bash
docker compose exec openclaw-gateway node openclaw.mjs config set \
  'channels.discord.groupPolicy' '"allowlist"' --json

docker compose exec openclaw-gateway node openclaw.mjs config set \
  'channels.discord.guilds.YOUR_SERVER_ID.requireMention' 'false' --json

docker compose exec openclaw-gateway node openclaw.mjs config set \
  'channels.discord.guilds.YOUR_SERVER_ID.users' '["YOUR_USER_ID"]' --json
```

Replace `YOUR_SERVER_ID` and `YOUR_USER_ID` with the IDs you copied in Step 6.

Setting `requireMention: false` means your agent responds to every message in allowed channels (ideal for private servers). Set it to `true` if you want the agent to only respond when @mentioned.

---

### Step 13 — Restart and Verify

```bash
docker compose restart openclaw-gateway
```

Go to any channel in your Discord server and send a message.  
Your agent should respond.

---

## Configuration Reference

Your final `openclaw.json` Discord section should look like:

```json5
{
  channels: {
    discord: {
      enabled: true,
      token: "YOUR_BOT_TOKEN",        // or use DISCORD_BOT_TOKEN env var
      groupPolicy: "allowlist",
      guilds: {
        "YOUR_SERVER_ID": {
          requireMention: false,      // false = respond to all messages in allowed channels
          users: ["YOUR_USER_ID"],    // only these users get responses
        },
      },
    },
  },
}
```

View and edit the config file directly on the VPS:

```bash
docker compose exec openclaw-gateway cat /home/node/.openclaw/openclaw.json
```

---

## IDs and Tokens Summary

| Item | Where to find it | Where it goes |
|---|---|---|
| Bot Token | Discord Developer Portal → Bot → Reset Token | `.env` or `openclaw config set` on VPS |
| Server ID | Discord app → right-click server icon → Copy Server ID | `openclaw.json` guilds key |
| User ID | Discord app → right-click your avatar → Copy User ID | `openclaw.json` users list |

---

## OpenAI Credit Grants — quick check

Exactly—that is exactly where you want to look. In OpenAI’s developer world, **"Credit Grants"** are essentially your "gas tank." Even though it sounds like a "gift," this is the section OpenAI uses for **prepaid balances** as well. Based on what you're seeing:

- **Balance: `$5.00 / $5.00`** means you have a full **$5.00** ready to use.
- **Expires: Mar 31, 2027** means you have plenty of time (basically a year) to use those credits before they disappear.
- **Usage:** Since it shows **$0.00** used so far, your bot hasn't actually spent anything yet!

### How far will $5.00 go?

If you chose **`gpt-4o-mini`** or **`gpt-5-nano`** during setup, $5.00 is actually a lot of money.

- **`gpt-4o-mini`**: You can send/receive roughly **8 to 10 million words**. That is about 20 average-sized novels.
- **`gpt-5-nano`**: Even more. You could practically run a small village on that $5.00.

---

### Is the bot talking yet?

Now that we know the "gas tank" is full, let's see if the engine is actually pulling from it:

1. **Check Discord:** Go to your server and @mention your bot: `@BotName are you there?`
2. **Check the usage:** Wait about 1 minute after talking to the bot, then refresh that **Credit Grants** page.
3. **The Result:** If it changes to something like **`$4.99 / $5.00`**, then everything is perfectly connected!

### If the bot isn't responding:

It usually means the **Discord Token** didn't save correctly or the **Intents** aren't turned on in the Discord Developer portal.

**Does the bot show up as "Online" (green light) in your Discord server?** If it's online but ignoring you, we just need to flip one switch in the Discord settings!

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Bot appears offline in Discord | Gateway not running or token wrong | Check `docker compose logs openclaw-gateway` for errors |
| Bot does not respond to DMs | Pairing not approved | Run `docker compose exec openclaw-gateway node openclaw.mjs pairing list discord` and approve the code |
| Bot does not respond in channels | Server not in allowlist | Complete Step 12; verify Server ID is correct |
| `[discord] Invalid token` in logs | Token was regenerated | Go back to Developer Portal → Bot → Reset Token; update `.env` |
| Cannot DM the bot | Server DMs disabled | Re-enable DMs in server Privacy Settings (Step 7) |
| Bot responds but cannot see message content | Message Content Intent missing | Enable it in Developer Portal → Bot → Privileged Gateway Intents |
