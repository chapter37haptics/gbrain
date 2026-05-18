---
status: complete
branch: customized-domain-superpowers
base_branch: chapter37haptics/customized-domain
timestamp: 2026-05-15T21:58:00Z
pr: https://github.com/chapter37haptics/gbrain/pull/5
approach: superpowers (writing-plans + subagent-driven-development)
files_modified: []
---

## Working on: Customized Domain (VC to Developer) via Superpowers

### Summary

Implemented the customized-domain spec using the superpowers skill chain. All code shipped, reviewed by Codex (GATE: PASS), tests improved with realistic examples, PR #5 open with 4 review comments posted. Clean working tree, 16 commits total.

### Approach Used

**Superpowers workflow:**
1. `/writing-plans` read the spec, read all 12 target files, produced a detailed 11-task plan with exact code, file paths, and test steps at `docs/superpowers/plans/2026-05-15-customized-domain.md`
2. `/subagent-driven-development` executed the plan:
   - Tasks 1-3 (code: types.ts, markdown.ts, link-extraction.ts): dispatched as sonnet subagents with TDD (write test, verify fail, implement, verify pass, commit)
   - Tasks 4-9 (skill files: quality.md, brain-ops, signal-detector, filing-rules, RESOLVER, brain-first): executed inline as direct edits since content was fully specified in the plan
   - Task 10 (verification): ran typecheck + 485 targeted tests (0 failures) + PageType consumer audit
   - Task 11 (doctor.ts): one-line SQL clause expansion
3. `/verification-before-completion` ran fresh typecheck and tests before claiming done
4. `/gstack-codex review` got an independent Codex review (GATE: PASS, 4 findings)
5. `/receiving-code-review` evaluated each finding against the codebase
6. Second `/gstack-codex` consult asked Codex to grade Claude's dismissals (1 VALID, 3 PARTIAL)
7. Fixed finding #2 (added `concept` to doctor.ts graph_coverage) based on Codex feedback
8. Third `/gstack-codex` consult on test plan inspired by real email-parsing spec
9. Fourth `/gstack-codex` audit: spec-vs-tests gap analysis, pinned ALL_PAGE_TYPES count at 25

### Decisions Made

- **Code-first execution order**: types.ts before markdown.ts before link-extraction.ts, so skill file changes could be tested immediately without transient type errors
- **Conservative RESOLVER.md rewrite**: kept all quoted trigger phrases unchanged because the resolver test (D5/C) fuzzy-matches them against skill frontmatter, and the underlying skills (query, enrich, data-research) were NOT modified
- **Singular/plural DIR_PATTERN**: only added plural forms (goals, decisions, processes) matching the existing codebase pattern where inferType accepts singulars as fallbacks but DIR_PATTERN only has plurals
- **llms-full.txt rebuild**: the `build:llms` CI check caught stale generated output after skill file changes; rebuilt and committed
- **gh repo set-default**: set after accidentally posting a PR comment to the upstream repo (garrytan/gbrain) instead of the fork (chapter37haptics/gbrain) due to `gh` CLI ambiguous remote resolution
- **Filing rules JSON is additive-only**: the JSON is a registry of valid directories (infrastructure), the MD is the behavioral layer (replaceable). Never delete from JSON.
- **Added concept to doctor.ts**: Codex cross-model review correctly flagged that touching the query and leaving concept out was a missed opportunity
- **Pinned ALL_PAGE_TYPES at 25**: Codex spec audit flagged the contract test didn't pin the count

### What Shipped (16 commits)

| Commit | Change |
|--------|--------|
| 91af5a7 | types.ts: goal, decision, process added to PageType |
| bd27f60 | markdown.ts: inferType() directory mappings |
| bc11839 | link-extraction.ts: DIR_PATTERN expansion |
| ba00bdf | quality.md: generalized Iron Law + notability |
| 661ed79 | brain-ops: 8 hard-gate sites patched |
| cc163ff | signal-detector: full rewrite |
| 3b7fb1c | filing rules: developer taxonomy |
| 3fff873 | RESOLVER.md: disambiguation updated |
| ba75466 | brain-first.md: entity conventions |
| 8a69e92 | llms-full.txt: rebuilt |
| d490a6f | doctor.ts: developer types in graph_coverage |
| 5856aa2 | doctor.ts: concept added (Codex finding) |
| b0894ab | superpowers plan file |
| ab93d6c | session context saved |
| 7c700c5 | realistic developer-entity tests |
| 05c9a66 | ALL_PAGE_TYPES count pinned at 25 |

### PR Review Comments Posted

1. [Codex review](https://github.com/chapter37haptics/gbrain/pull/5#issuecomment-4463344835): GATE PASS, 4 findings
2. [Claude's evaluation](https://github.com/chapter37haptics/gbrain/pull/5#issuecomment-4463503409): dismissed all 4
3. [Codex second opinion](https://github.com/chapter37haptics/gbrain/pull/5#issuecomment-4463505987): graded dismissals, finding #2 fixed
4. [Codex spec audit](https://github.com/chapter37haptics/gbrain/pull/5#issuecomment-4463798644): spec-vs-tests gap analysis

### Verification Evidence

- `bun run typecheck`: exit 0
- 497 tests across 8 affected test files: 0 failures (485 original + 10 new + 2 new assertions)
- `bun run verify` (CI pre-test gate): exit 0
- Full `bun run test` (8-shard parallel): exit 0
- Codex review: GATE PASS, 4 findings (1 fixed, 3 dismissed with justification)

### Known Limitations

1. **Singular/plural DIR_PATTERN**: inferType accepts `/goal/` but DIR_PATTERN only has `goals`. Pre-existing pattern (same as person/people). Low risk.
2. **inferLinkType**: classifies developer entity relationships as `mentions` (default fallback). Tier 2 follow-up.

### Remaining Work

1. **Merge PR #5** into `chapter37haptics/customized-domain`
2. **Compare approaches**: this was the superpowers approach. User plans to try GSD and other approaches on separate branches for comparison
3. **Follow-up (Tier 2)**: developer-specific inferLinkType heuristics
4. **Follow-up (devcontainer)**: update entrypoint.sh in practicespace-2 to load the rewritten skill files (out of scope for this spec, separate repo)

### Lessons Learned

- `gh` CLI in a fork with both `origin` and `upstream` remotes will ambiguously resolve. Always run `gh repo set-default <fork>` after cloning a fork. Detect forks with `gh repo view --json isFork,parent`.
- `bun run test` (npm script) includes pre-checks like `build:llms` that catch stale generated files. Changing CLAUDE.md or skill files requires `bun run build:llms` before the test suite passes.
- The superpowers subagent-driven-development skill worked well for the 3 code tasks but was overhead for the 6 skill file tasks where content was fully specified. Inline execution was faster for those.
- Codex CLI `codex review` doesn't support `--base` with `[PROMPT]` simultaneously. Use `codex exec` with embedded diff for custom-prompt reviews.
- Codex sandbox (bwrap) doesn't work in this devcontainer. Embedding the diff in the prompt works around it.
- llms-full.txt large deletion was from CLAUSE.md key-files section trimmed in an earlier commit, not caused by this PR.
- Filing rules JSON is additive-only (infrastructure registry); filing rules MD is replaceable (agent behavior). Different contracts.
