# Contributing

Thanks for looking. This is a working project rather than a maintained library, so contributions
are welcome but the bar is specific.

## The most useful contribution

**Something you proved by running it.**

If a node behaves differently on your n8n version, if a documented step is wrong, or if an approach
here breaks in a case this build never hit — that is worth more than any new feature.

**Say what you ran and what came back.** "This is wrong" is not actionable. "On n8n 1.x, this node
returned that instead" is.

## Before you open a pull request

1. **No secrets, ever.** Not in a file, not in a commit message, not in a screenshot. Check what
   you staged, not what you think you staged.
2. **Redact personal identifiers.** Instance URLs, emails, chat ids and document ids are replaced
   with angle-bracket placeholders like `<your-n8n-instance>` throughout. Keep it that way.
3. **Do not replace a placeholder with a plausible value.** `SET_ME_`, `<angle-bracket>` and
   `YOUR_` values are deliberate. Each fails loudly at its own node rather than half-working.
4. **Do not make a claim you have not verified.**

## Style, if you are touching a workflow

- Node names follow `Original Node Name - What It Does`. Both halves required.
- Set every parameter explicitly. Never rely on an n8n default — a default can change under you,
  and a reader cannot tell an intended value from an unset one.
- The Code node is a last resort. Try an expression, then Edit Fields, then Code.
- **Never retry a non-idempotent step.** Creating a record, sending a message and appending a row
  are single-attempt on purpose. A retry after a timeout that actually succeeded does the thing
  twice.
- Build the error path at the same time as the happy path.

## Reporting a problem

Open an issue. If it involves a secret being exposed, use a
[security advisory](../../security/advisories/new) instead — see [SECURITY.md](SECURITY.md).
