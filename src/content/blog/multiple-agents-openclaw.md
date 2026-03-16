---
title: "Running Multiple AI Agents on One Machine with OpenClaw"
description: "How I run a main assistant, a trading agent, and more on the same Ubuntu machine — different personalities, workspaces, and Telegram bots. Plus the hard lessons from migrating an agent from cloud to local."
pubDate: 2026-03-16
---

I've been running two independent AI agents on the same machine for a while now — one for general assistant work, one dedicated to trading. Different bots, different personalities, different Telegram groups, same hardware. No cloud dependency.

Here's how it works, and what I learned the hard way when I moved an agent from cloud to local.

---

## Why Multiple Agents?

One agent trying to be your trading desk, your personal assistant, your wellness coach, and your home automation hub is like hiring one person to be your accountant, your chef, your trainer, and your IT support — all at once. They might manage, but they'll be stretched thin and confused.

The better approach: specialized agents, each with their own focus.

Here's what a real multi-agent setup looks like:

| Agent | Role | Telegram |
|-------|------|----------|
| **aria** (main) | Personal assistant, system monitoring, general tasks | Personal DM + general group |
| **atlas** (trading) | Market research, stock watchlists, trading alerts | 3 trading groups |
| *(future)* **zen** | Habit tracking, health check-ins, reminders | Wellness group |
| *(future)* **haven** | Smart home scripts, device control | Home automation group |

Each agent gets its own:
- **Workspace** — files, memory, scripts, personality
- **Telegram bot** — separate bot token, separate groups
- **Context window** — no cross-contamination between domains
- **Cron jobs & heartbeats** — independent schedules

Atlas doesn't know about your calendar. Your main agent doesn't need to track options flow. Clean separation makes both better.

---

## How OpenClaw Routes Multiple Agents

OpenClaw uses a simple name-matching system. Each agent gets an entry in `agents.list`:

```json
"agents": {
  "list": [
    {
      "id": "main",
      "default": true,
      "name": "Aria",
      "workspace": "/home/user/.openclaw/workspace-aria"
    },
    {
      "id": "atlas",
      "name": "Atlas",
      "workspace": "/home/user/.openclaw/workspace-atlas"
    }
  ]
}
```

On the Telegram side, each agent gets its own account block under `channels.telegram.accounts`, **named to match the agent ID exactly**:

```json
"channels": {
  "telegram": {
    "accounts": {
      "default": {        
        "botToken": "YOUR_ARIA_BOT_TOKEN",
        ...
      },
      "atlas": {       
        "botToken": "YOUR_ATLAS_BOT_TOKEN",
        ...
      }
    }
  }
}
```

When a message arrives on a bot token, OpenClaw looks up which account it belongs to, matches the account name to an agent ID, and routes accordingly. Simple and effective.

> **Note:** The main agent uses the account named `default` — this is the fallback account. All other agents must have an account name that exactly matches their agent ID.

---

## The Config That Makes It Work

Here's the full working account config for each agent. Both fields that trip people up are highlighted:

```json
"accounts": {
  "default": {
    "dmPolicy": "allowlist",
    "botToken": "YOUR_ARIA_BOT_TOKEN",
    "allowFrom": ["your-telegram-id"],
    "groupAllowFrom": ["your-telegram-id"],
    "groupPolicy": "allowlist",
    "groups": {
      "-100xxxxxxxxx": {
        "requireMention": false,
        "enabled": true
      }
    },
    "streaming": "partial"
  },
  "atlas": {
    "dmPolicy": "allowlist",
    "botToken": "YOUR_ATLAS_BOT_TOKEN",
    "allowFrom": ["your-telegram-id", "trusted-user-id"],
    "groupAllowFrom": ["your-telegram-id", "trusted-user-id"],
    "groupPolicy": "allowlist",
    "groups": {
      "-100xxxxxxxxx": { "requireMention": false, "enabled": true },
      "-100yyyyyyyyy": { "requireMention": false, "enabled": true },
      "-100zzzzzzzzz": { "requireMention": false, "enabled": true }
    },
    "streaming": "partial"
  }
}
```

**Two fields people miss:**

- `allowFrom` — controls who can trigger the bot in **DMs**
- `groupAllowFrom` — controls who can trigger the bot in **groups** — separate field, must be set explicitly at the account level

Missing `groupAllowFrom` means group messages are silently dropped. No error, no log — just silence.

---

## Two Agents, Two Workspaces

Each agent has its own workspace directory with a complete identity:

