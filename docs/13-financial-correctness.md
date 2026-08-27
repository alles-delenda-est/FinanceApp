# Financial Correctness — Ledger Semantics, Valuation, Reconciliation and the Tax Lens

**Status:** Design. Binding on the meaning of every number the app displays.
**Owner:** Financial correctness agent
**Date:** 27 August 2026
**Inputs:** `01-feasibility-assessment.md`, `02-decisions-and-constraints.md` (D1–D5, INV1–INV11, threat model seed)
**Boundary:** This document defines *what the numbers mean*. `12-architecture.md` defines *how they move*.
Where a schema appears here it is a semantic contract, not a storage prescription — the architecture
agent chooses tables, services and transport.

> **⚠ INCOMPLETE — truncated at section 4.** The authoring agent was terminated by an API
> spend limit partway through writing. Sections 0–4 (ledger model, money/currency/FX, the
> current cash position) are written and usable. Still missing: reconciliation and idempotency,
> interpolation and staleness maths, projection and goal maths, and the tax lens structure.
> Do not treat this document as a complete specification of financial correctness.

---

## 0. Summary of the calls made

Twelve decisions, each argued below. If you read nothing else, read this.

| # | Decision | Section |
|---|---|---|
| **F1** | The ledger is **double-entry postings with first-class balance assertions and typed reconciliation postings**. Not balances-with-history, not pure double-entry. | [2](#2-the-ledger-model) |
| **F2** | Every transaction **balances within a single currency**. Cross-currency movements are two single-currency legs through a clearing account. There is no such thing as an FX posting. | [2.5](#25-cross-currency-movements-and-the-fx-clearing-account) |
| **F3** | Reconciliation postings are **derived artefacts**, recomputed deterministically on every ingest, never stored as user data. This is what makes INV7 hold under out-of-order arrival. | [2.6](#26-balance-assertions-and-the-reconciliation-posting) |
| **F4** | FX convention: **ECB reference rate for the figure's own date, last-observation-carried-forward across non-publication days, never interpolated, never forward-looking.** Flows use their own transaction date. | [3.2](#32-the-seven-rate-rules) |
| **F5** | INV10 decomposition: **contribution and return are valued at their own transaction-date rates; FX is the residual.** A fourth term, *unexplained*, is always shown and never folded into market. | [3.5](#35-inv10-decomposing-the-change) |
| **F6** | Cash position headline = **settled cash + cash-equivalents − all card liability − committed outflows within 30 days.** Sellable securities are a separate band, never in the headline. | [4](#4-the-current-cash-position) |
| **F7** | A holding inside the Julius Baer investment account is cash-equivalent only if it passes an explicit four-part test. **Default when unknown is *not* cash-equivalent.** | [4.3](#43-the-julius-baer-investment-account-the-hard-call) |
| **F8** | Card liability includes **statemented + post-statement posted + pending**, and for statement-only cards a separately-labelled **modelled unbilled** estimate, included by default. | [4.4](#44-six-credit-cards-in-three-currencies) |
| **F9** | Documents are idempotent on **content hash**, then on **logical period key**, then reconciled at transaction level by **multiset**, not set. Periods are replaced wholesale, never merged. | [5](#5-reconciliation-and-idempotency-inv7) |
| **F10** | Confidence **decays by the account's own observed volatility**, not by an arbitrary time constant. A dead feed is a **step function to unknown**, not a decay curve. | [6](#6-interpolation-staleness-and-confidence-inv4) |
| **F11** | Projections use **block-bootstrap** uncertainty bands, refuse to project beyond 3× the available history, and **never forecast FX** — FX is a separate, separately-drawn sensitivity band. | [7](#7-projections-and-goals) |
| **F12** | The tax lens is a **pure function over versioned rule packs**. Ship the **declaration-support mode (3916 / 3916-bis) first**; it is mechanical, verifiable and high value. Liability estimation ships behind it. | [8](#8-the-tax-lens) |

One invariant exception is declared, in [§9.1](#91-inv1-the-one-declared-exception).
Two additional invariants are proposed, in [§9.3](#93-two-invariants-i-want-to-add).

---

## 1. What the data actually is

The feasibility assessment classifies the 26 lines by *access route* (A/B/C). That is the right
taxonomy for the ingestion team and the wrong one for the ledger. A CBA CSV export and a Revolut
PSD2 feed are both tier-differentiated but both deliver **itemised dated movements**. A Hostplus
annual statement and a DVF property estimate are both tier C but mean completely different things.

The ledger cares about **observation type**:

| Type | Meaning | Lines |
|---|---|---|
| **O1 Posting stream** | Dated, signed, itemised movements. May arrive late and in batches. | CA joint current, Revolut ×2, Wise, CBA transaction, CBA cards ×2, TopCard ×4, JB current ×5 |
| **O2 Balance assertion** | One figure, true at one instant. No explanation of how it got there. | Livret A ×3, Pillar 2 certificate, Hostplus statement, every statement footer |
| **O3 Position set** | Quantity × instrument at an instant, priced by someone. | JB investment, Bourse Direct PEA, Linxea assurance vie |
| **O4 Model output** | Computed from contractual terms we hold. | Mortgage amortisation, Livret A interest accrual |
| **O5 External estimate** | Someone else's opinion of value. | The house (DVF comparables + override) |

Two facts follow, and they drive everything:

1. **Most account lines emit more than one observation type.** A Julius Baer current-account
   statement is simultaneously O1 (the movement list) and O2 (the closing balance). A Hostplus
   statement is O2 (the member balance) *and*, if we parse its breakdown, O1 (contributions,
   earnings, fees, insurance premiums, contributions tax). The ledger must accept all of them and
   let them corroborate each other — that corroboration is the single most valuable correctness
   mechanism available and it is free.
2. **Posting coverage is patchy, and the patchiness is knowable per account per period.** For the
   CA joint current account in a live-feed month, we have every movement. For a Livret A between
   January and July, we have none. A ledger that cannot represent "I know the balance but not how
   it got there" will either fabricate movements or lose the balance. Both are disqualifying.

So: **design for O1+O2 corroboration with explicit, typed ignorance.**

---

## 2. The ledger model

### 2.1 The call

**Double-entry postings, plus first-class balance assertions, plus typed reconciliation postings
that make the two agree.**

Rejected alternatives, and why:

- **Balances with history** (a time series of numbers per account). Cannot express that a €2,000
  move from the CA joint account to the PEA is not a €2,000 change in net worth. Makes INV10
  impossible — you cannot decompose a change you never observed the cause of. Makes transfers
  between the household's own accounts look like income and expenditure. Fails immediately.
- **Full double-entry, postings only.** Requires a movement behind every change. We do not have
  one for 11 of 26 lines. The alternative is to invent movements from balance deltas, which is
  exactly the fabrication we must avoid, or to refuse the data, which is INV5's opposite.
- **The hybrid above.** Keeps double-entry's arithmetic guarantees (transfers net to zero; the
  household's net worth changes only when something crosses its perimeter or an asset revalues)
  while representing ignorance honestly and in a typed way that INV10 can consume.

The lineage is Beancount's `balance` and `pad` directives. The departure is that our padding is
**typed** — the contra account records *why* we could not explain the delta, and that reason is
what feeds the contribution / market / FX / unexplained split. A generic "opening balances" plug
would destroy INV10.

### 2.2 Chart of accounts

Two kinds of account, and the distinction is load-bearing.

**Real accounts** — things the household owns or owes. One per account line, sub-divided by
currency, and for investment wrappers by instrument lot.

```
ASSET:CA:JOINT-CURRENT:EUR
ASSET:CA:LIVRET-A:PARTNER-A:EUR
ASSET:CA:LIVRET-A:SON:EUR          # separate subject — see §4.6
ASSET:JB:CURRENT:PARTNER-A:CHF
ASSET:JB:CURRENT:PARTNER-A:EUR
ASSET:JB:CURRENT:PARTNER-A:USD
ASSET:JB:INVEST:PARTNER-A:<instrument>:<lot-id>
ASSET:BOURSEDIRECT:PEA:PARTNER-A:<isin>:<lot-id>
ASSET:LINXEA:AV:PARTNER-B:<isin>:<lot-id>
ASSET:CBA:TRANSACTION:AUD
ASSET:HOSTPLUS:SUPER:PARTNER-?:AUD
ASSET:PILLAR2:PARTNER-A:CHF
ASSET:PROPERTY:MAISON:EUR
LIABILITY:TOPCARD:PARTNER-A:CHF
LIABILITY:TOPCARD:PARTNER-A:EUR
LIABILITY:CBA:CARD:PARTNER-B:AUD
LIABILITY:CA:MORTGAGE:EUR
```

**Perimeter accounts** — the boundary of the household. Every posting to a perimeter account is a
statement that value entered or left the household's control, or that an asset changed value on
its own. Each carries a mandatory `flow_class`.

```
flow_class = CONTRIBUTION            # the household's own saving/dissaving behaviour
  INCOME:SALARY
  INCOME:GIFT
  EXPENSE:LIVING:<category>
  EXPENSE:TAX:INCOME
  EXPENSE:TAX:PROPERTY
  EXPENSE:INSURANCE:BORROWER          # assurance emprunteur — a cost of borrowing, not of the debt
  EXPENSE:FX-COST                     # realised spread and fees on conversions

flow_class = RETURN                  # what the assets did on their own
  INCOME:INTEREST
  INCOME:DIVIDEND:GROSS
  EXPENSE:WITHHOLDING:<jurisdiction>  # Swiss impôt anticipé, Australian WHT — see §8.6
  EXPENSE:MORTGAGE-INTEREST
  EXPENSE:ACCOUNT-FEES
  VALUATION:MARKET                    # unrealised revaluation of a position
  VALUATION:MARKET:ADJUSTMENT         # snap to custodian's authoritative mark — §5.6

flow_class = UNEXPLAINED             # honest ignorance; never silently absorbed
  RECON:UNEXPLAINED
  RECON:MODEL-DRIFT
  RECON:ROUNDING

flow_class = UNOBSERVED              # known cause, unknown detail
  RECON:UNOBSERVED-CONTRIBUTION
  RECON:UNOBSERVED-RETURN

# internal, flow_class = NONE — never crosses the perimeter
  CLEARING:TRANSFER-IN-FLIGHT
  CLEARING:FX:<conversion-id>
```

`flow_class` on the perimeter account, not on the posting direction, is the whole trick. Interest
received on a Livret A and mortgage interest paid are both `RETURN` — the assets doing their thing.
Salary received and groceries bought are both `CONTRIBUTION` — the household's behaviour. Getting
this backwards (classifying by sign) produces a decomposition where paying down debt looks like
market movement.

### 2.3 Core records

```
Posting {
  posting_id        uuid
  txn_id            uuid            # postings group into transactions
  account           account_ref
  amount            Money           # signed, native currency, exact
  booking_date      date            # the date the institution booked it
  value_date        date?           # where it differs and matters (Livret A quinzaines)
  seq               int             # statement line order within a day; never used for correctness
  source            provenance_ref  # INV3
  derived           bool            # true for reconciliation and model postings
  estimated         bool            # true if produced by interpolation or a model, not observed
}

Transaction {
  txn_id            uuid
  description       text
  natural_key       bytes           # §5.3
  tags              set<text>
  # invariant: for each currency c, sum of postings in c == 0
}

BalanceAssertion {
  assertion_id      uuid
  account           account_ref
  currency          ccy
  as_of             date            # end of this day, account's home timezone
  amount            Money
  source            provenance_ref
  status            PENDING | CONFIRMED | RECONCILED | CONTRADICTED
}

Provenance {                        # INV3 — attached to every figure that reaches a screen
  kind              FEED | DOCUMENT | MODEL | MANUAL | DERIVED
  document_id       uuid?           # content-addressed; §5.1
  page              int?
  region            bbox?           # so the UI can show the figure beside the pixels it came from
  feed_connection   uuid?
  model_version     text?
  extracted_at      timestamp
  confirmed_by      HUMAN(user) | FOUR-EYES | AUTO
}
```

Instrument positions are held as **lots**, not as net quantities, because the tax lens needs
acquisition dates and cost basis and cannot recover them from a net figure:

```
Lot {
  lot_id            uuid
  account           account_ref
  instrument        isin | internal_id
  quantity          Quantity        # decimal(28,8)
  acquired_on       date
  cost_native       Money           # what was actually paid, in the currency paid
  cost_basis_eur    Money           # frozen at acquisition — see §9.1, the declared INV1 exception
  cost_basis_rate   Decimal         # the rate used, so cost_basis_eur is reproducible
  cost_basis_rate_src text
  disposal          {on: date, proceeds_native: Money, method: FIFO|SPECIFIC}?
}
```

Disposal method: **FIFO by default**, because French plus-value rules for securities held outside a
wrapper use a weighted-average cost (`prix moyen pondéré d'acquisition`) and Australian rules permit
specific identification. Store lots; compute whichever basis the rule pack asks for. Never store a
single blended cost — you cannot recover lots from it.

### 2.4 Balance derivation

```
balance(account, currency, D) =
    Σ { p.amount : p.account = account, p.currency = currency, p.booking_date ≤ D }
```

That is the *only* definition. There is no stored balance field anywhere. A balance is always a
fold over postings, and reconciliation postings are part of that fold. This makes late-arriving and
out-of-order data automatically correct, and makes the whole ledger reproducible from its
observations — see the proposed INV12 in §9.3.

Performance is the architecture agent's problem, not a reason to store balances. If a materialised
snapshot is needed, it must be a cache keyed by `(account, currency, date, ledger_version)` that is
invalidated, never reconciled.

### 2.5 Cross-currency movements and the FX clearing account

**Every transaction balances within each currency it touches.** There is no multi-currency
transaction that balances "at the rate of the day", because that would persist a converted value
and break INV1.

A Wise conversion of 5,000 CHF into 5,340 EUR on 12 July becomes:

```
txn 4f2a  "Wise CHF→EUR"
  ASSET:WISE:PARTNER-A:CHF        −5,000.00 CHF
  CLEARING:FX:4f2a                +5,000.00 CHF
  CLEARING:FX:4f2a                −5,340.00 EUR
  ASSET:WISE:PARTNER-A:EUR        +5,340.00 EUR
```

Balances per currency: CHF nets to zero, EUR nets to zero. The clearing account now holds
+5,000 CHF and −5,340 EUR. Valued at the ECB reference rate for 12 July, its residual is the
**realised FX cost** — the difference between the executed rate and the reference rate, which is
exactly the spread and fee the household paid. Post it, on the same date, in the currency it was
suffered in:

```
  CLEARING:FX:4f2a                −5,000.00 CHF
  CLEARING:FX:4f2a                +5,340.00 EUR
  EXPENSE:FX-COST                 (the residual, in the source currency)
```

after which the clearing account is flat. This gives the household a real, cumulative,
per-provider "what conversions cost us" figure, which for a four-country household is a number
worth having and which no aggregator will ever show them.

The executed rate is stored on the transaction as a fact (`executed_rate`, `executed_rate_source`).
It is not the same object as the reference rate and must never be written into the rate table.

**Transfers between institutions** that take days (CA → Julius Baer) use
`CLEARING:TRANSFER-IN-FLIGHT`. Debit it on the sending date, credit it on the arrival date. In
between, the money is visibly in flight rather than missing. Unmatched in-flight balances older
than 10 business days raise a discrepancy.

### 2.6 Balance assertions and the reconciliation posting

The mechanism that lets O1 and O2 coexist.

**Algorithm — run on every ingest, from scratch, for the affected account:**

```
for each assertion A on (account, currency), in as_of order:
    B_derived = balance(account, currency, A.as_of)   # excluding derived reconciliation postings
                                                      # generated by A itself or any later assertion
    delta = A.amount − B_derived

    if delta == 0:
        A.status = CONFIRMED
        emit nothing

    else if |delta| ≤ rounding_tolerance(account):     # typically 0.02 in minor units × line count
        A.status = RECONCILED
        emit derived posting: (account +delta) / (RECON:ROUNDING −delta), dated A.as_of

    else:
        contra = choose_contra(account, [prev_confirmed.as_of, A.as_of])
        A.status = RECONCILED if contra.flow_class == UNOBSERVED else CONTRADICTED
        emit derived posting: (account +delta) / (contra −delta), dated A.as_of
        if A.status == CONTRADICTED or |delta| > materiality(account):
            raise Discrepancy(account, A, delta, contra, diagnosis())
```

`choose_contra` is where the honesty lives:

| Posting coverage over the interval | Account class | Contra | Meaning |
|---|---|---|---|
| **Complete** (live feed, no gaps, no expiry) | any | `RECON:UNEXPLAINED` | A real error. We saw every movement and the balance still disagrees. Always escalates. |
| Incomplete | Livret A | `INCOME:INTEREST` | The only thing that moves a Livret A between statements is interest, and we can check that against the modelled accrual (§6.2). |
| Incomplete | Current / card | `RECON:UNOBSERVED-CONTRIBUTION` | Unseen spending or income. |
| Incomplete | Investment (positions unknown) | `RECON:UNOBSERVED-RETURN` | Market movement we could not itemise. |
| Incomplete | Pension / super | *prefer the statement's own breakdown* | See below. |
| Incomplete | Mortgage | `RECON:MODEL-DRIFT` | §5.7. |
| Incomplete | Property | `VALUATION:MARKET` (never auto-committed) | §6.1, `FROZEN` policy. |

**Prefer the document's own decomposition over a plug.** A Hostplus member statement itemises
opening balance, employer contributions, personal contributions, investment earnings, fees,
insurance premiums and contributions tax. A Swiss Pillar 2 certificate gives the opening *avoir de
vieillesse*, employee and employer contributions, interest credited, and the closing balance. Parse
those into explicit `CONTRIBUTION` and `RETURN` postings. This turns an annual O2 observation into
a proper INV10 decomposition once a year, for free, from a document we already have. Plug only when
the document genuinely gives one number.

**Reconciliation postings are derived.** They carry `derived = true`, are keyed by
`(assertion_id)`, and are deleted and regenerated in full whenever anything in their interval
changes. Concretely: if the July Julius Baer statement produced a €1,400 `RECON:UNOBSERVED-RETURN`
plug, and in September we ingest the actual portfolio movement list for July, the plug shrinks to
zero automatically on the next recompute. **This is the property that makes INV7 hold under
out-of-order arrival**, and it is the reason nothing derived may ever be stored as authoritative.

---
## 3. Money, currency and FX

### 3.1 Types (INV9)

```
Money    = { minor: int64, currency: ccy }      # scale from ISO 4217 exponent; 2 for EUR/CHF/AUD/USD
Quantity = decimal(28, 8)                        # instrument units — Linxea unit-linked quotes 4–6 dp
Price    = { value: decimal(28,10), currency }   # NAVs and market prices
Rate     = decimal(28,10)                        # ECB publishes 4–5 dp; cross-rates need the headroom
```

No binary floating point anywhere in the pipeline, including in the extraction adapters and the
projection engine. A `float` in a bootstrap resampler is as much a defect as a `float` in a balance.

**Rounding.** Round-half-away-from-zero (`ROUND_HALF_UP` in decimal terms), applied **once**, at
the point a figure becomes pixels. Never round an intermediate. A conversion is a single decimal
multiplication from native minor units to display minor units; conversions are never chained.

**Aggregation and the penny problem.** Summing exact converted decimals and rounding once gives the
mathematically correct total, but the displayed components will then sometimes not add up to it. In
a household ledger, columns that do not add up destroy trust faster than a one-cent inaccuracy.

*Rule:* compute the exact total and round it; round each component; distribute the residual minor
units to the components with the largest fractional remainders (largest-remainder / Hamilton
apportionment), preserving sign. **This applies to the display projection only.** Any figure feeding
further computation uses the unrounded decimal. A displayed table always adds up; the ledger is
never touched.

### 3.2 The seven rate rules

Rates come from the **ECB euro foreign exchange reference rates**, published around 16:00 CET on
each working day, and **not published on TARGET closing days**. TARGET closes on Saturdays, Sundays,
1 January, Good Friday, Easter Monday, 1 May, 25 December and 26 December. All four household
currencies are covered, daily, back to 1999, without an API key.

**R1 — Valuation rate.** A balance or position as of date *D* converts at the reference rate **for
date *D***. If no rate was published for *D*, use the **most recent preceding published rate**
(last observation carried forward).

**R2 — Never interpolate, never look forward.** FX is not interpolated between publication days, and
a rate published *after* *D* is never used to value *D*. Both would make history mutate: a chart
redrawn tomorrow would show different values for last Saturday.

**R3 — Transaction rate.** A flow dated *D* converts at the rate for *D* under R1, **except** where
the transaction carries an actually-executed rate (Revolut, Wise, a card issuer's conversion). Then
the executed rate is a fact about that transaction and is used for that transaction; the reference
rate is used only to measure the spread (§2.5).

**R4 — Today.** Before 16:00 CET there is no rate for today. "Today" converts at yesterday's rate,
and the UI states the rate date. Do not chase intraday quotes; the household is not trading.

**R5 — Cross rates triangulate through EUR, always.**
```
rate(A→B, D) = rate_EUR_per_A(D) / rate_EUR_per_B(D)
             = (1 / ecb[A](D)) / (1 / ecb[B](D))      # ECB quotes units of X per 1 EUR
             = ecb[B](D) / ecb[A](D)
```
computed at full decimal precision with no intermediate rounding. `rate(A→A, D) = 1` exactly, by
short circuit — never by division, which would introduce a rounding artefact.

**R6 — The rate table is append-only and versioned.** The ECB occasionally revises a published
rate. A revision creates a new `rate_version` rather than overwriting. Every report, chart and
export records the `rate_version` it used, so a figure reprinted six months later is bit-identical.
Without this, a rate correction silently rewrites history and INV3 becomes a lie.

**R7 — A missing rate is UNKNOWN, not zero and not stale.** If no rate exists within
`max_rate_staleness` (default 7 calendar days) of the requested date, the converted figure is
`Unknown` and propagates as unknown into every aggregate that contains it (§6.4). Silently reusing
a three-week-old rate is exactly the class of failure INV4 exists to prevent.

**Ingestion.** Both of the following, because they fail differently:
- `https://www.ecb.europa.eu/stats/eurofxref/eurofxref-daily.xml` (today) and
  `eurofxref-hist-90d.xml` / `eurofxref-hist.zip` (backfill) — plain files, no key, very hard to
  break.
- The SDMX API at `https://data-api.ecb.europa.eu/service/data/EXR/D.CHF+AUD+USD.EUR.SP00.A`
  with `startPeriod`, `endPeriod`, `format=csvdata` — for gap-filling and revision detection.

Poll after 16:15 CET on TARGET open days. Missing-rate alarm if the table has no new row for two
consecutive TARGET business days. *(Confirm the exact SDMX parameter list against the ECB Data
Portal at implementation time — the portal's URL scheme has changed once already.)*

### 3.3 Concrete non-business-day consequences

These are not hypotheticals; they will bite in the first year.

- **Statements dated 31 December.** TARGET is open on 31 December unless it falls at a weekend.
  31 Dec 2026 is a Thursday — fine. 31 Dec 2028 is a Sunday — the year-end valuation uses the
  **Friday 29 December** rate. Any year-end figure must therefore record the rate date, not the
  valuation date, or two reports will disagree.
- **1 January valuations (IFI, §8.5).** 1 January is *always* a TARGET closing day. A 1 January
  valuation *always* uses the preceding business day's rate — normally 31 December. Hard-code
  nothing; let R1 handle it and let the figure state its rate date.
- **Easter.** In 2026, Good Friday is 3 April and Easter Monday 6 April. The rate published on
  Thursday 2 April values 2, 3, 4, 5 and 6 April. A four-day flat segment in an FX chart is
  correct, not a bug, and the chart should render it as a step, not a slope.
- **Australian and Swiss holidays are irrelevant to the rate table** but very relevant to
  settlement and value dates. Do not conflate them.

### 3.4 Two currency series, never mixed

Every multi-currency time series must be one of exactly two kinds, and must say which:

- **Then-current** — each point converted at its own date's rate. This is "what our net worth was".
  It moves with FX.
- **Constant-currency** — every point converted at a single pinned rate date. This is "what our
  net worth would have been if rates had not moved". It isolates contribution and market.

Offering both, side by side, is the single most legible answer to INV10 and is the interface the
presentation agent should build against. **Mixing them within one series or one figure is a
defect**, and is the specific failure mode where a CHF/EUR swing reads as savings progress.

### 3.5 INV10: decomposing the change

Over an interval [t₀, t₁], in display currency *d*:

```
ΔNW_d  =  C_d  +  M_d  +  X_d  +  U_d
```

- **C_d — contribution.** Σ over all postings in the interval to perimeter accounts with
  `flow_class = CONTRIBUTION`, each converted at **its own booking date's rate**.
- **M_d — market and return.** Σ over all postings with `flow_class = RETURN` (interest, dividends
  net and gross, withholding, fees, mortgage interest, `VALUATION:MARKET`), each at **its own
  booking date's rate**.
- **U_d — unexplained.** Σ over `UNEXPLAINED` and `UNOBSERVED` postings, at their own dates.
  **Always displayed. Never folded into M.** If it is large, the app's job is to say so, not to
  make the chart look tidy.
- **X_d — FX.** Defined as the residual: `X_d = ΔNW_d − C_d − M_d − U_d`.

Defining FX as the residual is deliberate and correct. It makes the identity hold exactly with no
leftover interaction term, and it means each flow is valued at the moment it happened — which is
what a household means when it says "I put €500 in during March". Two sanity properties fall out:
with no flows, `X_d = ΔNW_d` (pure FX); with no rate change, `X_d = 0` identically.

**X_d decomposes by currency pair** — `X_d = Σ_c X_d(c)` where `X_d(c)` is computed by rerunning
the identity with only currency *c*'s rates allowed to move. This gives the presentation agent
"−€1,240 FX, of which −€1,180 CHF/EUR and −€60 AUD/EUR", which is what a cross-border household
actually wants to see.

**M_d splits into observed and estimated.** `M_observed` comes from priced positions and parsed
statement breakdowns; `M_estimated` comes from `estimated = true` postings produced by
interpolation or modelling (§6.1). Report both. A month whose entire "market movement" is
interpolation should look different from one backed by daily marks.

**Granularity is bounded by observation cadence, and must be admitted.** The Julius Baer investment
account is valued monthly, so a month's market movement is all valued at one day's rate, and the
market/FX split within that month is approximate. The PEA and (per D5) Linxea can be marked daily,
so their split is fine-grained. Attach `decomposition_granularity ∈ {DAILY, MONTHLY, QUARTERLY,
ANNUAL}` to each account's contribution to the decomposition and surface the worst one on any
aggregate. Pretending to a precision the data does not support is the failure this app exists to
avoid.

### 3.6 Worked example — verify your implementation against this

Display currency EUR. Period 1 July → 31 July 2026.

| Account | Native | N₀ | Movements | N₁ |
|---|---|---|---|---|
| JB current (Partner A) | CHF | 40,000.00 | −5,000.00 spending on 10 Jul; +30,000.00 salary on 25 Jul | 65,000.00 |
| CA joint current | EUR | 8,000.00 | −1,500.00 mortgage payment on 5 Jul | 6,500.00 |
| CA mortgage | EUR | −220,000.00 | +1,100.00 principal on 5 Jul | −218,900.00 |
| Bourse Direct PEA | EUR | 30,000.00 | +900.00 revaluation on 31 Jul | 30,900.00 |

The mortgage payment of 1,500.00 EUR splits into 1,100.00 principal (internal), 380.00 interest
(`EXPENSE:MORTGAGE-INTEREST`, RETURN) and 20.00 borrower insurance
(`EXPENSE:INSURANCE:BORROWER`, CONTRIBUTION).

ECB rates (CHF per EUR, and the derived CHF→EUR rate):

| Date | EUR/CHF | rate(CHF→EUR) |
|---|---|---|
| 30 Jun | 0.9400 | 1.0638298 |
| 10 Jul | 0.9350 | 1.0695187 |
| 25 Jul | 0.9250 | 1.0810811 |
| 31 Jul | 0.9200 | 1.0869565 |

```
NW₀ = 40,000×1.0638298 + 8,000 − 220,000 + 30,000  = −139,446.81 EUR
NW₁ = 65,000×1.0869565 + 6,500 − 218,900 + 30,900  = −110,847.83 EUR
ΔNW =                                                  +28,598.98 EUR

C_d = +30,000×1.0810811   (salary, 25 Jul)   = +32,432.43
      −5,000×1.0695187    (spending, 10 Jul) =  −5,347.59
      −20.00              (insurance, 5 Jul) =     −20.00
                                              -------------
                                                +27,064.84

M_d = −380.00  (mortgage interest, 5 Jul)
      +900.00  (PEA revaluation, 31 Jul)      =    +520.00

U_d =                                                   0.00

X_d = 28,598.98 − 27,064.84 − 520.00 − 0.00   =  +1,014.14
```

Independent check of X_d from first principles — opening CHF stock revalued, plus the timing effect
of each CHF flow:

```
  40,000 × (1.0869565 − 1.0638298)  =  +925.07   opening stock
+ 30,000 × (1.0869565 − 1.0810811)  =  +176.26   salary, held 25–31 Jul
−  5,000 × (1.0869565 − 1.0695187)  =   −87.19   spending, gone from 10 Jul
                                        --------
                                        +1,014.14  ✓
```

The household's story for July: **"You saved €27,065. Your assets returned €520. The Swiss franc
strengthening added €1,014 you did nothing to earn."** That sentence is the entire point of INV10,
and this arithmetic is what produces it. Make it a test case.

---

## 4. The current cash position

Priority-one view. It has to answer one question: **can we pay what is coming?** Every design choice
below serves that and nothing else.

### 4.1 Liquidity tiers

| Tier | Definition | Settlement | In the headline? |
|---|---|---|---|
| **T0 — Settled cash** | Cleared credit balances in transaction and current accounts, spendable today without instruction to anyone. | Immediate | Yes |
| **T1 — Cash equivalent** | Available within 3 business days, with no meaningful risk of change in value and no lock-up, notice period or gate. | ≤ 3 business days | Yes |
| **T2 — Liquid, at risk** | Listed securities. Sellable, but with market risk, settlement lag, and possibly a wrapper consequence to withdrawing. | T+2 today, **T+1 from 11 October 2027** in the EU, UK and Switzerland | **No** — separate band |
| **T3 — Illiquid** | Assurance vie, Pillar 2, superannuation, property, and the mortgage. | Weeks to decades | No |

**T2 is excluded from the headline, and this is the important call.** Including sellable ETFs makes
the "can we pay the bills" number move with the equity market, which is precisely when you least
want it to. Show T2 as an adjacent, clearly-separated "**available if we liquidate**" band, with its
settlement lag and — for the PEA — the wrapper consequence attached (§8.4). Two numbers, honestly
labelled, beats one number that means two things.

### 4.2 The headline formula

```
AVAILABLE(H) =   Σ T0 balances
               + Σ T1 balances
               − Σ card liabilities            (§4.4, all three components)
               − Σ committed outflows due within H days   (§4.5)

WITH LIQUIDATION(H) = AVAILABLE(H) + Σ T2 marks, less estimated dealing costs
```

`H` defaults to **30 days**, user-settable to 7 / 14 / 30 / 60. The horizon is part of the number's
meaning and must be on screen with it.

**Presented three ways, always:**
1. Consolidated in the viewer's display currency (INV2).
2. **Per currency.** A household with CHF-rich and EUR-poor has a real, immediate problem that a
   consolidated figure hides completely. This is not a nicety; for this household it is the point.
3. **"Usable without conversion"** — for each currency, T0+T1 in that currency minus that currency's
   commitments. AUD sitting in Sydney will not pay a French mortgage this week without a conversion
   that costs money and takes days.

### 4.3 The Julius Baer investment account — the hard call

The account holds a mixture chosen by a private bank. Some of it is genuinely cash. Some of it is a
structured note that will not be cash for three years. The bank's own statement is authoritative on
value (feasibility assessment, and correctly so) but its asset-allocation buckets are marketing
categories, not liquidity categories.

**A holding is T1 cash-equivalent only if all four hold:**

1. **Classification** — the custodian statement places it in a cash, liquidity, money-market or
   call-deposit bucket, **or** its ISIN appears on a household-maintained
   `money_market_instruments` allowlist.
2. **Maturity** — residual maturity ≤ 3 months, **or** the instrument is open-ended with daily
   dealing. (This is IAS 7's cash-equivalent test — "short-term, highly liquid, readily convertible
   to known amounts of cash", conventionally three months or less at acquisition.)
3. **No impediment** — no lock-up, notice period, redemption gate, minimum holding, or pledge as
   collateral. Swiss fiduciary call deposits with 48-hour notice pass. Anything with a swing-pricing
   gate does not.
4. **Value stability, made testable** — the instrument's NAV has not moved more than **±0.5% in any
   rolling 30-day window over the trailing 12 months**. This operationalises "insignificant risk of
   changes in value" into something a computer can check rather than a phrase a human can rationalise.

**Default when any of the four is unknown: T2, not T1.** Liquidity claims are earned, not assumed.
An account that has never been classified contributes nothing to the headline and shows a
"classify these holdings" prompt. This will feel pessimistic in month one and will be right in
month thirty-six.

**Staleness interacts.** The JB portfolio statement is monthly, so its T1 figure can be up to ~35
days old. Apply §6.3: it renders with a confidence band, and past the hard limit it becomes UNKNOWN
and drops out of the headline into the unknown-count (§6.4). It must **not** silently persist as a
confident number, and it must **not** vanish without the headline saying an account went unknown.

Also: **the JB current accounts in EUR, USD and CHF are T0**, not T1, even though they arrive by
monthly statement. Access route and liquidity tier are orthogonal — this is the distinction §1
insists on. Their *freshness* is poor; their *liquidity* is perfect.

### 4.4 Six credit cards in three currencies

TopCard ×4 (Partner A CHF + EUR, Partner B CHF + EUR) and CBA ×2 (AUD). Every one of them has three
distinct liabilities and most tools conflate them.

| Component | What it is | Where it comes from | In the headline |
|---|---|---|---|
| **L1 Statemented** | Closing balance of the last statement, not yet paid, with a known due date. | Statement | Always |
| **L2 Post-statement posted** | Spend booked since the statement close, not yet billed. | Feed, or the *next* statement (i.e. retrospectively) | Always, when known |
| **L3 Pending authorisations** | Authorised, not posted. Invisible on statements entirely. | Live feed only | Always, when known — but see §5.5, pendings never enter the ledger |
| **L4 Modelled unbilled** | Estimated L2 for cards with no feed, between statements. | Model | **Yes by default**, separately labelled |

`card_liability = L1_unpaid + L2 + L3 + L4_if_modelled`, all in the card's **billing currency**.

**L4 is the honest response to a real gap.** None of the six cards has a live feed. Between a
TopCard statement on the 25th and the next one, the app's view of that card's liability is frozen at
L1 and is therefore **understated, monotonically, by up to a month of spending.** A cash position
that flatters you the week before the direct debit is worse than useless.

Model: `L4 = median_daily_spend(trailing 3 statements) × days_since_statement_close`, with a P10–P90
band from the same three statements' daily distributions. Present it as its own line —
*"estimated unbilled spend: €410 (€260–€680)"* — never blended into the observed figure. Offer one
toggle, **conservative (default, includes L4)** versus **observed only**. Default to the pessimistic
reading; a household that is surprised by a card bill has been failed by the tool.

**Currency on cards.** A CHF-billed TopCard used for a €60 purchase in France produces a CHF charge
at the issuer's own rate. **The CHF amount is the transaction amount** — it is the actual liability.
Store the original EUR amount and the issuer's implied rate as metadata so the FX spread is
measurable and comparable against Revolut and Wise, but never store the EUR figure as the amount.
Getting this backwards understates the liability by whatever the issuer's margin is, on every
foreign transaction, forever.

**Due dates are first class.** TopCard settles by LSV direct debit on a monthly cycle; CBA cards
carry a due date on each statement. Both feed §4.5 and §7 — the cash position needs to know not just
*how much* is owed but *when it leaves*.

### 4.5 Committed outflows

Netted off only if **committed**: contractual, direct-debited, or already scheduled. Sources, in
order of authority:

1. The mortgage amortisation model (§5.7) — payment date and amount are known exactly.
2. Recurring-payment detection over the four live feeds and statement history: same counterparty,
   same approximate amount (±15%), regular interval, ≥3 occurrences. Detected recurrences are
   **proposed**, not committed, until confirmed once by a person; thereafter they self-maintain.
3. Manually entered scheduled items — school fees, insurance premiums, the annual French income tax
   balance, taxe foncière.
4. Card direct debits (§4.4).

**Forecast discretionary spending is not netted off the cash position.** It belongs to the
projection (§7). The cash position answers "what is already spoken for"; the projection answers
"what will probably happen". Conflating them produces a headline that drifts for reasons the user
cannot see.

### 4.6 Two exclusions, stated explicitly

- **The son's Livret A is not household liquidity.** It is a third person's money held under power
  of attorney. It is excluded from T0/T1, from the headline, from goals, and from household net
  worth aggregates, and appears only in a separately-labelled custodial section. This is a
  correctness position before it is an ethical one: including it would let the household plan
  around money it does not own. The feasibility assessment flags this account as deserving its own
  retention and access story; the ledger's contribution is to keep it arithmetically separate from
  day one, so that separating it later is not a migration.
- **The mortgage principal is not a short-term liability.** Only the payments falling inside horizon
  `H` are netted. The outstanding principal belongs to net worth, not to the cash position. Netting
  the whole mortgage against cash produces a permanently, uselessly negative headline.

---
