---
summary: "Frequently asked questions about Clawdis setup, configuration, and usage"
---
# FAQ 🦞

Common questions from the community. For detailed configuration, see [configuration.md](./configuration.md).

## Installation & Setup

### Where does Clawdis store its data?

Everything lives under `~/.clawdis/`:

| Path | Purpose |
|------|---------|
| `~/.clawdis/clawdis.json` | Main config (JSON5) |
| `~/.clawdis/credentials/` | WhatsApp/Telegram auth tokens |
| `~/.clawdis/sessions/` | Conversation history & state |
| `~/.clawdis/sessions/sessions.json` | Session metadata |

Your **workspace** (AGENTS.md, memory files, skills) is separate — configured via `agent.workspace` in your config (default: `~/clawd`).

### What platforms does Clawdis run on?

**macOS and Linux** are the primary targets. Anywhere Node.js 22+ runs should work in theory.

- **macOS** — Fully supported, most tested
- **Linux** — Works great, common for VPS/server deployments
- **Windows** — Should work but largely untested! You're in pioneer territory 🤠

Some features are platform-specific:
- **iMessage** — macOS only (uses `imsg` CLI)
- **Clawdis.app** — macOS native app (optional, gateway works without it)

### I'm getting "unauthorized" errors on health check

You need a config file. Run the onboarding wizard:

```bash
pnpm clawdis onboard
```

This creates `~/.clawdis/clawdis.json` with your API keys, workspace path, and owner phone number.

### How do I start fresh?

```bash
# Backup first (optional)
cp -r ~/.clawdis ~/.clawdis-backup

# Remove config and credentials
rm -rf ~/.clawdis

# Re-run onboarding
pnpm clawdis onboard
pnpm clawdis login
```

### Something's broken — how do I diagnose?

Run the doctor:

```bash
pnpm clawdis doctor
```

It checks your config, skills status, and gateway health. It can also restart the gateway daemon if needed.

### Terminal onboarding vs macOS app?

**Use terminal onboarding** (`pnpm clawdis onboard`) — it's more stable right now.

The macOS app onboarding is still being polished and can have quirks (e.g., WhatsApp 515 errors, OAuth issues).

### How do I log Claude usage over time?

Run the collector once:

```bash
pnpm clawdis collect-usage
```

It reads `~/.clawdis/rate-limits.json` and appends a line to `~/.clawdis/usage-history.jsonl`.

Cron every 5 minutes:

```cron
*/5 * * * * cd /home/admin/clawdis && pnpm clawdis collect-usage
```

---

## Authentication

### OAuth vs API key — what's the difference?

- **OAuth** — Uses your Claude Pro/Max subscription ($20-100/mo flat). No per-token charges. ✅ Recommended!
- **API key** — Pay-per-token via console.anthropic.com. Can get expensive fast.

They're **separate billing**! An API key does NOT use your subscription.

**For OAuth:** During onboarding, pick "Anthropic OAuth", log in to your Claude account, paste the code back.

**If OAuth fails** (headless/container): Do OAuth on a normal machine, then copy `~/.clawdis/` to your server. The auth is just a JSON file.

### OAuth callback not working (containers/headless)?

OAuth needs the callback to reach the machine running the CLI. Options:

1. **Copy auth manually** — Run OAuth on your laptop, copy `~/.clawdis/credentials/` to the container.
2. **SSH tunnel** — `ssh -L 18789:localhost:18789 user@server`
3. **Tailscale** — Put both machines on your tailnet.

---

## Migration & Deployment

### How do I migrate Clawdis to a new machine (or VPS)?

1. **Backup on old machine:**
   ```bash
   # Config + credentials + sessions
   tar -czvf clawdis-backup.tar.gz ~/.clawdis
   
   # Your workspace (memories, AGENTS.md, etc.)
   tar -czvf workspace-backup.tar.gz ~/path/to/workspace
   ```

2. **Copy to new machine:**
   ```bash
   scp clawdis-backup.tar.gz workspace-backup.tar.gz user@new-machine:~/
   ```

3. **Restore on new machine:**
   ```bash
   cd ~
   tar -xzvf clawdis-backup.tar.gz
   tar -xzvf workspace-backup.tar.gz
   ```

