---
name: setup-routine-fetch
description: >
  Set up the recurring Teams transcript fetch as a scheduled task on this machine.
  Use when the user explicitly invokes setup-routine-fetch, or asks to "set up the
  routine fetch", "schedule the transcript fetch", "automate my Teams transcripts",
  or similar. This is a one-time setup helper: it interviews the user for working
  hours and confirms configuration, then creates a scheduled task named
  `teams-transcripts-routine` that runs hourly on weekdays and invokes the
  `teams-transcript-fetch` skill for each newly-finished meeting. It does NOT fetch
  transcripts itself. Do NOT trigger from casual meeting mentions.
---

# Set up the routine fetch

This skill creates a **scheduled task** on the user's machine that automatically pulls
newly-finished Teams transcripts during their workday. It replaces the old hand-copy
workflow (copying a prompt into a scheduled task by hand) with a guided setup.

The task it creates is named **`teams-transcripts-routine`**, runs **hourly on weekdays**,
and on each run invokes the companion **`teams-transcript-fetch`** skill for any meeting
that has genuinely finished. This skill only *creates* the task — it never fetches
transcripts itself.

> **Why a setup skill and not a bundled cron file?** A Claude Code plugin cannot ship a
> live scheduled task — scheduled tasks are machine-local, stored as
> `~/.claude/scheduled-tasks/<taskId>/SKILL.md` and registered with the scheduler at
> creation time. So the portable way to package a recurring routine is a setup skill that
> creates the task on the adopting user's machine. That's what this is.

---

## Prerequisites

1. **The `teams-transcript-fetch` skill must be installed** (it ships in this same
   `teams-transcripts` plugin). The scheduled task this skill creates calls it on each run.
   If it is not available, tell the user to install the plugin first and stop.
2. **Scheduled-task tooling.** This skill needs a way to register a scheduled task. Use the
   `create_scheduled_task` tool if it is available in the session; otherwise fall back to the
   `schedule` skill, passing it the equivalent cron expression, task id, and prompt described
   below. If neither is available, tell the user their Claude Code build does not expose
   scheduled tasks and stop.
3. **A Microsoft 365 / Outlook MCP connector** is required *at run time* by the fetch skill,
   not by this setup step. You may check for one (a tool whose name ends with
   `__outlook_calendar_search`) and warn the user if it's missing, but do not block setup on
   it — the routine discovers the connector on each run and logs a skip if none is connected.

---

## Step-by-step process

### 1. Resolve the output directory (do not hardcode a path)

The routine, like the fetch skill, resolves its output directory at run time from an
environment variable. Resolve and show the current value:

```bash
TRANSCRIPTS_DIR="${TEAMS_TRANSCRIPTS_DIR:-$HOME/Documents/Transcripts}"
echo "$TRANSCRIPTS_DIR"
```

Tell the user: *"Transcripts will be saved to `<resolved path>`. To use a different folder,
set `TEAMS_TRANSCRIPTS_DIR` in your shell profile (`~/.zshrc` / `~/.bashrc`) before the task
runs — I won't bake a literal path into the task."* Do **not** write an absolute path into
the task prompt; the task resolves the env var itself on every run.

### 2. Interview the user (use AskUserQuestion)

Confirm only what's genuinely per-user. Keep it short.

- **Working hours** → these become the cron hour range. Default: **8am–5pm, weekdays**
  (`0 8-17 * * 1-5`). Offer the default plus a couple of alternatives (e.g. 9am–6pm
  `0 9-18 * * 1-5`, 10am–5pm `0 10-17 * * 1-5`), and let them give a custom range. The task
  runs at the top of each hour within the range; cron is evaluated in the user's **local**
  timezone.
- **Output directory** → confirm the resolved path from step 1 looks right (see note above
  about the env var). No need to ask if they've clearly already set it.

Do not ask about the activation cutoff — it is always "now" (next step).

### 3. Compute the activation cutoff (= now)

The routine must never backfill historic meetings, so it ignores anything that ended before
the moment it was set up. Capture the current local time as the cutoff:

```bash
date "+%Y-%m-%dT%H:%M:%S"
```

Use that value as the `ACTIVATION CUTOFF` in the task prompt below. This is set once, at
creation, and is what stops the first run from pulling the user's entire meeting history.

### 4. Check for an existing routine task

List scheduled tasks and look for one with id `teams-transcripts-routine`. If it already
exists, tell the user and ask whether to overwrite it (re-create with the new settings) or
leave it as-is — do **not** silently create a duplicate. Creating with the same `taskId`
overwrites the existing task.

### 5. Create the scheduled task

Create a task with:

- **taskId:** `teams-transcripts-routine`
- **description:** `Fetch newly-finished Teams transcripts hourly on weekdays`
- **cronExpression:** the range from step 2 (default `0 8-17 * * 1-5`), local time
- **prompt:** the routine prompt below, with `<ACTIVATION-CUTOFF>` replaced by the timestamp
  from step 3. Do not substitute anything else — the path stays as the env-var expression so
  it resolves at run time.

