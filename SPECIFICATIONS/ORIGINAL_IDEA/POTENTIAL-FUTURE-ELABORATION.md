# Simmer: exploration of a possible future elaboration

> **Status: parked, not the build target.** This is the full design exploration from September/October 2026. It was critically reviewed and judged to be nine-tenths about what happens once the writing habit exists and one-tenth about creating it. The actual build target is **MVP-IDEA.md** in this repo. Everything here (modes as a schedule, the people list, corpus revisits, the base/orbit dial, the no-praise rule) is deferred until the MVP has produced at least thirty sessions and a corpus to reason about. Treat this as a reference for *why* the data model has the shape it has, and as a menu for later, not as a spec.

*Duolingo for thinking in writing.* An email-driven service that builds a writing habit and breaks writer's block by handing you a question (prompt) via email the night before and an email for writing the response the morning after.

Status: pre-build. Author: Magnus. Date: 2026-09-28.
This document is the starting brief for Claude Code. Sections marked **[decided]** are settled; sections marked **[open]** are for refinement. Treat everything as editable, but change a [decided] item only with a stated reason.

---

## 1. The problem

Most people who want a writing habit fail for two separate reasons, and existing tools only address the second:

1. **They don't sit down.** Habit forming issue mechanics: no cue / inspiration, no small enough first step (thinking way too big), no reward to look forward to, don't prioritise it.
2. **When they do, they have nothing to say.** The diary/"just write anything" advice fails because a blank page has no question and no anticipation of a reader.

Writing tools (Marker, iA Writer, Ulysses) solve the room. Prompt-a-day apps supply random questions nobody cares about and that doesn't resonate. Journalling apps drift into mood tracking. Nobody owns the moment *before* the writer comes up with an idea worth spending time on.

## 2. Positioning [decided]

- **What it is:** a habit and unblocking service. The output is a person who writes 300 words most weekdays without a fight and reaches for writing when confused.
- **What it is not:** an editor, a word processor, a journalling app, a wellness app. Explicitly a complement to tools like Marker, sitting one step before them.
- **Audience for v1:** people who aim to think for a living (PMs, engineers, analysts, founders, creative writers) and want to think better, not feel better.
- **Tone:** dry, respectful, encouraging but no cheerleading, no guilt.

## 3. Day-90 user (work backwards from this)

Sixty-odd short pieces they'd be mildly proud to show a colleague. A sense of having got better at distinct kinds of thinking and the writing that shows it. A few positions they have visibly expressed their mind on, maybe even changed it, in writing. They open the morning email to do a spot of writing before they open Slack.

## 4. Principles / hard rules for the agent [decided]

These are constraints, not tone guidance. Enforce them in code / hooks where possible (e.g. output length limits), not just in the system prompt.

1. **The agent never writes prose for the user.** Its only moves when nudging: ask a question, point at the weakest sentence in the draft so far, or offer a counterexample. Never a sentence the user could paste.
2. **The agent never comments on prose quality.** Style is someone else's job. The agent engages only with whether the thinking holds: the argument, the stakes, the decision logic, the clarity.
3. **Prompts are questions with tension, never topics.** "Reflect on AI in product management" is forbidden. "Your team ships a feature nobody asked for because AI made it cheap. Argue the side you don't believe." is the standard.
4. **No links in the evening email.** Incubation, not research. The evening email ends with "don't answer this yet, just mull it over."
5. **Sessions, never streaks.** Nothing in any email or UI counts consecutive days or mentions missed days.
6. **The user's writing is the user's.** Never used to train third-party models. Exportable at any time. State this on the signup page.

## 5. The five modes [decided]

Each session has one mode. Modes are the "skill tree": distinct thinking muscles, rotated so the user feels progress in each.

| Mode | What it trains | Prompt shape | Uses a named person? | Corpus revisit reaches back for |
|---|---|---|---|---|
| **Argue** | Holding and testing a position | A claim + "defend/attack it" | Optional, as the sceptic | A position |
| **Explain** | Making something clear to a specific person | A thing + a named naive reader | Required, as the reader | A position |
| **Decide** | Reasoning to a choice under uncertainty | A scenario + "what would you need to believe?" | Optional, as the stakeholder | A position |
| **Predict** | Committing to a forecast and its reasoning | A question about the future + a horizon | Never | A prediction (to score later) |
| **Imagine** | Invention under constraint (fiction) | A premise + a constraint (length, POV, object) | Never | A world or character |

Rotation rule [open]: default round-robin weighted by the user's settings; user can exclude modes.

