---
name: teams-transcript-fetch
description: >
  Fetch and save Microsoft Teams meeting transcripts as local markdown files. Use
  when the user explicitly invokes the teams-transcript-fetch skill, OR when invoked
  by an approved scheduled/automated routine (e.g. a daily-teams-transcripts task).
  Do NOT trigger from casual meeting mentions in conversation, sync requests, or
  other implicit signals. The user provides a meeting name (or names) and an optional
  date or date range. The skill searches the Outlook calendar via a Microsoft 365 MCP
  connector, retrieves the transcript from Teams, formats it, and saves it to a local
  folder (configurable). Also delivers a short summary of key topics.
---

# Teams Transcript Fetch

Fetch one or more Microsoft Teams meeting transcripts and save them as readable local markdown files.

**Run this skill when the user explicitly invokes it, or when an approved scheduled/automated routine (such as a `daily-teams-transcripts` task) invokes it.** Do not trigger it from meeting names mentioned in casual conversation, sync requests, or any other implicit signal.

---

## Prerequisites & configuration

This skill has **no hardcoded credentials or server IDs**. It relies on two things the user configures once:

1. **A Microsoft 365 / Outlook MCP connector** must be connected in this Claude Code session. It must expose (at minimum) a calendar-search tool and a resource-read tool whose names end in the suffixes `__outlook_calendar_search` and `__read_resource`. Any Microsoft 365 connector that surfaces `meetingTranscriptUrl` on calendar events works — the server's ID/prefix does not matter (see the preflight step below). Transcription must have been enabled on the Teams meetings you want to fetch.

2. **Output directory** — where transcripts are saved. Resolved at runtime as:
   ```
   ${TEAMS_TRANSCRIPTS_DIR:-$HOME/Documents/Transcripts}
   ```
   i.e. the `TEAMS_TRANSCRIPTS_DIR` environment variable if set (add it to your `~/.zshrc` / `~/.bashrc` to customize), otherwise `~/Documents/Transcripts`. Always resolve this value at the start of a run and use it everywhere below — never assume a literal path.

3. **Timezone** — meeting times in the saved file are shown in the machine's **local timezone** by default. If `TEAMS_TRANSCRIPTS_TZ` is set (an IANA name like `America/Los_Angeles`), use that instead.

---

## Inputs

- **Meeting name(s)**: Required. One or more names or partial names to search for.
- **Date or date range**: Optional. Defaults to the past 7 days if omitted.

---

## Step-by-step process

### 0. Preflight — locate the connector and resolve config

- **Find the connector.** Scan the tools available in this session for one whose name **ends with** `__outlook_calendar_search`. The text before that suffix is the connector's server prefix (e.g. a UUID or a named server). Reuse that exact same prefix for its companion tools — `<prefix>__read_resource` (and `<prefix>__outlook_email_search` if ever needed). **Do not hardcode or assume any specific server ID.**
  - If no such tool is present, stop and tell the user: *"No Microsoft 365 / Outlook MCP connector is connected in this session. Connect one that exposes calendar search and reports `meetingTranscriptUrl` on events, then re-run."* Do nothing else.
- **Resolve the output directory** into a variable, e.g.:
  ```bash
  TRANSCRIPTS_DIR="${TEAMS_TRANSCRIPTS_DIR:-$HOME/Documents/Transcripts}"
  mkdir -p "$TRANSCRIPTS_DIR"
  ```

Below, `<prefix>` means the connector prefix discovered here.

### 1. Search the calendar

Use `<prefix>__outlook_calendar_search` with the meeting name as the `query` and appropriate date bounds. Default: past 7 days.

If the search returns multiple meetings with the same name (e.g., a recurring series), pick the most recent one within the date range — or ask the user to clarify if ambiguous.

### 2. Read the full calendar event

Call `<prefix>__read_resource` with the event URI returned by the calendar search. This gives you the full event object, which may contain a `meetingTranscriptUrl` field.

**If `meetingTranscriptUrl` is absent:** Tell the user the meeting was found but has no transcript — it may not have been recorded or transcription wasn't enabled. Stop here for that meeting.

### 3. Fetch the transcript

