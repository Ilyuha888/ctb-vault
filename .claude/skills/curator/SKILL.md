---
name: curator
description: Generate a vault curation report — stale inbox, draft promotions, orphan notes, MOC gaps. Trigger when user asks for a vault report, "what needs attention", "what's stale", "vault health", "clean up my vault", or wants a curation summary.
argument-hint: "(no arguments needed)"
---

# Curator — Weekly Vault Curation

## Task: $ARGUMENTS

Generate a curation report for the vault. Read-only — no writes, no commits.

### 0. The rule that governs every section

**Never propose an action on a file you have not opened.** Counts and frontmatter locate
candidates; they do not classify them. A `status:` field cannot distinguish a spent note from a
live one nobody re-tagged, and a filename cannot distinguish a deliberate import from an
abandoned draft.

This has produced a wrong proposal in every report so far:

| Reported | Actually |
|---|---|
| 15 orphan notes, "course leftovers" | 7 were the vault's only calculus/linear-algebra material |
| 12 stale inbox notes | 3 were spent; 4 were finished reference mislabelled `raw` |
| 13 of 17 areas have no owning project | 10 did, and having none is what an area *is* |
| 114 promotion-ready drafts | one bulk import whose `created` is the import date |
| 3 done rows to clean up | a 4-row table replaced months ago by per-project `next_action` |

Each collapsed on reading the files. None would have collapsed on re-running the query. So:
**a section may report a count, but every named proposal must cite something from inside the
file.** If reading the candidates would exceed the budget, report the count and say the
candidates were not read — an unverified count is a lead, not a finding, and must be labelled
as one.

### 1. Inbox scan

List all files in `inbox/`. For each, read frontmatter only (first 10 lines). Classify:
- `status: raw` and `created` older than 7 days → **stale inbox** (needs processing)
- `status: raw` and `created` within 7 days → **fresh** (skip)

**Before proposing any inbox note for archiving, check for inbound links** — inbox notes
are linked from projects and areas, and those links carry the `inbox/` prefix:

```
Grep(pattern: "\\[\\[([^]|]*/)?<notename>", path: ".", glob: "**/*.md")
```

A note with live inbound links can still be archived, but every referring file must be
repointed to `archive/` in the same commit. Report the referring paths alongside the
proposal so the cost is visible before the move, not after.

### 2. Draft promotion candidates

Per `va-contract.md`: drafts live in *target folders*, not `inbox/`. Scan `notes/`, `projects/`, `areas/`, and `resources/` for notes with `status: draft` using:

```
Grep(pattern: "status: draft", path: "<folder>", glob: "**/*.md")
```

Run for each folder in parallel. For each match, read the first 30 lines plus a word count of the body (lines after the closing `---`). A draft is **promotion-ready** if ALL of the following hold (per `meta/va-contract.md` § Draft Promotion-Ready Rule):
- `status: draft`
- base frontmatter complete and non-empty: `type`, `status`, `created`, `source` all present
- body word count ≥ 300 (excluding frontmatter block)
- `created` date ≥ 14 days ago
- no raw-capture markers in body: `TODO:`, `WIP`, `[draft]`, `raw transcript`

⚠️ **First exclude the migration import, or this section is meaningless.** Check the date spread
before applying the age clause:

```bash
cd ${CTB_VAULT_DIR}
grep -rh "^created:" $(grep -rl "^status: draft" notes resources) | sort | uniq -c | sort -rn | head
```

**115 drafts share `created: 2026-04-15`** (63 in `resources/`, 52 in `notes/`) and 15 more share
`2026-04-25` — these are bulk migration imports, where `created` records *the import*, not
authorship. The "≥14 days ago" clause passes automatically for all of them, so it discriminates
nothing and the section reports ~114 candidates every single month. **A dominant repeated
`created` date is an import signature: exclude that cohort from promotion candidates entirely
and say so in one line, rather than reporting it as a backlog.**

What remains after the exclusion is the real candidate pool — typically single digits. If it is
empty, that is the correct answer and `✓ nothing to do` is the honest report.

Propose up to 3 promotions with:
- Absolute path
- Suggested target folder (use `meta/vault-index.md` for routing)
- One-sentence rationale

