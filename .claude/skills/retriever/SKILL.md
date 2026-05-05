---
name: retriever
description: Answer a question grounded in vault content with note citations, no hallucination. Trigger when user asks "what do I know about X", "have I thought about Y", "do I have notes on Z", "search my vault for", "what did I capture about", or any intent to retrieve personal knowledge.
argument-hint: "what do I know about X / find notes on Y"
---

# Retriever — Vault Search

## Task: $ARGUMENTS

Answer the query strictly from vault content. No hallucination — every claim must cite a note.

### 1. Parse the query

Extract the core topic from `$ARGUMENTS`. Identify likely language (Russian or English) and domain (analyst / student / life / therapy) for scoped search.

### 2. Read the index

Read `meta/vault-index.md` to identify the 1–3 most relevant folder paths for this topic.

### 3. Search

Run in parallel using the built-in Grep tool:
- `Grep(pattern: "<query>", path: ".", glob: "**/*.md")` — broad search across the vault
- `Grep(pattern: "<russian synonym if applicable>", path: ".", glob: "**/*.md")` — Russian fallback (skip if query is English-only)
- `Grep(pattern: "<query>", path: "<most relevant folder>", glob: "**/*.md")` — scoped search in the likely folder

For fuzzy/semantic matching where exact keyword may not appear, also try:
- `Glob(pattern: "**/*<keyword>*.md")` — filename-based match
- Synonyms and related terms as separate Grep calls

Collect matching note paths. Deduplicate.

### 4. Read relevant notes

Read the top 5 matching notes. For each, extract the relevant passages.

### 5. Synthesise and respond

Write a direct answer that:
- Cites each claim with the note path: `([[path/to/note.md]])`
- Groups related points thematically
- Ends with a **Scope** statement if the answer is partial: "I only found notes on X; no coverage of Y."

**Hard rules:**
- Never state a fact not found in a note you actually read.
- If search returns nothing relevant, say: "No vault notes found on this topic."
- Do not suggest the user might have notes you haven't checked — either check or say nothing.

### 6. Offer follow-up

After the answer, offer one of:
- "Want me to create an inbox note summarising this?" → `/scribe`
- "Want me to find related notes on [adjacent topic]?" → re-invoke `/retriever`
