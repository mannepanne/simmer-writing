# Simmer — MVP

**A service that makes you write something most days, even if it's a snippet.**
A question arrives by email before bed. You reply to it in the morning. If you're stuck, something answers back.

Tagline: *A question at night. Your reflection by morning.*
Status: build target. Author: Magnus. Date: 2026-10-04.
The fuller design exploration lives in FUTURE-ELABORATION.md; it is parked, not part of this build.

---

## 1. The problem, and which half this build addresses

People who want a writing habit fail for two reasons: they don't sit down, and when they do they have nothing to say. Diaries fail because a blank page has no question and no reader.

This MVP mostly fixes the second: it supplies the question and a reader. For the first, it offers only a cue (the morning email) and the lowest possible friction (reply to an email). Whether that is enough is the thing being tested. Do not add features to compensate until the test says so.

## 2. The loop [decided]

One email thread per session.

1. **Evening email**, an hour before the user's bedtime. The question, one line of framing, and "don't answer this yet, let it simmer." No links. No stats. Nothing else.
2. **Morning email**, a reply in the same thread at the user's cue time. Restates the question. One line of context: the most recent completed session ("Session 12, two days ago: 340 words on X"). This is the email to reply to.
3. **The user replies.** Any length. Any time. A late reply still lands in its thread and still counts.
4. **`stuck`**: reply with the word `stuck` on the last line, with whatever draft exists above it (or nothing). The agent answers with one thing: a question, a pointer to the weakest sentence, or a counterexample. Never prose. Any number of times.
5. **Receipt**, on the first non-`stuck` reply: word count, session number, the week at a glance, and one specific sentence from the agent about the thinking (a question, or a precise observation, never generic praise). Later replies in the same session are appended; no second receipt.

## 3. Setup [decided]

No form. Signup is an email address and a timezone. Everything else is one email exchange:

> **What are you simmering?** A project, a book idea, a discipline, a question you can't leave alone, or "I don't know yet." A sentence or a paragraph, whatever you have. Also tell me roughly when you go to bed and when you'd like the morning question.

The reply is stored verbatim as the user's `brief` and is the only input the generator has on day one. "I don't know yet" is a valid answer; the generator then ranges across whatever else the reply mentioned. Users can change their brief at any time by replying to any email with `brief:` on the first line.

Defaults: evening send = bedtime minus 60 min; morning send = 07:00 local if not stated; word floor 150 (counts as a session), target 300 (mentioned once in the morning email, never enforced).

## 4. The first week [decided]

Where it lives or dies. Design it, don't let it happen.

- **Day 0**: the setup exchange above. The first evening question arrives the same night if the reply came before 6pm local, otherwise the next.
- **Days 1–3**: easy questions, close to the brief, answerable from what the user already thinks. The point is to get a reply, not a good piece. The receipt's one sentence on day 1 is the most important sentence the product ever sends; it must prove the agent read the piece.
- **Day 4–7**: one question with real tension, one that changes register (if the brief is work, something playful; if the brief is a novel, something argumentative). Watch which gets the better reply.
- **If no reply by day 4**: one plain morning email: "No rush. The questions keep coming; reply to any of them." Then nothing further about it.
- Week two is the test (section 9).

## 5. Hard rules for the agent [decided]

Enforce in code where possible, not just in the system prompt.

1. **Never writes prose for the user.** Nudges are a question, a pointer to a sentence, or a counterexample. Hard `max_tokens`; post-check rejects anything else.
2. **Engages with the thinking, not the style.** Argument, stakes, clarity of the idea. Never sentence quality, word choice, grammar.
3. **Questions, never topics.** Every prompt contains a tension or demands a stance. "Reflect on X" is forbidden.
4. **Specific, never generic.** Encouragement is allowed when it names something in the piece. "Great work" is forbidden; "You changed your mind halfway through and the second half is stronger" is fine.
5. **Sessions, never streaks.** No consecutive-day counts, no missed-day mentions, anywhere.
6. **The writing is the user's.** Never used to train models. Exportable on request as a Markdown zip.

## 6. Prompt generation [open, the only thing that matters]

Inputs: the brief, the last 14 prompts (avoid repetition), the last 5 pieces (for context, not for revisiting yet), the day number.

A good question is one to three sentences, has a tension, can be answered in 300 words from what the user already knows, stays inside the brief, and ends with "let it simmer."

