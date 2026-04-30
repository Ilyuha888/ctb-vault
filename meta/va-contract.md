---
type: reference
status: evergreen
created: 2026-04-17
source: haft-dec-20260417-002
---

# V-A Methodology Contract

## Note Types

| type | purpose | default status | terminal status | target folder |
|---|---|---|---|---|
| `evergreen` | Permanent knowledge, refined over time | draft | **evergreen** | notes/ |
| `inbox` | Raw capture, unprocessed | raw | archived | inbox/ |
| `project` | Active work with concrete outcome | active | **archived** | projects/ |
| `area` | Ongoing responsibility domain | active | **archived** | areas/ |
| `person` | Contact, person of interest | active | archived | areas/people/ |
| `reference` | External source, third-party material | draft | **evergreen** | resources/ |

> **Type/status rule:** `status: evergreen` is only valid for `type: evergreen` and `type: reference` notes.
> Projects, areas, and inbox items are temporary by nature — their terminal status is `archived`, not `evergreen`.
> A project note marked `status: evergreen` is a miscategorisation — either the type is wrong or the status is wrong.

## Lifecycle States

| status | meaning | valid for types |
|---|---|---|
| `raw` | Unprocessed inbox capture — needs triage | inbox |
| `active` | In progress, being worked on | project, area, person |
| `paused` | Deliberately stopped; will resume later | project |
| `blocked` | Cannot proceed — waiting on external dependency | project |
| `draft` | Being developed; not yet canonical | evergreen, reference |
| `evergreen` | Stable, retrievable, regularly linked | evergreen, reference only |
| `archived` | Retired or superseded; preserved in archive/ | all types |

> **Paused vs blocked:** `paused` is a deliberate choice (deprioritised). `blocked` means there is a specific external dependency (`waiting_on`) preventing progress. A blocked project with no `waiting_on` field is miscategorised — it's paused.

## Frontmatter Schema

### Base schema (all note types — minimum 4 fields required)

```yaml
type: <evergreen|inbox|project|area|person|reference>
status: <raw|active|paused|blocked|draft|evergreen|archived>
created: <YYYY-MM-DD>
source: <telegram|migrated|manual|...>
aliases: [<alternative titles>]  # always include Cyrillic original for migrated notes
```

### Extended schema for `type: project`

```yaml
type: project
status: active          # active | paused | blocked | archived
created: <YYYY-MM-DD>
source: <telegram|manual|...>
outcome: "<what done looks like — one sentence>"
next_action: "<present-tense verb phrase — the single next step>"
waiting_on: null        # or "<person/event/decision>" — required when status: blocked
due: <YYYY-MM-DD>       # optional soft deadline
energy: deep            # deep | shallow | admin — effort type needed
last_reviewed: <YYYY-MM-DD>  # set by weekly review, not by file edits
tags: [...]
```

**Field rules:**
- `next_action` — absence means the project is stalled. Weekly review must always produce one.
- `last_reviewed` — if today minus `last_reviewed` > 7 days, flag in weekly curator.
- `waiting_on` — required when `status: blocked`. Null otherwise.
- `energy` — used by daily focus to match project to current energy state.
- `outcome` — forces clarity on what "done" means. Required for new projects.

## Capture Pipeline (3 steps)

1. **Capture** — create note with `type: inbox, status: raw` in `inbox/`. Raw content only, no editing.
2. **Draft** — agent transforms raw capture into structured note with correct type, frontmatter, wikilinks. Moves to target folder, `status: draft`.
3. **Promote** — human reviews draft, confirms quality, updates `status: draft → evergreen`. Moves to `notes/` if not already there.

## Draft Promotion-Ready Rule

A draft is a candidate for promotion when ALL of the following hold. This is the criterion the curator uses to surface proposals; the promotion act (flipping `status: draft → evergreen`) still requires human confirmation.

| Clause | Check |
|---|---|
| Status | `status: draft` |
| Frontmatter complete | `type`, `status`, `created`, `source` all present and non-empty |
| Body length | ≥ 300 words (excluding the YAML frontmatter block) |
| Age | `created` date ≥ 14 days ago |
| No raw-capture markers | body contains none of: `TODO:`, `WIP`, `[draft]`, `raw transcript` |

The `tags` field is **not** a promotion criterion (many migrated notes lack it by design). A note that passes all five clauses is promotion-ready regardless of tags.

## MOC Creation Rule

Create a MOC in `mocs/` when:
- ≥5 notes share a topic cluster (common wikilinks or title-keyword overlap)
- A topic appears in ≥3 different evergreen notes
- A new domain arrives in inbox with no existing navigation entry

MOC frontmatter: `type: reference, status: evergreen`
MOC body: one section per sub-cluster, wikilinks to every note in cluster, backlink from `meta/topic-index.md`.

## Vault Folders

| Folder | Role | Notes |
|---|---|---|
| `inbox/` | Raw captures | All new notes land here; see Capture Pipeline |
| `notes/` | Evergreen knowledge | Promoted from draft via Capture Pipeline step 3 |
| `projects/` | Active project notes | Archived on completion |
| `areas/` | Ongoing responsibility domains | Persistent; not archived until domain ends |
| `areas/people/` | Contact / person notes | Sub-folder of areas/ |
| `resources/` | External reference material | type: reference notes |
| `mocs/` | Maps of Content | Created per MOC Creation Rule above; type: reference, status: evergreen |
| `assets/` | Binary attachments | Images, PDFs referenced from notes; no sub-folders |
| `archive/` | Superseded / retired notes | Write-once; never edit archived notes |
| `templates/` | Obsidian template plugin sources | Currently empty; reserved |
| `meta/` | Vault-level config and indices | This file, vault-index.md, topic-index.md, reports/ |
| `meta/reports/` | Routine scheduler outputs | monthly_audit → `YYYY-MM.md`; quarterly_review → `YYYY-Qn.md`; auto-generated, read-only for humans |

## projects/tasks.md Schema

`tasks.md` is the singleton task-table in `projects/`. It is a `type: reference, status: evergreen` file — a structural index, not a project note, and exempt from the `outcome`/`next_action` project schema.

| Column | Position | Format | Allowed values |
|---|---|---|---|
| Task | 1 | free text | any |
| Due | 2 | `YYYY-MM-DD` or empty | — |
| Status | 3 | exact lowercase | `todo` · `in-progress` · `done` |

**Status vocabulary is strict**: use exactly `todo`, `in-progress`, `done`. Any other casing or spacing (e.g. `In Progress`, `in progress`) will be silently skipped by `daily_focus`.

**Cleanup rule**: done rows are retained indefinitely. `weekly_curator` flags done rows for awareness but does not delete them (curator is read-only). Cleanup deferred until file exceeds ~5 KB or `daily_focus` token cost becomes measurable.

**Graduation**: a `tasks.md` row is never auto-promoted to a project note. To manually expand a row into a project: create `projects/<slug>.md` with full project frontmatter (`outcome`, `next_action`, etc. per Note Types schema), remove the row from `tasks.md`, commit. The agent does this only under explicit user instruction.

## Query Order (topic lookup)

1. Read `meta/topic-index.md` — find relevant MOC
2. Navigate MOC → follow wikilinks to candidate notes
3. Fall back to Grep (built-in tool) only if no MOC or index entry exists
