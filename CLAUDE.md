# CLAUDE.md

Navigation index and quick reference for working with this project.

## Rules of engagement

Collaboration principles and ways of working: @.claude/CLAUDE.md
When asked to remember anything, add project memory in this CLAUDE.md (project root), not @.claude/CLAUDE.md.

## Project overview

**Simmer** - an email service that gets you writing something most days. A question arrives by email before bed; you reply to it in the morning.

Tagline: *A question at night. Your reflection by morning.* The name: a question simmers overnight and you write toward a conclusion in the morning.

**Core workflow:**
1. The user emails simmer@hultberg.org; Simmer asks "What are you simmering?" and stores the answer as the **brief**
2. Each evening, Claude generates a question from the brief and Simmer emails it ("let it simmer")
3. Each morning, Simmer replies in the same thread restating the question; the user replies with their piece (or `stuck` for a nudge)
4. Simmer sends a receipt with one specific sentence about the thinking

**Source of truth:**
- [project-outline.md](./SPECIFICATIONS/ORIGINAL_IDEA/project-outline.md) - what Simmer is and why
- [MVP-IDEA.md](./SPECIFICATIONS/ORIGINAL_IDEA/MVP-IDEA.md) - the detailed build target
- [POTENTIAL-FUTURE-ELABORATION.md](./SPECIFICATIONS/ORIGINAL_IDEA/POTENTIAL-FUTURE-ELABORATION.md) - parked larger design; not a spec

## Architecture overview

**Stack:**
- **Runtime**: Cloudflare Workers + Cron Triggers
- **Database**: Cloudflare D1
- **Email**: Cloudflare Email Service on hultberg.org. Inbound: an Email Routing address rule sends simmer@hultberg.org to the Worker's `email()` handler. Outbound: `send_email` binding, plain text only, with `In-Reply-To`/`References` for threading
- **Model**: Anthropic API, Claude Opus 5.5 (`claude-opus-5-5`)
- **Web**: none in the MVP; any later web part is a Worker at simmer.hultberg.org

