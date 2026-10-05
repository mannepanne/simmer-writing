# Simmer

*A question at night. Your reflection by morning.*

Simmer is an email service that gets you writing something most days. Each evening it emails you one question with some tension in it, and tells you not to answer yet. Each morning it replies in the same thread with the question again, and you answer that email. If you're stuck, reply with `stuck` and it answers back with a question, never with prose. After you write, you get a short receipt with one specific sentence about your thinking.

It exists because a blank page has no question and no reader. Simmer supplies both.

## Status

Planning is complete; no code yet. The MVP has one user, its author. The first build task is an eval of the questions Claude generates, because question quality is the product.

## How it works

1. Email simmer@hultberg.org. Simmer asks what you're simmering, plus your bedtime, morning time and timezone. Your answer becomes your **brief**.
2. An hour before bed, a question arrives. In the morning, the same thread comes back; you reply with whatever you have.
3. Reply with `stuck` on the last line to get one question, one pointer to your weakest sentence, or one counterexample.
4. Change settings by email: `brief:`, `times:`, `weekdays:`, `pause`, `resume`.

No streaks, no missed-day counts, no comments on style. Your writing is yours and is never used to train models.

## Stack

- Cloudflare Workers, Cron Triggers and D1
- Cloudflare Email Service on hultberg.org: Email Routing in, `send_email` binding out, plain text only
- Claude Opus 5.5 through the Anthropic API

## Documentation

- [Project outline](./SPECIFICATIONS/ORIGINAL_IDEA/project-outline.md) - what Simmer is and why
- [MVP build target](./SPECIFICATIONS/ORIGINAL_IDEA/MVP-IDEA.md) - the loop, commands, hard rules, data model and stack in detail
- [CLAUDE.md](./CLAUDE.md) - navigation index for working on the project with Claude Code
- [Environment setup](./REFERENCE/environment-setup.md) - accounts, secrets and Cloudflare configuration

## Working on it

All changes go through a feature branch and a pull request, reviewed with the `/review-pr` skill. The project was set up from Magnus Hultberg's [project template for AI-assisted development](https://github.com/mannepanne/useful-assets-template), which supplies the collaboration guidance in `.claude/` and the review skills.
