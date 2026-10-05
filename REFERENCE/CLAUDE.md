# Reference documentation library

Auto-loaded when working with files in this directory. How-it-works documentation for implemented features.

## Files in this directory

### [testing-strategy.md](./testing-strategy.md)
**When to read:** Writing tests, setting up test coverage, or working on the question-generation eval.

Tests for deterministic code (email parsing, classification, attribution, scheduling, hard-rule post-checks) versus the prompt eval for model output; Vitest in the Workers runtime; coverage targets.

### [environment-setup.md](./environment-setup.md)
**When to read:** Setting up local development, adding or rotating a secret, or changing the Cloudflare email configuration.

Accounts, the `ANTHROPIC_API_KEY` secret, Wrangler login, and the hultberg.org Email Routing and Email Sending setup for simmer@hultberg.org.

### [troubleshooting.md](./troubleshooting.md)
**When to read:** Mail not arriving or not sending, Claude API errors, Wrangler problems, failing tests.

Known problems and fixes, grouped by area. Add an entry whenever a problem takes more than a few minutes to solve.

### [pr-review-workflow.md](./pr-review-workflow.md)
**When to read:** Starting a new feature, creating PRs, or running any kind of review.

How to use `/review-spec` (pre-implementation), `/review-pr`, and `/review-pr-team` skills.

### [profile-setup-hook.md](./profile-setup-hook.md)
**When to read:** The profile setup prompt fired and you want to understand what's going on, you want to suppress or disable the hook, or you want to extend the setup flow.

What the hook detects, the `.local` file pattern, how the signal is consumed by Claude, test command, extension points, and removal steps. Decision rationale at [`decisions/2026-05-24-personal-profile-setup-sentinel.md`](./decisions/2026-05-24-personal-profile-setup-sentinel.md).

### [safety-harness.md](./safety-harness.md)
**When to read:** A safety-harness block or ask dialog fired and you want to understand what's going on, you want to add a pattern, or you want to bypass the hook for a legitimate use.

What's caught at block / ask tier, what's deliberately not caught, how the inline `SAFETY_HARNESS_OFF=1` bypass works (and its limits), how the hook composes with the allowlist, how to extend patterns + tests.

### [scratch-write-hook.md](./scratch-write-hook.md)
**When to read:** Reviewing or extending the `Write` auto-approval for `<project>/SCRATCH/`, debugging a SCRATCH/ Write prompt that fired unexpectedly, or removing the hook if upstream Claude Code fixes the underlying matcher.

What the hook approves and why, where it sits in the call path alongside `safety-harness.sh`, what's deliberately out of scope (symlinks, exotic filenames), how to extend, and the rollback path if the upstream defect is fixed. Decision rationale at [`decisions/2026-04-26-scratch-write-pretooluse-hook.md`](./decisions/2026-04-26-scratch-write-pretooluse-hook.md).

### [decisions/](./decisions/)
**When to read:** Making architectural decisions, choosing between alternatives, or looking up why something was built the way it was.

Architecture Decision Records (ADRs) — permanent log of significant technical choices, alternatives considered, and trade-offs accepted.

### [TEMPLATE-UPDATES/](./TEMPLATE-UPDATES/)
**When to read:** Rolling out a template improvement to a derivative project, or formalising a new improvement so future derivative projects can adopt it.

Migration packets that describe template changes in conceptual terms plus a file manifest, so a Claude in another project can compare against local files and apply the improvement without clobbering customisations.