4. **Install Clawdis** (Node 22+, pnpm, clone repo, `pnpm install && pnpm build`)

5. **Start gateway:**
   ```bash
   pnpm clawdis gateway
   ```

**Note:** WhatsApp may notice the IP change and require re-authentication. If so, run `pnpm clawdis login` again. Stop the old instance before starting the new one to avoid conflicts.

### Can I run Clawdis in Docker?

There's no official Docker setup yet, but it works. Key considerations:

- **WhatsApp login:** QR code works in terminal — no display needed.
- **Persistence:** Mount `~/.clawdis/` and your workspace as volumes.
- **pnpm doesn't persist:** Global npm installs don't survive container restarts. Install pnpm in your startup script.
- **Browser automation:** Optional. If needed, install headless Chrome + Playwright deps, or connect to a remote browser via `--remote-debugging-port`.

**Volume mappings (e.g., Unraid):**
```
/mnt/user/appdata/clawdis/config    → /root/.clawdis
/mnt/user/appdata/clawdis/workspace → /root/clawd
/mnt/user/appdata/clawdis/app       → /app
```

**Startup script (`start.sh`):**
```bash
#!/bin/bash
npm install -g pnpm
cd /app
pnpm clawdis gateway
```

**Container command:**
```
bash /app/start.sh
```

Docker support is on the roadmap — PRs welcome!

### Can I run Clawdis headless on a VPS?

Yes! The terminal QR code login works fine over SSH. For long-running operation:

- Use `pm2`, `systemd`, or a `launchd` plist to keep the gateway running.
- Consider Tailscale for secure remote access.

### bun binary vs Node runtime?

Clawdis can run as:
- **bun binary** — Single executable, easy distribution, auto-restarts via launchd
- **Node runtime** (`pnpm clawdis gateway`) — More stable for WhatsApp

If you see WebSocket errors like `ws.WebSocket 'upgrade' event is not implemented`, use Node instead of the bun binary. Bun's WebSocket implementation has edge cases that can break WhatsApp (Baileys).

**For stability:** Use launchd (macOS) or the Clawdis.app — they handle process supervision (auto-restart on crash).

**For debugging:** Use `pnpm gateway:watch` for live reload during development.

### WhatsApp keeps disconnecting / crashing (macOS app)

This is often the bun WebSocket issue. Workaround:

1. Run gateway with Node instead:
   ```bash
   pnpm gateway:watch
   ```
2. In **Clawdis.app → Settings → Debug**, check **"External gateway"**
3. The app now connects to your Node gateway instead of spawning bun

This is the most stable setup until bun's WebSocket handling improves.

---

## Multi-Instance & Contexts

### Can I run multiple Clawds (separate instances)?

The intended design is **one Clawd, one identity**. Rather than running separate instances:

- **Add skills** — Give your Clawd multiple capabilities (business + fitness + personal).
- **Use context switching** — "Hey Clawd, let's talk about fitness" within the same conversation.
- **Use groups for separation** — Create Telegram/Discord groups for different contexts; each group gets its own session.

Why? A unified assistant knows your whole context. Your fitness coach knows when you've had a stressful work week.

If you truly need full separation (different users, privacy boundaries), you'd need:
- Separate config directories
- Separate gateway ports
- Separate phone numbers for WhatsApp (one number = one account)

### Can I have separate "threads" for different topics?

Currently, sessions are per-chat:
- Each WhatsApp/Telegram DM = one session
- Each group = separate session

**Workaround:** Create multiple groups (even just you + the bot) for different contexts. Each group maintains its own session.

