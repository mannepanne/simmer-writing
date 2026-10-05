# Project outline

Source of truth for what Simmer is and why. The detailed build target is [MVP-IDEA.md](./MVP-IDEA.md); the parked larger design is [POTENTIAL-FUTURE-ELABORATION.md](./POTENTIAL-FUTURE-ELABORATION.md). Where this outline summarises, the MVP doc has the detail.

---

## What this is

Simmer is an email service that gets you writing something most days. A question arrives by email before bed; you reply to it in the morning. If you're stuck, it answers back with a question, never with prose.

Tagline: *A question at night. Your reflection by morning.*

---

## Why this exists

People who want a writing habit fail for two reasons: they don't sit down, and when they do they have nothing to say. A diary fails because a blank page has no question and no reader. Simmer supplies both: a question with some tension in it, planted the night before so it works on you overnight, and a reader that responds to the thinking.

The author has tried and failed to build a writing habit several times, dislikes diaries, and often feels he has nothing to write about. Simmer is built to test whether a good question and a reader fix that.

---

## Who it's for

- **Primary:** the author, Magnus. The MVP has one user, and the test is whether he keeps replying.
- **Later, if the test passes:** people who think for a living (product managers, engineers, analysts, founders, writers) and want to think better on the page, not feel better.

---

## Core features (in scope)

- **Setup by email.** Emailing simmer@hultberg.org from an allowlisted address starts it. Simmer asks "What are you simmering?" plus bedtime, morning time and timezone. The answer is stored as the **brief**, the base every question comes from.
- **The daily loop.** Each session is one email thread:
  1. Evening: the question, ending "let it simmer".
  2. Morning: a reply in the same thread restating the question. This is the one you answer.
  3. Your reply, any length, any time.
  4. A receipt: word count, session number, the week at a glance, and one specific sentence about your thinking.
- **`stuck`.** A reply ending in `stuck` gets one question, one pointer to the weakest sentence, or one counterexample.
- **Question generation with Claude.** Mostly independent questions from the brief, with recent pieces as background. An occasional follow-up when the last piece left a thread worth pulling. Generation never waits for a reply.
- **Commands by email.** `brief:`, `times:`, `weekdays:`, `pause` and `resume`, each confirmed by a short reply.
- **Every day by default**, limited to chosen days with `weekdays:`.
- **Pausing.** An automatic pause after five unanswered sessions, lifted by any reply. A manual pause, lifted only by `resume`.
- **Six hard rules for the agent**, enforced in code where possible ([MVP-IDEA.md section 5](./MVP-IDEA.md#5-hard-rules-for-the-agent-decided)). The two that shape the whole product: it never writes prose for the user, and it counts sessions, never streaks.

---

## Explicitly out of scope

- A web editor, word counter or any web interface. A Worker at simmer.hultberg.org is possible later.
- Visible question modes, the people list, revisiting old pieces, the base/orbit dial, weekly digests. These are in the parked design and wait for the secondary test.
- Other users, cohorts, payments, current-events questions, an app.
- Comments on writing style. Style is the job of tools like Marker; Simmer only engages with the thinking.

---

## What "good" looks like

- **Primary test:** the author still replies at all in week two. If not, the cue and friction are wrong, and no feature will fix that.
- **Secondary test:** thirty sessions in roughly six weeks, then an honest answer to "were the questions good enough to get out of bed for?"
- **Before any email plumbing:** an eval in which a week of generated questions for ten imagined briefs, two of them "I don't know yet", meets the standard in [MVP-IDEA.md section 6](./MVP-IDEA.md#6-prompt-generation-open-the-only-thing-that-matters).

---

## Constraints and assumptions

- **Stack:** Cloudflare Workers with Cron Triggers, D1, and Cloudflare Email Service on hultberg.org (Email Routing in, `send_email` binding out). Claude Opus 5.5 through the Anthropic API.
- **Email:** simmer@hultberg.org, plain text only. Mail for that address is routed to the Worker by an address rule. All other hultberg.org mail is untouched, and the catch-all forwards to Gmail.
- **Cost:** the free Workers plan covers sending to verified addresses, which is enough for one user. Inviting others needs the Workers Paid plan. Claude costs are small at one user, and a monthly spending limit sits on the API key.
- **Threading:** sessions are matched by `In-Reply-To`, never by date. This assumes mail clients keep that header, which the major ones do.
- **Assumption:** a fixed morning cue plus the lowest-friction possible reply (just hit reply) is enough to get the author writing. This is the thing being tested.

---

## Open questions

- **How questions get generated.** The system prompt, how the kind of question varies, and how often a follow-up is chosen over an independent question. The eval answers the first part; real use answers the rest.
- **Quote stripping.** How reliably the quoted history can be removed across Gmail, Apple Mail, Outlook and mobile clients.
- **The receipt.** What its one sentence and its "week at a glance" look like without implying a streak.
- **Replies under the word floor.** They get a receipt but don't count as a session. What session number, if any, does that receipt show?
- **Model choice.** Whether a cheaper model writes questions as good as Opus 5.5. The eval decides.
- **Before inviting anyone:** the Workers Paid plan, and trademark and other checks on the name "Simmer".

---

## Naming and inspiration

- **Simmer:** a question simmers overnight, on low heat while you're not looking, and you write toward a conclusion in the morning.
- **Research behind the loop:**
  - Boice: brief scheduled sessions beat waiting for inspiration.
  - Lally et al.: missed days barely matter, hence no streaks.
  - Gollwitzer: a fixed cue roughly doubles follow-through, hence the morning email.
  - The incubation effect: a question before sleep gets worked on overnight.
- **Marker** (marker.page) is the admired neighbour. Its stance ("won't do the hard part for you") is adopted wholesale. Simmer is its complement, not its competitor: Marker solves the room, Simmer solves showing up with a question.

---

*Last updated: 2026-10-05*
