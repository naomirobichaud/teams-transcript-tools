# Teams Transcript Fetch

A Claude Code plugin that fetches Microsoft Teams meeting transcripts and saves them as
clean, readable local markdown files — one file per meeting, with an organizer/attendee
header and a speaker-grouped transcript. Includes a guard that refuses to save a partial
transcript from a meeting that's still in progress.

This repo is **both a plugin and a single-plugin marketplace**, so anyone can install it
without a hosted registry.

---

## What it needs (prerequisites)

1. **Claude Code** (recent version — plugin/marketplace support).
2. **A Microsoft 365 / Outlook MCP connector** connected in your Claude Code session that:
   - exposes a calendar-search tool (name ending in `__outlook_calendar_search`) and a
     resource-read tool (name ending in `__read_resource`), and
   - surfaces a `meetingTranscriptUrl` field on calendar events.
   The skill discovers this connector at runtime by tool-name suffix — it does **not**
   hardcode any server ID, so it works regardless of your connector's install ID or name.
3. **Teams meetings with transcription enabled** (no transcript exists otherwise).
4. **Python 3** — only used as a fallback for very large transcripts; already present on macOS/most Linux.

> **No secrets live in this plugin.** Your Microsoft 365 credentials stay in your own MCP
> connector configuration (e.g. `~/.claude.json`). Nothing is embedded in the skill.

---

## Install

From a local clone:

```
/plugin marketplace add /path/to/teams-transcript-tools
/plugin install teams-transcripts@teams-transcript-tools
```

Or straight from a git repo (once you push this folder somewhere):

```
/plugin marketplace add naomirobichaud/teams-transcript-tools
/plugin install teams-transcripts@teams-transcript-tools
```

Then reload:

```
/reload-plugins
```

---

## Configure (optional)

Both settings are environment variables read at runtime — set them once in your shell
profile (`~/.zshrc` / `~/.bashrc`). Shell-profile vars are available to the skill.

| Variable | Default | Purpose |
|---|---|---|
| `TEAMS_TRANSCRIPTS_DIR` | `~/Documents/Transcripts` | Where transcript `.md` files are saved. |
| `TEAMS_TRANSCRIPTS_TZ` | machine local timezone | IANA name (e.g. `America/Los_Angeles`) for meeting times shown in the file. |

Example:

```bash
echo 'export TEAMS_TRANSCRIPTS_DIR="$HOME/work/transcripts"' >> ~/.zshrc
```

---

## Use

Invoke the skill (plugin skills are namespaced by plugin name):

```
/teams-transcripts:teams-transcript-fetch
```

Give it a **meeting name** (or several) and an optional **date / date range** (defaults to
the past 7 days). It searches your calendar, pulls the transcript, saves the markdown file,
and reports the path plus a short summary of key topics.

---

## Optional: set up the daily auto-fetch routine

Want new transcripts pulled automatically during your workday? See
[`examples/daily-teams-transcripts.md`](examples/daily-teams-transcripts.md) for a ready-to-copy
scheduled-task prompt. In Claude Code, use the `schedule` skill (or ask: *"schedule this to run
hourly on weekdays"*) and paste in that prompt, setting the two marked values (activation cutoff
+ your working hours). The routine only ever fetches meetings that have genuinely finished, and
retries anything still in progress on the next run.

---

## How it avoids truncated transcripts

Teams will hand back a *partial* transcript for a meeting that's still live or running past
its scheduled end. Two layers prevent saving those:

- The **skill** computes when the last words were spoken and, if that's within the last ~15
  minutes, treats the meeting as still in progress and declines to save.
- The **scheduled routine** (if you use it) won't even attempt a meeting until 20 minutes
  past its scheduled end, and re-evaluates deferred meetings on the next hourly run.

If an earlier fetch caught a meeting mid-flight, a later fetch with a longer transcript
overwrites the shorter file.

---

## License

MIT