Feature request? Open a [GitHub discussion](https://github.com/steipete/clawdis/discussions)!

### How do groups work?

Groups get separate sessions automatically. By default, the bot requires a **mention** to respond in groups.

Per-group activation can be changed by the owner:
- `/activation mention` — respond only when mentioned (default)
- `/activation always` — respond to all messages

See [groups.md](./groups.md) for details.

---

## Context & Memory

### How much context can Clawdis handle?

Claude Opus has a 200k token context window, and Clawdis uses **autocompaction** — older conversation gets summarized to stay under the limit.

Practical tips:
- Keep `AGENTS.md` focused, not bloated.
- Use `/new` to reset the session when context gets stale.
- For large memory/notes collections, use search tools like `qmd` rather than loading everything.

### Where are my memory files?

In your workspace directory (configured in `agent.workspace`, default `~/clawd`). Look for:
- `memory/` — daily memory files
- `AGENTS.md` — agent instructions
- `TOOLS.md` — tool-specific notes

Check your config:
```bash
cat ~/.clawdis/clawdis.json | grep workspace
```

---

## Platforms

### Which platforms does Clawdis support?

- **WhatsApp** — Primary. Uses WhatsApp Web protocol.
- **Telegram** — Via Bot API (grammY).
- **Discord** — Bot integration.
- **iMessage** — Via `imsg` CLI (macOS only).
- **Signal** — Via `signal-cli` (see [signal.md](./signal.md)).
- **WebChat** — Browser-based chat UI.

### Can I use multiple platforms at once?

Yes! One Clawdis gateway can connect to WhatsApp, Telegram, Discord, and more simultaneously. Each platform maintains its own sessions.

### WhatsApp: Can I use two numbers?

One WhatsApp account = one phone number = one gateway connection. For a second number, you'd need a second gateway instance with a separate config directory.

---

## Skills & Tools

### How do I add new skills?

Skills are auto-discovered from your workspace's `skills/` folder. After adding new skills:

1. Send `/reset` (or `/new`) in chat to start a new session
2. The new skills will be available

No gateway restart needed!

### How do I run commands on other machines?

Use **[Tailscale](https://tailscale.com/)** to create a secure network between your machines:

1. Install Tailscale on all machines (it's separate from Clawdis — set it up yourself)
2. Each gets a stable IP (like `100.x.x.x`)
3. SSH just works: `ssh user@100.x.x.x "command"`

Clawdis can use Tailscale when you set `bridge.bind: "tailnet"` in your config — it auto-detects your Tailscale IP.

For deeper integration, look into **Clawdis nodes** — pair remote machines with your gateway for camera/screen/automation access.

---

## Troubleshooting

### Build errors (TypeScript)

If you hit build errors on `main`:

1. Pull latest: `git pull origin main && pnpm install`
2. Try `pnpm clawdis doctor`
3. Check [GitHub issues](https://github.com/steipete/clawdis/issues) or Discord
4. Temporary workaround: checkout an older commit

### WhatsApp logged me out

WhatsApp sometimes disconnects on IP changes or after updates. Re-authenticate:

```bash
pnpm clawdis login
```

Scan the QR code and you're back.

### Gateway won't start

Check logs:
```bash
cat /tmp/clawdis/clawdis-$(date +%Y-%m-%d).log
```

Common issues:
- Port already in use (change with `--port`)
- Missing API keys in config
- Invalid config syntax (remember it's JSON5, but still check for errors)

**Debug mode** — use watch for live reload:
```bash
pnpm gateway:watch
```

**Pro tip:** Use Codex to debug:
```bash
cd ~/path/to/clawdis
codex --full-auto "debug why clawdis gateway won't start"
```

### Processes keep restarting after I kill them (Linux)

Something is supervising them. Check:

```bash
# systemd?
systemctl list-units | grep -i clawdis
sudo systemctl stop clawdis

# pm2?
pm2 list
pm2 delete all
```

Stop the supervisor first, then the processes.

### Clean uninstall (start fresh)

```bash
# Stop processes
pkill -f "clawdis"

# If using systemd
sudo systemctl stop clawdis
sudo systemctl disable clawdis

# Remove data
rm -rf ~/.clawdis

# Remove repo and re-clone
rm -rf ~/clawdis
git clone https://github.com/steipete/clawdis.git
cd clawdis && pnpm install && pnpm build
pnpm clawdis onboard
```

---

## Chat Commands

Quick reference (send these in chat):

| Command | Action |
|---------|--------|
| `/status` | Health + session info |
| `/new` or `/reset` | Reset the session |
| `/think <level>` | Set thinking level (off\|minimal\|low\|medium\|high) |
| `/verbose on\|off` | Toggle verbose mode |
| `/activation mention\|always` | Group activation (owner-only) |

---

*Still stuck? Ask in [Discord](https://discord.gg/qkhbAGHRBT) or open a [GitHub discussion](https://github.com/steipete/clawdis/discussions).* 🦞
