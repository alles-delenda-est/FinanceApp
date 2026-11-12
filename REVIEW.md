# REVIEW.md — guidance for the AI code reviewer

> Owner: Patrick. Loaded by every automated review of this repo, alongside CLAUDE.md.
> Tune this for a few weeks: whenever a human repeats the same review comment, move it here.

## What failure looks like in this repo
- It leaks: personal financial data or bank/API credentials end up in logs, third-party requests, unencrypted storage, or the code.
- It is silently wrong: money maths breaks on rounding, currency or FX rate date, sign (debit vs credit), or period (accrual, cadence).
- It misroutes: an account or transaction is posted to the wrong household member, perimeter, or ledger account.

## What to check first
1. Correctness of any calculation or economic parameter (units, indexation year, currency, nominal vs real).
2. Security: input validation, secrets in code or logs, injection, unsafe deserialisation, dependency risk (OWASP Top 10).
3. Error handling at system boundaries (user input, external APIs, files).
4. Whether the change matches CLAUDE.md conventions — and whether CLAUDE.md now has stale lines.

## Rules for the reviewer
- **Evidence required.** Every finding cites a file and line and says what input breaks it. No guesses from a name.
- **Label every comment** `Important` (a bug / security issue), `Nit` (style), or `Pre-existing` (not introduced by this PR — never blame the author).
- **Cap nits at 3** per review. Drop style-only findings if there is a real bug to report.
- **Start the summary with a count** ("2 Important, 1 Nit, 0 Pre-existing") and say plainly "Nothing to fix" when clean.
- **Do not repeat** a finding already posted on this PR.
- **After 3 rounds**, stop line-by-line review and ask the author to reconsider the approach.
- **After the first few passes, review only the new diff.**
- When you find a bug, **name the bug class** so the author fixes every instance, not just the one quoted.

## Skip these paths
- `package-lock.json`, `pnpm-lock.yaml`, `yarn.lock`
- `dist/`, `build/`, `.next/`, `node_modules/`

## Always needs a human (never auto-approve)
- Anything under `.github/workflows/`
- Authentication, payments/billing, secrets handling
- Changes to economic assumptions or parameters (rates, mortality tables, inflation) — a person must confirm the source
- Deployment / release configuration
