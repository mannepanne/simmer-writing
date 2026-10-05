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

**Current phase:** none started. Phases are not defined yet.

The suggested order comes from [MVP-IDEA.md section 12](./ORIGINAL_IDEA/MVP-IDEA.md#12-first-prompts-for-claude-code). Question quality comes before any plumbing:

1. Prompt-generation system prompt and eval. The first phase that adds code, so it also scaffolds the project and updates the status lines in `README.md` and root `CLAUDE.md`
2. D1 data model, migrations and the derived session view
3. Inbound `email()` handler: allowlist, quote stripping, classification, commands, attribution. Creates the simmer@ routing rule, so this phase also updates `REFERENCE/environment-setup.md` and `REFERENCE/troubleshooting.md`
4. Evening and morning sends, receipts, the day-4 email and pausing
5. Nudge responder and receipt sentence, with post-checks for hard rules 1 and 4

Each becomes a numbered phase file (`01-…md`) based on [00-TEMPLATE-phase.md](./00-TEMPLATE-phase.md) when it is planned.

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
