# Phase 1: Question generation and eval

## Phase overview

**Phase number:** 1
**Phase name:** Question generation and eval
**Dependencies:** none. Needs `ANTHROPIC_API_KEY` in `.dev.vars` before any paid run.

**Brief description:**
Build the code that turns a brief into the evening question, and an eval that shows whether those questions are good. Question quality is the product ([MVP-IDEA.md section 6](./ORIGINAL_IDEA/MVP-IDEA.md#6-prompt-generation-open-the-only-thing-that-matters)), so it comes before any email or database work. The phase ends when the author has read a full week of generated questions for ten briefs and judged them.

---

## Scope and deliverables

### In scope
- [ ] Project scaffold: `package.json`, TypeScript (strict), Vitest, the Anthropic TypeScript SDK (`@anthropic-ai/sdk`)
- [ ] The generator system prompt, as a versioned file in the repo
- [ ] A generator module: brief and history in, three candidates with critiques and the chosen question out
- [ ] Deterministic checks on a question (shape rules that need no judgement)
- [ ] Unit tests for prompt assembly, output parsing and the deterministic checks, with the Anthropic client mocked
- [ ] The eval: case set, runner, grader, report
- [ ] A baseline run on Claude Opus 5.5, then a comparison run on Claude Sonnet 5.5 if the author wants one
- [ ] Docs: commands in root `CLAUDE.md`, how to run the eval in `REFERENCE/question-generation.md`, status lines in `README.md` and root `CLAUDE.md`

### Out of scope
- Cloudflare Workers project, `wrangler.jsonc`, D1, the Workers test pool (phase 2)
- Sending or receiving email (phases 3 and 4)
- Nudges and the receipt sentence (phase 5)
- Tuning the prompt against the eval over many rounds. One or two revisions after the first read are in scope; systematic hill-climbing is a later decision.

### Acceptance criteria
- [ ] The author has approved the eval's input briefs (gate 1)
- [ ] The author has approved the grading method after reading a graded pilot (gate 2)
- [ ] A full baseline run on Claude Opus 5.5 is complete, with a report the author has read
- [ ] The author's answer to "would these questions get you out of bed?" is recorded in the PR, along with any prompt revisions it led to
- [ ] Every generated question passes the deterministic checks, or each failure is explained
- [ ] Unit tests pass with 95%+ lines, functions and statements and 90%+ branches on `src/`
- [ ] `npx tsc --noEmit` passes

---

## Technical approach

### Architecture decisions

**One generator module, used by both the eval and production**
- Choice: `generateQuestion(input, client)` is a plain TypeScript function with the Anthropic client passed in. No Node-only or Worker-only APIs.
- Rationale: the eval must exercise the real code path, not a copy of the API call. The same function runs inside the Worker in phase 4.
- Alternatives considered: a separate eval-only prompt (rejected: the eval would measure something production never runs).

**Three candidates, a critique each, then a choice, in one call**
- Choice: one request returns three candidate questions, a critique of each against the rules, and the index of the chosen one, as structured output.
- Rationale: MVP section 6 requires all three and the critique to be logged; one call keeps cost and latency down. The eval grades the chosen question and also shows the other two, so the author can judge the selection.
- Alternatives considered: three generation calls plus a separate critic call (more cost, more code, no clear gain until the eval says otherwise).

**System prompt as a versioned file**
- Choice: the prompt text lives in `src/generation/systemPrompt.ts` with a `PROMPT_VERSION` constant. Every generated result records the version.
- Rationale: questions must be traceable to the prompt that wrote them, as they are to the brief.

### Model and API rules

These come from the current Claude API behaviour for Claude Opus 5.5 and apply to every call in this phase.

- Model `claude-opus-5-5`. Thinking cannot be turned off on this model; do not send a `thinking: disabled` setting.
- Set `output_config.effort` explicitly. The default on Claude Opus 5.5 is `medium`; the baseline uses `medium`, and the eval records the effort level per run.
- Return the candidates through structured outputs (`output_config.format` with a JSON schema), not "reply only in JSON" instructions. No assistant prefill (rejected on this model) and no forced `tool_choice` (rejected).
- Check `stop_reason` before reading content. `refusal` and `max_tokens` are recorded as their own outcomes, never parsed as a question.
- **Refusal fallbacks.** Production calls (from phase 4) enable server-side fallbacks (`fallbacks: "default"`), so a refusal is retried on another model instead of leaving the user with no question. The eval turns fallbacks **off** and fails any case where the model that answered differs from the model requested, so results always describe the model under test.
- Brief and piece text from users is data, not instructions. The system prompt says so, and user content is passed in clearly delimited fields.

### Generator input and output

**Input**
- `brief`: the current brief text
- `dayNumber`: days since setup (drives the first-week rules in [MVP section 4](./ORIGINAL_IDEA/MVP-IDEA.md#4-the-first-week-decided): days 1–3 easy and close to the brief; days 4–7 include one question with real tension and one change of register)
- `recentPrompts`: up to the last 14 questions sent, to avoid repetition
- `recentPieces`: up to the last 5 pieces, each with its session id, date and text
- `latestPieceSessionId`: the most recent piece that arrived before generation, or none

**Output**
- `candidates`: three items, each with `text`, `kind` (defend a claim, explain to someone, decide between options, predict, invent a scene, or other), `followsUpSessionId` (or null) and `critique`
- `chosenIndex` and a one-line `reason`
- `promptVersion`, `model`, `effort`, `usage`, `stopReason`

A follow-up may only reference a session id that appears in `recentPieces`. With no pieces, every candidate is independent: generation never waits for a reply.

### Deterministic checks

Applied to every candidate, in unit tests and in the eval:
- One to three sentences
- Ends with "let it simmer"
- Contains no URL
- Does not start with or contain "reflect on" (case-insensitive)
- Within a word limit (proposed: 80 words including the ending)
- `followsUpSessionId` is null or matches a piece in the input

### Key files and components

```
src/generation/
  systemPrompt.ts          # prompt text + PROMPT_VERSION
  generateQuestion.ts      # the generator
  checks.ts                # deterministic checks
  types.ts
tests/generation/
  generateQuestion.test.ts # mocked client: request shape, parsing, stop reasons
  checks.test.ts
eval/question-generation/
  cases/                   # public synthetic cases (committed)
  run.mjs                  # runner, adapted from the Claude API eval scaffold
  grade.ts                 # deterministic checks + rubric judge
  rubric.md                # the judge rubric, in plain words
```

Private cases and all run output go in gitignored directories (see decision 2 below), because the repository is public.

---

## The eval

### Cases
- **Ten briefs.** The author's own brief, two or three others the author considers realistic, and synthesised variations of those to reach ten. Two of the ten are "I don't know yet" briefs.
- **A week per brief.** Days 1 to 7 run in order, each day's question added to the next day's `recentPrompts`.
- **Follow-up coverage.** On some days the case supplies a synthetic piece for the previous day, written to leave a thread worth pulling; on others it supplies none. At least one day per brief has no new piece, to show generation proceeds without one.
- Sequential within a brief, briefs run concurrently.

**Gate 1:** the author reads every brief and every synthetic piece and approves the set before any full run.

### Grading
1. **Deterministic checks** (above), per candidate.
2. **Rubric judge**, pointwise, on the chosen question. Draft criteria, each pass or fail:
   - Contains a tension or demands a stance; is not a topic
   - Answerable in about 300 words from what the user already knows, without research
   - Stays inside the brief (or, for "I don't know yet", inside what the brief mentions)
   - Fits the day: easy on days 1–3; by day 7 the week includes one real-tension question and one change of register
   - If it is a follow-up, it engages with something specific in the piece
   - Not a near-repeat of a question earlier in the week
3. **Week-level checks** per brief: variety of `kind`, no repeats, register change present by day 7.
4. **The author's read.** The report shows all 70 chosen questions, with their rejected siblings and critiques. The author's judgement is the verdict; the scores are there to find problems fast.

The judge returns its verdict through structured outputs and treats the question as data. It must not be the model under test.

**Gate 2:** the runner grades a pilot of about five cases; the author reads the questions next to the grades and says whether they would have graded any differently. The rubric is revised until the answer is no.

### Running it
- The runner is based on the Claude API skill's eval scaffold: per-case wall-clock limit, backoff on rate limits with retries recorded, results written as each case finishes, resume without duplicates, a sidecar file for failed attempts, and an assertion that the answering model is the requested one.
- The scaffold's harness-approval flag (`--approve-harness`) is passed only by the author, never by Claude.
- **Cost:** estimated from the pilot's measured token usage, shown with the arithmetic, and approved by the author before the full run. No estimate is made before a pilot exists.
- The report is built with the skill's report builder, not hand-written HTML.

---

## Testing strategy

### Unit tests
- Request assembly: model, effort, structured output schema, no prefill, no forced tool choice, brief and pieces delimited as data
- Parsing: valid output; malformed output; `refusal`; `max_tokens`
- Follow-up guard: a candidate referencing an unknown session id is rejected
- Each deterministic check, with passing and failing examples
- No test calls the real API

### Manual testing checklist
- [ ] One live call with the author's brief produces three candidates and a choice
- [ ] The pilot report opens and shows candidates, critiques and grades
- [ ] Full report read by the author

---

## Pre-commit checklist

- [ ] `npm test` passes with coverage thresholds met
- [ ] `npx tsc --noEmit` passes
- [ ] No API key, private brief or eval output staged (`git status`)
- [ ] Docs updated (root `CLAUDE.md` commands and status, `README.md` status, `REFERENCE/question-generation.md`)

---

## PR workflow

- Branch: `feature/question-generation`
- Review with `/review-pr`. The prompt and eval are the heart of the product; consider `/review-pr-team`.
- The PR description includes the baseline headline numbers, the author's verdict, and the measured cost of the runs.

---

## Edge cases and considerations

### Known risks
- **The judge rewards the wrong thing.** Mitigated by gate 2 and by the author reading every question.
- **Synthetic briefs are too easy.** Mitigated by anchoring them on the author's real brief and examples.
- **Questions converge on one style.** A per-question judge can't see this; the week-level variety check and the author's read can.

### Security considerations
- The author's real brief and any personal material stay out of the public repository.
- Brief and piece text is treated as data in both the generator and the judge prompts.

---

## Decisions for the author before work starts

1. **Judge model.** It cannot be Claude Opus 5.5, the model under test. Proposed: Claude Sonnet 5.5, with the author's own read as the real verdict.
2. **Where private cases and results live.** Proposed: `eval/question-generation/private/` for the author's brief and `eval/question-generation/runs/` for output, both gitignored. Public synthetic cases are committed.
3. **The author's brief and two or three realistic examples.** Needed for the case set; supplied in chat or written straight into the private folder.

---

## Related documentation

- [MVP-IDEA.md](./ORIGINAL_IDEA/MVP-IDEA.md) sections 4, 5, 6 and 12
- [testing-strategy.md](../REFERENCE/testing-strategy.md) - tests versus the eval
- [environment-setup.md](../REFERENCE/environment-setup.md) - the API key