Call `<prefix>__read_resource` with the `meetingTranscriptUrl` value verbatim (it's an opaque URI — pass it exactly as-is).

**Handling large transcripts:** The tool may save the result to a file path instead of returning it inline when the content is large. If the result contains a file path reference rather than inline content, read the file in Python chunks:

```bash
python3 -c "
with open('<path>') as f:
    content = f.read()
import json
data = json.loads(content)
# extract data['transcripts'][0]['content']
"
```

Read in slices of ~12,000 characters if needed to avoid truncation. (Python 3 is the only local dependency, and only for oversized transcripts.)

### 4. Parse the WEBVTT transcript

The transcript content is in WEBVTT format inside `data['transcripts'][0]['content']`. Parse it into readable prose:

- **Group consecutive utterances by the same speaker** into a single block — don't emit a new line for every subtitle chunk.
- **Format**: `**Speaker Name** (HH:MM): text`
- Use the timestamp of the first cue in each speaker block for the `(HH:MM)` marker.
- Strip the `<v SpeakerName>` WEBVTT tags.
- Ignore the `WEBVTT` header and cue timestamps.

**Example output:**
```
**Alex Rivera** (00:01): Okay, here we go. Pretty quick sync for today. I've only got one topic on the agenda...

**Jordan Kim** (00:01): Yeah, absolutely. So let's start with the roadmap...
```

### 4.5 Completeness check — never save an in-progress meeting

Before building the file, confirm the meeting has actually finished. A meeting that is still live — or running past its scheduled end — returns a partial transcript, and saving it would lock in a truncated record (e.g. only the first 30 minutes of a longer meeting).

- Determine the meeting's scheduled START time from the calendar event (step 2).
- Find the LAST cue in the parsed WEBVTT and take its timestamp offset (time elapsed from meeting start). Approximate the wall-clock time of the last spoken words as: `meeting_start + last_cue_offset`.
- If that approximate last-spoken time is within the last ~15 minutes of the current time, the transcript is still growing — the meeting is almost certainly still in progress or only just ended and not yet finalized by Teams.
  - Do NOT write a file. Report: "<meeting> appears to still be in progress (transcript still growing) — skipping; a scheduled run will retry it once it has finished." Move on to the next meeting.
- Only proceed to save when the last-spoken time is comfortably in the past (older than ~15 minutes), indicating the meeting has genuinely concluded.

If a target file already exists but the freshly fetched transcript is longer (the meeting had run over on a prior fetch), overwrite it with the fuller version.

### 5. Build the markdown file

Structure (times shown in the configured local timezone, see Prerequisites):
```markdown
# <Meeting Title>

**Date:** <Day, Month DD YYYY>  
**Time:** <HH:MM – HH:MM TZ> (translate from UTC to the local/configured timezone)  
**Organizer:** <name>  
**Attendees:** <comma-separated list of names>

---

## Transcript

<formatted transcript>
```

### 6. Save the file

- Directory: the resolved `$TRANSCRIPTS_DIR` from the preflight step (already created).
- Filename: `<YYYY-MM-DD>_<meeting-name-slug>.md`
  - Slug: lowercase, spaces → hyphens, strip special chars (e.g., `|`, `/`)
  - Example: `2026-06-03_project-sync.md`

### 7. Report back to the user

After saving:
1. State the file path.
2. Give a **3–5 bullet summary** of the key topics discussed — concrete decisions, blockers, action items, notable names or dates mentioned.

---

## Handling multiple meetings

If the user requests transcripts for multiple meetings, process them sequentially and report all file paths + summaries together at the end.

---

## Edge cases

| Situation | Behavior |
|---|---|
| No Microsoft 365 connector in session | Stop; tell the user to connect one (see Preflight, step 0). |
| No calendar match | Tell the user, suggest trying a broader date range or different name |
| Multiple matches (same name, different dates) | Use the most recent; if truly ambiguous, list options and ask |
| `meetingTranscriptUrl` absent | Report: "Meeting found but no transcript available — may not have been recorded" |
| Transcript `content` is empty | Report it; don't save an empty file |
| Meeting still in progress / running past its scheduled end | Transcript is partial. Do NOT save. Report it as still-in-progress; a scheduled run will retry once it has finished. (See step 4.5.) |
| File already exists but new fetch is longer | The earlier fetch caught the meeting mid-flight. Overwrite with the fuller transcript. |
| File already exists at target path (same length / complete) | Overwrite silently (user re-fetching is intentional) |
