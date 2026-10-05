# Implementation specifications library

Auto-loaded when working with files in this directory. Forward-looking plans for features being built.

## Purpose of this folder

The SPECIFICATIONS folder contains **forward-looking plans** for features you're actively building. These are living documents that guide development and evolve as you learn more.

### Key principles

1. **Specifications are active work** - They describe what you're building *now* or *next*
2. **One phase at a time** - Focus on clear, sequential implementation phases
3. **Move completed specs to ARCHIVE/** - Keep this folder focused on current/upcoming work
4. **Reference ORIGINAL_IDEA/** - Link back to master vision for context

## How to structure implementation phases

Break your project into numbered sequential phases (e.g., 01-foundation.md, 02-authentication.md, etc.).

### What each phase file should include

1. **Phase overview**
   - Phase number and name
   - Brief description
   - Estimated timeframe
   - Dependencies on previous phases

2. **Scope and deliverables**
   - What will be built in this phase
   - What's explicitly out of scope
   - Acceptance criteria

3. **Technical approach**
   - Architecture decisions (document significant choices as ADRs in REFERENCE/decisions/)
   - Technology choices (check existing ADRs for precedent before deciding)
   - Key files and components
   - Database schema changes (if applicable)

4. **Testing strategy**
   - Unit test requirements
   - Integration test requirements
   - Coverage targets
   - Manual testing checklist

5. **Pre-commit checklist**
   - [ ] All tests passing
   - [ ] Type checking passes
   - [ ] Coverage meets targets
   - [ ] Manual verification complete
   - [ ] Documentation updated

6. **PR workflow**
   - Branch naming convention
   - PR review requirements
   - Deployment steps

7. **Edge cases and considerations**
   - Known risks or challenges
   - Alternative approaches considered
   - Future optimization opportunities

### Example phase structure

See [00-TEMPLATE-phase.md](./00-TEMPLATE-phase.md) for a complete example.

## Supporting folders

### ORIGINAL_IDEA/

Store your initial project concept documents here:
- Master specification and product vision
- Naming rationale and inspiration
- Early brainstorming and requirements
- Competitive analysis or market research

These documents are the "source of truth" for the project's intent and typically don't change during implementation.

### ARCHIVE/

Move completed phase files here after:
1. Phase implementation is complete
2. PR is merged to main
3. Features are deployed/verified

Archive serves as historical record. For current implementation details, see `REFERENCE/` documentation instead.

## Workflow

1. Break the next piece of work into a numbered phase file (e.g. `01-prompt-eval.md`)
2. Review it with `/review-spec` before writing code
3. Work through phases in order
4. Move completed specs to `ARCHIVE/`
5. Write how-it-works docs in `REFERENCE/` for what was built

**Current phase tracking:** update the "Current phase" line in both the root `CLAUDE.md` and this file.

---

## Simmer's implementation phases

**Current phase:** 1, question generation and eval ([01-question-generation.md](./01-question-generation.md)), specified, not started.

Six phases. Question quality comes first; real email works from phase 3; the author starts using Simmer for real after phase 5.

1. **[Question generation and eval](./01-question-generation.md).** Project scaffold, the generator system prompt and module, deterministic checks, and an eval centred on the author's own brief over 14 sessions, with a few other briefs over 7. Ends with the author's marks on their own questions. Also updates the status lines in `README.md` and root `CLAUDE.md`.
2. **Data model.** Workers project and `wrangler.jsonc`, D1 with migrations for `user`, `brief`, `prompt`, `session` and `session_event`, the derived session view, and the Workers test pool. Tests for attribution by `In-Reply-To` and the append rule.
3. **Email in and out, first deploy.** The `email()` handler: allowlist, quote stripping against real client samples, classification, commands (`brief:`, `times:`, `weekdays:`, `pause`, `resume`) with quoted-back confirmations, and the setup exchange. Outbound sending with threading headers, since setup and confirmations are emails. First deploy and the simmer@ routing rule, so the setup exchange works with real Gmail. Updates `REFERENCE/environment-setup.md` and `REFERENCE/troubleshooting.md`.
4. **The daily loop.** Cron Triggers for evening and morning sends in each user's timezone, `weekdays:`, generation from phase 1 with fallbacks on, the receipt (word count, session number, week at a glance), the word floor, the day-4 no-reply email, and the automatic and manual pauses.
5. **The reader.** The `stuck` nudge and the receipt's one sentence, each with a post-check in code for hard rules 1 and 4. The author starts using Simmer daily at the end of this phase.
6. **Export and hardening.** Markdown export of all pieces (hard rule 6), logging of unanswered sessions as a tuning signal, a check of every email template against rule 5, and a docs sweep.

The week-two test and the thirty-session test ([MVP-IDEA.md section 9](./ORIGINAL_IDEA/MVP-IDEA.md#9-the-test-decided)) happen after the build; they are not phase deliverables.

Each phase gets a numbered spec based on [00-TEMPLATE-phase.md](./00-TEMPLATE-phase.md) when it is next up, reviewed with `/review-spec` before work starts.

### Supporting documentation

**[ORIGINAL_IDEA/](./ORIGINAL_IDEA/)**
- `project-outline.md` - what Simmer is and why
- `MVP-IDEA.md` - the detailed build target
- `POTENTIAL-FUTURE-ELABORATION.md` - the parked larger design; a menu for later, not a spec

**[ARCHIVE/](./ARCHIVE/)** - completed specifications (moved here when a phase is done)

**[REFERENCE/decisions/](../REFERENCE/decisions/)** - Architecture Decision Records. Search here before making architectural decisions.

## When specs move to archive

After completing a phase and merging the PR:
1. Move the phase file to `ARCHIVE/`
2. Update implementation docs in `REFERENCE/` if needed
3. Update this index to reflect current phase
