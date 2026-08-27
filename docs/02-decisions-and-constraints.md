# Decisions and Constraints

**Status:** Settled input for all design work. Supersedes any contrary assumption.
**Date:** 27 August 2026

This document is the shared brief for every design agent. Read it in full alongside
`01-feasibility-assessment.md` before producing anything.

---

## 1. What we are building

A private household financial ledger for **two adults** — referred to here as **Partner A**
(male, the primary operator) and **Partner B** (his wife) — covering assets and liabilities
across France, Switzerland, Australia and Belgium in EUR, CHF, AUD and USD.

Delivered as **one codebase**: an installable progressive web app serving Android and desktop.

The four views requested, in priority order:

1. **Current cash position** — short-term liquidity, including money-market-type holdings in
   accessible investment accounts, netted against pending credit card liabilities and bills.
2. **Projected earnings, savings and deficits.**
3. **Progress toward savings objectives** — at individual account level and on aggregate.
4. **Tax impact** — as an explicitly-labelled estimation lens, deferred to a later phase.

All four must account for FX. See invariants below.

There is also a third data subject: **Partner A and B's son**, whose Livret A is held under
power of attorney. He is a minor now and will not always be.

---

## 2. Settled decisions

**D1 — Extraction pipeline: redact locally, then extract remotely, then automated four-eyes.**
Personal data is redacted on infrastructure we control *before* any document content leaves it.
A model then performs structured extraction returning a consistent schema. Review is
**automated four-eyes**, not manual sign-off on every figure.

This is a deliberate change from the feasibility document's recommendation of mandatory human
confirmation, and the design must earn it back. The intended reading of "automated four-eyes"
is two independent extraction passes that must agree before a figure is committed, with
disagreement, low confidence, or a materiality breach escalating to a human. Auto-commit is
permitted only when the cross-check agrees *and* the result is plausible against account
history. Silent auto-commit of an unverified number is not acceptable at any confidence level.

**D2 — Both partners can upload any document.** Upload is not access-controlled by account
ownership, and onboarding need not treat ingestion as a privileged operation. Note this
interacts with D3: uploading a document must not become a route to seeing a private account's
contents.

**D3 — Privacy model: private accounts sit outside shared aggregates** (option 1 of the three
in the feasibility assessment). Each partner sees a complete and true picture of their own
position; shared household views cover only jointly-visible accounts and are labelled as
partial. The model must be configurable so the household can move to another option later,
and a change to visibility policy must be visible to both partners.

*Caveat carried forward:* Partner A has accepted this on the household's behalf. It is the one
decision here that warrants explicit confirmation with Partner B before launch, because the
failure mode is domestic rather than technical.

**D4 — Tax context: Partner A is a frontalier, resident in France, working in the Canton of
Geneva.** This is materially different from the other Swiss cantons. Geneva taxes frontaliers
at source (*impôt à la source*); France then taxes worldwide income with a credit mechanism.
Consequences for modelling:

- French tax residence governs: worldwide income reporting, PFU on investment income, PEA and
  assurance vie wrapper rules, IFI on real property.
- Swiss wealth tax does not reach a French resident except on Swiss-situated real property.
  Pillar 2 is untaxed until withdrawal, and its withdrawal treatment is a cross-border question
  in its own right.
- Australian assets and superannuation are held by a non-resident for Australian tax purposes,
  which changes withholding and CGT treatment.
- Foreign account and foreign life-insurance declaration obligations almost certainly apply
  given Swiss, Australian and Belgian holdings.

None of the above is advice, and the app must never present it as such.

**D5 — Instrument pricing is viable for both investment wrappers.** Partner B's Linxea
assurance vie is almost entirely ETFs, so it can be marked to market by ISIN rather than
interpolated between statements. This upgrades it from the feasibility document's tier C.
Still reconcile against the insurer's statement, which is authoritative on unit counts, fees
and any gross/net difference.

---

## 3. Working assumptions (not yet confirmed)

**A1 — Running cost.** Assume a modest monthly budget: managed hosting and paid market data are
acceptable where they buy reliability or save weekend maintenance; five-figure annual licensing
is not. Flag any recommendation whose cost profile is materially above "a streaming
subscription", rather than silently designing around a constraint nobody set.