## 6. User flow (v1, email-only) [decided]

### Signup + settings (web, minimal)
- Email, timezone, bedtime (evening send = bedtime minus 60 min), cue time (morning send), word floor/target (default 150 / 300).
- **Topic configuration is mandatory.** Free-text topics plus mode weights. No "random" option.
- **People (optional).** A short list of named people from the user's world, each with a name and a one-line description of what they know and care about ("my CFO — numbers-first, allergic to hand-waving"; "my 12-year-old nephew — curious, no jargon"). People belong to the user, not to a topic; the generator decides when to cast them per the mode table in section 5. If the list is empty, Explain prompts invent a generic reader ("someone smart who has never heard of X").

### The daily loop (one email thread per session)
1. **Evening email**: the prompt, its mode, one line of framing, "don't answer this yet, just mull it over." No stats, no links, nothing else.
2. **Morning email** (reply in same thread): restates the prompt, arrives at cue time, one line of context ("Session 12. Yesterday you argued X in 340 words."). This is the email the user replies to.
3. **User replies** with their piece. Reply may arrive any time; attribute by thread, not by date.
4. **`stuck` reply** (any time, any number of times): the user replies with `stuck` on its own line, usually with the draft so far above or below it. Agent responds *to the draft* with one question / weakest-sentence / counterexample. Never prose.
5. **Receipt email** on the first submission: word count, session number, week at a glance, and one sentence from the agent about the thinking (a question, not praise). Later submissions in the same session are appended (see below) and get no receipt.

### Classifying an inbound reply [decided]
Every inbound email is processed in this order:

1. **Strip quoted history.** Remove everything the mail client quoted (`>` lines, "On … wrote:" blocks, signature separators). Only the new, unquoted body is ever classified or stored. This means an earlier `stuck` reply, or the agent's response to it, can never be re-read as a trigger when the client quotes it under a later reply.
2. **Check for the trigger.** If the last non-empty line of the new body is `stuck` (case-insensitive, trailing punctuation allowed), or the body is nothing but that word, the reply is a `nudge_request`. Everything above the trigger line is the draft so far and is passed to the agent.
3. If the body is `resume` (or has it on the first or last line) and the user is paused, it is a `resume_request`; the user is unpaused and no session is involved.
4. **Otherwise it is a `submission`.** "Stuck" appearing anywhere else in the text is just a word.

The natural gesture is draft first, `stuck` last, send; the rule follows that. No override word for now: the failure modes are cheap (a misread submission delays a receipt until the next reply; a misread nudge fires a receipt on a partial piece, which the append rule absorbs). Add an override only if the thirty-session test surfaces a real case.

### Submissions [decided]
- **First submission** = the first inbound reply in the session classified as `submission`. It fires the receipt.
- **Later submissions** happen for ordinary email reasons: "also, one more thing", a corrected paragraph, a piece finished in two sittings. For v1 the rule is **append**: the piece is all submissions concatenated in order, the word count is the total, and no further receipt is sent. (A "replace" rule only makes sense once there is a web editor with a real edit action; leave it for the future UI.)

### Unanswered sessions [decided]
A session with a `cue_sent` and no `submission` is simply "not started": never "missed", never a failure state in code or copy.

- **The thread stays open indefinitely.** A reply days later is attributed by thread to the original session and counts as a session; timestamps show it was late, nothing else comments on it. (This is the main reason attribution is by thread, not by date.)
- **The next evening proceeds as normal**: a fresh prompt in a new thread, with no reference to the unanswered one. The morning email's context line refers to the most recent *completed* session, however old. Receipts and digests count sessions written, never sessions offered.
- **Never**: resend the unanswered prompt, nudge about it, mention a count of unanswered days, or recycle the question later with "you never got to this."
- **Log unanswered prompts as a tuning signal.** If they cluster by mode or topic, the generator is being told something.
- **Auto-pause after 5 consecutive unanswered sessions.** Stop sending evening prompts and send one plain email: "Paused. Reply `resume` whenever." This is inbox hygiene, not guilt: a service that keeps emailing someone who has stopped replying gets marked as spam. `resume` on the first or last line of a reply (same rules as `stuck`) restarts the loop from the next scheduled evening. A manual pause/holiday toggle on the settings page covers the planned case.

### Corpus revisit
- Roughly one session in six (tunable), the evening prompt is generated *from the user's own past writing* rather than fresh: "Three weeks ago you argued X. Since then Y. Still true?" For Imagine: "return to [world/character]."
- Predict mode revisits when the horizon passes: "You predicted X by [date]. Score yourself."
- Weekly digest email [open]: what you argued/decided/predicted this week, in your own words.

