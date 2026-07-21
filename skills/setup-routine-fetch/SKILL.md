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

---

## Prerequisites

1. **The `teams-transcript-fetch` skill must be installed** (it ships in this same
   `teams-transcripts` plugin). The scheduled task this skill creates calls it on each run.
   If it is not available, tell the user to install the plugin first and stop.
2. **Desktop app vs. CLI — this decides how the task is registered.** Durable local scheduled
   tasks are a **Claude Code desktop app** feature: the desktop app ships a built-in
   `create_scheduled_task` tool (backing `~/.claude/scheduled-tasks/`) out of the box, with no
   configuration. The standalone **terminal CLI does not have it**, and its schedulers don't fit
   this routine — cloud `schedule` / `RemoteTrigger` runs in an isolated sandbox with no access to
   your local files or Microsoft 365 connector, and session-scoped `CronCreate` / `/loop` jobs are
   in-memory and expire in ~7 days. So branch on the tool:
   - **If `create_scheduled_task` is available** (desktop app) → do the **Automatic setup** below
     (steps 1–6).
   - **If it is NOT** (terminal CLI, or any session without it) → do **not** substitute the cloud or
     in-memory schedulers. Go to **Manual setup** (the section right after step 6): still interview
     and generate the fully filled-in routine, then hand it to the user with registration steps,
     leading with the representative message quoted there so it's clear *why* automatic setup isn't
     possible in that session.
3. **A Microsoft 365 / Outlook MCP connector** is required *at run time* by the fetch skill,
   not by this setup step. You may check for one (a tool whose name ends with
   `__outlook_calendar_search`) and warn the user if it's missing, but do not block setup on
   it — the routine discovers the connector on each run and logs a skip if none is connected.

---

## Automatic setup (Claude Code desktop app)

Use this path when `create_scheduled_task` is available. Steps 1–3 (resolve dir, interview,
compute cutoff) are shared with Manual setup; steps 4–6 do the actual registration.

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
the moment it was set up. Capture the current time as the cutoff **with an explicit UTC offset**
so the comparison against calendar events (which the connector returns in UTC) is unambiguous:

```bash
date "+%Y-%m-%dT%H:%M:%S%z"
```

This produces e.g. `2026-07-17T18:00:00-0700` — the offset (`-0700`) is what lets the routine
compare the cutoff against UTC event times correctly. A naive timestamp with no offset would be
off by the local–UTC difference (hours) near the cutoff instant. Use this value as the
`ACTIVATION CUTOFF` in the task prompt below. This is set once, at creation, and is what stops
the first run from pulling the user's entire meeting history.

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
- **prompt:** the routine prompt. Read the bundled [`routine-prompt.md`](routine-prompt.md)
  (sibling of this file) and use the fenced prompt block from it verbatim, substituting
  `<ACTIVATION-CUTOFF>` with the timestamp from step 3. Do not substitute anything else — the
  path stays as the env-var expression so it resolves at run time.

`routine-prompt.md` is the single source of truth for the prompt text — do not paraphrase it or
keep a second copy here.

### 6. Report back

Confirm to the user, plainly:

1. The task `teams-transcripts-routine` was created, with its schedule in words
   (e.g. "top of each hour, 8am–5pm, weekdays, local time") and the activation cutoff used.
2. Where transcripts will be saved (the resolved `TRANSCRIPTS_DIR`).
3. **Scheduled tasks run only while the Claude Code app is open**; if it's closed when a run
   is due, it runs on next launch.
4. **How to test it now:** run the task manually ("Run now") — on an empty day it should just
   append a `checked, nothing new` line to `$TRANSCRIPTS_DIR/_log.md`.
5. **How to pause or remove it later** (this skill doesn't manage that): pause with
   `update_scheduled_task` (`enabled: false`) and remove with `delete_scheduled_task`, both
   keyed off the id `teams-transcripts-routine`.

---

## Manual setup (terminal CLI, or any session without `create_scheduled_task`)

Reach here when the scheduled-task tool is not available (see Prerequisite 2). Do **not** fall
back to the cloud `schedule` skill or `CronCreate` — neither can run this routine. Instead:

### M1. Lead with the representative message

Tell the user plainly *why* you can't create it here, and that you'll hand them a ready-to-add
routine:

> *"Heads up — this routine has to run on the **Claude Code desktop app**. Durable local scheduled
> tasks (the kind that run on a schedule and can reach your local Transcripts folder and your
> Microsoft 365 connector) are a built-in desktop-app feature; the terminal CLI you're in right now
> doesn't provide that scheduler. The CLI's alternatives don't fit — cloud routines can't see your
> local files or connector, and `/loop` jobs are session-only and expire after about a week. So I
> can't create the task from here, but I've filled in the exact routine for you to add in the
> desktop app — it takes about a minute."*

### M2. Still run steps 1–3

Resolve the output dir, interview working hours, and compute the activation cutoff (= now), exactly
as in the automatic path, so you can produce a complete, filled-in routine.

### M3. Output the ready-to-register routine

Print the full task for the user to copy, with the cutoff already substituted and the cron they chose:

~~~
Task name / id:  teams-transcripts-routine
Description:     Fetch newly-finished Teams transcripts hourly on weekdays
Schedule (cron): 0 8-17 * * 1-5      (or the range they chose; local time)

Prompt:
<the "Routine task prompt" from step 5, with ACTIVATION CUTOFF filled in>
~~~

### M4. Give the registration steps

- Open the **Claude Code desktop app** → **Routines** in the sidebar → **New routine** → **Local**.
- Paste the prompt, set the schedule to the cron shown, name it `teams-transcripts-routine`, and save.
- (Equivalently, from *any* desktop-app session you can just ask: *"create a local scheduled task
  named `teams-transcripts-routine`, cron `0 8-17 * * 1-5`, with this prompt: …"* — the desktop app
  will create it.)
- The prompt to paste is the fenced block in [`routine-prompt.md`](routine-prompt.md), which also
  documents the two set-once values and the by-hand steps.

Once added on the desktop app, it behaves identically to the automatic path.

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
