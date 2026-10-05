# Phase 1: Question generation and eval

## Phase overview

**Phase number:** 1
**Phase name:** Question generation and eval
**Dependencies:** none. Needs `ANTHROPIC_API_KEY` in `.dev.vars` before any paid run.

**Brief description:**
Build the code that turns a brief into the evening question, and an eval that shows whether those questions are good. Question quality is the product ([MVP-IDEA.md section 6](./ORIGINAL_IDEA/MVP-IDEA.md#6-prompt-generation-open-the-only-thing-that-matters)), so it comes before any email or database work. The eval centres on the author's own brief over 14 sessions, because the MVP's real test is whether the author still replies in week two ([section 9](./ORIGINAL_IDEA/MVP-IDEA.md#9-the-test-decided)). A few other briefs check that the prompt doesn't only work for one person.

---

## Scope and deliverables

### In scope
- [ ] `.gitignore` entries for the eval's private and output folders, in the branch's first commit, before any private brief is written to disk
- [ ] Project scaffold: `package.json`, TypeScript (strict), Vitest with istanbul coverage, `tsx`, the Anthropic TypeScript SDK (`@anthropic-ai/sdk`)
- [ ] The generator system prompt, as a versioned file in the repo
- [ ] A generator module: brief and history in, three candidates with critiques and a chosen question out, or a typed failure
- [ ] Deterministic checks on a question
- [ ] Unit tests for request assembly, output parsing, candidate selection and the checks, with the Anthropic client mocked
- [ ] The eval: case files, a runner, week-level checks, and a Markdown report for the author to read and mark
- [ ] Docs: commands in root `CLAUDE.md`; how to run the eval in `REFERENCE/question-generation.md`, listed in the REFERENCE index in root `CLAUDE.md` and `REFERENCE/CLAUDE.md`; status lines in `README.md` and root `CLAUDE.md`; the test setup in `REFERENCE/testing-strategy.md` (plain Vitest in phase 1, the Workers pool from phase 2)

### Out of scope
- Cloudflare Workers project, `wrangler.jsonc`, D1, the Workers test pool (phase 2)
- Receiving and sending email (phase 3)
- The evening email template and its framing around the question, including "let it simmer" (phase 4)
- Nudges and the receipt sentence (phase 5)
- A model-graded rubric judge, and comparing models. Both wait until there is a reason, such as inviting other users.
- Systematic prompt tuning against the eval. One or two revisions after the author's read are in scope.

### Acceptance criteria
- [ ] The author has approved every case: briefs, synthetic pieces and the questions they answer
- [ ] A full run on Claude Opus 5.5 is complete, and the author has marked every question for their own brief as *would reply*, *might* or *wouldn't*. The counts are recorded in the PR.
- [ ] The author has made a blind pick among the three candidates for each of their own sessions. How often it matches the generator's choice is recorded in the PR.
- [ ] Any prompt revisions, and why, are recorded in the PR with the `PROMPT_VERSION` each run used. After a revision, the author re-marks their own brief's questions; the other briefs are re-run and checked, not re-marked.
- [ ] Every chosen question passes the deterministic checks, or each failure is explained
- [ ] Unit tests pass with 95%+ lines, functions and statements and 90%+ branches on `src/`
- [ ] `npx tsc --noEmit` passes

---

## Technical approach

### Architecture decisions

**One generator module, used by both the eval and production**
- Choice: `generateQuestion(input, client)` is a plain TypeScript function with the Anthropic client passed in. No Node-only or Worker-only APIs; the prompt is a TypeScript string, not a file read at runtime.
- Rationale: the eval must exercise the code that will run in production, not a copy of the API call. The same function runs inside the Worker in phase 4.

**Three candidates, a critique each, then a choice, in one call**
- Choice: one request returns three candidate questions, a critique of each against the rules, and which one is chosen, as structured output.
- Rationale: MVP section 6 requires all three and the critique to be logged. One call is the cheapest design that does this. A separate critic call would double cost and latency for no measured gain; the author's blind pick tests whether the self-selection is any good.

**System prompt as a versioned file**
- Choice: the prompt lives in `src/generation/systemPrompt.ts` with a `PROMPT_VERSION` constant, bumped on any change to the text. Every result records it.

**"Let it simmer" belongs to the email, not the question**
- Choice: the generator returns only the question. The evening email template (phase 4) adds the framing and "don't answer this yet, let it simmer", as [MVP section 2](./ORIGINAL_IDEA/MVP-IDEA.md#2-the-loop-decided) describes.
- Rationale: one place owns the phrase, so it can never be printed twice, and it doesn't distort sentence and word counts.

**The generator never retries**
- Choice: one call per invocation. Retrying on rate limits or failures is the caller's job: the eval runner in this phase, the scheduled send in phase 4.
- Rationale: retries in two layers multiply silently and break cost estimates.

### Model and API rules

These come from the Claude API skill's documentation of Claude Opus 5.5. Confirm each against the API docs when writing the code.

- Model `claude-opus-5-5`. Thinking cannot be turned off on this model; never send a disabled-thinking setting.
- Set `output_config.effort` explicitly (its default on this model is `medium`). The run uses `medium` and records it.
- Return the candidates through structured outputs (`output_config.format` with a JSON schema). No assistant prefill and no forced `tool_choice`; this model rejects both.
- The schema uses three named candidate fields (`candidateA`, `candidateB`, `candidateC`) and a `chosen` field of `"A"`, `"B"` or `"C"`, because a schema can't enforce an array of exactly three. Enum values such as `kind` are lower-cased in code before validation.
- Read the answer from the text block, selected by type. With thinking on, other blocks come first.
- Check `stop_reason` before parsing. `refusal` and `max_tokens` become typed failures, never a question.
- `max_tokens` is set from the pilot's measured usage, since thinking tokens count against it, and recorded per run.
- **Refusal fallbacks.** Production calls (from phase 4) enable the API's server-side refusal fallback, so a refusal doesn't leave the user without a question. The eval turns it off and fails any case where the model that answered differs from the model requested. Confirm the parameter and the response field that names the answering model against the API docs when writing this code.
- Brief and piece text is data, not instructions. The system prompt says so, and user content goes in clearly delimited fields.

### The brief

A brief is one text, of any length. It may have two parts: a **core** (what the user is simmering, in their own words) and **background** (supporting material). The generator prompt treats them differently: questions orbit the core, and the background supplies specifics and depth. A brief with no background is all core. A brief of "I don't know yet" lets the generator range across anything else the reply mentions.

Questions are written in the language of the brief.

### Generator input

- `brief` and `briefId`
- `today`: the user's local date, so a question can ask for a prediction with a date on it
- `sessionNumber`: sessions sent under any brief, counting unanswered ones. Not calendar days, so `weekdays:` doesn't distort it.
- `repliesReceived`: how many sessions have a submission
- `recentPrompts`: up to the last 14 questions, each with its `kind`, its `orbit` and whether it was answered
- `recentPieces`: up to the last 5 pieces, each with session id, date and text
- `latestPieceSessionId`: the newest piece that arrived before generation, or none

### Rules the generator follows

From [MVP sections 4 and 6](./ORIGINAL_IDEA/MVP-IDEA.md#4-the-first-week-decided):

- **Sessions 1–3:** easy, close to the core, answerable from what the user already thinks.
- **Sessions 4–7:** include one question with real tension and one change of register. If `repliesReceived` is 0 by session 4, stay easy instead.
- **Session 8 onward:** roughly half close to the core, a quarter at its edges, a quarter a change of register, judged over the last 14 prompts.
- **Follow-ups:** only on `latestPieceSessionId`, and only when that piece left a thread worth pulling. With no new piece, the question is independent.
- Vary `kind` without announcing it. Never repeat or near-repeat a recent prompt.
- A changed brief doesn't reset `sessionNumber`.

### Generator output

A result with a `status`:

- `ok`: the chosen question's `text`, `kind`, `orbit` (`close`, `edge` or `register`) and `followsUpSessionId`; all three candidates with their critiques; which was chosen and why; and `promptVersion`, `briefId`, `model`, `effort`, `usage`, `stopReason`
- `refusal`, `max_tokens` or `invalid_output`: the raw response details, for logging
- `check_failed`: no candidate passed the deterministic checks; all three are included

**Selection:** if the model's chosen candidate fails a check, the generator takes the first other candidate that passes, records the switch, and returns `ok`. If none pass, it returns `check_failed`.

### Deterministic checks

- One to three sentences, split by a rule written down as tested examples (question marks, abbreviations such as "e.g.", ellipses, quoted speech, dates)
- At most 60 words
- No URL
- Does not contain "reflect on" (case-insensitive)
- Does not contain "let it simmer" (the email adds it)
- `followsUpSessionId` is null or equals `latestPieceSessionId`
- `kind` and `orbit` are from their allowed sets

These checks cover shape only. Hard rule 3 (questions, never topics) beyond the "reflect on" phrase rests on the model's own critique and the author's read.

### Key files and components

```
src/generation/
  systemPrompt.ts          # prompt text + PROMPT_VERSION
  generateQuestion.ts      # the generator
  checks.ts                # deterministic checks and sentence splitting
  types.ts                 # input and result types
tests/generation/
  generateQuestion.test.ts # mocked client
  checks.test.ts
eval/question-generation/
  cases/                   # public synthetic cases (committed)
  private/                 # the author's briefs (gitignored)
  runs/                    # all run output (gitignored)
  run.ts                   # the runner, executed with tsx
  weekChecks.ts            # week-level checks
  report.ts                # writes the Markdown report
```

---

## The eval

### Cases

Each case file holds a brief and a schedule of sessions. A session entry may supply a synthetic piece for the previous session; when it does, it also pins the question that piece answers, so the piece and the history agree.

- **The author's book, 14 sessions.** Core plus background ("Memory Scam"). Shown first in the report.
- **The author's book, core only, 14 sessions.** Same prompt and settings as the full-brief run. The author marks both side by side; the result decides whether the setup email invites long briefs.
- **Four or five other briefs, 7 sessions each**, based on the author's realistic examples. At least one is "I don't know yet" and at least one is short.
- **Piece coverage per brief:** some sessions with a piece that leaves a thread, some with a flat piece that doesn't, and some with no piece. The eval checks that the generator follows up on the first kind and stays independent on the others.
- **One injection case:** a piece containing an instruction to the model, to check it is treated as data.

The author reads and approves every case, including the synthetic pieces and pinned questions, before the full run.

### Running it

- `npx tsx --env-file=.dev.vars eval/question-generation/run.ts`
- Sessions within a brief run in order; up to three briefs run at once.
- Results are stored per (brief, session) as each finishes. If a session fails after the runner's retries, that brief stops and a re-run resumes from the failed session.
- The runner retries rate limits and overloads with jittered backoff, caps attempts, and records retries. Failures are recorded by type: refusal, max tokens, invalid output, check failed, timeout, API error.
- Fallbacks off; any answer from a different model than requested fails the session.
- The report includes the switch rate: how often the generator replaced the model's chosen candidate because it failed a check.
- **Pilot first:** two sessions of the author's brief. The full run's cost is estimated from the pilot's measured usage, with the arithmetic shown, and approved by the author before it starts.

### Grading

1. **Deterministic checks** on every candidate.
2. **Week-level checks** per brief: at least three different `kind` values in any 7 sessions; tension and a register change present in sessions 4–7 (unless no replies); the `orbit` mix from session 8 onward; follow-ups present after thread pieces and absent after flat or missing pieces; no near-repeats, flagged for the author's read.
3. **The author's read.** The Markdown report lists, per brief and session: the chosen question, the two rejected candidates and all three critiques. For the author's own brief, the candidates are shown unlabelled first, for the blind pick, then with the generator's choice. The author marks each of their own questions *would reply*, *might* or *wouldn't*. Those marks are the verdict.

---

## Testing strategy

### Unit tests
- Request assembly: model, effort, schema, no prefill, no forced tool choice, user text delimited as data
- Parsing: valid output; thinking block before the text block; malformed JSON; wrong enum casing; `refusal`; `max_tokens`
- Selection: chosen candidate passes; chosen fails and another passes; none pass
- Follow-up guard against session ids other than `latestPieceSessionId`
- Each deterministic check, with the sentence-splitting examples
- Week-level checks on fixed inputs
- No test calls the real API

### Manual testing checklist
- [ ] One live call with the author's brief returns three candidates and a choice
- [ ] The pilot report shows candidates, critiques and choices correctly
- [ ] The full report is read and marked by the author

---

## Pre-commit checklist

- [ ] `npm test` passes with coverage thresholds met
- [ ] `npx tsc --noEmit` passes
- [ ] Nothing from `eval/question-generation/private/` or `runs/`, and no key, is staged (`git status`)
- [ ] Docs updated (root `CLAUDE.md` commands and status, `README.md` status, `REFERENCE/question-generation.md`)

---

## PR workflow

- Branch: `feature/question-generation`
- Review with `/review-pr`; the prompt is the heart of the product, so consider `/review-pr-team`.
- The PR description includes the author's marks, the blind-pick agreement, the measured cost of each run, and any prompt revisions.

---

## Edge cases and considerations

### Known risks
- **The prompt suits the author's brief only.** The other briefs, especially "I don't know yet", are the guard.
- **Questions converge on one style.** The week-level variety checks and the author's read catch it.
- **Repeats beyond the last 14 prompts go unseen.** The generator only sees 14, so a 30-session run can repeat an early question. Acceptable for the MVP; revisit if the author notices.
- **Long background pulls questions into side details.** The core/background rule addresses it; the core-only comparison measures it.

### Security considerations
- The author's book material and other private briefs stay in gitignored folders and never reach the public repository.
- Brief and piece text is treated as data in the generator prompt; the injection case tests it.

---

## Decisions for the author

1. **Other briefs:** two or three realistic examples, in chat, to base the other cases on.

Decided: the core-only comparison for the book brief is included.

---

## Related documentation

- [MVP-IDEA.md](./ORIGINAL_IDEA/MVP-IDEA.md) sections 2, 4, 5, 6, 9 and 12
- [testing-strategy.md](../REFERENCE/testing-strategy.md) - tests versus the eval
- [environment-setup.md](../REFERENCE/environment-setup.md) - the API key
