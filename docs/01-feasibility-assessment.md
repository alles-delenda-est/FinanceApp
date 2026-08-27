# Cross-Border Household Ledger — Feasibility Assessment

**Status:** Step 1 of 2 (feasibility). Step 2 — the comprehensive multi-agent plan — follows discussion of this document.
**Date:** 27 August 2026
**Published version:** https://claude.ai/code/artifact/c8b202db-32d0-4e3a-ba7d-68a46d5f69cd

---

## Verdict

Everything requested is buildable. Almost none of it is buildable the way a personal-finance
aggregator normally works.

Of **26 account lines across 11 institutions in 4 jurisdictions**:

| Tier | Count | Meaning |
|---|---|---|
| A — live API feed | 4 | Automatable today, zero licensing cost, via EU open banking |
| B — structured export | 17 | No API, but a predictable download cycle. Semi-automatable |
| C — manual / modelled | 5 | Moves slowly, arrives annually, or is computed from known terms |

The correct framing is **a household financial ledger with a few automated feeds attached**, not a
connection-driven aggregator. Designing for a 4/22 split succeeds; designing for full automation
stalls in month three.

This is acceptable. The value described — one true cross-currency picture, actionable liquidity,
savings progress, projections — comes from the ledger and the views, not the plumbing. The plumbing
is the least valuable part, and this household is in an unusually poor position to automate it.

---

## Account-line inventory

### Crédit Agricole des Savoie (France)

