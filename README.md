# ctb-vault — Knowledge Base Scaffold

A clean Obsidian vault scaffold designed to pair with [claude-telegram-bot](https://github.com/Ilyuha888/claude-telegram-bot). Provides the PARA folder structure, the V-A note-type methodology, and three PKM skills (Scribe, Retriever, Curator) that the bot uses to capture, retrieve, and curate your notes.

This repo is **structure + methodology only** — zero personal content. Clone it and start populating your own knowledge.

---

## Why this exists

Most "personal AI assistant" projects treat the assistant as a chat interface to an LLM. CTB is different: it's a Claude Code agent with a **knowledge base as a first-class citizen**. The vault is where your captures, decisions, projects, and references live, all under your own git history.

The PKM skills bridge the gap between Telegram (capture surface) and the vault (knowledge store):

- Voice memo on the go → `/scribe` → frontmatter-correct note in `inbox/` with commit-confirm
- Question about something you read months ago → `/retriever` → vault-grounded answer with citations
- Vault drift over time → `/curator` → weekly health report (stale inbox, draft promotions, orphans)

Together with the scheduler routines (daily focus, weekly curator, monthly audit, quarterly review), this becomes a feedback loop: capture → triage → promote → review.

---

## What you get

| Item | Where | Purpose |
|---|---|---|
| **V-A methodology spec** | `meta/va-contract.md` | Defines six note types (evergreen, inbox, project, area, person, reference), lifecycle states, frontmatter schema, draft promotion rules |
| **Vault index** | `meta/vault-index.md` | Folder map with English aliases — what each path is for |
| **Topic index** | `meta/topic-index.md` | Stub for navigating by topic cluster (you populate as your vault grows) |
| **Vault agent instructions** | `CLAUDE.md` | Tells Claude Code how to search, write, and commit — read by the bot at startup |
| **PKM skills** | `.claude/skills/scribe/`, `retriever/`, `curator/` | Invoked from Telegram via the bot |
| **Scheduler prompts** | `prompts/scheduler/*.md` | Optional override templates for the bot's 4 V-rich routines |
| **PARA folders** | `inbox/`, `notes/`, `projects/`, `areas/`, `resources/`, `archive/`, `mocs/`, `assets/`, `templates/` | Empty — you fill them with your own notes |

---

## Quick start (with the bot)

This vault is one of three repos in the full setup. See the [bot's README](https://github.com/Ilyuha888/claude-telegram-bot) for end-to-end clone-and-run instructions. Short version:

```bash
git clone https://github.com/Ilyuha888/claude-telegram-bot ~/repos/claude-telegram-bot
git clone https://github.com/Ilyuha888/ctb-vault          ~/repos/ctb-vault
git clone https://github.com/Ilyuha888/bot-data           ~/repos/bot-data

# Configure the bot's .env to point at this vault:
#   CTB_VAULT_DIR=~/repos/ctb-vault
#   CLAUDE_WORKING_DIR=~/repos/ctb-vault
```

Then start writing notes via Telegram (`/scribe ...`) or directly in your editor.

---

## Standalone use (Obsidian vault, no bot)

You can use this repo as a starting point for your own Obsidian vault even without the bot:

```bash
git clone https://github.com/Ilyuha888/ctb-vault ~/repos/my-vault
# Open ~/repos/my-vault in Obsidian
# Read meta/va-contract.md to learn the V-A methodology
# Start writing notes in inbox/
```

The skills and scheduler prompts are bot-specific but harmless without it. The folder structure and methodology stand on their own.

---

## V-A methodology (one-paragraph summary)

Every note declares a `type` in its frontmatter: `evergreen` (permanent knowledge), `inbox` (raw capture), `project` (active work with an outcome), `area` (ongoing responsibility), `person`, or `reference` (external source). Lifecycle goes: `raw` → `draft` → `evergreen` (terminal for evergreen/reference) or `archived` (terminal for inbox/project/area). Notes start in `inbox/` as raw, get triaged into a target folder as drafts, and the human (not the agent) promotes them to evergreen once they meet the promotion-ready rule (≥300 words, ≥14 days since creation, no raw markers, full frontmatter).

Read `meta/va-contract.md` for the full spec — it's pure spec, no personal content, the load-bearing document of this whole methodology.

---

## License

MIT — paired with the bot's [LICENSE](https://github.com/Ilyuha888/claude-telegram-bot/blob/main/LICENSE).

Fork of methodology and structure inspired by Tiago Forte's PARA system and Maggie Appleton's evergreen note philosophy.
