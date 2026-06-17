<p align="center">
  <img src="assets/icon-256.png" width="128" height="128" alt="proactive-slacks icon">
</p>

<h1 align="center">proactive-slacks</h1>

<p align="center">Let an AI assistant reach you on Slack <b>on purpose</b> — without becoming a noise machine.</p>

Most assistants are silent until you message them. This is a small plugin that
gives yours a way to reach out first: when a draft is ready, when a build is
blocked, when a number looks wrong. It pings you **only when something actually
changed**, and it lands in the **right channel** at the **right volume**.

## How it works

Three pieces:

1. **A routing table** (`config/notifications.txt`) — plain text you edit by
   hand. Each line maps a *topic* to a *channel* and a *level*:

   ```
   reports.daily     #my-reports    fyi
   ops.deploy_fail   #my-ops        urgent   push
   private           DM             ask
   ```

2. **A helper** (`bin/proactive-slacks`) — one command your assistant calls:

   ```
   proactive-slacks reports.daily "Daily numbers are in: https://..."
   ```

   It reads the table, posts to the channel, at the level the table says.
   **Channel decides where. Level decides how loud. They're independent.**

3. **A heartbeat** — a scheduled wake (hourly is a good default) that checks
   what changed and pings only on a real delta. A small state file remembers
   what it already told you, so nothing nags. See [`SKILL.md`](./SKILL.md).

## Levels

| level    | what happens                                            |
|----------|---------------------------------------------------------|
| `silent` | log only, no message                                    |
| `fyi`    | post in the channel, no push                            |
| `ask`    | post; the first line is what the assistant needs from you |
| `urgent` | post + lock-screen push                                 |

A topic with no row falls back to its category, then to `default`. Nothing is
ever silently dropped.

## Setup

```bash
export PSLACK_TOKEN="xoxb-..."      # or PSLACK_TOKEN_CMD to fetch it
export PSLACK_DM_USER="U0XXXXXXX"   # only if you route any topic to DM
# optional: a command that fires a lock-screen push, gets the message as last arg
export PSLACK_PUSH_CMD="/usr/local/bin/my-push"

cp config/notifications.txt config/mine.txt   # edit to your channels
export PSLACK_TABLE="$(pwd)/config/mine.txt"

proactive-slacks --list                        # see your routes
proactive-slacks reports.daily "hello" --dry-run
```

**Slack scopes:** `chat:write` (plus `chat:write.public`, or invite the bot to
each target channel), and `im:write` if you use DM rows.

## Why not just DM everything?

Because a wall of DMs is the same as no signal. Topic channels let you mute the
noisy ones, keep the loud ones loud, and skim the rest when you have a minute.
The point isn't to message you more. It's to message you **well**.

## License

MIT.
