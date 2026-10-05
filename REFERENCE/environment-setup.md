# Environment and secrets setup

**When to read this:** setting up local development, adding or rotating a secret, or changing Simmer's Cloudflare email configuration.

**Related documents:**
- [CLAUDE.md](../CLAUDE.md) - project navigation index
- [MVP-IDEA.md section 10](../SPECIFICATIONS/ORIGINAL_IDEA/MVP-IDEA.md#10-stack-decided) - the stack and why
- [troubleshooting.md](./troubleshooting.md) - when something doesn't work

---

## Accounts

| Service | Used for | Account |
|---|---|---|
| Cloudflare | Workers, Cron Triggers, D1, Email Routing and Email Sending | M Hultberg |
| Anthropic | Claude API for questions, nudges and receipts | The author's Anthropic Console account |
| GitHub | Code and pull requests | [mannepanne/simmer-writing](https://github.com/mannepanne/simmer-writing) (public) |

The repository is public. Never commit email addresses, account IDs, API keys or anything from `.dev.vars`.

---

## Secrets

| Name | What it is | Local | Production |
|---|---|---|---|
| `ANTHROPIC_API_KEY` | Anthropic API key for Claude Opus 5.5 | `.dev.vars` | Worker secret |

Email needs no key: the Worker sends through the `send_email` binding and receives through its `email()` handler.

### Anthropic API key

1. Create a key in the Anthropic Console, named for this project (for example `simmer`).
2. Set a monthly spending limit on the account.
3. Add it to `.dev.vars` in the project root yourself; never paste it into chat:
   ```
   ANTHROPIC_API_KEY=...
   ```
4. For production, once the Worker exists:
   ```bash
   npx wrangler secret put ANTHROPIC_API_KEY
   ```

`.dev.vars` and `.env` are gitignored. Wrangler loads `.dev.vars` automatically for `wrangler dev` and tests.

---

## Wrangler

Wrangler is the Cloudflare CLI. It runs through `npx`, so no global install is needed.

```bash
npx wrangler login     # one-time browser approval; credentials stay in Wrangler's own config, outside the repo
npx wrangler whoami    # confirm the account and scopes
```

The login needs the `email_routing` and `email_sending` scopes, which the default login includes.

---

## Cloudflare email configuration

Simmer uses the address simmer@hultberg.org. The hultberg.org zone also carries the author's personal mail, so changes here are limited to that one address.

### Already in place on hultberg.org

| Setting | State |
|---|---|
| Email Routing | Enabled |
| Email Sending | Enabled, so mail can be sent from any @hultberg.org address |
| Verified destination | The author's Gmail, so sending to it is free on the Workers Free plan |
| Rule for the author's personal address | Forwards to Gmail; untouched by Simmer |
| Catch-all rule | Forwards to Gmail. Until Simmer's own rule exists, mail to simmer@ lands there |

Check the current state with:

```bash
npx wrangler email routing settings hultberg.org
npx wrangler email routing rules list hultberg.org
npx wrangler email routing addresses list
npx wrangler email sending list
```

### Added when the inbound Worker is built

An Email Routing rule that sends simmer@hultberg.org to the Simmer Worker ("Send to a Worker"). A specific address rule takes priority over the catch-all, so all other hultberg.org mail keeps going to Gmail. The Worker must be deployed before the rule can point at it. Ask before creating or changing any routing rule.

### Sending to anyone else

Sending to addresses that are not verified destinations needs the Workers Paid plan. Until then, every address on Simmer's allowlist must also be a verified destination in Email Routing.

---

## D1 database

Created when the data model is built ([MVP-IDEA.md section 12](../SPECIFICATIONS/ORIGINAL_IDEA/MVP-IDEA.md#12-first-prompts-for-claude-code), prompt 3), with migrations kept in the repo. The database name and binding are recorded here at that point.

---

## Security practices

- Secrets live only in `.dev.vars` (local) and Worker secrets (production).
- Inbound email is untrusted input: only allowlisted senders are processed, and email content is never treated as instructions to the model.
- Check `git status` and `git diff` before every commit for stray secrets or personal data.
- If a secret is committed, rotate it first, then clean history. See [troubleshooting.md](./troubleshooting.md).
