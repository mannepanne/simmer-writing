# Simmer — MVP

**A service that makes you write something most days, even if it's a snippet.**
A question arrives by email before bed. You reply to it in the morning. If you're stuck, something answers back.

Tagline: *A question at night. Your reflection by morning.*
Status: build target. Author: Magnus. Last updated: 2026-10-05.
The fuller design exploration lives in POTENTIAL-FUTURE-ELABORATION.md; it is parked, not part of this build.

---

## 1. The problem, and which half this build addresses

People who want a writing habit fail for two reasons: they don't sit down, and when they do they have nothing to say. Diaries fail because a blank page has no question and no reader.

This MVP mostly fixes the second: it supplies the question and a reader. For the first, it offers only a cue (the morning email) and the lowest possible friction (reply to an email). Whether that is enough is the thing being tested. Do not add features to compensate until the test says so.

## 2. The loop [decided]

One email thread per session. A session runs every day by default; the user can limit it to chosen days of the week (section 3).

1. **Evening email**, an hour before the user's bedtime. The question, one line of framing, and "don't answer this yet, let it simmer." No links. No stats. Nothing else.
2. **Morning email**, a reply in the same thread at the user's cue time. Restates the question. One line of context: the most recent completed session ("Session 12, two days ago: 340 words on X"). This is the email to reply to.
3. **The user replies.** Any length. Any time. A late reply still lands in its thread and still counts.
4. **`stuck`**: reply with the word `stuck` on the last line, with whatever draft exists above it (or nothing). The agent answers with one thing: a question, a pointer to the weakest sentence, or a counterexample. Never prose. Any number of times.
5. **Receipt**, on the first non-`stuck` reply: word count, session number, the week at a glance, and one specific sentence from the agent about the thinking (a question, or a precise observation, never generic praise). Later replies in the same session are appended; no second receipt.

## 3. Setup [decided]

No form, no signup page. The user emails simmer@hultberg.org with anything. An address Simmer doesn't know yet gets one reply:

> **What are you simmering?** A project, a book idea, a discipline, a question you can't leave alone, or "I don't know yet." A sentence or a paragraph, whatever you have. Also tell me roughly when you go to bed, when you'd like the morning question, and which timezone you're in.

The reply to that email (attributed by `In-Reply-To`, like everything else) is stored verbatim as the user's **brief**, never as a submission: the base every question is generated from, and the only input the generator has on day one. "I don't know yet" is a valid answer; the generator then ranges across whatever else the reply mentioned.

