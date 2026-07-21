# Teams Transcript Fetch

A Claude Code plugin that fetches Microsoft Teams meeting transcripts and saves them as
clean, readable local markdown files — one file per meeting, with an organizer/attendee
header and a speaker-grouped transcript. Includes a guard that refuses to save a partial
transcript from a meeting that's still in progress.

The plugin ships **two skills**:

- **`teams-transcript-fetch`** — pull and save one or more transcripts on demand.
- **`setup-routine-fetch`** — a one-time helper that creates a scheduled task
  (`teams-transcripts-routine`) which runs the fetch hourly on weekdays and grabs
  newly-finished meetings automatically.

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

## Set up the routine fetch

Want new transcripts pulled automatically during your workday? Invoke the setup skill — it does
the wiring for you, no hand-copying:

```
/teams-transcripts:setup-routine-fetch
```

It interviews you (working hours, confirms your output dir), sets the activation cutoff to
**now**, and produces a scheduled task named **`teams-transcripts-routine`** that runs **hourly
on weekdays** and calls `teams-transcript-fetch` for each newly-finished meeting. The only value
baked in at creation is the setup timestamp — the output path stays an env var and the Microsoft
365 connector is discovered at run time.

### Requires the Claude Code desktop app

The routine runs as a durable local scheduled task, which is a **Claude Code desktop app** feature.
This applies to the VS Code/JetBrains extensions too — only the desktop app can schedule a routine
with access to your local files and connector; the on-demand fetch, though, works everywhere.

- **On the desktop app** → the skill creates `teams-transcripts-routine` for you automatically.
- **On the terminal CLI** → the skill fills in the complete routine and hands it to you with steps
  to add it via the desktop app's **Routines** panel (**New routine → Local**). That same
  ready-to-paste routine lives in
  [`skills/setup-routine-fetch/routine-prompt.md`](skills/setup-routine-fetch/routine-prompt.md).

### Good to know

- **Scheduled tasks run only while the desktop app is open.** If it's closed when a run is due, it
  runs on next launch. To cover an overnight close, each run looks back over the **last ~48 hours**
  (yesterday and today), so a meeting that finished late yesterday is still picked up the next
  morning rather than being missed.
- The routine only ever fetches meetings that have **genuinely finished** (20-minute settle buffer)
  and retries anything still in progress on the next hourly run — so it never saves a truncated
  transcript.
- **Pause or remove it later** (the setup skill only creates it) with `update_scheduled_task`
  (`enabled: false`) or `delete_scheduled_task`, keyed off the id `teams-transcripts-routine`.

---

## License

MIT
