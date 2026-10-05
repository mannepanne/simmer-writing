# Testing strategy

**When to read this:** writing tests, setting up coverage, or working on the question-generation eval.

**Related documents:**
- [CLAUDE.md](../CLAUDE.md) - project navigation index, including the hard rules for model-calling code
- [.claude/CLAUDE.md](../.claude/CLAUDE.md) - collaboration principles (testing section)
- [pr-review-workflow.md](./pr-review-workflow.md) - PR review process
- [MVP-IDEA.md](../SPECIFICATIONS/ORIGINAL_IDEA/MVP-IDEA.md) - the behaviour the tests specify

Test tooling is not set up yet. This document describes the approach the first build phases follow.

---

## Two kinds of checking

Simmer has deterministic code (parsing email, classifying replies, attributing sessions, scheduling) and model output (questions, nudges, receipt sentences). They are checked differently.

| | Deterministic code | Model output |
|---|---|---|
| Checked by | Unit and integration tests | The prompt eval, plus post-checks in production code |
| Runs | Every commit and PR | On demand, when prompts or the model change |
| Pass means | Exact expected behaviour | Passes the checks, and the author would reply to it |
| Calls the Anthropic API | Never; the client is mocked | Yes |

---

## Principles

Tests serve two purposes: **validation** (the code works) and **directional context** (they tell anyone, human or AI, what the code is supposed to do).

1. **Tests are specifications.** Write the failing test first, from the MVP doc, then the code.
2. **High coverage.** 95%+ lines, functions and statements; 90%+ branches. A gap usually means a missing specification.
3. **Clear failures.** A failing test says what was expected, what happened, and where to look.
4. **Structure mirrors the code.** `src/email/classify.ts` is tested by `tests/email/classify.test.ts`.
5. **Self-contained.** Each test sets up its own data and runs in isolation.

---

## Framework

- **Runner:** [Vitest](https://vitest.dev/), with Cloudflare's Workers integration so tests run in the Workers runtime with D1 and bindings available
- **Coverage:** Vitest coverage, with the thresholds above enforced in config
- **Type checking:** `tsc --noEmit`, run alongside the tests

Commands are recorded in [CLAUDE.md](../CLAUDE.md) once the project is scaffolded.

---

## What must be tested

These are the behaviours most likely to break silently. Each needs tests before it ships.

### Inbound email
- **Quote stripping**, against real reply samples from Gmail, Apple Mail, Outlook and at least one mobile client, kept as fixtures in `tests/fixtures/email/`. `raw_content` is always kept so the stripper can be re-run.
- **Classification order:** command on the first line (`brief:`, `times:`, `weekdays:`, `pause`, `resume`), then `stuck` on the last line, then submission.
- **Edge cases:** `stuck` mid-text is just a word; "Pause for a moment…" is a submission; a quoted earlier `stuck` never triggers; the reply to the setup question becomes the first brief.
- **Allowlist:** mail from any other sender is dropped with no reply.

### Sessions
- **Attribution by `In-Reply-To`**, never by date, including replies days late.
- **Append rule:** later submissions join the piece; only the first fires a receipt.
- **Derived status:** not started, in progress and submitted come from events, never stored.
- **Word floor:** a reply under 150 words gets a receipt but doesn't count as a session.

### Scheduling
- Evening and morning sends at the right local time across timezones and daylight-saving changes.
- `weekdays:` skips inactive days.
- Pausing: the automatic pause after five unanswered sessions lifts on any reply; a manual pause lifts only on `resume`.
- The day-4 no-reply email is sent once.

### Hard rules in code
- **Rule 1 post-check:** a nudge that isn't a question, a pointer to a sentence, or a counterexample is rejected.
- **Rule 4 post-check:** a receipt sentence with generic praise is rejected.
- **Rule 5:** no email template or code path can produce a streak or missed-day count. Test the rendered emails, not just the functions.

---

## Mocking

Mock the Anthropic API client and the `send_email` binding. Tests assert on what would be sent, not on what was delivered.

Don't mock Simmer's own logic: classification, attribution, scheduling and event derivation run for real against a test D1 database.

---

## The prompt eval

The eval checks question quality, which unit tests cannot. It is the first build task ([MVP-IDEA.md section 12](../SPECIFICATIONS/ORIGINAL_IDEA/MVP-IDEA.md#12-first-prompts-for-claude-code), prompt 2).

- **Input:** the author's own brief over 14 sessions, plus four or five other briefs over 7 sessions, including an "I don't know yet" brief. Private briefs stay in gitignored folders.
- **Grade:** deterministic checks on every question, week-level checks (variety, the first-week rules, follow-ups), and the author's read. The author marks their own questions *would reply*, *might* or *wouldn't*; that is the verdict.
- **Keep:** every candidate and critique is logged, with the prompt version, so prompt changes can be compared.
- **Not yet:** a model-graded judge and model comparisons. They wait until there is a reason, such as other users.

Details are in [01-question-generation.md](../SPECIFICATIONS/01-question-generation.md).

The eval calls the real API, so it never runs in CI or on every commit.

---

## Before every commit

Once the project is scaffolded:

```bash
npm test
npx tsc --noEmit
```

Both must pass. PRs must keep coverage at or above the thresholds.
