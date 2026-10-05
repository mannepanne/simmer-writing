# Troubleshooting

**When to read this:** something in Simmer isn't working: mail not arriving, mail not sending, Wrangler errors, tests failing.

**Related documents:**
- [environment-setup.md](./environment-setup.md) - accounts, secrets and the Cloudflare email configuration
- [testing-strategy.md](./testing-strategy.md) - how tests are organised
- [debugging-mindset.md](../.claude/COLLABORATION/debugging-mindset.md) - the general approach: read the error, find the root cause, change one thing at a time

Add an entry here whenever a problem takes more than a few minutes to solve.

---

## Inbound email

### Replies to simmer@ land in Gmail instead of reaching the Worker
The catch-all rule forwards every address without its own rule to Gmail. Check that the simmer@ rule exists, is enabled and points at the Worker:
```bash
npx wrangler email routing rules list hultberg.org
```

### A reply reached the Worker but wasn't attributed to a session
Attribution uses `In-Reply-To`. Check the stored `raw_content` for that header and compare it with the `provider_message_id` of the outbound emails in the session. A client that drops the header is a real case to handle, not a reason to fall back to dates.

### A reply was classified wrongly
Re-run classification on the stored `raw_content`. If the quote stripper left quoted text behind, add the email as a fixture in `tests/fixtures/email/` and fix the stripper against it.

### Mail from a new address gets no reply
Expected: only allowlisted senders are processed. Add the address to the allowlist, and as a verified destination while on the Workers Free plan.

---

## Outbound email

### `E_SENDER_NOT_VERIFIED`
The sending domain isn't onboarded to Email Sending. Check:
```bash
npx wrangler email sending list
```

### Sending to an address fails, or is silently not delivered
On the Workers Free plan, sending only works to verified destination addresses. Check:
```bash
npx wrangler email routing addresses list
```

### Replies don't thread in the mail client
Outbound mail must set `In-Reply-To` and `References` to the Message-ID of the session's earlier email, and keep the subject. Check the headers of the sent message in Gmail ("Show original").

### Mail lands in spam
Plain text, a consistent sender, and only sending to people who asked help most. Check SPF, DKIM and DMARC in the "Show original" view; all three should pass.

---

## Claude API

### `401` or "invalid API key"
Check that `ANTHROPIC_API_KEY` is in `.dev.vars` locally, or set as a Worker secret in production (`npx wrangler secret list`).

### A nudge or receipt sentence was rejected by a post-check
That is the post-check doing its job (hard rules 1 and 4). Log the rejected output; if rejections are frequent, the prompt needs work, not the check.

### Unexpected cost
Check usage in the Anthropic Console. The monthly spending limit caps it.

---

## Wrangler

### "Not logged in" or a missing scope
```bash
npx wrangler login
npx wrangler whoami
```

### A command is labelled "open beta"
The `wrangler email` commands are in open beta. If one misbehaves, check the result in the Cloudflare dashboard.

---

## Tests

### Tests pass locally but fail in CI
Look for timezone assumptions (scheduling tests must set the timezone explicitly), test order dependence, and missing `.dev.vars` values that CI doesn't have.

---

## Git

### A secret was committed
1. Rotate the secret first. Assume it's compromised, since the repository is public.
2. Remove it from history, for example with `git filter-repo`, and force-push after checking with the author.