## 7. Prompt-generation brief (draft, for the agent system prompt) [open]

Inputs available to the generator: user topics and mode weights, the user's people list, the full corpus (final pieces, modes, dates), recent prompts (to avoid repetition), today's mode, and optionally a current-events feed if the user enabled it.

A good prompt:
- Is one to three sentences.
- Contains a tension: two things that can't both be fully true, or a stance the user must take.
- Is answerable in 300 words without deep research.
- Is specific to the user's stated topics but not so narrow it presumes knowledge they may lack.
- Names the mode. For Explain, names the reader (from the people list, or an invented generic one). For Argue and Decide, may cast a person as the sceptic or stakeholder when it sharpens the tension. For Imagine, names the constraint.
- Ends with "don't answer this yet, just mull it over."

A bad prompt: a topic, a listicle request, anything beginning "reflect on", anything that could be answered by a search, anything requiring the user to have had a specific experience.

Generate 3, self-critique against the rules above, send the best. Log all 3 with the critique (useful for later prompt tuning).

## 7a. Generator policy: base and orbit [open]

Two schools on writer's block, and Simmer treats them as phases of one loop rather than a fork:

- **Hemingway:** block is an ordinary struggle, overcome by persistence (with technique: stop mid-sentence so tomorrow has a running start; when stuck, write one true sentence).
- **Bradbury:** block is a signal that you are on the wrong topic; change tack and write about something else.

Bradbury's advice is relational: "change tack" only means something if you have a tack. So both need a **base**, and the difference is how tightly prompts **orbit** it.

### Setup
- The setup question is "**What are you simmering?**" One base: a project, a book idea, a discipline, a question. "**I don't know yet**" is a legitimate answer.
- The existing topic list is the wider field the orbit can range over. The people list and mode weights apply to both.

### The dial: orbit width
- **Tight** (Hemingway phase): prompts circle the base — different aspects, different modes, same centre of gravity. Default when a base is given. Risk: depletion, the orbit decaying into the same three thoughts.
- **Wide** (Bradbury phase): prompts range deliberately across the user's broader topics to find what catches. Default when the answer is "I don't know yet". Risk: dilettantism; nothing compounds, and the corpus revisit has nothing to bite on.
- The user can see and set the dial. The system can nudge it, as below. Never random: wide still means "inside this user's world."

### Signals that move the dial
Bradbury's "subliminal signal" is not subliminal here; it is already in the event stream:
- Unanswered sessions, short pieces, sessions with repeated `stuck` nudges, long cue-to-submit times → base prompts going dead → **widen** for a few sessions.
- A quick, long, nudge-free response out wide → **candidate base**. Surface it once, lightly, in the next morning context line: "That one came easily. Want to simmer on it for a while?" The user's reply (or a settings change) sets the new base; the system never switches base on its own.
- Nothing about any of this is phrased as failure. Widening is a change of scenery, not a diagnosis.

