# Security Policy

## What this repo is

Workflow definitions and documentation. Nothing here runs on its own. The risk surface is what the
workflow does **once you import it into your own n8n instance and attach your own credentials.**

## Reporting a vulnerability

Please open a [security advisory](../../security/advisories/new) rather than a public issue, so the
problem is not disclosed before it can be looked at.

Expect a first reply within seven days. This is a personal project rather than a funded product, so
please set expectations accordingly.

## What counts as a vulnerability here

- A credential, key, token or other secret found anywhere in this repo or its history.
- A pattern in a workflow that would leak data from the importing user's own accounts.
- Anything in the documentation that would lead someone to expose a key by following it.

## What does not

- **Placeholders.** Any `SET_ME_`, `<angle-bracket>` or `YOUR_` value is deliberate. They fail
  loudly rather than half-working with a plausible guess, and that is the point.
- **Credential ids.** An id is a reference. It is useless without the instance and an account on it.

## Secrets and redaction

No key, token or credential value appears in this repo. Keys belong in the n8n credential store or
in environment variables, never in a workflow file or a document.

**Personal identifiers are redacted** and replaced with angle-bracket placeholders such as
`<your-n8n-instance>`. If you find one that was missed, that is worth reporting.

**If you fork this and add your own values, check your `.gitignore` before your first commit.**
Anything holding a literal key must stay out of git. Check what you actually staged, not what you
assume was staged.