Vary the *kind* of question without telling the user: defend a claim, explain something to a specific kind of person, decide between two bad options, predict something with a date on it, invent a scene. This is seasoning, not a schedule. Over a week, roughly: half close to the brief, a quarter at its edges, a quarter a change of register. Adjust toward whatever gets replies.

Generate three candidates, self-critique against the rules, send the best, log all three with the critique. The log is the product's most valuable asset in the first month.

## 7. Unanswered sessions and pausing [decided]

An unanswered session is "not started," never "missed." The thread stays open; a late reply counts. The next evening proceeds normally with no reference to it. Unanswered questions are logged as a tuning signal (which kinds go dead?). After five consecutive unanswered sessions: one email, "Paused. Reply `resume` whenever," and nothing more. Any reply to any Simmer email unpauses.

## 8. Data model [decided in shape]

Email is the transport, not the model. Every session is an ordered event stream so a future web view renders the same data.

```
user           id, email, timezone, bedtime, cue_time, brief, word_floor, word_target,
               paused (bool), paused_at, created_at
prompt         id, user_id, text, kind, candidates (json), generated_at
session        id, user_id, prompt_id, scheduled_for, created_at
session_event  id, session_id, seq, type, direction, channel, content, raw_content,
               word_count, occurred_at, provider_message_id, in_reply_to
```

- `type` ∈ {prompt_sent, cue_sent, nudge_request, nudge_response, submission, receipt_sent, pause_sent}
- Inbound mail is attributed by `In-Reply-To`, never by date. Quoted history is stripped before classification; `raw_content` keeps the original.
- Session status is derived from events. The piece is all submissions concatenated.
- Every row has a timestamp.

## 9. The test [decided]

Primary: **does the author reply at all in week two?** Not every day. At all. If yes, the loop works and the next question is prompt quality. If no, the cue and friction are wrong and no feature in FUTURE-ELABORATION.md will fix it.

Secondary, if primary passes: thirty sessions in roughly six weeks, then an honest answer to "were the questions good enough to get out of bed for?"

## 10. Stack [decided]

- Cloudflare Workers + Cron Triggers; D1 for the tables above
- Resend for both directions: Send API out; Receiving on a dedicated subdomain of echoreflex.me, `email.received` webhook into a Worker, signature verified
- Anthropic API for generation and nudges
- Signup: one static page with an email field, or simply "email hello@simmer.echoreflex.me"
- Lives at simmer.echoreflex.me

## 11. Explicitly not in this build

Modes as a visible schedule, the people list, corpus revisits, the base/orbit dial, weekly digests, any web editor or word counter, cohorts, payments, current events, an app. All described in FUTURE-ELABORATION.md. Revisit after the secondary test.

## 12. First prompts for Claude Code

1. "Read MVP.md. Propose a repo layout and a CLAUDE.md that encodes the six hard rules in section 5 as constraints on any code that calls the model."
2. "Draft the prompt-generation system prompt from section 6. Write an eval: 10 briefs (including two 'I don't know yet'), generate a week of questions each, grade against section 6. Make the day-1 questions deliberately easy."
3. "Implement the data model in D1 with migrations and the derived session view; test thread attribution, quote stripping, and the append rule."
4. "Build the Resend `email.received` webhook Worker: verify signature, strip quotes, classify (`stuck` last line / `brief:` first line / otherwise submission), attribute, store raw and clean."
5. "Build the two cron sends and the receipt, including the day-4 no-reply email and the pause logic."
6. "Build the nudge responder with the post-check for rule 1 and the receipt's one-sentence observation with the post-check for rule 4."

## 13. Background, in brief

The author has repeatedly failed to build a writing habit, dislikes diaries, and feels he has nothing to write about. The insight was that the diary fails for lack of a question and a reader. Research that shaped the loop: Boice (brief scheduled sessions beat inspiration; accountability helps), Lally et al. (missed days barely matter, so no streaks), Gollwitzer (a fixed cue doubles follow-through), and the incubation effect (a question before sleep gets worked on overnight). Marker (marker.page) is the admired neighbour and this is its complement: Marker solves the room, Simmer solves showing up with a question; style is Marker's job, the thinking is ours. A fuller design was explored first and then cut back to this after a critical review found it was mostly about what happens after the habit exists.
