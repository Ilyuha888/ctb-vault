---
name: scribe
description: Capture a voice memo or text note into the vault inbox as a correctly-formatted Obsidian note
argument-hint: "the content to capture (text, or 'last voice message')"
---

# Scribe — Vault Capture

## Task: $ARGUMENTS

Capture the input as a new inbox note in `inbox/`.

### 1. Determine content

If `$ARGUMENTS` is empty or says "last voice message", the content is the most recent message in this conversation before the `/scribe` invocation. Otherwise use `$ARGUMENTS` as the content verbatim.

### 2. Classify and title

- Read the content and identify: topic, language (Russian or English), and note type.
- Choose a short, descriptive English filename in kebab-case (e.g. `ask-nastya-grade-salary.md`).
- Derive a `type` tag: use `inbox` for raw captures.
- Extract 1–3 relevant tags from the content for the `tags` field (lowercase, no spaces, use existing vault vocabulary if possible).
- Detect time references: if the content contains "tomorrow", "in N days", "by [date]", a specific day name, or any scheduling language, extract a `reminder_date` in `YYYY-MM-DD` format. Use today's date plus the offset (e.g. "tomorrow" = today + 1 day).

### 2.5. Detect task intent — route to tasks.md or inbox

Determine whether this capture is an **actionable task** or a **note/idea**.

**Task signals** (any match → task):
- Explicit keywords: "todo", "task", "add task", "напомни" (remind me), "нужно сделать" (need to do), "сделай" (do this)
- Due-date language combined with an action verb: "by May 3 do X", "до пятницы сделать Y", "present X on May 4"
- Imperative + deadline pattern

**If task detected → go to step 2.6 (Task route).** Otherwise continue to step 3 (inbox route).

**When ambiguous** (could be a task or a note), default to inbox. The weekly curator will flag it later.

### 2.6. Task route — append to tasks.md

Instead of creating an inbox file:

1. Read `projects/tasks.md` to see the current table.
2. Compose a new row per schema in `meta/va-contract.md` § projects/tasks.md Schema:
   `|<task description>|<due date YYYY-MM-DD or empty>|todo|`
   Column order: Task · Due · Status. Status must be exactly `todo` (lowercase).
3. Append the row to the markdown table in tasks.md.
4. Show the user the new row. Ask: "Commit to vault?"
5. Commit and push only after confirmation. Use commit message: `task(scribe): <short task description>`
6. If a `reminder_date` was detected, set a one-shot reminder (same as step 6 below).
7. **Stop here** — do not continue to steps 3–5.

### 3. Check for duplicates and related vault entities

**3a. Duplicate check:**
Use `Grep(pattern: "<key phrase from content>", path: ".", glob: "**/*.md")` to check whether a closely related note already exists. Use 2–3 distinctive words from the content as the pattern. If an obvious duplicate is found, report it and ask whether to append or create new.

**3b. Related entity scan — do this even if no duplicate found:**
Actively look for existing vault entities this capture should link to. Check all of the following:

- **Projects**: does this relate to an active project in `projects/`? If yes, the note body must include a wikilink to that project file (e.g. `[[projects/career-resume-linkedin]]`).
- **Areas**: does this touch a standing area (e.g. health, finances, relationships)? Link to the relevant `areas/` note.
- **MOCs**: does this fall under a role domain (analyst, therapist, living)? Link to the relevant MOC in `mocs/`.
- **People**: does this mention a named person who has a note in `areas/people/`? Link to them.
- **Resources**: does this reference a resource, course, or tool that has a note in `resources/`? Link to it.

Use `Glob` and `Grep` to confirm the target files exist before adding links. Do not invent links to files that don't exist.

Add a `## Related` section at the bottom of the note body listing all found wikilinks. Minimum effort: check projects/ and mocs/ for every capture — these are the most commonly missed connections.

### 4. Write the note

Create `inbox/<filename>.md` with this frontmatter:

```yaml
---
type: inbox
status: raw
created: <YYYY-MM-DD>
source: telegram
tags: [<tag1>, <tag2>]
reminder_date: <YYYY-MM-DD>   # omit this line if no time reference found
---
```

Then the content body. **Keep it verbatim in the original language — do NOT translate, paraphrase, or summarize.** If the user wrote in English, the body is in English. If in Russian, the body is in Russian.

If the conversation includes an attachment hint (`[Attachments on disk: ...]`), copy each file into the vault's `assets/` directory and reference it in the note body as `[[assets/filename.ext]]`.

### 5. Show and confirm

Display the full note content to the user. If a `reminder_date` was detected, include a line:
> "⏰ Reminder will be set for <date> at 09:00 MSK."

Ask:
> "Commit to vault?"

Commit and push **only after** explicit confirmation. Use commit message: `inbox(scribe): <filename without extension>`

### 6. Set reminder (only if reminder_date was detected)

After the commit is confirmed and pushed, append a one-shot entry to the scheduler:

```bash
bun -e "
const fs = require('fs');
const path = process.env.BOT_DATA_DIR + '/schedules.json';
const data = JSON.parse(fs.readFileSync(path, 'utf8'));
data.push({
  id: 'remind-' + crypto.randomUUID().slice(0, 8),
  cron: '',
  tz: 'Europe/Moscow',
  prompt_key: 'scribe_reminder',
  last_fired: 'FIRE_AT_ISO',
  one_shot: true,
  payload: {
    reminder_message: 'REMINDER_TEXT',
    note_path: 'NOTE_ABSOLUTE_PATH'
  }
});
fs.writeFileSync(path, JSON.stringify(data, null, 2) + '\n');
"
```

Replace:
- `FIRE_AT_ISO` — `<reminder_date>T09:00:00+03:00` (e.g. `2026-04-25T09:00:00+03:00`)
- `REMINDER_TEXT` — short title of what to remember (derived from note title)
- `NOTE_ABSOLUTE_PATH` — absolute path to the committed note

The running scheduler watches `schedules.json` and will register the timer automatically within seconds. Confirm to the user: "⏰ Reminder set for <date>."