### 3. Orphan detection

Use `git log` (not `find -mtime`, which is unreliable in git-managed repos) to find notes with no recent commits. Run in the vault repo:

```bash
cd ${CTB_VAULT_DIR} && \
for f in notes/*.md; do \
  last=$(git log -1 --format="%ar" -- "$f" 2>/dev/null); \
  echo "$last | $f"; \
done | grep -E "month|week" | head -10
```

Then for each candidate, check if any other note links to it. **Links in this vault are written both
with and without a folder prefix** — `[[notes/foo]]` and `[[foo]]` both occur — so the pattern must
allow an optional path segment:
```
Grep(pattern: "\\[\\[([^]|]*/)?<notename>", path: ".", glob: "**/*.md")
```
A bare `\\[\\[<notename>` pattern silently misses every prefixed link and will report linked notes as
orphans. Notes with no incoming wikilinks and no recent commits are orphan candidates. Read 3–5 to
spot patterns.

### 4. MOC gap identification

List all `.md` files in `mocs/` with `ls ${CTB_VAULT_DIR}/mocs/`. For each MOC stem
(filename without `.md`), count inbound wikilinks **across the whole vault, in both link forms**:

```bash
cd ${CTB_VAULT_DIR}
for f in mocs/*.md; do s=$(basename "$f" .md)
  n=$(grep -rl -- "\[\[\(mocs/\)\?$s" . --include="*.md" 2>/dev/null \
      | grep -v "^./mocs/$s.md" | grep -v "^./meta/" | wc -l)
  printf "%-34s %s\n" "$s" "$n"
done
```

Two things this gets right that the obvious version does not, and both must stay:

- **Match `[[mocs/<stem>]]` as well as `[[<stem>]]`.** Most links in this vault carry the folder
  prefix. Counting only the bare form reports nearly every MOC as an orphan.
- **Search the whole vault, not `notes/`.** MOCs are linked mainly from `projects/`, `areas/` and
  `inbox/`. Scoping to `notes/` is what made `being-analyst` (12 real inbound links) and
  `living-this-life` (10) read as zero on 2026-08-16.

Exclude the MOC's own file and `meta/` — self-links and index entries otherwise inflate every count.

Flag:
- **Orphan MOC**: fewer than 3 inbound links vault-wide
- **Missing MOC**: 5+ notes in `notes/` share a common filename keyword but no `mocs/<keyword>.md` exists (scan top-frequency title words, skip stop-words)

### 5. Deliver the report

Format as Telegram-friendly markdown (short sections, bullets, no tables):

```
📋 Weekly Curation Report

🗂 Stale inbox (N)
• <filename> — <age>d, topic: <one-line>

📤 Draft promotions (N)
• <draft name> → <target folder> — <rationale>

👻 Orphan candidates (N)
• <note name> — <age>d since last edit

🗺 MOC gaps (N)
• <folder> — <N> notes, no MOC
```

If everything is clean in a section, write "✓ nothing to do".

Mark every proposal with how it was established:
- **read** — the file was opened this run. Only these may be proposed as actions.
- **count only** — measured but unread. State it as a lead: *"N candidates, not read."*

Never let a count-only line carry a verb like *archive*, *promote* or *clean up*.

**Constraints:**
- Read-only — do NOT write, edit, or commit anything.
- **Read every file you name in a proposal.** Reading is cheap; a wrong proposal costs a note.
  Skim frontmatter to shortlist, then open the shortlist. Reading ~6 candidate files is the
  expected cost of this report, not an overrun.
- Keep the report under 1500 characters total for Telegram readability. **If depth and the
  character budget conflict, cut sections, not verification** — three checked findings beat six
  guesses. A section with unread candidates says so explicitly.

**Two shell traps that have each produced a silent false negative here:**
- `grep -r --include="*.md"` fails under `ugrep` with
  `warning: --include=*.md: No such file or directory` and returns **empty**, which reads as
  "clean". Use explicit `-e` patterns and no `--include`. **A grep that warns has not answered
  the question — treat it as unrun, not as evidence of absence.**
- A bare `\[\[<notename>` pattern misses every prefixed link. Always allow the optional path
  segment, as in sections 1, 3 and 4.