| Line | Ccy | Route | Cadence | Tier |
|---|---|---|---|---|
| Joint current account | EUR | PSD2 API via licensed aggregator | Daily, automatic | A |
| Livret A ×3 (hers / yours / son's under PoA) | EUR | PSD2 *if exposed*, else manual | Twice yearly | B |
| Mortgage | EUR | Modelled from loan terms | Computed daily | C |

Livret A is not a payment account and therefore outside the PSD2 mandate, though French banks often
surface livrets anyway — test at link time. The fallback costs little since the balance only moves on
interest. Credit is out of PSD2 scope entirely; model the amortisation schedule from the contract and
reconcile against the annual statement.

### Bank Julius Baer (Switzerland)

| Line | Ccy | Route | Cadence | Tier |
|---|---|---|---|---|
| Current accounts — yours EUR/USD/CHF, hers EUR/CHF | 5 lines | e-banking document download | Monthly | B |
| Investment account | Multi | Portfolio statement download | Monthly/quarterly | B |

Switzerland has deliberately declined to mandate open banking. No aggregator can compel access. Take
investment valuations from the bank's own statement rather than pricing positions independently — it
is authoritative and already handles FX.

### Commonwealth Bank of Australia

| Line | Ccy | Route | Cadence | Tier |
|---|---|---|---|---|
| Transaction account | AUD | NetBank export — CSV/OFX/QIF | Monthly | B |
| Credit cards ×2 (yours, hers) | AUD | NetBank export | Monthly, pre-due-date | B |

CDR access requires ACCC accreditation or a paid sponsorship arrangement — disproportionate for a
household of two. Practical export limits: 600 transactions, two years, desktop NetBank only (the
mobile app has no export), and large date ranges fail silently — pull in three-month slices.

### Neobanks (EEA licensed)

| Line | Ccy | Route | Cadence | Tier |
|---|---|---|---|---|
| Revolut — yours (CH profile) | Multi | PSD2 API | Daily, automatic | A |
| Revolut — hers (FR) | EUR | PSD2 API | Daily, automatic | A |
| Wise — yours (BE) | Multi | PSD2 API, **not** Wise's own API | Daily, automatic | A |

The Swiss-facing Revolut profile is provided by Revolut Bank UAB (Lithuania), so EEA rules should
reach it — verify first. Counter-intuitively, Wise's *personal API token* explicitly cannot read
balances or statements for EU/UK accounts, precisely because of PSD2; reach Wise as a bank instead.

### Investment wrappers (France)

| Line | Ccy | Route | Cadence | Tier |
|---|---|---|---|---|
| Bourse Direct PEA | EUR | Position export + market prices | Positions monthly, prices daily | B |
| Linxea assurance vie (hers) | EUR | Insurer statement | Quarterly/annual | C |

Do not chase NAVs for unit-linked funds — free ISIN coverage of French UCITS is patchy and the
insurer's valuation is authoritative. Interpolate between statements and label it as interpolated.

### Cards, pensions, property

| Line | Ccy | Route | Cadence | Tier |
|---|---|---|---|---|
| TopCard ×4 (yours CHF+EUR, hers CHF+EUR) | CHF/EUR | Monthly PDF statement | Monthly | B |
| Swiss Pillar 2 | CHF | Annual certificate | Yearly, Q1 | C |
| Hostplus superannuation | AUD | Member statement | Quarterly/annual | C |
| The house (and future purchases) | EUR | DVF comparables + manual override | Annual | C |

Superannuation is not a designated CDR sector, so no open-banking route exists even in principle.
France publishes every notarised sale price as open data (DVF), covering Savoie — use it as a sanity
anchor, revalue annually, and never let an estimate move the net-worth line month to month.

---

## Obstacles and workarounds

**EU open banking is licensed, and the licence is expensive.** Calling a PSD2 endpoint requires being
a regulated third-party provider holding an eIDAS certificate (~€3,000–8,000/year).
*Workaround:* use a licensed aggregator's **restricted production** mode. Enable Banking offers this
free — you operate under their AISP licence, whitelisted to accounts you personally authenticate to.
That is exactly this use case. Verify terms at signup before the architecture depends on it.

**The obvious free option just closed.** GoCardless Bank Account Data (formerly Nordigen) has stopped
onboarding new users; free open-banking tiers are contracting industry-wide.
*Workaround:* Enable Banking is the closest self-serve replacement. More importantly, treat any
aggregator as a component that will be discontinued and put a thin adapter interface behind it. This
is not hypothetical — it already happened once.

**Switzerland has no open banking mandate.** A deliberate policy choice, not a coverage gap. Six
account lines sit behind this.
*Workaround:* accept it and build excellent document ingestion. Statement download is a first-class
data path here, and is arguably more stable than an API.

**Australian CDR is gated behind accreditation.** *Workaround:* NetBank export on a monthly cycle.

**Connections go stale every six months.** PSD2 links require strong customer authentication roughly
every 180 days; live feeds silently expire twice a year, per connection.
*Workaround:* treat freshness as a first-class property of every figure. A stale connection must
render as *unknown* with a re-link prompt, never as a stale balance. Silent staleness destroys trust
permanently.

**Ten statement formats, four languages, permanent drift.** Hand-written parsers are a maintenance tax
forever and break silently on redesign — the worst failure mode for financial data.
*Workaround:* model-based extraction against a strict schema with a **mandatory human confirmation
step**. Propose figures beside the source page; approve before committing. Never auto-commit an
extracted number. Store and hash source documents for idempotency, and make every figure traceable to
the page it came from.

> **Decision needed early:** model-based extraction means statements leave your infrastructure. Three
> coherent positions: accept it under a no-training commitment; crop and redact before sending; or run
> a local model at home. This changes the deployment shape, so it must be settled before architecture
> work starts.

---

## The hard part: independent visibility is an arithmetic problem

Per-account permissions are trivial. The problem is what follows.

If an account is hidden from one partner but still counted in household net worth, its balance is
recoverable by subtracting the visible accounts — and its *movement over time* is observable, which is
more revealing than a single balance. No engineering fixes this: a total containing a hidden number
leaks that number to anyone who knows the other terms.

Three coherent options, to be chosen explicitly and together:

1. **Private accounts sit outside shared totals** *(recommended)* — each person sees a true, complete
   picture of their own position; the shared view covers only jointly-visible accounts and is honestly
   labelled partial. Nothing leaks, nobody is misled.
2. **Included but coarsened** — private accounts contribute to totals that are banded or rounded so
   small movements can't be inverted. Preserves a true-ish total; costs precision; reduces rather than
   eliminates the leak.
3. **Included, leak accepted** — full accuracy, with both parties explicitly informed the sum is
   derivable. Reasonable if it is a mutual, informed decision rather than an assumption baked in by
   the software.

Two rules hold regardless. The privacy policy must not be changeable by one person without the other
seeing that it changed. And **the son's Livret A deserves separate treatment** — it is a third
person's financial data held under power of attorney, he will eventually be an adult with his own view
on it, and it should carry its own retention and access story.

This is the requirement most likely to cause real trouble and least likely to be caught by any
technical agent. It is not a security bug; it is a family design decision wearing one's clothing.

---

## FX

ECB reference rates cover EUR, CHF, AUD and USD, free without an API key, back to 1999, daily on
business days — the right resolution for a household.

The correctness risk reduces to one rule: **store every amount in the currency it actually exists in
and never persist a converted value.** Conversion is a display-time function of amount, currency, date
and the viewer's display currency. Persisting a converted figure silently corrupts history.

Three consequences to build in from the start:

- **Display currency is per user, possibly per view.** One partner may think in CHF, the other in EUR;
  the mortgage is a EUR problem and the super an AUD one.
- **Decompose the change.** When net worth moves, say how much was contribution, market and pure FX.
  Otherwise a EUR/CHF swing reads as savings progress and projections quietly lie.
- **Pick and record a rate convention.** Which day's rate, for valuations and transactions, is
  arbitrary but must be consistent and stated, or two views will disagree.

---

## Tax

French residence, Swiss pension and employment exposure, Australian assets and superannuation, plus EU
wrappers with their own regimes (PEA five-year rule, assurance vie eight-year abatement, PFU, IFI,
Swiss wealth tax, Australian CGT and preservation). Modelling this is possible; modelling it correctly
across three interacting treaty relationships is a specialist project.

**Recommendation:** build tax as a pluggable, clearly-labelled estimation lens with configurable
assumptions, shipped after the core ledger works. A wrong tax number presented confidently is worse
than none, because decisions get made on it.

**Useful side effect:** French tax residents are likely required to declare foreign accounts annually,
and this household holds Swiss, Australian and Belgian accounts plus foreign life insurance. The app
will already hold exactly the list needed. Worth confirming the precise obligation with an accountant,
but it may be the most valuable "tax" feature available and is far simpler than liability modelling.

---

## Platform

Android and web from a single codebase as an installable progressive web app. Android supports
installation and push notifications for update reminders; tables, charts, file upload and
review-and-confirm flows are all web-native. Two codebases would double the cost of every view.

The one strong reason to wrap natively later is the **Android share target** — sharing a statement PDF
from a bank app directly into the ledger removes the largest friction point in a document-driven
system. Worth doing, but after the thing it feeds exists.

EU hosting residency is the natural default. Managed vs. self-hosted depends on the document-extraction
decision above.

---

## Build order

Each phase either proves something that could invalidate the design, or delivers something usable.

1. **Prove the connection before building around it.** Restricted-production signup; link one Revolut
   account; confirm terms permit this; confirm what Crédit Agricole exposes for a Livret A. A week that
   determines whether tier one is four accounts or one.
2. **The multi-currency ledger and its two logins.** Accounts, balances, history, per-user permissions
   under the chosen privacy model, display-time FX. Fully manual entry at this stage. If this layer is
   wrong, no automation saves it.
3. **Document ingestion with human confirmation.** Upload, extract, review, approve. Start with TopCard
   and Julius Baer — highest volume, most tedious, clearest payoff.
4. **The requested views.** Current liquidity including pending card liabilities, savings goals per
   account and aggregate, projected surplus/deficit.
5. **Live feeds and the staleness engine.** Wire in the four automated connections, per-account refresh
   cadences, reminders addressed to the right person, a clear "what needs updating" surface.
   Deliberately after the manual path works — the app must be fully usable when every feed is down.
6. **Tax lens, property revaluation, refinement.** Once numbers have been trustworthy for months.

---

## Open questions (blocking step 2)

1. **Tax residency, precisely.** French resident working in Switzerland — frontalier or otherwise?
   Determines the tax lens, reporting obligations, and default household currency.
2. **Which privacy model, agreed by both partners.** Shapes the data model at its foundation.
3. **Where statements may be processed.** Hosted, redacted, or fully local extraction. Determines
   deployment architecture.
4. **Who operates the connections.** Does each partner link and maintain their own accounts, or is one
   the household administrator? Interacts with question 2.
5. **Running cost tolerance.** A near-zero stack is achievable; modest monthly spend buys managed
   hosting, better market data, and less weekend maintenance.

---

## Proposed agent roster for step 2

The five proposed agents are well chosen. Two adjustments:

**Security by design and data security overlap heavily** and will contradict each other if run
independently. Give them a shared threat model as common input, with distinct remits — one owning
authentication, authorisation and the privacy model; the other owning encryption, retention, document
handling, and device loss.

**Nothing in the roster owns financial correctness**, which on this evidence is the highest-risk area
in the system. An architecture agent will design a fine system that computes the wrong number.

| Agent | Remit |
|---|---|
| Security by design | Threat model, authn/authz, two-user privacy model, session and device security |
| Data security | Encryption at rest/in transit, key management, document storage, retention, third-party exposure |
| Architecture | System shape, aggregator adapter boundary, deployment, resilience when every feed is down |
| Ease of use | Onboarding, review-and-confirm ingestion flow, staleness and reminder experience for two people |
| Graphical presentation | Liquidity, projection and goal-progress views; making freshness and FX decomposition legible |
| **Financial correctness** *(addition)* | Ledger semantics, multi-currency valuation, reconciliation, idempotent ingestion, projection maths |

All agents should receive this assessment in full plus the answers to the five open questions —
including constraints they might otherwise design around, such as the 4/22 automation split and the
arithmetic leak. Agents given partial context solve the problem they were shown rather than the one
that exists.

---

*Institutional access routes verified August 2026. Aggregator terms, free-tier availability and export
limits change frequently — the restricted-production route in particular should be confirmed at signup
before the architecture depends on it. Tax observations are structural, not advice.*
