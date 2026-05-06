# Calipokehouse TikTok → Discord Notifier

Fully automated replacement for Pipeline Dream. Drop-in compatible with your existing message style — preserves all the day-specific notifications you had set up, just runs reliably and free.

## What it does

1. **Detects when your TikTok stream is actually live** — only fires notifications when TikTok confirms the stream is up. Solves the false-positive issue you had with Pipeline Dream.
2. **Posts day-specific messages** for each shift, matching your Pipeline Dream message style exactly.
3. **Handles the continuous Thu-Mon stream** — fires a notification at each shift change even though TikTok sees one continuous session.
4. **Enforces "max one notification per shift per day"** cooldown.
5. **Runs entirely free on GitHub Actions** — no servers, no subscriptions.

## How notifications fire

Every 5 minutes, the script asks one question: *"Is the stream live AND is there a scheduled shift active right now AND have we not yet announced this shift today?"* If yes to all three, it posts the right message. Otherwise, silence.

This single rule handles every case you described:
- 1st shift streamer running 30 min late → no false ping; fires when they actually go live
- Stream briefly disconnects and reconnects within a shift → no duplicate
- Continuous Thu-Mon stream's overnight handoff → fires the new shift's message
- Stream stays live past shift end → no spam, just waits for next shift

## Files

| File | What it does |
|---|---|
| `notifier.py` | Main script |
| `shifts.json` | Shift schedule — which shift type runs when |
| `messages.json` | All Discord messages (the part you'll edit most) |
| `state.json` | Auto-managed; tracks what's been announced |
| `requirements.txt` | Python deps |
| `.github/workflows/notifier.yml` | GitHub Actions cron (every 5 min) |

## Setup — one time, ~15 minutes

### 1. Regenerate your Discord webhook

**Important:** The webhook URL `1422634450273046629/Ulk8KqdfafVdnjw3FGwNnikbff1DpsF4...` was hardcoded in your old Pipeline Dream scripts and shared in plain text. Anyone with that URL can post fake messages to your channel.

In Discord: right-click your notification channel → **Edit Channel** → **Integrations** → **Webhooks** → click your webhook → **Copy Webhook URL** at the top, then click the red **"X"** to delete it. Create a new webhook to replace it. Copy the new URL — you'll paste it into GitHub in step 4.

### 2. Create a private GitHub repository

Go to [github.com/new](https://github.com/new). Name it `calipokehouse-notifier`. Set to **Private**. Upload all files from this folder.

### 3. (Optional) Customize messages

Open `messages.json` and tweak any wording. The structure is one section per shift type, with one message per day of the week. Edit anything you want — emojis, time windows, verbiage. No code changes needed.

### 4. Add GitHub secrets

In your repo: **Settings** → **Secrets and variables** → **Actions** → **New repository secret**.

| Secret name | Value |
|---|---|
| `DISCORD_WEBHOOK_URL` | The new webhook URL from step 1 |
| `TIKTOK_USERNAME` | `calipokehouse` |

### 5. Enable Actions write permission

**Settings** → **Actions** → **General** → scroll to **Workflow permissions** → select **"Read and write permissions"** → **Save**.

### 6. Test it

Go to **Actions** tab → click **Calipokehouse Notifier** → **Run workflow** → **Run workflow**.

Wait 30 seconds, then click into the run. Should see logs like:

```
[info] Tick at 2026-05-05T18:30:00-07:00
[info] is_live=False  active_shift=None
[info] Done.
```

### 7. Remove Pipeline Dream

Disable Pipeline Dream's scheduled actions and remove its access to your Discord channel before the next stream. Otherwise you'll get duplicate notifications until Pipeline Dream's subscription lapses.

## Roster

Current shift roster (per the screenshot you provided):

| Shift | Days | Streamers |
|---|---|---|
| Shift 1 | Mon-Thurs | Austen & Aryton |
| Shift 1 | Fri-Sun | Shannon & Jack |
| Shift 2 | Mon-Weds | Shannon & Slick |
| Shift 2 | Thu-Sat | Apollo & Ahnist |
| Shift 2 | Sun | Shannon & Slick |

To update roster: edit `messages.json`. The streamer names appear in the message text, so changing them there is all you need to do.

## Maintenance

- **Streamer changes** (someone calls out, day swaps): edit `messages.json` for the affected day, commit to GitHub. Next 5-minute tick uses the new wording.
- **Schedule changes** (DST, holidays): the script uses `America/Los_Angeles` and auto-handles DST. For one-off holiday changes, edit `shifts.json`.
- **Health check**: glance at the **Actions** tab occasionally. Should see green checkmarks every 5 minutes.

## Troubleshooting

**Notifications not firing when stream is live**
TikTok occasionally changes their HTML structure. Look at `notifier.py` → `is_tiktok_live()` and update the `live_signals` list. Recent Actions logs will show what TikTok is currently returning.

**Notifications firing when stream isn't live (the original Pipeline Dream problem)**
This shouldn't happen — the script only fires when TikTok confirms live state. If it does, check Actions logs to see what `is_live` returned. Usually means a `live_signals` pattern is matching too loosely.

**GitHub Actions cron delays**
GitHub's free cron can have 5-15 minute delays during high traffic. Acceptable for stream announcements (a few-minute delay vs. a 24/7 sub fee). If sub-minute precision becomes critical, the script also runs on any always-on machine — see `notifier.py`, just call it on a 60-second loop.

**Streamer goes live before their shift starts**
Currently the script will not announce them. To fix: open `shifts.json` and adjust the shift's `start` time earlier (e.g., 12:30 instead of 13:00). The shift window is the only time the corresponding message fires.

**Streamer disconnects mid-shift, comes back**
No duplicate notification — once a shift is announced for the day, the cooldown holds.
