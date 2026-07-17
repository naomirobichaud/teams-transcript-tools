# Optional recipe: daily auto-fetch routine (runs hourly on workdays)

This is a **template** for a scheduled task that runs during your workday and fetches
any newly-finished Teams meetings automatically. It is optional — the skill works fine
on its own. Copy the prompt below into a Claude Code scheduled task (see README →
"Optional: set up the daily auto-fetch routine"). Nothing here is machine-specific except
the two clearly-marked values you set once.

## Two values to set

- **ACTIVATION CUTOFF** — an ISO timestamp of roughly when you set the task up. The routine
  ignores any meeting that ended before this, so it never backfills your entire history.
  Set it to "now" when you create the task.
- **Output directory** — controlled by the `TEAMS_TRANSCRIPTS_DIR` env var (default
  `~/Documents/Transcripts`), same as the skill. You do not edit a path in this prompt.

## Suggested schedule

`0 10-17 * * 1-5` — top of each hour, 10am–5pm, weekdays (cron is evaluated in your local
timezone). Adjust to your working hours.

## Task prompt (copy this)

```
You are a lightweight transcript fetcher. Today's date is available from the system. Follow this order exactly and minimize token use — most runs find nothing new and should stop after a single calendar check plus one log line.

ACTIVATION CUTOFF: <SET-TO-YOUR-SETUP-TIME, e.g. 2026-07-17T18:00:00> (local time). This routine only handles meetings going forward from when it was set up. NEVER fetch or consider any meeting that ENDED before this cutoff timestamp — treat those as out of scope, even on a manual "Run now". This prevents backfilling historic meetings.

Resolve the output directory at the start: TRANSCRIPTS_DIR="${TEAMS_TRANSCRIPTS_DIR:-$HOME/Documents/Transcripts}"; create it if missing. Use it everywhere below.

1. Check $TRANSCRIPTS_DIR for transcript files already saved today (files whose name starts with today's date, YYYY-MM-DD).

2. Find the Microsoft 365 / Outlook MCP connector in this session (a tool whose name ends with __outlook_calendar_search); if none is connected, append "YYYY-MM-DD HH:MM — no M365 connector, skipped" to $TRANSCRIPTS_DIR/_log.md and STOP. Otherwise check the calendar for today's meetings, noting each meeting's scheduled START and END time. Consider only meetings that ended at or after the ACTIVATION CUTOFF above; ignore any meeting that ended before the cutoff.

3. COMPLETENESS GATE — a meeting is eligible to fetch only if it has ACTUALLY finished, not merely if its scheduled slot is past. A meeting that is still live, or running over its scheduled end, returns a PARTIAL transcript; saving that would lock in a truncated record. Use a settle buffer of 20 minutes. Defer (skip this run) any meeting where: its scheduled end is still in the future; OR its scheduled end was less than 20 minutes ago (still settling / may be running over); OR it otherwise appears in progress. Deferring is safe: deferred meetings are never saved, and because this routine runs hourly, the next run re-evaluates them automatically — nothing is lost.

4. Cross-reference: from the meetings that PASS the completeness gate, identify those that do NOT yet have a complete saved transcript file from step 1. If a file saved earlier today looks truncated (its last spoken timestamp falls well before the meeting actually ended), treat it as not-yet-complete and re-fetch/overwrite once the meeting has settled.

5. If there are NO eligible new meetings: append one line to $TRANSCRIPTS_DIR/_log.md: "YYYY-MM-DD HH:MM — checked, nothing new" (append " (N deferred, still in progress)" if you skipped any at the gate). Then STOP. Do not call any other tools.

6. If there ARE eligible new meetings: invoke the teams-transcript-fetch skill to retrieve and save each one, passing today's date so it does not search historic dates. The skill runs its own final completeness check and will REFUSE to save a transcript that is still growing; if it reports a meeting as still in progress, do not force it — log it as deferred and let the next run pick it up. After fetching, append a line to _log.md noting which meeting(s) were saved (and any deferred).

Notes:
- Keep the empty-run path as cheap as possible: one calendar check, one log append, stop.
- Never trade completeness for a single early fetch: when in doubt about whether a meeting is finished, defer it.
```
