# FinanceApp

A private household financial ledger for a two-person, cross-border household (France / Switzerland / Australia / Belgium), delivered as an installable web app on Android and desktop. The project is currently in its design phase; see `docs/` for the design documents.

## Review process

- **Checks** (run before every PR and paste the real output; never claim they pass without running them): No automated checks yet: propose adding some before larger changes.
- **Before a PR**: fresh-context subagent review of the diff + `/code-review` + `/security-review`. See `REVIEW.md` for what reviewers must check in this repo.
- **PR body** must have **Testing**, **Risk** (Low/Medium/High + why) and **How to revert** (`.github/pull_request_template.md`).
- **Risk is set by the change, not the author.** High (never auto-merged): auth, secrets, CI/deploy config, economic parameters or calculations, anything hard to undo.
- **Findings**: label Important / Nit / Pre-existing; fix each or resolve it with a written reason. Max 3 review-and-fix rounds, then rethink the approach.
- Patrick is the required human reviewer on every agent-written PR. One concern per PR.