### Consequences
- Day-90 outcomes differ by phase and both count: the tight-orbit user ends with a project's worth of fragments; the wide-orbit user ends with knowing what they actually want to write about, which for many target users is the more valuable result.
- Corpus revisits are mostly a tight-orbit feature; in wide orbit they should be rare and reserved for the pieces that came easily.
- Data model: add `base` (nullable text) and `orbit` (`tight` | `wide`, plus a nudge history) to the user; add `orbit` and `distance_from_base` (the generator's own estimate) to the prompt, so the signals above can be analysed per prompt later.

The thresholds (how many dead sessions before widening, what counts as "came easily") are for the thirty-session test to find. Start with simple rules and log everything.

## 8. Data model [decided in shape, open in detail]

Email is the transport, not the model. Every session is an ordered event stream; a future web/Android UI renders the same stream.

```
user           id, email, timezone, bedtime, cue_time, word_floor, word_target,
               topics (json), mode_weights (json), people (json),
               paused (bool), paused_at (nullable), created_at
prompt         id, user_id, mode, text, source ('fresh' | 'revisit'),
               revisit_of_session_id (nullable), person_id (nullable),
               orbit ('tight' | 'wide'), distance_from_base (nullable),
               candidates (json), generated_at
session        id, user_id, prompt_id, scheduled_for (date), created_at
session_event  id, session_id, seq, type, direction ('out' | 'in'), channel,
               content, raw_content (nullable), word_count (nullable), occurred_at,
               provider_message_id, in_reply_to
```

- `type` ∈ {prompt_sent, cue_sent, nudge_request, nudge_response, submission, receipt_sent, digest_sent, pause_sent, resume_request}
- `channel` ∈ {email} for now; {web, android} later.
- `content` is the cleaned, unquoted body; `raw_content` keeps the original inbound email for debugging the quote-stripper and re-classifying if the rules change.
- Session status (not started / in progress / submitted) is **derived** from events, never stored as truth.
- Inbound emails are attributed to a session via `in_reply_to` → `provider_message_id`, never by date.
- The piece is all `submission` events in the session concatenated in `seq` order. Receipt fires on the first.
- A derived, cacheable `session_view` (final text, word count, nudge count, cue-to-submit time) feeds the digest, the corpus revisit generator, and any future UI. The event stream remains the source of truth.
- Every row has a timestamp. No exceptions.

## 9. Scope

### In (v1)
- Signup + settings page
- Evening / morning / receipt emails
- `stuck` nudge via reply; auto-pause / `resume`
- Corpus revisit (basic: reach back for a position in the same mode)
- Event-stream storage with export (Markdown zip of all pieces)

### Out (v1)
- Any web editor, word counter, or live nudge button
- Android app
- Cohorts / shared prompts / seeing others' pieces
- Weekly digest (unless trivial once session_view exists)
- Prediction scoring
- Payments
- Current-events prompt source

### The v1 success test
The author uses it for 30 consecutive weekdays (sessions, not streaks: 30 sessions in ~6 weeks) and answers one question honestly: *were the prompts good enough to get me out of bed for?* If no, fix prompt generation before building anything else.

## 10. Suggested stack [open, author's default preferences]

- Cloudflare Workers + Cron Triggers for the two scheduled sends and the generator
- Cloudflare D1 for the tables above
- **Email in both directions via Resend [decided].** Account exists and the domain is already verified there. Outbound via the Send API. Inbound via Resend Receiving: a receiving MX record on a dedicated subdomain (so it never interferes with any ordinary mail on the root domain), a webhook on the `email.received` event hitting a Worker, webhook signature verification on, and `In-Reply-To` parsed from the payload for session attribution. Resend stores inbound mail even if the webhook is down, which is a useful safety net during development. Cloudflare Email Routing is not used.
- **Domain: echoreflex.me [decided].** The service lives here (web, sending address, reply address) whatever it ends up being called
- Anthropic API for prompt generation and nudges; enforce rule 1 with a hard `max_tokens` and a post-check that the response is a question/pointer, not prose
- Minimal signup/settings as a static page + Worker

## 11. Open questions for refinement in Claude Code

1. Mode rotation algorithm and how user weights interact with the revisit cadence.
2. Exact heuristics for choosing *which* past piece to revisit.
3. How to handle a `stuck` reply with no draft associated with it (the "one sentence you'd say out loud" opener?).
4. Quote-stripping robustness across mail clients (Gmail, Outlook, Apple Mail, mobile clients all quote differently). Keep `raw_content` so classification can be re-run when the stripper improves.
5. What the receipt's "one sentence about the thinking" is allowed to be, precisely, given rule 2.
6. Orbit thresholds (section 7a): how many dead sessions trigger widening, and what "came easily" means numerically.
7. **Name: Simmer [decided].** Something simmering in the mind overnight, developing on low heat while you're not looking, moving toward a conclusion by morning. Lives at simmer.echoreflex.me (the root domain is the studio, not the product). Tagline [decided]: *"A question at night. Your reflection by morning."* Rejected: "EchoReflex" (reads as automatic repetition), "Mull" (Australian slang for cannabis), "Nightcap" (same substance-before-bed problem), "Steep" (stray connotations). Still to do before launch: app-store, UK/US trademark and Urban Dictionary checks on "Simmer".

## 12. Suggested first prompts for Claude Code

In this order; the prompt-generation eval comes before any plumbing because the v1 success test is entirely about prompt quality.

1. "Read POTENTIAL-FUTURE-ELABORATION.md. The product is called Simmer. Propose a repo layout and a CLAUDE.md that encodes the six hard rules in section 4 as constraints on any code that calls the model."
2. "Draft the prompt-generation system prompt from section 7 and write a small eval: 20 topic/people configs, generate prompts across all five modes in both tight and wide orbit, grade against the good/bad rules. Include at least 3 revisit cases built from a fake corpus."
3. "Implement the data model in D1 with migrations, plus the session_view derivation, with tests for thread-based attribution and the append rule for multiple submissions."
4. "Build the inbound Worker as a Resend `email.received` webhook (verify the signature): strip quoted history, classify as nudge_request / submission using the last-line `stuck` rule, attribute to session by In-Reply-To, store both raw and cleaned content. Test against reply samples from at least three mail clients."
5. "Build the nudge responder with a post-check that rejects any model output that is not a question, a pointer to a sentence in the draft, or a counterexample."

## 13. Background: how this document came about

Context for Claude Code on the reasoning behind the decisions above, so they aren't re-litigated without cause.

**Starting point.** The author has repeatedly tried and failed to build a writing habit, for two reasons: not prioritising it, and struggling when actually sitting down. Standard advice ("just write anything", "keep a diary") doesn't work for him: diary writing feels self-indulgent, produces nothing worth rereading, and drifts into complaints. An Obsidian-based attempt also failed. The insight that reframed the problem: the diary fails because it has no *question* and no *reader*. Habit mechanics only solve half the problem; supplying the question and the reader solves the other half.

**Research that shaped the design.** Boice's studies of academic writers: brief, scheduled, slightly-too-short daily sessions dramatically outproduce writing when inspired, and accountability to another person adds further. Lally et al. on habit formation: missing single days barely matters, so streaks are motivationally noisy and were deliberately excluded. Gollwitzer's implementation intentions: a fixed cue ("after X, I will…") roughly doubles follow-through, hence the morning cue email. The incubation effect and Wagner et al. (2004) on sleep and insight: giving the brain a question and then stepping away from the keyboard is a real, measurable trick, hence the evening prompt with "don't answer this yet." Pennebaker's expressive-writing work was noted and set aside: it validates short bursts of emotional writing, not a daily diary.

**Key design decisions and why.**
- *Email-first, no editor.* Zero install, works everywhere, and a bad editor is a feature: it removes the temptation to format and fiddle. A web page with a text box and a nudge button is the obvious next step but is explicitly deferred until reply-by-email has actually failed.
- *Prompt the night before, write in the morning.* Incubation plus a fixed cue. The evening email carries no stats and no links so it doesn't become a scoreboard or a research rabbit hole at bedtime.
- *Three emails per session.* Evening (plant the question), morning (the cue and the thing you reply to), receipt (the reward, after the routine, where progress data belongs).
- *The five modes* (argue, explain, decide, predict, imagine) are the Duolingo "skill tree": distinct thinking muscles. "Imagine" replaced an earlier "tell" because fiction is the mode furthest from the target user's day job and therefore where block is felt most, and it is the only purely playful mode.
- *Corpus revisit.* Roughly one session in six the prompt is generated from the user's own past writing. This is spaced repetition applied to one's own ideas, it trains changing one's mind on the page (a distinct skill from forming an opinion), and it makes the agent a reader with memory rather than a prompt jar with amnesia. It is also the moat: nobody else has the user's corpus.
- *People list.* Named readers were originally an Explain-only feature; generalised to a per-user list the generator can cast as reader (Explain), sceptic (Argue) or stakeholder (Decide). Per-topic assignment was rejected as complexity nobody would miss.
- *Hard rules on the agent* came partly from Marker (marker.page), which the author uses and admires. Marker's stance ("won't do the hard part for you", process over output, writing stays the user's) is adopted wholesale. The rule that the agent never comments on prose quality exists to keep this product a complement to Marker, not a competitor: Marker solves the room, this service solves showing up with a question. Style is Marker's job; whether the thinking holds is ours.
- *Event-stream data model.* Email is one transport; a future web/Android UI should render the same stream. Sessions are attributed by email thread, never by date. Later submissions append rather than replace, because in email addenda are far more common than resends.
- *Base and orbit.* Hemingway (persist) and Bradbury (change topic) are treated as phases of one loop: every user has a base (or "not yet"), prompts orbit it tightly or widely, and the engagement signals already logged move the dial. The author himself starts in the wide phase.
- *No streaks, ever.* Sessions are counted; consecutive days are not, and missed days are never mentioned. An unanswered session stays open and is never chased; after five in a row the service pauses itself rather than keep emailing, which is inbox hygiene rather than a nudge.

**Positioning.** "Duolingo for thinking in writing." Named Simmer: a question simmers overnight and you write toward a conclusion in the morning. A habit-and-unblocking service for people who think for a living and want to think better, not feel better. Not an editor, not a journal, not a wellness app.

**The v1 test** is deliberately narrow: the author uses it for thirty sessions and answers whether the prompts were good enough to get out of bed for. If not, the prompt generator is the thing to fix before anything else is built. Prompt quality is the product.
