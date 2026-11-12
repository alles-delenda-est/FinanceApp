## What and why
<!-- One or two sentences: what changes, and the problem it solves. -->

## Testing
<!-- What was actually run, with real output (test run, build, lint, screenshots). "Looks fine" is not testing. -->

## Risk
<!-- Low / Medium / High — and why. -->
<!-- Low: docs, copy, additive change behind a flag. -->
<!-- Medium: ordinary product change; calculations, caching, config. -->
<!-- High: auth, payments, deploy/CI config, economic parameters, anything hard to undo. -->

## How to revert
<!-- Exact steps: e.g. "Revert this PR's merge commit" / "Turn off flag X". Note any data or migration that is NOT undone by a revert. -->

## Review checklist
- [ ] Checks (tests / build / lint) pass — output pasted above
- [ ] Independent review done (`/code-review` and a fresh-context subagent)
- [ ] Security pass done (`/security-review`), results in `~/securitylog.md` where required
- [ ] Every review finding is fixed, or resolved with a written reason
- [ ] One concern per PR (no unrelated features bundled in)
