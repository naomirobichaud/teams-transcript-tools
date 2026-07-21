# The recurring routine-fetch task prompt

This is the single source of truth for the `teams-transcripts-routine` task prompt. The
`setup-routine-fetch` skill reads the fenced block below at run time and writes it into the
scheduled task; a human setting the task up by hand copies the same block.

The easiest way to set this up is the **`setup-routine-fetch` skill** — it interviews you
(working hours, output dir), sets the activation cutoff to "now", and creates the task for you.
Everything below is for reviewing or setting it up by hand.

**Where this can run.** The on-demand `teams-transcript-fetch` skill works in any Claude Code
surface — CLI, the VS Code/JetBrains extensions, and the desktop app. The **recurring routine
below requires the desktop app**, because only it can schedule a task that reaches your local
Transcripts folder and your Outlook/M365 connector. If you're CLI-only, run the fetch on demand,
or install the desktop app just to hold the routine.

## The prompt (copy this verbatim)

Paste this as the task prompt, replacing `<ACTIVATION-CUTOFF>` with your setup time (see below).
Change nothing else — the output path stays as the env-var expression so it resolves at run time.

```
You are a lightweight transcript fetcher. Today's date is available from the system. Follow this order exactly and minimize token use — most runs find nothing new and should stop after a single calendar check plus one log line.

ACTIVATION CUTOFF: <ACTIVATION-CUTOFF> (an ISO timestamp WITH a UTC offset, e.g. 2026-07-17T18:00:00-0700). This routine only handles meetings going forward from when it was set up. NEVER fetch or consider any meeting that ENDED before this cutoff timestamp — treat those as out of scope, even on a manual "Run now". This prevents backfilling historic meetings. When comparing, normalize BOTH the cutoff and each meeting's end time to UTC (calendar events are typically returned in UTC) so the comparison is not off by the local-UTC offset near the cutoff instant.

Resolve the output directory at the start: TRANSCRIPTS_DIR="${TEAMS_TRANSCRIPTS_DIR:-$HOME/Documents/Transcripts}"; create it if missing. Use it everywhere below.

1. Check $TRANSCRIPTS_DIR for transcript files already saved today OR yesterday (files whose name starts with today's or yesterday's date, YYYY-MM-DD). Looking back two days is deliberate: if the app was closed overnight, this run must still catch a meeting that finished late yesterday after the last successful run.

2. Find the Microsoft 365 / Outlook MCP connector in this session (a tool whose name ends with __outlook_calendar_search); if none is connected, append "YYYY-MM-DD HH:MM — no M365 connector, skipped" to $TRANSCRIPTS_DIR/_log.md and STOP. Otherwise check the calendar for meetings from yesterday and today (roughly the last 48 hours), noting each meeting's scheduled START and END time. Consider only meetings that ended at or after the ACTIVATION CUTOFF above (comparing in UTC as noted); ignore any meeting that ended before the cutoff.

3. COMPLETENESS GATE — a meeting is eligible to fetch only if it has ACTUALLY finished, not merely if its scheduled slot is past. A meeting that is still live, or running over its scheduled end, returns a PARTIAL transcript; saving that would lock in a truncated record. Use a settle buffer of 20 minutes. Defer (skip this run) any meeting where: its scheduled end is still in the future; OR its scheduled end was less than 20 minutes ago (still settling / may be running over); OR it otherwise appears in progress. Deferring is safe: deferred meetings are never saved, and because this routine runs hourly, the next run re-evaluates them automatically — nothing is lost.

4. Cross-reference: from the meetings that PASS the completeness gate, identify those that do NOT yet have a complete saved transcript file from step 1. If a file saved earlier today looks truncated (its last spoken timestamp falls well before the meeting actually ended), treat it as not-yet-complete and re-fetch/overwrite once the meeting has settled.

5. If there are NO eligible new meetings: append one line to $TRANSCRIPTS_DIR/_log.md: "YYYY-MM-DD HH:MM — checked, nothing new" (append " (N deferred, still in progress)" if you skipped any at the gate). Then STOP. Do not call any other tools.

6. If there ARE eligible new meetings: invoke the teams-transcript-fetch skill to retrieve and save each one, passing that meeting's own date (today or yesterday) so it does not search further-back historic dates. (This skill is approved for scheduled/automated invocation by this task.) The skill runs its own final completeness check and will REFUSE to save a transcript that is still growing; if it reports a meeting as still in progress, do not force it — log it as deferred and let the next run pick it up. After fetching, append a line to _log.md noting which meeting(s) were saved (and any deferred).

Notes:
- Keep the empty-run path as cheap as possible: one calendar check, one log append, stop.
- Never trade completeness for a single early fetch: when in doubt about whether a meeting is finished, defer it. The hourly cadence means a deferred meeting is fetched, in full, within the next hour or two.
```

## Two values you set once

- **ACTIVATION CUTOFF** — an ISO timestamp *with* a UTC offset (e.g. `2026-07-17T18:00:00-0700`)
  of roughly when the task is set up. The offset matters: the routine compares this against
  calendar event times in UTC, so a naive offset-less timestamp could be off by hours near the
  cutoff. The routine ignores any meeting that ended before this, so it never backfills your
  entire history. Capture it with `date "+%Y-%m-%dT%H:%M:%S%z"`. Set it to "now".
- **Output directory** — controlled by the `TEAMS_TRANSCRIPTS_DIR` env var (default
  `~/Documents/Transcripts`), same as the fetch skill. You do not edit a path in this prompt;
  it is resolved at run time.

## Suggested schedule

`0 8-17 * * 1-5` — top of each hour, 8am–5pm, weekdays (cron is evaluated in your local
timezone). Adjust to your working hours.

## Adding it by hand (Claude Code desktop app)

Durable local scheduled tasks live in the **Claude Code desktop app** — the standalone terminal
CLI has no such scheduler. Open the desktop app → **Routines** → **New routine** → **Local**, name
it `teams-transcripts-routine`, set the schedule to the cron above, and paste the prompt block
(fill in the activation cutoff with your setup time). The `setup-routine-fetch` skill does all of
this for you.
