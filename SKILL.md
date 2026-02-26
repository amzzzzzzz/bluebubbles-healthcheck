---
name: bluebubbles-healthcheck
description: "Diagnoses and auto-heals BlueBubbles ↔ OpenClaw iMessage connectivity. Use when: iMessages stop arriving after a gateway restart, webhook connection is broken, or user reports messages not coming through. Runs a 4-step diagnostic and auto-fixes webhook backoff, stale registrations, and gateway issues."
homepage: https://github.com/amzzzzzzz/bluebubbles-healthcheck
metadata: { "openclaw": { "emoji": "🩺", "requires": { "bins": ["curl", "python3"] } } }
---

# BlueBubbles Healthcheck Skill

## When to Use This Skill

Use this skill when:
- iMessages aren't being delivered to/from OpenClaw
- After restarting the OpenClaw gateway
- User reports "messages not coming through"
- Periodic healthcheck (can be added to HEARTBEAT.md)
- Debugging BlueBubbles ↔ OpenClaw connectivity

## What It Does

Diagnoses and auto-heals the webhook connection between BlueBubbles and OpenClaw. This is a common failure mode: after gateway restarts, BlueBubbles can lose its webhook or enter backoff state.

**Diagnostic checks:**
1. BlueBubbles server reachable
2. Webhook registered pointing to OpenClaw
3. OpenClaw gateway endpoint responding
4. Recent webhook delivery activity

**Auto-healing:**
- Restarts OpenClaw gateway if endpoint is down
- Deletes stale webhooks and re-registers fresh
- Verifies fix after healing

## How to Use

### Quick Check (Read-Only)

```bash
BB_URL="http://127.0.0.1:1234" \
BB_PASSWORD="your-password" \
~/.openclaw/workspace/skills/bluebubbles-healthcheck/scripts/diagnose.sh
```

**Interpret the output:**
- All ✅ = healthy, no action needed
- Any ❌ = issue detected, consider running heal

### Auto-Heal

```bash
BB_URL="http://127.0.0.1:1234" \
BB_PASSWORD="your-password" \
~/.openclaw/workspace/skills/bluebubbles-healthcheck/scripts/heal.sh
```

This will:
1. Run diagnostics
2. Identify what's broken
3. Attempt to fix it (gateway restart, webhook reset)
4. Re-run diagnostics to verify

### Dry Run (See What Would Happen)

```bash
BB_URL="http://127.0.0.1:1234" \
BB_PASSWORD="your-password" \
~/.openclaw/workspace/skills/bluebubbles-healthcheck/scripts/heal.sh --dry-run
```

## Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `BB_URL` | Yes | `http://127.0.0.1:1234` | BlueBubbles server URL |
| `BB_PASSWORD` | Yes | — | BlueBubbles API password |
| `OPENCLAW_WEBHOOK_URL` | No | `http://127.0.0.1:18789/bluebubbles-webhook` | OpenClaw webhook endpoint |

You can also pass these as args: `--bb-url`, `--password`, `--webhook-url`

## Agent Decision Flow

```
User reports iMessage issue
         ↓
    Run diagnose.sh
         ↓
    ┌────┴────┐
    │ All ✅? │
    └────┬────┘
    Yes  │  No
    ↓    │  ↓
 Report  │  Run heal.sh
 healthy │      ↓
         │  ┌───┴───┐
         │  │Fixed? │
         │  └───┬───┘
         │  Yes │ No
         │  ↓   │ ↓
         │Report│ Escalate to user:
         │fixed │ - BB app not running?
         │      │ - Network issue?
         └──────┴─ Manual intervention needed
```

## Common Failure Patterns

### Pattern 1: Gateway restart broke webhooks
**Symptoms:** Messages stop after `openclaw gateway restart`
**Fix:** `heal.sh` will reset webhook

### Pattern 2: BlueBubbles in backoff
**Symptoms:** Webhook exists but BB stopped trying to deliver
**Fix:** `heal.sh` deletes and re-registers webhook (clears backoff state)

### Pattern 3: Gateway not running
**Symptoms:** Check 3 fails (port 18789 not listening)
**Fix:** `heal.sh` runs `openclaw gateway restart`

### Pattern 4: BlueBubbles.app not running
**Symptoms:** Check 1 fails (HTTP 000)
**Fix:** Manual — user must start BlueBubbles.app on the Mac

## Files

```
skills/bluebubbles-healthcheck/
├── SKILL.md           ← You are here
├── README.md          ← GitHub docs
└── scripts/
    ├── diagnose.sh    ← Read-only diagnostics (exit 0 = healthy)
    ├── heal.sh        ← Auto-heal orchestrator
    └── reset-webhook.sh ← Atomic webhook delete+re-register
```

## Adding to Heartbeat

To run periodic healthchecks, add to `HEARTBEAT.md`:

```markdown
## BlueBubbles Health
Every 4 hours, run the BlueBubbles healthcheck skill.
If any checks fail, run heal and report results.
```
