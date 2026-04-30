# Scheduler Prompts

These files are the prompt bodies for the 4 built-in V-rich scheduler routines.

The bot ships with these prompts as built-in defaults. If you set `CTB_VAULT_DIR` in your `.env`, the bot will prefer the versions here over the built-ins — allowing you to customize the prompts without editing bot source code.

Path references use `$CTB_VAULT_DIR` as a placeholder for your vault root. The bot substitutes the actual path at runtime.

| File | Routine | Default schedule |
|---|---|---|
| `daily_focus.md` | Daily digest of active projects + tasks | Daily 09:00 |
| `weekly_curator.md` | Vault curation: stale inbox, drafts, project momentum | Sunday 20:00 |
| `monthly_audit.md` | Monthly project health + area coverage review | 1st of month 10:00 |
| `quarterly_review.md` | Strategic synthesis of what shipped, slipped, and patterns | Quarterly |
