---
name: proactive-slacks
description: Let an AI assistant reach its human on Slack on purpose — one routing table maps topics to channels and loudness, one helper sends, one heartbeat pings only when something actually changed. Load this when running the heartbeat or sending any proactive notification.
---

# proactive-slacks

Most assistants are silent until you message them. This gives yours a way to
reach out first, without turning into a noise machine. Two ideas:

- The **helper** handles *where* a message lands and *how loud* it is.
- The **heartbeat** handles *when* — it wakes on a schedule, checks what changed
  since last time, and pings only on a real delta.

## The helper

    proactive-slacks <topic> "<message>"

Reads the routing table (`config/notifications.txt`) on every call. The table
maps a topic to a channel and a level. Edit + save = live, no restart.

- `silent`  log only
- `fyi`     post in the channel, no push
- `ask`     post; the FIRST line of the message is what you need from the human
- `urgent`  post + lock-screen push

Channel decides WHERE. Level decides HOW LOUD. They are independent — an `ask`
still lands in its topic channel, it does not get buried in a DM.

Flags: `--level <l>` overrides for one call, `--push` forces a push,
`--dry-run` resolves the route without sending, `--list` prints the table.

### Setup (env vars)

- `PSLACK_TOKEN` or `PSLACK_TOKEN_CMD` — Slack bot token, or a command that
  prints it (keeps secrets out of the environment).
- `PSLACK_DM_USER` — Slack user ID, required if any row routes to `DM`.
- `PSLACK_PUSH_CMD` — optional command for lock-screen push; gets the message
  as its last argument. Without it, `urgent` just posts.
- `PSLACK_TABLE`, `PSLACK_LOG` — optional path overrides.

Bot scopes: `chat:write` (+ `chat:write.public` or invite the bot to each
channel), `im:write` for DM rows.

## The heartbeat

Schedule a recurring wake (hourly is a good default). Each wake, run this and
stay quiet unless something genuinely moved:

1. **Read state.** A small JSON file holds the last-known state of each thread
   you track, plus when you last pinged it.
2. **Read reality.** Whatever your assistant uses to track work — notes, a task
   list, a threads file. For each active item, ask: *did something change that
   the human needs to know or act on?*
3. **Compare.** An item in the same state as last wake is NOT news. Only a delta
   pings: a draft finished, a blocker cleared or appeared, a job failed, a
   number went out of range, or an item has waited too long for the human.
4. **Route.** For each delta, pick a topic and call `proactive-slacks`.
5. **Update state.** Write the new state + `last_pinged` back. This is what
   stops the same thing pinging every wake.
6. **Exit quiet.** No deltas = no message. Silence is the common, correct
   outcome. Most wakes do nothing.

## Throttle (don't nag)

- Never ping the same item in the same state twice. The state file enforces it.
- An `ask` the human hasn't acted on gets at most one nudge per 48h.
- When torn between pinging and waiting: wait. A missed `fyi` costs nothing; a
  nagging loop costs trust.

## Invariants

- A topic with no row falls back to its category, then `default`. Never silently
  drop.
- The message carries the substance. For `ask`/`urgent`, the first line is the
  ask, in plain words.
- Keep the assistant's own voice. Short. No filler.

## Files

    bin/proactive-slacks        the helper
    config/notifications.txt    the routing table (human-editable)
    examples/heartbeat-state.json   starter state file for the heartbeat
    logs/notify.log             audit log (one JSON line per call)