**A2 — Hosting residency.** EU, given French residence and EU-origin data. D1's local-redaction
requirement means some component must run on infrastructure the household controls; the
boundary between that and any managed platform is an architectural question, not a settled one.

---

## 4. Invariants

These bind every agent. If your design requires breaking one, say so explicitly and argue it —
do not quietly work around it.

**INV1 — Native currency is the source of truth.** Every amount is stored in the currency it
actually exists in. A converted value is never persisted. Conversion is a display-time function
of amount, currency, date and the viewer's display currency.

**INV2 — Display currency is per user, and possibly per view.** Partner A may think in CHF,
Partner B in EUR. A single global setting is a defect.

**INV3 — Every figure carries provenance.** Source document or feed, as-of date, and how it was
derived. Any number on screen must be traceable to where it came from.

**INV4 — Freshness is a property of every figure, and staleness is visible.** An expired
connection renders as *unknown*, never as a stale balance. PSD2 links require re-authentication
roughly every 180 days; assume they will silently expire.

**INV5 — The app is fully usable with zero live feeds.** Only 4 of 26 account lines have an API.
Manual and document-driven paths are the primary experience, not a degraded fallback.

**INV6 — No bank credentials are ever stored.** Only OAuth/PSD2 tokens and uploaded documents.

**INV7 — Ingestion is idempotent.** Re-uploading a statement must never double-count. Key on
institution, account, period and document hash.

**INV8 — Redaction precedes egress.** No unredacted document content leaves controlled
infrastructure, per D1.

**INV9 — Monetary arithmetic uses exact types.** Integer minor units or decimal. Never binary
floating point.

**INV10 — Net worth change is decomposable** into contribution, market movement and FX. A
EUR/CHF swing must never read as savings progress.

**INV11 — Aggregation is a privilege, not a convenience.** This app maps an entire household's
finances in one place, making it a more attractive target than any individual bank login. The
threat model includes a compromised partner device and a compromised host, not only an external
attacker.

---

## 5. Threat model seed

Shared input for the two security agents so their outputs do not contradict. Both must work
from this list; either may extend it.

| Actor | Capability | Primary concern |
|---|---|---|
| External attacker | Network access, credential stuffing, stolen session | Full household financial map; identity data in statements |
| Compromised partner device | Authenticated session, local storage | Access to the *other* partner's private accounts |
| Curious partner | Legitimate credentials, ordinary UI | Deriving a private balance by arithmetic (see D3) |
| Hosting provider / insider | Storage and database access at rest | Statement contents, identity data, full position |
| Extraction model provider | Whatever is sent to it | Residual identity data surviving redaction (see D1, INV8) |
| Future subject | The son, on reaching majority | His right of access to, and erasure of, his own account data |

---

## 6. Agent remits

Six agents work in parallel from this brief. Each owns its scope and decides its own approach;
each writes one design document into `docs/`.

| Agent | Owns | Output |
|---|---|---|
| Security by design | Threat model, authn/authz, the two-user privacy model and its enforcement, session and device security | `docs/10-security-by-design.md` |
| Data security | Encryption at rest and in transit, key management, document storage and retention, redaction pipeline, third-party exposure, subject rights | `docs/11-data-security.md` |
| Architecture | System shape, aggregator adapter boundary, ingestion pipeline, deployment topology, resilience with all feeds down | `docs/12-architecture.md` |
| Financial correctness | Ledger semantics, multi-currency valuation, reconciliation, idempotent ingestion, projection and goal maths, tax lens structure | `docs/13-financial-correctness.md` |
| Ease of use | Onboarding, the ingestion and review flow, staleness and reminder experience for two different people | `docs/14-ease-of-use.md` |
| Graphical presentation | The four views, information hierarchy, making freshness and FX decomposition legible at a glance | `docs/15-presentation.md` |

**Boundaries that matter:** Security-by-design owns *who may see what and how that is enforced*;
data security owns *how data is protected at rest, in transit and in third-party hands*. Where
they meet — the privacy model's cryptographic enforcement — security-by-design sets the
requirement and data security specifies the mechanism.

Financial correctness owns *what the numbers mean*; architecture owns *how they move*. Where
they meet — the ledger schema — financial correctness sets the semantics and architecture
places it in the system.