```
/home/user/.openclaw/workspace-aria/      ← main agent
├── SOUL.md          personality & values
├── IDENTITY.md      name, emoji, vibe
├── USER.md          who it's helping
├── MEMORY.md        long-term memory
├── HEARTBEAT.md     periodic check tasks
├── TOOLS.md         local notes, script paths
├── AGENTS.md        session instructions
└── memory/          daily logs

/home/user/.openclaw/workspace-atlas/     ← trading agent
├── SOUL.md          Atlas's personality
├── IDENTITY.md      Atlas's identity
├── USER.md
├── MEMORY.md        Atlas's memories (trading context)
├── TOOLS.md         trading scripts, data sources
├── stocks-in-watch.md
├── trading/         domain-specific files
└── skills/          custom skills
```

The agents don't share context. Atlas's memory is full of tickers, entry points, and market observations. Aria's memory is full of project notes and personal context. Neither bleeds into the other.

---

## Migrating an Agent from Cloud to Local

This is where things get interesting — and where most people hit problems. When Atlas moved from cloud to local, three things broke silently.

### 1. Auth doesn't migrate automatically

Every agent needs an `auth-profiles.json` in its agent directory:

```
~/.openclaw/agents/atlas/agent/auth-profiles.json
```

When you set up a new agent locally, this file isn't created automatically. Without it, the agent can't authenticate to call the AI API — it fails silently when processing messages.

**Fix:**
```bash
cp ~/.openclaw/agents/main/agent/auth-profiles.json \
   ~/.openclaw/agents/atlas/agent/auth-profiles.json
```

### 2. `groupAllowFrom` must be set at account level

The top-level `channels.telegram.groupAllowFrom` exists, but when you have per-account configs, the account level takes precedence. If `groupAllowFrom` isn't set in the account block, group messages are silently dropped even though DMs work fine.

**Fix:** Add `groupAllowFrom` explicitly to each account (see config above).

### 3. Telegram group subscriptions reset on bot changes

Any time a bot's settings change in BotFather — or a token is used on a new server — Telegram quietly stops delivering group messages to it. The bot still shows as a member. No errors. Just silence.

**Fix:** Remove the bot from the group, wait a few seconds, re-add it. Do this for every group.

### 4. Bring the whole workspace

A cloud agent has built up a life — don't leave it behind. When migrating, bring everything:

- **SOUL.md, IDENTITY.md, USER.md** — who the agent is. Without these it starts fresh with no personality.
- **MEMORY.md + memory/** — all historical context and learned knowledge
- **TOOLS.md** — but update any paths that differ on the new machine
- **skills/** — any custom skills. Check that referenced script paths still exist locally
- **Scripts** — if the agent relied on Python scripts, scrapers, or automation tools, they need to exist at the correct local paths (or TOOLS.md needs updating to point to the new locations)
- **Cron jobs** — scheduled jobs don't migrate. Recreate them manually via `openclaw cron add` or the gateway cron tool

---

## Migration Checklist

Use this every time you move an agent:

- [ ] Copy `auth-profiles.json` to the new agent directory
- [ ] Set `groupAllowFrom` explicitly in the Telegram account config
- [ ] Verify group IDs are correct (forward a group message to @userinfobot to get the ID)
- [ ] Copy the full workspace: SOUL.md, IDENTITY.md, USER.md, MEMORY.md, memory/, TOOLS.md
- [ ] Copy or recreate skills and scripts — update any hardcoded paths in TOOLS.md and skill files
- [ ] Recreate cron jobs
- [ ] Restart the gateway: `openclaw gateway restart`
- [ ] Remove and re-add the bot to every Telegram group
- [ ] Test DMs and groups separately — they can fail independently

---

## Tips & Lessons Learned

**Always restart after config changes.** OpenClaw hot-reloads via SIGUSR1 but some changes (especially channel config) need a clean restart to fully take effect.

**Test DMs and groups separately.** They use different auth paths and can fail independently. A working DM doesn't mean groups are working.

**BotFather privacy mode.** Make sure it's disabled for each bot (`/setprivacy → Disable`). Confirm it worked by calling the Telegram API — look for `can_read_all_group_messages: true` in the response.

**`groupPolicy: allowlist` is strict.** The group ID must be explicitly listed in the `groups` block. A new group won't work until you add it to the config and restart.

**Name your accounts to match agent IDs exactly.** The routing is purely by name match — `atlas` account → `atlas` agent. A typo means messages go nowhere.

**One bot token per agent.** Never share a token between two agents. OpenClaw uses the token to identify which account (and therefore which agent) a message belongs to.

---

## Final Thoughts

Running multiple agents on one machine is genuinely powerful. Atlas handles the trading domain with full context and dedicated memory. Aria stays focused on general work. Neither knows nor cares about the other.

The setup cost is low — an extra workspace directory, a second bot token from BotFather, and a few config lines. The migration gotchas are real but easy to avoid once you know them.

If you're running OpenClaw and haven't tried a second agent yet, start simple — pick one domain you want to separate out, spin up a new agent, give it a personality, and point a fresh bot at it. You'll wonder why you didn't do it sooner.