**Routine task prompt (fill in the cutoff, then use verbatim):**

```
You are a lightweight transcript fetcher. Today's date is available from the system. Follow this order exactly and minimize token use — most runs find nothing new and should stop after a single calendar check plus one log line.

ACTIVATION CUTOFF: <ACTIVATION-CUTOFF> (local time). This routine only handles meetings going forward from when it was set up. NEVER fetch or consider any meeting that ENDED before this cutoff timestamp — treat those as out of scope, even on a manual "Run now". This prevents backfilling historic meetings.

Resolve the output directory at the start: TRANSCRIPTS_DIR="${TEAMS_TRANSCRIPTS_DIR:-$HOME/Documents/Transcripts}"; create it if missing. Use it everywhere below.

1. Check $TRANSCRIPTS_DIR for transcript files already saved today (files whose name starts with today's date, YYYY-MM-DD).

2. Find the Microsoft 365 / Outlook MCP connector in this session (a tool whose name ends with __outlook_calendar_search); if none is connected, append "YYYY-MM-DD HH:MM — no M365 connector, skipped" to $TRANSCRIPTS_DIR/_log.md and STOP. Otherwise check the calendar for today's meetings, noting each meeting's scheduled START and END time. Consider only meetings that ended at or after the ACTIVATION CUTOFF above; ignore any meeting that ended before the cutoff.

3. COMPLETENESS GATE — a meeting is eligible to fetch only if it has ACTUALLY finished, not merely if its scheduled slot is past. A meeting that is still live, or running over its scheduled end, returns a PARTIAL transcript; saving that would lock in a truncated record. Use a settle buffer of 20 minutes. Defer (skip this run) any meeting where: its scheduled end is still in the future; OR its scheduled end was less than 20 minutes ago (still settling / may be running over); OR it otherwise appears in progress. Deferring is safe: deferred meetings are never saved, and because this routine runs hourly, the next run re-evaluates them automatically — nothing is lost.

4. Cross-reference: from the meetings that PASS the completeness gate, identify those that do NOT yet have a complete saved transcript file from step 1. If a file saved earlier today looks truncated (its last spoken timestamp falls well before the meeting actually ended), treat it as not-yet-complete and re-fetch/overwrite once the meeting has settled.

5. If there are NO eligible new meetings: append one line to $TRANSCRIPTS_DIR/_log.md: "YYYY-MM-DD HH:MM — checked, nothing new" (append " (N deferred, still in progress)" if you skipped any at the gate). Then STOP. Do not call any other tools.

6. If there ARE eligible new meetings: invoke the teams-transcript-fetch skill to retrieve and save each one, passing today's date so it does not search historic dates. (This skill is approved for scheduled/automated invocation by this task.) The skill runs its own final completeness check and will REFUSE to save a transcript that is still growing; if it reports a meeting as still in progress, do not force it — log it as deferred and let the next run pick it up. After fetching, append a line to _log.md noting which meeting(s) were saved (and any deferred).

Notes:
- Keep the empty-run path as cheap as possible: one calendar check, one log append, stop.
- Never trade completeness for a single early fetch: when in doubt about whether a meeting is finished, defer it. The hourly cadence means a deferred meeting is fetched, in full, within the next hour or two.
```

### 6. Report back

Confirm to the user, plainly:

1. The task `teams-transcripts-routine` was created, with its schedule in words
   (e.g. "top of each hour, 8am–5pm, weekdays, local time") and the activation cutoff used.
2. Where transcripts will be saved (the resolved `TRANSCRIPTS_DIR`).
3. **Scheduled tasks run only while the Claude Code app is open**; if it's closed when a run
   is due, it runs on next launch.
4. **How to test it now:** run the task manually ("Run now") — on an empty day it should just
   append a `checked, nothing new` line to `$TRANSCRIPTS_DIR/_log.md`.
5. **How to pause or remove it later** (this skill doesn't manage that): pause with the
   scheduled-task tooling (`update_scheduled_task` with `enabled: false`, or the `schedule`
   skill), and remove with `delete_scheduled_task`. Both key off the id
   `teams-transcripts-routine`.

---

## Notes on portability

- **No secrets, no server IDs.** The created task discovers the M365 connector at run time by
  the `__outlook_calendar_search` tool-name suffix, exactly like the fetch skill — nothing
  connector-specific is written into the task.
- **No literal paths.** The task resolves `${TEAMS_TRANSCRIPTS_DIR:-$HOME/Documents/Transcripts}`
  on every run.
- **Completeness is preserved.** The 20-minute settle buffer and defer/retry logic live in the
  task prompt, and the fetch skill's own step-4.5 in-progress guard is the final backstop.
- The only per-machine value baked in at creation is the **activation cutoff** (the setup
  timestamp), which is exactly what should be machine/time-specific.