**Cloudflare account:** M Hultberg. hultberg.org already has Email Routing (catch-all forwards to the author's Gmail, which is a verified destination) and Email Sending enabled. Wrangler is logged in locally.

**Current status:** planning complete; no code yet. The first task is the prompt-generation eval ([MVP-IDEA.md section 12](./SPECIFICATIONS/ORIGINAL_IDEA/MVP-IDEA.md#12-first-prompts-for-claude-code), prompt 2).

## Hard rules for any code that calls the model

From [MVP-IDEA.md section 5](./SPECIFICATIONS/ORIGINAL_IDEA/MVP-IDEA.md#5-hard-rules-for-the-agent-decided). These are constraints on code, not tone guidance. Enforce them in code (length limits, post-checks that reject output) wherever possible, not only in the system prompt.

1. **Never writes prose for the user.** A nudge is one question, one pointer to a sentence in the draft, or one counterexample. Hard `max_tokens`; a post-check rejects anything else.
2. **Engages with the thinking, not the style.** Argument, stakes, clarity of the idea. Never sentence quality, word choice or grammar.
3. **Questions, never topics.** Every question contains a tension or demands a stance. "Reflect on X" is forbidden.
4. **Specific, never generic.** Encouragement must name something in the piece. "Great work" is forbidden.
5. **Sessions, never streaks.** No consecutive-day counts and no mention of missed days, in any email or code path.
6. **The writing is the user's.** Never used to train models. Exportable on request as a Markdown zip.

Related invariants:
- Inbound mail is attributed to a session by `In-Reply-To`, never by date.
- Session status is derived from the event stream, never stored as truth. `raw_content` keeps the original inbound email.
- Question generation never waits for a reply.

## Implementation phases

Not yet defined. Phases go in `SPECIFICATIONS/` as numbered files based on [00-TEMPLATE-phase.md](./SPECIFICATIONS/00-TEMPLATE-phase.md); the suggested order is in [MVP-IDEA.md section 12](./SPECIFICATIONS/ORIGINAL_IDEA/MVP-IDEA.md#12-first-prompts-for-claude-code).

### SPECIFICATIONS/
- **Implementation phases** (numbered files) - active work-in-progress
- **ORIGINAL_IDEA/** - project outline, MVP build target, parked design
- **ARCHIVE/** - completed specs (move here when a phase is complete)

### REFERENCE/
How-it-works documentation for implemented features:
- [testing-strategy.md](./REFERENCE/testing-strategy.md) - testing philosophy and approach
- [environment-setup.md](./REFERENCE/environment-setup.md) - API keys and environment configuration (template, not filled in yet; see Secrets below)
- [troubleshooting.md](./REFERENCE/troubleshooting.md) - common issues and solutions
- [pr-review-workflow.md](./REFERENCE/pr-review-workflow.md) - how the review skills work
- [decisions/](./REFERENCE/decisions/) - Architecture Decision Records
- [TEMPLATE-UPDATES/](./REFERENCE/TEMPLATE-UPDATES/) - migration packets for template improvements

*Keep CLAUDE.md files short (<300 lines). Details go in separate reference files.*

## Code conventions

### File headers
```typescript
// ABOUT: Brief description of file purpose
// ABOUT: Key functionality or responsibility
```

### Naming
- Use the product's own words: `brief`, `session`, `prompt`, `piece`, `receipt`, `nudge`, `submission`
- TypeScript conventions: camelCase (variables), PascalCase (types)
- Avoid temporal references: no "new", "improved", "old"

### Comments
- Evergreen (describe what code does, not recent changes)
- Minimal (code should be self-documenting)
- Explain complex logic and non-obvious decisions

## Development workflow

**⚠️ CRITICAL: ALL CHANGES REQUIRE A FEATURE BRANCH + PR ⚠️**

**Step 0 (BEFORE making ANY changes):**
- [ ] On feature branch (not main)?
- [ ] If on main: create feature branch first

**Implementation steps:**
1. Create feature branch (feature/, fix/, refactor/, docs/)
2. Check SPECIFICATIONS/ for relevant specs
3. Review spec with **`/review-spec`** before starting non-trivial features
4. Implement with tests (run tests + type checking)
5. Open a PR on [mannepanne/simmer-writing](https://github.com/mannepanne/simmer-writing) and review it:
   - **`/review-pr`** - triages the change and routes to light / standard / team review
   - **`/review-pr-team`** - forces the full four-perspective review
   - **See:** [pr-review-workflow.md](./REFERENCE/pr-review-workflow.md)

## TypeScript configuration

Not set up yet. Expected: TypeScript in strict mode with `@cloudflare/workers-types`.

## Testing

Tests serve dual purpose:
1. **Validation** - verify code works
2. **Directional context** - guide AI development

Commands are not set up yet.

**Coverage target:** 100% (enforced minimums: 95% lines/functions/statements, 90% branches)

**See:** [testing-strategy.md](./REFERENCE/testing-strategy.md)

## Secrets

- `ANTHROPIC_API_KEY` lives in `.dev.vars` locally (gitignored) and as a Worker secret in production. Never in the repo or in chat.
- Email needs no API key: sending uses the `send_email` binding.

## Quick reference links

- **Project outline** → [project-outline.md](./SPECIFICATIONS/ORIGINAL_IDEA/project-outline.md)
- **Build target** → [MVP-IDEA.md](./SPECIFICATIONS/ORIGINAL_IDEA/MVP-IDEA.md)
- **Environment setup** → [environment-setup.md](./REFERENCE/environment-setup.md) (template, not filled in yet)
- **Known issues / technical debt** → GitHub Issues with `technical-debt` label
- **Getting unstuck** → [troubleshooting.md](./REFERENCE/troubleshooting.md)
- **Architecture decisions** → [decisions/](./REFERENCE/decisions/)

## Project-specific notes

- **Prompt quality is the product.** The question-generation eval comes before any email plumbing.
- **Plain-text email only.** No HTML, images or tracking.
- **One user.** The allowlist holds the author's Gmail. Sending to anyone else needs the Workers Paid plan.
- **Commands by email:** `brief:`, `times:`, `weekdays:`, `pause`, `resume` on the first line; `stuck` on the last line. See [MVP-IDEA.md section 3](./SPECIFICATIONS/ORIGINAL_IDEA/MVP-IDEA.md#3-setup-decided).