Simmer only answers senders on an allowlist (initially the author's Gmail), so mail from anyone else gets no reply. Inviting someone means adding their address.

**Commands.** Settings change by email. A command goes on the first line of a reply to any Simmer email, or of a fresh email to simmer@hultberg.org. A command email is never counted as a submission.

| Command | What it does |
|---|---|
| `brief: <text>` | Replaces the whole brief with everything after `brief:`. Earlier briefs are kept, so questions can be traced to the brief they came from. Applies from the next evening question not yet generated. |
| `times: <text>` | Sets any of bedtime, morning time and timezone, in plain words (`times: bed 23:00, morning 06:30, Europe/Stockholm`). Applies from the next send not yet made. |
| `weekdays: <text>` | Sets the days sessions run, in plain words (`weekdays: mon-fri`, `weekdays: every day`). The default is every day. |
| `pause` | Stops evening and morning emails until `resume`. |
| `resume` | Lifts any pause. Sends restart from the next scheduled evening. |

- Every command gets one short confirming reply.
- `brief:`, `times:` and `weekdays:` take free text. Simmer interprets it, saves it, and quotes back what it understood, so a misreading is visible straight away.
- `pause` and `resume` match only when the first line is that word alone (case-insensitive, trailing punctuation allowed). A submission that starts "Pause for a moment and consider…" is a submission.
- While paused, replies to earlier threads still count as submissions and still get receipts.

Defaults: every day; evening send = bedtime minus 60 min; morning send = 07:00 local if not stated; word floor 150, target 300 (mentioned once in the morning email, never enforced). A reply under the floor still gets a receipt, because every reply deserves a reader, but does not count as a session.

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

Inputs: the brief, the last 14 prompts (avoid repetition), the last 5 pieces (for context), the day number.

**Mostly independent, occasionally a follow-up.** Most evenings the question stands on its own: it comes from the brief, with recent pieces as background. Now and then, when the most recent piece clearly left a thread worth pulling (an unresolved tension, a claim made in passing, a change of mind), the question follows it up instead. The generator decides, and logs which piece it followed and why. Generation never waits for a reply: if no new piece has arrived when the evening question is due, it is an ordinary independent question, so a skipped or late morning never stalls the loop.

A good question is one to three sentences, has a tension, can be answered in 300 words from what the user already knows, and stays inside the brief. The evening email adds "let it simmer"; the question itself never includes it.

Vary the *kind* of question without telling the user: defend a claim, explain something to a specific kind of person, decide between two bad options, predict something with a date on it, invent a scene. This is seasoning, not a schedule. Over a week, roughly: half close to the brief, a quarter at its edges, a quarter a change of register. Adjust toward whatever gets replies.

Generate three candidates, self-critique against the rules, send the best, log all three with the critique. The log is the product's most valuable asset in the first month.

## 7. Unanswered sessions and pausing [decided]

An unanswered session is "not started," never "missed." The thread stays open; a late reply counts. The next evening proceeds normally with no reference to it. Unanswered questions are logged as a tuning signal (which kinds go dead?). After five consecutive unanswered sessions: one email, "Paused. Reply `resume` whenever," and nothing more. This automatic pause lifts on `resume` or on any reply to any Simmer email. A pause the user asked for with `pause` lifts only on `resume`, so a late reply to an old thread never cuts a holiday short. Section 8 records which kind of pause applies.

## 8. Data model [decided in shape]

Email is the transport, not the model. Every session is an ordered event stream so a future web view renders the same data.

```
user           id, email, timezone, bedtime, cue_time, active_days (json, default all 7),
               word_floor, word_target, paused_reason ('manual' | 'auto' | null),
               paused_at, created_at
brief          id, user_id, text, created_at            (current brief = latest row)
prompt         id, user_id, brief_id, text, kind, follows_up_session_id (nullable),
               candidates (json), generated_at
session        id, user_id, prompt_id, scheduled_for, created_at
session_event  id, session_id, seq, type, direction, channel, content, raw_content,
               word_count, occurred_at, provider_message_id, in_reply_to
```

- `type` ∈ {prompt_sent, cue_sent, nudge_request, nudge_response, submission, receipt_sent, pause_sent}. Commands and setup are not session events; they change `user` or `brief`.
- Inbound mail is attributed by `In-Reply-To`, never by date. Quoted history is stripped before classification; `raw_content` keeps the original.
- Session status is derived from events. The piece is all submissions concatenated.
- Every row has a timestamp.

## 9. The test [decided]

Primary: **does the author reply at all in week two?** Not every day. At all. If yes, the loop works and the next question is prompt quality. If no, the cue and friction are wrong and no feature in POTENTIAL-FUTURE-ELABORATION.md will fix it.

Secondary, if primary passes: thirty sessions in roughly six weeks, then an honest answer to "were the questions good enough to get out of bed for?"

## 10. Stack [decided]

- Cloudflare Workers + Cron Triggers; D1 for the tables above
- Cloudflare Email Service for both directions: Email Routing sends mail for simmer@hultberg.org (an address rule on the existing hultberg.org setup) to the Worker's `email()` handler; mail goes out from the same address through the `send_email` binding, with `In-Reply-To`/`References` headers for threading. No API key.
- Anthropic API for generation, nudges and receipts, using Claude Opus 5.5 (`claude-opus-5-5`). Whether a cheaper model is good enough is tested later, when there is a reason such as other users.
- Plain-text emails only: no HTML, no images, no tracking. They read like a letter, quote cleanly in replies, and are less likely to be filed as marketing.
- Signup: email simmer@hultberg.org (see section 3); no signup page
- Email address: simmer@hultberg.org. Any web part is a separate Worker at simmer.hultberg.org

Why Cloudflare Email Service on hultberg.org rather than Resend on echoreflex.me: one platform; inbound mail arrives in the Worker directly, with no webhook, signature check or separate fetch; no API key to manage; sending to verified destination addresses is free on any Workers plan, which covers the author-only test. hultberg.org already runs Email Routing and sending for other projects, and an address rule for simmer@ leaves the rest of its mail alone, so no subdomain is needed. Inviting other users needs the Workers Paid plan (sending to unverified recipients), and a later domain move would orphan old threads.

## 11. Explicitly not in this build

Modes as a visible schedule, the people list, corpus revisits, the base/orbit dial, weekly digests, any web editor or word counter, cohorts, payments, current events, an app. All described in POTENTIAL-FUTURE-ELABORATION.md. Revisit after the secondary test.

## 12. First prompts for Claude Code

1. "Read MVP-IDEA.md. Propose a repo layout and a CLAUDE.md that encodes the six hard rules in section 5 as constraints on any code that calls the model."
2. "Draft the prompt-generation system prompt from section 6. Write an eval: 10 briefs (including two 'I don't know yet'), generate a week of questions each, grade against section 6. Make the day-1 questions deliberately easy."
3. "Implement the data model in D1 with migrations and the derived session view; test thread attribution, quote stripping, and the append rule."
4. "Build the Worker's `email()` handler for inbound mail: drop senders not on the allowlist; parse the raw message, strip quotes; send the setup question to an allowlisted address with no user yet, and store the reply to it as the first brief; classify the rest in this order (command on the first line: `brief:`, `times:`, `weekdays:`, `pause`, `resume`, each confirmed by quoting back / `stuck` last line / otherwise submission), attribute, store raw and clean."
5. "Build the two cron sends and the receipt, including the day-4 no-reply email and the pause logic."
6. "Build the nudge responder with the post-check for rule 1 and the receipt's one-sentence observation with the post-check for rule 4."

## 13. Background, in brief

The author has repeatedly failed to build a writing habit, dislikes diaries, and feels he has nothing to write about. The insight was that the diary fails for lack of a question and a reader. Research that shaped the loop: Boice (brief scheduled sessions beat inspiration; accountability helps), Lally et al. (missed days barely matter, so no streaks), Gollwitzer (a fixed cue doubles follow-through), and the incubation effect (a question before sleep gets worked on overnight). Marker (marker.page) is the admired neighbour and this is its complement: Marker solves the room, Simmer solves showing up with a question; style is Marker's job, the thinking is ours. A fuller design was explored first and then cut back to this after a critical review found it was mostly about what happens after the habit exists.

**On writer's block.** The framing comes from Seth Thomas, "Wrestling with Yourself: Franz Kafka and Writer's Block" (Focus 82, the British Science Fiction Association's magazine for writers), which sets out four schools. Three of them shape this build:

- *Hemingway*: block is an ordinary struggle, beaten by persistence. Hence a small session most days, and `stuck` as a way to keep going rather than stop.
- *Bradbury*: block is a signal that you are on the wrong topic, so change tack. Hence the variation in section 6, adjusting toward whatever gets replies, and logging unanswered questions as a tuning signal rather than a failure.
- *Morrison*: sometimes the piece is not ready yet, and the answer is patience. Hence "let it simmer," and "not started," never "missed."

The article's closing advice, to let the writing be flawed and let it be read, is the case for the receipt: every reply gets a reader. The fourth school, Kafka (total commitment, writing to exhaustion, burning most of the result), is noted for curiosity only. It is the article's own cautionary tale and nothing here is modelled on it. One detail is worth keeping: when the real work stalled he wrote letters instead, which is a small argument for reply-by-email.
