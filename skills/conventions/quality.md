# Quality Convention

Cross-cutting quality rules for all brain-writing skills.

## Citations (MANDATORY)

Every fact written to a brain page must carry an inline `[Source: ...]` citation.

- **User's statements:** `[Source: User, {context}, YYYY-MM-DD]`
- **Meeting data:** `[Source: Meeting "{title}", YYYY-MM-DD]`
- **Email/message:** `[Source: email from {name} re: {subject}, YYYY-MM-DD]`
- **Web content:** `[Source: {publication}, {URL}, YYYY-MM-DD]`
- **Social media:** `[Source: X/@handle, YYYY-MM-DD](URL)`
- **Synthesis:** `[Source: compiled from {sources}]`

### Source precedence (highest to lowest)

1. User's direct statements (highest authority)
2. Compiled truth (brain's synthesized understanding)
3. Timeline entries (raw evidence)
4. External sources (API enrichment, web search)

## Back-Linking (MANDATORY)

Every mention of an entity WITH a brain page MUST create a back-link
FROM that entity's page TO the page mentioning it.

Entities: goals, decisions, processes, concepts — any page in a recognized
entity directory.

Format: `- **YYYY-MM-DD** | Referenced in [page title](path) -- context`

An unlinked mention is a broken brain.

## Notability Gate

Before creating a new brain page, check notability:

- **Goals:** Is this a distinct execution arc worth documenting? (Not a sub-step of an existing goal)
- **Decisions:** Does this choice govern future work beyond the current goal?
- **Processes:** Is this repeatable and handoff-worthy? (Not a one-off sequence)
- **Concepts:** Reusable across goals? Stable? Non-procedural? (If it's steps, it's a process)

When in doubt, capture in the current goal page first. Promote to its own page
only when reuse is clear. A missing page can be created later. A junk page
wastes attention and degrades search quality.
