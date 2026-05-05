---
name: curator
description: Generate a vault curation report — stale inbox, draft promotions, orphan notes, MOC gaps. Trigger when user asks for a vault report, "what needs attention", "what's stale", "vault health", "clean up my vault", or wants a curation summary.
argument-hint: "(no arguments needed)"
---

# Curator — Weekly Vault Curation

## Task: $ARGUMENTS

Generate a curation report for the vault. Read-only — no writes, no commits.

### 1. Inbox scan

List all files in `inbox/`. For each, read frontmatter only (first 10 lines). Classify:
- `status: raw` and `created` older than 7 days → **stale inbox** (needs processing)
- `status: raw` and `created` within 7 days → **fresh** (skip)

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

Propose up to 3 promotions with:
- Absolute path
- Suggested target folder (use `meta/vault-index.md` for routing)
- One-sentence rationale

### 3. Orphan detection

Use `git log` (not `find -mtime`, which is unreliable in git-managed repos) to find notes with no recent commits. Run in the vault repo:

```bash
cd $CTB_VAULT_DIR && \
for f in notes/*.md; do \
  last=$(git log -1 --format="%ar" -- "$f" 2>/dev/null); \
  echo "$last | $f"; \
done | grep -E "month|week" | head -10
```

Then for each candidate, check if any other note contains a `[[<notename>]]` wikilink using:
```
Grep(pattern: "\\[\\[<notename>", path: ".", glob: "**/*.md")
```
Notes with no incoming wikilinks and no recent commits are orphan candidates. Read 3–5 to spot patterns.

### 4. MOC gap identification

List all `.md` files in `mocs/` with `ls $CTB_VAULT_DIR/mocs/`. For each MOC stem (filename without `.md`), count inbound wikilinks from `notes/`:
```bash
grep -rl "\[\[<stem>" $CTB_VAULT_DIR/notes/ | wc -l
```
Flag:
- **Orphan MOC**: fewer than 3 inbound links from `notes/`
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

**Constraints:**
- Read-only — do NOT write, edit, or commit anything.
- Do not read full note bodies unless needed for draft promotion assessment.
- Keep the report under 1500 characters total for Telegram readability.
