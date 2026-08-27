# Financial Correctness — Ledger Semantics, Valuation, Reconciliation and the Tax Lens

**Status:** Design. Binding on the meaning of every number the app displays.
**Owner:** Financial correctness agent
**Date:** 27 August 2026
**Inputs:** `01-feasibility-assessment.md`, `02-decisions-and-constraints.md` (D1–D5, INV1–INV11, threat model seed)
**Boundary:** This document defines *what the numbers mean*. `12-architecture.md` defines *how they move*.
Where a schema appears here it is a semantic contract, not a storage prescription — the architecture
agent chooses tables, services and transport.

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

One invariant exception is declared, in [§9.1](#91-inv1--the-one-declared-exception).
Two additional invariants are proposed, in [§9.3](#93-two-invariants-i-want-to-add).

**Rulings on the other agents' requests**, argued where cited:

| Request | Ruling | Section |
|---|---|---|
| Security **S8** — perimeter-scoped HMAC idempotency keys instead of bare document hashes | **Adopted, and extended.** Every idempotency index is perimeter-keyed, not only the document one; the duplicate check moves into the uploader's client; perimeter change requires a re-key or ingestion halts | [5.2](#52-ruling-on-s8-perimeter-scoped-hmac-idempotency-keys) |
| Presentation **C2** — measure staleness in cadence units, `expectedCadenceDays` per observation | **Confirmed as the display ladder; rejected as the arithmetic.** Two numbers, not one: `r` drives badges and upkeep, `σ` drives bands and Unknown | [6.3](#63-confidence-decay-and-a-ruling-on-cadence-units) |
| Presentation **C8** — projections forecast no FX rate, flat with a shaded variability band | **Confirmed.** The band's shape and sample are corrected: √t-growing, from ten years of ECB data, applied to the exposed share only | [7.4](#74-the-fx-ruling--c8-confirmed-the-band-corrected) |
| Presentation **§11.1** — `decompose()` with exactly three cause terms summing to the delta, `unknown` as a list | **Granted, with the residual not folded into `fx`.** Three cause terms stand; `unexplained` and `perimeter` render outside the cause palette | [10.2](#102-decompose--the-contract-amended) |
| Presentation **§12** — persist a rendered display snapshot as an INV1 exception | **Refused as scoped; granted as glyphs.** Store formatted strings, not values, behind a lint-checked module boundary. Then it is not an INV1 exception at all | [9.2](#92-ruling-on-presentations-requested-inv1-exception-its-12) |
| Security **INV12 / INV13 / INV14** | **All three adopted.** INV13's cost falls on the owner's own view; INV14 is what forces the perimeter term in `decompose()` | [9.4](#94-on-securitys-inv12-inv13-and-inv14) |
| Architecture **§10.5** — eight questions | Answered individually, including one amendment architecture did not ask for | [10.1](#101-architectures-eight-questions-12-105) |

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

**One term is added later.** Once INV14 makes every figure audience-relative (adopted in §9.4), the
identity gains a fifth, **boundary** term for value crossing a perimeter edge: `ΔNW_d = C + M + X + U + P`.
`P` is zero by construction in a single-principal audience and non-zero only in the household view, where
it says *"this money left the shared view; it did not leave the household."* It is not a cause and never
wears a cause hue. The full contract is in [§10.2](#102-decompose--the-contract-amended).

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

## 5. Reconciliation and idempotency (INV7)

INV7 says re-uploading a statement must never double-count, and proposes keying on institution,
account, period and document hash. That is the right instinct and it is not sufficient, because the
three things that actually go wrong in this household's inventory are not re-uploads:

- the **same period arriving twice by different routes** — a Revolut PSD2 backfill after a re-link
  covering days already imported from a monthly PDF;
- a **statement contradicting a live feed** for an instant both claim to describe;
- the **modelled mortgage drifting** from the bank's annual `capital restant dû`.

Only the first is a duplicate. The other two are disagreements, and a dedup key cannot resolve a
disagreement. So this section has two halves: identity (§5.1–§5.4), which prevents double-counting,
and adjudication (§5.5–§5.7), which decides who is right when two honest sources differ.

### 5.1 Three layers, three keys

Architecture's `12-architecture.md` §7.3 already proposes three layers. I adopt the structure and
change two things: what the keys are made of (§5.2) and what happens on a hit at layer 2 (§5.6).

| Layer | Identity of | Key | Catches |
|---|---|---|---|
| **L1 — document** | a byte sequence | `idem_doc = HMAC(K_idem,p , "doc-v1" ‖ SHA-256(plaintext))` | the same file, uploaded twice, from any path |
| **L2 — period** | a claim about an interval | `idem_period = HMAC(K_idem,p , "period-v1" ‖ canonical(institution, account, period_start, period_end, doc_kind))` | the same statement reissued as a different file; a corrected statement; a CSV slice overlapping a PDF |
| **L3 — line** | a movement | `external_ref` if the source has one, else `natural_key` (§5.3) | overlapping export ranges, feed backfill, a transaction seen by both a feed and a statement |

`K_idem,p = HKDF(K_perimeter, "idem-v1")` — a key derived for this purpose only, so that leaking an
idempotency index never threatens content. `p` is the perimeter of the **target account**, not of the
uploader.

### 5.2 Ruling on S8: perimeter-scoped HMAC idempotency keys

Security's S8 replaces the bare document hash with `HMAC(K_perimeter, H)` to close the dedup oracle —
the channel by which Partner A could test whether a guessed statement had already been ingested into
Partner B's private perimeter, and thereby learn that the account exists.

**Ruling: adopt S8, and extend it. It is compatible with everything in this document, and the
extension is not optional.**

**Why it is compatible.** INV7's guarantee is *within an account*: the same movement must not be
counted twice on the same balance. An account sits in exactly one perimeter (security §3.1), and a
document takes the sensitivity of its most-restricted line (S11), so every posting a document produces
lands in one perimeter. Dedup scoped to a perimeter therefore covers every case where double-counting
is arithmetically possible. Scoping it more widely buys nothing a ledger cares about.

The three cases worth walking, since this is the invariant most likely to be broken by accident:

| Case | Under a bare hash | Under a perimeter HMAC |
|---|---|---|
| A uploads the joint CA statement twice | deduped | deduped — both hold `K_H`, so both compute the same `idem_doc` |
| A and B each upload the same joint statement | deduped once | deduped — same key, same value |
| A and B each upload the same consolidated JB statement, routed to different accounts in different perimeters | deduped, and A learns B's document existed | stored twice as ciphertext; each perimeter's postings deduped within itself; **no account double-counts** |

The third row is the only behaviour change, and it costs a duplicated blob — a storage cost, which
data security's DS5 already permits to be collapsed by content hash at the storage layer provided the
result never surfaces in a response. There is no correctness cost.

**Three conditions, all load-bearing.**

**Condition 1 — every idempotency index is perimeter-keyed, not only the document one.** S8 specifies
the HMAC over the document hash and stops there. A bare L2 period key is exactly as good an oracle:
`(Julius Baer, ‹B's account›, 2026-07-01, 2026-07-31, portfolio_statement)` is guessable in a few
attempts, and a uniqueness violation on it leaks the same bit. So L2 is HMAC'd too, over a canonical
encoding, with its own domain separator. L3 needs no separate treatment — it is scoped to an account,
which is scoped to a perimeter by construction, and it is never exposed to a caller who cannot see the
account.

**Condition 2 — the duplicate check runs in the client, against a synced perimeter-scoped index.**
The server cannot compute `K_idem,p` for a private perimeter, so it cannot answer "is this a
duplicate". It holds a unique index over an opaque 32-byte value and rejects a second insert. That is
enough for correctness but it inverts the flow: the *uploader's client* fetches its own perimeter's
idem set (RLS permits this — it is the caller's own perimeter) and detects the collision locally,
before upload. The server never differentiates a duplicate from a novel document in any response, per
S9. Architecture's `12` §7.3 L1 behaviour — *"No-op; return the existing document and its state"* — is
therefore **amended: return the constant response, and let the client that already holds the key tell
the user.**

**Condition 3 — perimeter change requires re-keying, and it is a named migration step.** D3 and
security §3.1 both allow an account to move between perimeters. Its historical idem values were
computed under the old perimeter's key; new uploads compute under the new one; a re-upload of an old
statement after the move would therefore **not** dedup and would double-count. This is a real
correctness bug created by an otherwise correct security control, and it does not announce itself —
it produces a plausible-looking duplicated month six months later.

The fix is available because whoever can move an account holds both keys: `HOUSEHOLD → B` is performed
by B, who holds `K_H` and `K_B`; `B → HOUSEHOLD` likewise. So perimeter change is a three-part
transaction: move the account, re-key its entire idem index on the owner's device, journal both. If
the re-key cannot complete, the move does not commit. An account whose idem index is partially re-keyed
must refuse ingestion rather than accept it.

**One consequence handed back to data security.** `11-data-security.md` §6 keeps a permanent hash tail
so that erasure and INV7 coexist. Under S8, that tail must be the **HMAC**, never the bare SHA-256 —
otherwise the erasure tombstone becomes the oracle that S8 just closed, with the added charm of being
permanent by design.

**One residual, stated rather than solved.** Perimeter-scoped keys mean the *household* can no longer
detect that the same document was ingested twice into two different perimeters. That is intentional
and it is also a small loss of a real signal, since the honest reading of "A and B both uploaded the
same consolidated statement" is usually "one of them is about to double-enter something manually".
Accepted: the alternative is the oracle.

### 5.3 Line identity, and the natural key

`external_ref` — the ASPSP's `transactionId`, the card issuer's reference — is authoritative **when
present and stable**. Two of the four live feeds are known to renumber on backfill, so stability is a
per-connection property that must be measured, not assumed: on each poll, re-observe the refs for a
7-day overlap window; if any ref changes for an otherwise-identical transaction, mark the connection
`unstable_refs` and fall through to the natural key for that source, permanently.

```
natural_key(txn) = BLAKE2s-256(
      account_id ‖ booking_date ‖ amount_minor ‖ currency
    ‖ normalise(counterparty) ‖ normalise(remittance) ‖ ordinal )

normalise(s) = NFKC(s) → casefold → collapse whitespace
             → strip issuer decorations: statement sequence numbers, terminal ids,
               "CARTE 1234 LE 12/07" prefixes, trailing reference blocks
```

`ordinal` disambiguates genuinely identical same-day movements (two €4.50 coffees) by position within
the source batch. **It is stable only within one rendering of one source.** The same two coffees seen
by a feed and by a PDF may be ordered differently, so `ordinal` must never be the sole discriminator
across sources. That is precisely why cross-source reconciliation happens per-period by multiset
(§5.4) rather than per-transaction by key.

### 5.4 A period is replaced wholesale, by multiset diff

**F9's third clause.** When a period's posting set is re-observed — a better source, a corrected
statement, a feed backfill — the ledger does not merge. It computes a multiset difference and applies
it as a whole.

*Multiset*, not set, is the entire point. Two identical €4.50 coffees are two elements. Set semantics
silently deletes one of them, the balance assertion then fails by €4.50, and a plug appears in
`RECON:UNEXPLAINED` for a transaction that was real, observed, and thrown away by a data structure.

```
observe(account, period, source, S_new):
    S_old  = current committed multiset for (account, period)
    add    = S_new ⊖ S_old        # multiset difference
    remove = S_old ⊖ S_new

    if source_rank(source) < source_rank(incumbent) and remove ≠ ∅:
        raise Discrepancy(...)     # a weaker source may add, never retract

    commit(add) ; reverse(remove)  # both append-only, §5.6
    recompute_derived(account, from = period.start)
```

Three rules around it:

1. **Periods are half-open on the right**, `[start, end)`, in the account's own booking-date
   convention. A closing balance dated 31 July belongs to the period ending at the start of 1 August.
   Off-by-one on period boundaries is the most common cause of a double-counted month in every ledger
   system I know of, and it is prevented by a convention, not by care.
2. **Overlapping periods from the same source are merged into their union before diffing.** The CBA
   three-month-slice workaround (feasibility §CBA) guarantees overlaps; they must be a non-event.
3. **A partial period is never committed as a period.** A CSV covering 1–14 July does not close July.
   It contributes postings and no closure claim. Only a document that asserts a closing balance closes
   a period, and that closure is what licenses the completeness check in §2.6's `choose_contra`.

### 5.5 A statement contradicting a live feed

Architecture's §8.4 asked for *a* declared precedence and proposed statement-over-feed. I am overriding
the global rule, because a single precedence answers a question that is usually not being asked.

**Precedence is per observation type, not per source.**

| | The feed is authoritative for | The statement is authoritative for |
|---|---|---|
| **O1 postings** | identity, timing (booking *and* value date), pendings, counterparty detail | **completeness** — it is a closed set over its period |
| **O2 balances** | *today* | **its own as-of instant** |

Most apparent conflicts dissolve at this table. A feed balance and a statement balance rarely describe
the same instant; when they do not, they are not in conflict and must not be compared. The
reconciliation task is *only* raised when both assert the same `(account, currency, as_of)`.

**The balance-type trap, which will otherwise generate a false conflict every week.** PSD2 exposes
several balance types — `closingBooked`, `expected`, `interimAvailable`, `forwardAvailable` — and
ASPSPs differ on which they populate. Comparing a statement's booked closing balance against a feed's
*available* balance produces a difference equal to the outstanding card authorisations, every single
day, which is not an error and must never be reported as one. This answers architecture's question 8:

> **`closingBooked` is the ledger balance.** It is what §2.6 asserts against. `interimAvailable` (or
> `expected`, where an ASPSP populates only that) is the **spendable** figure and feeds T0 in §4.1.
> Store both, label both, never compare across kinds, and record which type each connection actually
> returns as a per-connection fact.

**When the same instant genuinely disagrees beyond tolerance:**

1. The **statement wins the assertion.** It is a document the institution will stand behind; a feed is
   a convenience API with a known-variable balance-type contract.
2. The delta posts to `RECON:UNEXPLAINED`, never to an `UNOBSERVED` plug. Coverage was complete on
   both sides, so this is an error, and §2.6 escalates it.
3. The **connection is marked `contradicted`** and enters the upkeep list. A feed that disagrees with a
   statement has told you something about itself, and quietly preferring the statement forever throws
   that away.
4. If a connection is contradicted twice in a row on the same account, it drops to *corroboration only*:
   its postings are still ingested and matched, but it may no longer satisfy the completeness test in
   `choose_contra`. That is the safe failure: the account stops being able to produce
   `RECON:UNEXPLAINED` and starts producing honest `UNOBSERVED` plugs instead.

**Pending authorisations never enter the ledger.** This is the promise made in §4.4 for L3.

Pendings mutate, change amount on settlement, and vanish without a trace. They have no stable identity,
so they cannot satisfy INV7, and in an append-only ledger they would generate a reversal for every
authorisation the household ever made. They live in a separate `pending_authorisation` overlay keyed by
the ASPSP's own id, with a TTL, and they are added to the *cash view* (§4.2) and to nothing else. They
never affect net worth, never appear in INV10's decomposition, and never enter a projection.

Matching a pending to its settled posting — architecture's question 3 — is a three-tier rule, applied on
each ingest and never irreversible:

| Tier | Condition | Result |
|---|---|---|
| **Exact** | same card, same amount, same currency, settled within 7 days | matched; pending retired |
| **Probable** | same card, same merchant after normalisation, amount within the larger of ±20% or 5 units, settled within 10 days | matched; the amount difference is recorded as `authorisation_drift` (tips, fuel holds, hotel pre-auths, issuer FX) |
| **Expired** | neither, and older than 10 days | pending dropped from the cash view, and a one-line note kept for 30 days so a genuinely lost authorisation is visible |

`authorisation_drift` accumulated per merchant is worth keeping: it is how the app eventually learns
that this restaurant always authorises 20% over.

### 5.6 Restatement, correction, and the custodian's mark

**Architecture's question 6, answered: append-only reversal. Never amendment in place.**

A reissued or corrected statement does not edit the postings it supersedes. It produces two batches —
a full reversing batch tagged `supersedes: <document_id>`, and the new batch — both retained, both
carrying their own provenance. The UI shows the net; the audit shows both. Three reasons, in order of
weight:

1. INV3 requires every figure to be traceable to where it came from. An amended posting is traceable to
   a document that no longer says that.
2. F3 requires derived reconciliation postings to be regenerable from observations. If observations can
   be edited, "regenerable" means "regenerable from whatever they say now", and reproducing last
   month's report becomes impossible.
3. Ease-of-use asked for superseding corrections that retain the old value with a reason (`14` §1058).
   This is that, at ledger level.

Stated as a general principle, because it decides a dozen smaller questions and I am promoting it to an
invariant in §9.3:

> **Observations are append-only. Derivations are disposable.** A correction is a new observation. A
> reconciliation posting, a plug, a model output, a cached balance and a converted figure are all
> derivations: delete them all and the system must rebuild bit-identical state from the observations
> and the rate table alone.

**The custodian's mark — `VALUATION:MARKET:ADJUSTMENT`.** D5 lets us price the PEA and the Linxea
assurance vie by ISIN. The custodian's statement remains authoritative on unit counts, fees, and any
gross/net difference. When our marked value for a position disagrees with the custodian's statement
value at the same instant, **we snap to the custodian** and post the difference to
`VALUATION:MARKET:ADJUSTMENT` (`flow_class = RETURN`), dated the statement date.

The running total of that account is a **health metric on the pricing pipeline, not an accounting
figure.** Ordinary causes are rounding, an accrued management fee, a distribution booked on a different
date. A monotone, same-signed accumulation means the pricing is structurally wrong — the wrong share
class, an accumulating/distributing mix-up, a missing ongoing charge on a unit-linked fund — and it
should raise a pipeline alarm long before it becomes material to net worth. Alarm rule: same sign for
three consecutive statements, or an absolute total exceeding 0.5% of the position.

### 5.7 The mortgage: modelled amortisation against the annual statement

The mortgage is the one line where the app claims to know a balance to the cent every day without ever
observing it. That claim is defensible only if the drift is measured.

**The model is the bank's own amortisation table, not a formula.** Store the issued
*tableau d'amortissement* row by row as dated `MODEL` observations. Recompute only to *extend* the
table after a contractual event, and store the recomputation as a new table version with an effective
date and the event that caused it. Reimplementing a French lender's rounding — échéance rounded to the
cent, the residue absorbed into the final payment — is a losing game with no upside, and the table is a
document we already have.

**Split every échéance into three, always:**

```
CA joint current          −1,500.00 EUR    the direct debit
LIABILITY:CA:MORTGAGE     +1,100.00 EUR    principal          (internal — not a perimeter crossing)
EXPENSE:MORTGAGE-INTEREST   +380.00 EUR    RETURN
EXPENSE:INSURANCE:BORROWER   +20.00 EUR    CONTRIBUTION
```

The borrower insurance is a cost of *borrowing*, not of the *debt*, and classifying it as RETURN would
make the household's own insurance premium look like a market outcome. Where the premium is computed on
the capital restant dû rather than on the initial capital, it declines every month and the split is not
constant — which is one of the six drift sources below.

**The annual statement is a balance assertion** on `LIABILITY:CA:MORTGAGE:EUR` and goes through §2.6
unchanged, with contra `RECON:MODEL-DRIFT`. What is specific to the mortgage is the budget:

| Drift | Verdict | Action |
|---|---|---|
| `\|Δ\| ≤ 0.02 × months_elapsed` | rounding | silent; `RECON:ROUNDING` |
| `≤ €50`, sign alternating across years | model noise | recorded as `RECON:MODEL-DRIFT`, not escalated, table re-fit proposed |
| `> €50`, or **same sign two years running** | structural | escalate with a diagnosis |

The same-sign rule matters more than the threshold. A consistent one-directional drift of €12 a year is
far more diagnostic than a random €80, because it means the model is wrong in a way that will grow: a
rate revision never ingested, a modulation d'échéance, a partial early repayment, a *report d'échéance*,
a degressive insurance premium modelled as constant, or a garantie refund at term. The diagnosis routine
should test those six hypotheses explicitly and name the most likely, because "the mortgage is €47 out"
is not actionable and "your échéance changed in March and we never saw it" is.

**Neither side silently overwrites the other.** The model never overwrites the assertion. The assertion
never silently re-fits the model — a re-fit is *proposed*, with the diagnosis, and applied by a person.
Between statements the balance renders as **Modelled**, not as Current: it is exact given the terms, and
the risk is that the terms changed, which is not a freshness problem and must not wear a freshness badge
(this confirms presentation's §4.2 treatment of `ƒ`).

### 5.8 Materiality, plausibility and tolerance — the numbers

Ease-of-use and architecture both asked for these. Here they are, with the derivations, because
thresholds nobody can derive get tuned until they never fire.

**Rounding tolerance** (§2.6):

```
rounding_tolerance(account, interval) = min( 1.00 , 0.01 × (n_postings + 1) )   # account ccy
```

Each posting can carry at most one minor unit of rounding; the cap stops a 4,000-line CBA import from
tolerating a €40 error.

**Materiality — escalate regardless of extraction agreement.** I adopt ease-of-use's §5.2 table, with
the absolute floors confirmed and one row added:

| Rule | Threshold |
|---|---|
| Absolute | > EUR 1,000 / CHF 1,000 / AUD 1,700 / USD 1,100 |
| Relative | > 15% of the account's prior value |
| Household impact | > 1% of the **audience-closed** net worth (INV14 — never the true household total) |
| Sign change | any balance crossing zero |
| Liquidity-feeding | any figure entering §4.2: all thresholds halved |
| First figure | the first value ever recorded for an account — always human |
| **Cost-basis-bearing** *(added)* | any figure that establishes or alters a lot's `acquired_on` or `cost_native` — always human, at any size |

The added row exists because a cost basis is the one number in this system that **cannot be corrected
later from any source we will still have**. A wrong balance is fixed by the next statement. A wrong
acquisition date is discovered in 2041 by an accountant. §8.3 explains why.

**Plausibility bands.** Ease-of-use's three volatility classes are right in shape. Two corrections:

- The *Drifting* rule — "exceeds a reference-index move by more than 8 percentage points" — is a fixed
  band over a quantity whose natural scale differs by an order of magnitude across the three accounts it
  governs. The Linxea ETF portfolio's tracking difference against its own holdings-weighted benchmark is
  tens of basis points; 8pp would never fire. A Julius Baer discretionary mandate's is percent-scale;
  8pp would fire and be ignored. Replace with: benchmark = the account's **own holdings-weighted**
  composite, threshold = `3 × stdev(tracking_difference)` over the trailing 24 months, floored at 2pp
  and capped at 8pp. Self-scaling, and it degrades to the original number for an account we know nothing
  about.
- **Property is not Stable; it is its own class.** A DVF-derived estimate has no observations to be
  stable about. Its band is the dispersion of its own method (§6.3), and it is `FROZEN` (§6.1). Putting
  it in the Stable class would make any revaluation an escalation.

**Where these live.** All of the above are per-account rows in a parameters table with defaults by
account class, versioned, and shown to the household in plain language on the account screen. A
threshold nobody can see is a threshold nobody trusts.

---

## 6. Interpolation, staleness and confidence (INV4)

The question this section answers is the one the feasibility assessment identified as fatal if got
wrong: **what does a figure mean on a day nobody looked?**

### 6.1 Five carry policies — what a figure means between observations

There are exactly five honest answers, and every account line gets assigned one. This replaces the
feasibility assessment's blanket suggestion to "interpolate between statements and label it as
interpolated", which D5 already partly overtook.

| Policy | Between observations, the figure is… | Applies to | Confidence |
|---|---|---|---|
| **HOLD** | still literally true; nothing can have moved | Pillar 2 between certificates, Hostplus between statements, a closed account | unchanged |
| **ACCRETE** | last observation + a contractually determined accrual | Livret A ×3 (§6.2), the mortgage (§5.7), a term deposit | unchanged where every input is contractual; degraded where one is assumed |
| **REPRICE** | last observed **quantity** × today's **price** | Bourse Direct PEA, Linxea (D5), any JB position with a usable ISIN | as fresh as the **quantity**, never as the price — see the weakest-link rule below |
| **DECAY** | last observation, with a widening band, until it is not worth showing | every current account, every card, JB investment holdings we cannot price | widens with the account's own volatility (§6.3) |
| **FROZEN** | exactly the last valuation, as a step; it does not move at all | the house, and any O5 external estimate | the method's own dispersion |

Three rules bind the table:

**Linear interpolation of a balance is banned outright.** Not discouraged — banned, for the same reason
as R2 bans interpolating FX. Interpolating a balance fabricates dated movements, and INV10 will then
faithfully attribute those fabrications to contribution or market. When the next statement arrives the
delta is posted **at the assertion date** (§2.6 already does this), producing a step. A monthly chart of
a quarterly account is a staircase, and a staircase is what is true. Presentation may draw a dashed
connector between two observed points for legibility; that connector is a line on a screen and never a
figure, never a posting, and never an input to anything.

**FROZEN is a step function, and the step is attributable.** The feasibility assessment is explicit that
a property estimate must never move the net-worth line month to month. So the house contributes zero to
every monthly delta except the month it is revalued, where its entire change appears at once, labelled,
in `VALUATION:MARKET`, and never auto-committed. A household that watches its net worth drift upward on
an algorithm's opinion of its house learns to distrust the whole app, and correctly.

**The weakest-link rule.** REPRICE is the trap in this table and it generalises:

> A derived figure is exactly as fresh, and exactly as uncertain, as its **stalest input**.

A Linxea position priced from this morning's NAV against a unit count from a statement four months old
is a four-month-old figure wearing a one-day-old badge. That is the single most convincing way this app
could lie, because everything about it looks live. Freshness, confidence and the Unknown state propagate
through conversion, aggregation, decomposition, projection and goal progress, and no derivation may ever
report a freshness better than the worst thing it was built from. I am proposing this as INV15 (§9.3).

### 6.2 The Livret A accrual, exactly

Three Livret A lines, the model that governs them between the annual statements, and the check that
proves it. This is worth doing precisely because it is the one ACCRETE model we can verify to the cent.

Interest is computed **par quinzaine** — two periods per month, the 1st–15th and the 16th–end — and
capitalised once a year on **31 December**, appearing on the account in early January:

- a deposit earns from the **1st or 16th following** it;
- a withdrawal stops earning from the **1st or 16th preceding** it;
- there are 24 quinzaines in a year, and the annual rate is divided by 24, not by 365.

```
interest(year) = Σ over the 24 quinzaines q of  ( qualifying_balance(q) × rate(q) / 24 )
```

`rate(q)` is a dated series, never a constant. The rate is revised on **1 February and 1 August** — both
of which fall exactly on quinzaine boundaries, which is why the convention composes cleanly. The rate has
been **1.70% since 1 August 2026**. The ceiling is **€22,950 of deposits**, and capitalised interest may
carry the balance above it, so a balance over the ceiling is not an error.

Two consequences for the ledger:

1. **The `INCOME:INTEREST` contra in §2.6 gets a check.** When the January statement asserts a balance
   and the plug is classified as interest, compare the plug against `interest(year)` computed from the
   movements we hold. Agreement within €0.02 confirms both the model and the coverage. A difference of
   more than €1 means the plug is **not** interest — it is an unobserved movement wearing interest's
   clothes — and it escalates instead of being absorbed. Without this check, a €400 unrecorded deposit
   into a Livret A disappears into an interest line forever.
2. **The 1 February and 1 August rate revisions are a first-class scheduled obligation**, one tap, not a
   portal visit — confirming ease-of-use §2.3, whose correction of the brief ("interest is annual, the
   *rate* is twice-yearly") is right and should be treated as settled.

The son's Livret A uses identical arithmetic in the `CUSTODIAL` perimeter and contributes to no household
figure (§4.6, FC3).

### 6.3 Confidence decay, and a ruling on cadence units

Presentation's C2 measures staleness as `r = age_days ÷ expected_cadence_days` and builds its entire
freshness design on it, with a worked table showing that a days-based badge cries wolf on a pension
certificate while staying silent on a dead daily feed. Ease-of-use independently arrives at per-class
grace periods. Both need a ruling.

**Ruling: C2 is right, and it is answering a different question from the one confidence answers.
Confirmed as the display ladder. Rejected as the arithmetic. The app needs two numbers, not one.**

| | `r = age ÷ expectedCadenceDays` | `σ(t)` — uncertainty, in the figure's own currency |
|---|---|---|
| Answers | *is this line late?* | *how wrong might this number be?* |
| Property of | the **document schedule** | the **account's behaviour** |
| Drives | the ladder, the badge, the sort order, the Upkeep list, reminders | the ± band, the Unknown threshold, the coverage bar's weighting, interval propagation |
| Owner | presentation and ease-of-use | this document |

The reason they cannot be the same number is a two-line argument. A CBA transaction account at `r = 0.9`
and a Pillar 2 certificate at `r = 0.9` are equally *on time* and are not remotely equally *trustworthy*:
one has had a month of unobserved salary and spending pass through it, the other cannot have moved. If
the ladder alone decided what enters a total, the app would be making a decision about arithmetic using a
fact about chores. `expectedCadenceDays` is confirmed as required per-line data (§6.5) and C2 stands
unmodified for everything it governs.

**How σ is computed — from the account's own history, not from a time constant.** This is F10.

> For a DECAY account, the band at horizon *t* is the **empirical P10–P90 of that account's own observed
> *t*-day balance changes** over the trailing 24 months.

No distributional assumption, no fitted volatility parameter, and it captures salary timing, the mortgage
direct debit and quarterly insurance automatically, because those are in the history. If fewer than 90
days of complete posting coverage exist, fall back to the account-class default and label the band
*"wide — not enough history"* rather than producing a confident narrow one.

The other policies fall out:

| Policy | σ(t) |
|---|---|
| HOLD | 0, plus the model-input uncertainty, which is 0 for a certificate |
| ACCRETE | the accrual's own input uncertainty — for a Livret A, the empirical distribution of *unobserved deposits*, which is near zero for most households and measurable for this one |
| REPRICE | quantity uncertainty (empirical unit-count change per statement period) × price, **plus** price staleness if prices are stale. Per the weakest-link rule, the *quantity* clock dominates |
| DECAY | as above |
| FROZEN | the method's dispersion. For DVF comparables: the interquartile range of €/m² for comparable sales in the commune over 24 months, × surface. This will be embarrassingly wide — ±12–20% is normal — and that is the correct and useful answer |

**Unknown is a step function, not the end of a curve.** Three triggers, any of which fires immediately:

1. **The connection is expired, or has failed twice consecutively.** INV4's literal case. No decay, no
   grace: the moment the PSD2 consent lapses, the figure is Unknown. This is the case the feasibility
   assessment says destroys trust permanently if got wrong.
2. **`σ(t) > max( 2% of the figure , materiality(account) )`.** When the band is wider than the number is
   useful, stop showing the number. This is better than a day count because it self-calibrates, and
   pleasingly it *reproduces* ease-of-use's per-class grace periods rather than contradicting them: on
   this household's shape a volatile current account crosses it at roughly two weeks, a drifting
   investment account at six to eight weeks, and a Livret A or a Pillar 2 never crosses it at all. Two
   agents reasoning from different premises landing on the same numbers is the best evidence either of us
   has that the numbers are right.
3. **An absolute backstop**, so that a bug in σ cannot keep a figure alive forever: 90 days for DECAY,
   400 days for HOLD / ACCRETE / FROZEN.

Ease-of-use's framing — *INV4 applies to figures that have become unknowable, not to figures that are
simply old* — is exactly right and I adopt it verbatim. §6.4 is how it is enforced.

### 6.4 Unknown propagation: a total containing an unknown is an interval

> A total containing an Unknown line is **not a number**. It is an interval. There is no third option:
> excluding the line understates, treating it as zero lies, and carrying its last value pretends.

```
value(known)   = [x, x]
value(unknown) = [lastKnown − σ_lo , lastKnown + σ_hi]      # widened to the backstop horizon
value(never observed) = [0, ⊤]                              # ⊤ renders as "unknown", never as a number

[a,b] + [c,d] = [a+c, b+d]        −[a,b] = [−b,−a]        [a,b] × k = [ka, kb] for k > 0
```

Because liabilities are negative, an unknown card balance widens a total **downward**, which is the
direction that matters.

**Which end of the interval is the headline depends on what the surface is for**, and this is a call the
app must make rather than leave to a component:

| Surface | Headline shows | Why |
|---|---|---|
| **Cash position** (§4.2) | the **lower bound** | it is a decision surface. Being surprised by a card bill is the failure it exists to prevent |
| **Net worth, decomposition, history** | the **interval**, symmetric | it is an information surface; skewing it in either direction is editorialising |
| **Goal progress** | the **interval**, with status computed on the conservative end | the percentage is information; the status is a decision |

This is what produces presentation's `57.0% – 62.3%` on a goal with a dark funding account, and its
`+ up to CHF 12,400 unknown (2 accounts)` appendix under a hero figure. Both are correct renderings of
the same interval arithmetic.

**On counts, and an apparent conflict between two sibling documents.** Security's S5 forbids emitting any
count, cardinality or coverage ratio computed over rows outside the caller's perimeter. Presentation's
coverage bar emits *"2 unknown"*. These do not conflict, and the adjudication should be recorded once so
nobody re-litigates it:

> A count computed **over the audience-closed slice** is a legitimate figure and is permitted. A count
> that spans a perimeter boundary — "3 accounts excluded", "n accounts you cannot see" — is forbidden,
> at any granularity, including as a chart axis, a bar segment or a percentage denominator.

The coverage bar sits entirely inside the closed slice, so it is fine. What would not be fine is a
coverage bar on the household view whose denominator was the true household total.

### 6.5 `expectedCadenceDays` — confirmed, seeded, and refined

Presentation asks for `expectedCadenceDays` per account line, stored as data rather than inferred.
**Confirmed, with one addition:** each line also carries `cadence_source ∈ {CONTRACTUAL, OBSERVED,
ASSUMED}`. Seeded from the table below, refined from observed inter-arrival times once three
observations exist, and a refinement is *proposed to the owner*, never applied silently — because the
cadence is also the household's chore schedule, and changing someone's schedule without telling them is
how an app loses an argument it did not know it was having.

| Line | Days | Source |
|---|---|---|
| CA joint current (feed) | 1 | CONTRACTUAL |
| Revolut ×2, Wise (feeds) | 1 | CONTRACTUAL |
| Livret A ×3 — **balance** | 365 | CONTRACTUAL — interest capitalises 31 December |
| Livret A — **rate input** | 182 | CONTRACTUAL — 1 February, 1 August |
| CA mortgage | — | `never_stale`; MODELLED, not on the ladder (§5.7) |
| JB current ×5 | 31 | CONTRACTUAL |
| JB investment | 31 | CONTRACTUAL — quarterly if the household elects it; then 92 |
| CBA transaction, CBA cards ×2 | 31 | CONTRACTUAL |
| TopCard ×4 | 31 | CONTRACTUAL |
| Bourse Direct PEA — positions | 31 | ASSUMED |
| Bourse Direct PEA / Linxea — prices | 1 | CONTRACTUAL |
| Linxea assurance vie — units & fees | 92 | ASSUMED |
| Pillar 2 certificate | 365 | CONTRACTUAL — Q1 |
| Hostplus | 365 | CONTRACTUAL — year to 30 June, issued ~September |
| The house | 365 | ASSUMED; FROZEN |

Two of these correct the brief, and both corrections come from ease-of-use §2.3, which I confirm: the
Livret A **balance** obligation is annual (not twice-yearly — that is the rate), and Hostplus issues an
**annual** statement in September (a quarterly *document* obligation asks three times a year for a
document that does not exist; a quarterly optional balance snapshot is fine).

---

## 7. Projections and goals

### 7.1 What a projection is, and the three things it refuses to do

A projection here is **a distribution over future ledger states**, presented as a central path with
bands, produced from the household's own history and its own contracts. It is not a target, not a
plan, and not a forecast of anything external.

Three refusals, each of which removes a way for the app to be confidently wrong:

1. **No market growth by default.** The default assumption is 0% real. Presentation's §8.5 already
   labels it *"deliberately none"*, and that label is the design. A household projection with 6% equity
   growth baked in is mostly a chart of its own assumption; over ten years the assumption contributes
   more of the terminal figure than the household's savings do. Growth is configurable, in one place,
   visibly, and any non-zero setting propagates a marker onto every figure it touches.
2. **No FX forecast.** §7.4.
3. **No horizon beyond the evidence.** §7.3.

### 7.2 Decompose the projection by generative kind, not by account

This is the structural decision, and getting it wrong is what makes most personal-finance projections
useless — they bootstrap the *total* monthly net flow, which treats a contractual mortgage payment as a
random variable and produces bands wide enough to contain any outcome.

| Kind | Lines | Method | Uncertainty |
|---|---|---|---|
| **Contractual** | mortgage échéances (the table, §5.7), borrower insurance, school fees, taxe foncière, the French income-tax balance, card direct debits, Livret A interest at the published rate | the contract itself | ~zero; only rate-revision risk, modelled as a discrete scenario, not a band |
| **Recurring-observed** | salaries, rent, utilities, subscriptions — detected per §4.5 and confirmed once | last observed amount × detected schedule | small; the empirical dispersion of that series alone |
| **Discretionary** | the residual: everything not in the two rows above | **block bootstrap** (§7.3) | large; this is the inner band, and it is the only genuinely stochastic term |
| **Valuation** | PEA, Linxea, JB investment, Pillar 2, super, the house | flat in native currency at 0% real | a separate scenario band, off by default |
| **FX** | every cross-currency term | flat at the last published rate | a separate, differently-shaded band (§7.4) |

Only the discretionary row is resampled. Bootstrapping a €1,500 mortgage payment that is written in a
table would add fabricated uncertainty to the single most certain thing in the household's finances, and
would do it in the direction that makes the projection *look* more sophisticated.

### 7.3 The bootstrap, and the horizon limit

- The series is **monthly discretionary net flow, per currency, in native currency**, over the longest
  history with complete posting coverage.
- **Stationary block bootstrap** with geometric block lengths, mean 3 months. Blocks rather than
  independent draws because household spending is autocorrelated and because quarterly items cluster;
  geometric rather than fixed lengths so the resampled series carries no artefact at a fixed period.
- 2,000 resamples. Report P10 / P50 / P90.
- **Exact arithmetic throughout** (INV9). Resampling is over integer minor units; percentiles are order
  statistics with no interpolation between them, since an interpolated percentile is generally not a
  representable amount. A `float` in the resampler is as much a defect as a `float` in a balance.

**Horizon limits — F11, stated as a refusal rather than a caveat:**

- Fewer than **12 months of complete coverage** ⇒ **no band at all.** The projection renders as
  contractual items only, labelled *"not enough history for a range"*, and the trajectory line is drawn
  without a fan. A fan built on four months of data is a decoration.
- With a band, the maximum horizon is **3 × the available complete history**, capped at 120 months.
  A request beyond it is **refused, not silently truncated** — the API returns `maxHorizonMonths` and the
  UI shows why. Silent truncation is how a user comes to believe they asked for ten years and got it.
- Contractual-only projections may run to the end of the amortisation table regardless, because they are
  not forecasts.

### 7.4 The FX ruling — C8 confirmed, the band corrected

**Ruling: presentation's C8 is confirmed and adopted. The app forecasts no exchange rate. The central
path holds the last published ECB rate flat for the entire horizon — no drift, no forward points, no
purchasing-power adjustment, no mean reversion.**

Three reasons, in increasing order of importance:

1. At household horizons the spot rate is not usefully predictable, and any drift term is an
   unfalsifiable opinion embedded in a family's plans.
2. A forward curve is tempting because it is *available* and looks rigorous. It is an interest-rate
   differential — a market price, not a prediction — and every user will read it as a prediction. Showing
   a EUR/CHF forward and calling it "not a forecast" is a distinction that survives exactly one
   conversation.
3. It is INV10's whole point, projected forward. If FX movement is not achievement in the past, a
   projected FX movement is not projected achievement, and folding it into the central path makes part of
   the household's forecast progress imaginary. Presentation's hero reading `FX 0 (held)` is the honest
   version and it should stay exactly that literal.

**But the outer band as specified is the wrong shape, and I am asking for it to change.** Presentation's
§5.5 uses *"the 10th and 90th percentile of the last five years of the relevant pairs' 12-month moves"*.
Two problems:

- **Wrong shape.** A fixed 12-month move applied at every horizon gives a band of constant width. FX
  uncertainty grows roughly with √t. The band should be narrow next month and wide in year three, which
  is also the shape that stops a reader over-trusting the near term.
- **Wrong sample.** Five years of *overlapping* 12-month moves contains about four independent
  observations. A P10/P90 estimated from four observations is noise, and it will conspicuously exclude
  the tail this particular household is exposed to.

Replacement, using data we already ingest for free:

```
outer_band(h) = empirical P10 / P90 of overlapping h-day log changes in each
                relevant pair, over the trailing 10 years of the ECB series,
                applied only to the projected balance denominated in that currency
```

Ten years, because the ECB series is free back to 1999 and because ten years of EUR/CHF spans the January
2015 de-peg — a 15%-in-a-day move in the exact pair that determines whether a Geneva salary pays a Savoie
mortgage. A band that excludes it is not a band about this household. State on the legend that the
observations overlap and the band is indicative; that is honest and it is still far better than four
points.

Two further constraints:

- The band applies to the **currency-exposed share** of the projected position, not to the whole
  projection. A EUR figure funded from EUR accounts has no outer band, and its absence is informative.
- The two bands are **not summed in quadrature and not stacked as if independent.** The outer band is the
  inner band's envelope re-evaluated at the P10/P90 rate paths — i.e. it is the same projection under a
  different rate assumption, which is what presentation's *"plus exchange-rate variability"* legend
  already implies.

**The frontalier corridor** (presentation §8.4) is the right chart and deserves a defined quantity rather
than an illustration:

```
corridor(t)  = salary_CHF × rate(CHF→EUR, payment_date(t))     # monthly, at its own date's rate (R3)
exposure(t)  = ( inflow_CHF(t) − outflow_CHF(t) ) / inflow_total(t)
```

`exposure` is the number that should size the outer band, and it is also the most decision-relevant single
figure in this household's finances: it says what fraction of the family's cash flow is a bet on one pair.

### 7.5 Goals

A goal is `{ target: Money in goal_ccy, by: Date, funding: AccountId[], created_on: Date,
rate_set_at_creation: RateSetVersion }`.

**Goal currency governs — C7 confirmed.** The ratio is computed in `goal_ccy` and is therefore identical
for both partners regardless of display currency. Two people looking at the same goal must not disagree
about whether it is on track; that is not a display preference, it is a shared fact. The viewer's display
currency appears only as the `≈ CHF 56,470` annotation.

**`goal_ccy` and `rate_set_at_creation` are immutable.** Changing either retroactively rewrites the
progress history and invalidates the ghost track. Changing them means closing the goal and opening a new
one; the old one is retained.

**Progress decomposes by §3.5, restricted to the funding accounts over `[created_on, now]`:**

```
value_now − value_at_creation  =  C + M + X + U        (all in goal_ccy, exactly)
```

**A correction to presentation's §9.2, which makes the graphic more correct rather than less.** The mock
splits the meter fill into *contributed* and *"growth + FX"*, and separately draws a ghost track whose gap
from the main track is captioned as the FX contribution. Those overlap: the FX euros are drawn once inside
the fill and once as the gap. In the worked figures, the fill's second segment is €1,400 and the ghost gap
is €960 of the same money.

> **Rule: the meter fill is `C` and `M` only. `X` is the ghost-track gap and appears nowhere else. `U`,
> if non-zero, is a hollow extension segment.** Each euro is drawn exactly once, and the three segments
> plus the gap reconcile to the total.

Applied to presentation's own example: fill = contributed €32,800 + valuation €440; ghost gap = FX €960;
total €34,200; ratio 57.0%; ghost ratio 55.4%. Same picture, one fewer double count, and the caption
*"the gap is what exchange rates lent you: EUR 960"* becomes literally true.

**Required and actual run-rate — the dumbbell (§9.3):**

```
required_monthly = (target − value_now) / months_remaining        # in goal_ccy
actual_monthly   = trailing 6-month mean of C into the funding accounts
```

`required_monthly` assumes **no growth and no FX movement**, consistent with §7.1 and §7.4. `actual_monthly`
counts **contribution only** — never valuation — because a bull market must not be able to make the
household look like savers. This is §5.4/§9.2's "the rule that keeps goals honest", stated arithmetically.

**Status is a decision, so it is computed on the conservative end of the interval:**

| Status | Rule |
|---|---|
| `on-track` | projected completion (P50 contribution path, §7.3) ≤ target date |
| `behind` | projected completion ≤ target date + 6 months |
| `at-risk` | later than that, or unreachable at any contribution rate the household has ever achieved |
| `unknown` | any funding account is Unknown **and** the interval spans two statuses |

That last row matters: a goal whose status is genuinely ambiguous because a feed died must say so, not
pick the flattering end. Sorting is by months behind, descending — presentation's rule, confirmed.

### 7.6 Aggregate goals, and the double-counting problem

The brief asks for goal progress *"at individual account level and on aggregate"*, and aggregation is
where goals quietly go wrong, because one account can fund several goals.

- An account is **allocated** to goals in shares that must sum to **≤ 1**. The unallocated share is shown
  as *"unallocated"* rather than hidden. Over-allocation is rejected at entry, not reconciled later.
- Aggregate progress is reported as **two figures, never one**: total contributed toward goals (a Money,
  in the display currency) and the per-goal ratios (each in its own goal currency).
- **There is no single household "goal completion percentage."** A ratio of sums across goals in different
  currencies with different deadlines is a ratio of incommensurable things; it is dominated by whichever
  goal is largest, and it moves for reasons no one can explain. Refusing to emit it is the correct
  behaviour and should be treated as a requirement, not an omission.
- The son's Livret A may not fund a household goal (§4.6, FC3). It may carry its own goal inside the
  custodial view.

### 7.7 Eight things a projection must never do

1. Extend past `maxHorizonMonths`, or truncate silently instead of refusing.
2. Include a non-zero market growth assumption without marking every figure it touches.
3. Forecast a rate (§7.4).
4. Treat a contractual payment as stochastic (§7.2).
5. Include a figure whose source is Unknown as though it were zero (§6.4).
6. Include the custodial perimeter (FC3), or any account outside the audience-closed slice (INV14).
7. Net forecast discretionary spending into the **cash position** — that is §4.5's boundary, and
   conflating the two makes the priority-one headline drift for invisible reasons.
8. Present a band as a guarantee. The legend says *likely range*; the P10 is not a floor, and the copy
   must never imply it is.

---

## 8. The tax lens

D4 makes Partner A a **frontalier resident in France working in the Canton of Geneva** and lists four
consequences. Those consequences check out (§8.10) and they are not the whole picture. This section
specifies the lens's shape, its shipping order, and — the part that actually matters now — **what the
ledger must record from day one so that the lens is possible later** (§8.3). Everything here is
structural. None of it is advice, and the app must never present it as such.

### 8.1 Shape: a pure function over versioned rule packs

```
estimate( slice: LedgerSlice,        # audience-closed, INV14
          pack:  RulePackVersion,    # dated data, not code
          assumptions: AssumptionSet )
    →  { figures: Estimate[], citations: Citation[], unknowns: MissingInput[] }
```

- **Rule packs are data**: `{ jurisdiction, tax_year, effective_from, rates, thresholds, citation,
  confidence }`. Never constants in code, never a migration.
- **Pure and reproducible.** Same slice, same pack, same assumptions ⇒ identical output, forever. Every
  figure carries its pack version and its citation, so a number reprinted in two years can be explained.
- **A missing input yields `Unknown`, never a default.** A defaulted tax assumption is a fabricated
  number that looks like a computed one.
- **Confidence is a field on the rule, not on the output.** Where the law is genuinely unsettled — the
  French characterisation of Australian superannuation, whether the 2026 autonomy contribution follows
  the CSG exemption — the rule says so and the figure inherits it.

**The evidence that this is not over-engineering.** France's `loi de financement de la sécurité sociale`
for 2026, adopted in December 2025, raised CSG on capital income by 1.4 points via a new *contribution
financière pour l'autonomie*, taking the aggregate social levy on **dividends, fixed-income products and
securities gains to 18.6%**, while **retaining 17.2%** for life-insurance products, rental income, real
estate gains and certain regulated savings. A single hard-coded 17.2% would now be simultaneously wrong
for Partner A's PEA and right for Partner B's assurance vie, in the same app, on the same day, for
holdings this household already owns. That is what a versioned pack is for.

### 8.2 Shipping order: the mechanical part first

| Phase | What ships | Why first |
|---|---|---|
| **1 — Declaration support** (§8.3.4) | the foreign-account and foreign-contract checklist (forms 3916 / 3916-bis) | mechanical, verifiable, no modelling, and the app already holds exactly the required list. Highest value per unit of risk in the entire tax area |
| **2 — Wrapper clocks and thresholds** | PEA five-year date and €150,000 payment ceiling; assurance vie eight-year date and the €150,000 outstanding-premium threshold; Pillar 2 and super component splits | date and running-total arithmetic, not tax law, and decision-relevant *now* |
| **3 — Liability estimation** | PFU, abatements, IFI, cross-border credits | last, behind the estimation frame, and never with a hero figure |

This confirms presentation's C12 and its §15: tax is a **lens over existing views**, not a fifth
destination, and the largest an estimated tax figure may ever render is stat-tile size. Type size is how
confidence is communicated, and a wrong tax number in 48px is worse than no tax feature at all.

### 8.3 What the ledger must record from day one

This is the section that has to be right now, because everything in it is **unrecoverable later**. A wrong
balance is corrected by the next statement. A missing acquisition date is discovered by an accountant in
2041, when the broker's records have been purged and the counterparty has been through two mergers. This
is why §5.8 puts cost-basis-bearing figures on the always-human escalation row at any size.

#### 8.3.1 Per lot

The `Lot` record in §2.3 already carries most of this. Three additions and one generalisation:

| Field | Why |
|---|---|
| `acquired_on` **and** `trade_date` | record both. French and Australian rules do not agree on which governs, and the difference straddles a year end roughly 1 trade in 250 |
| `cost_native`, `currency_paid` | the fact of what was paid |
| `cost_basis_{ccy}` **per jurisdiction currency**, frozen, with `rate` and `rate_version` | **generalised from §2.3.** Australian CGT computes the cost base in AUD at the acquisition-date rate; France computes in EUR. Freezing only EUR loses the Australian basis irrecoverably. Freeze in every jurisdiction currency the household is exposed to — EUR and AUD today, CHF if Partner A ever becomes Swiss-resident |
| `acquisition_channel` ∈ purchase / transfer-in / corporate-action / reinvested-distribution | a transfer-in inherits a basis; a reinvested distribution creates a **new lot at a new date**, which is how a DRIP silently destroys a cost basis in every naive system |
| `parent_lot_id` | corporate-action lineage. A split, merger or spin-off must preserve the original acquisition date, and it cannot be reconstructed from a post-event position |

Disposal method stays as §2.3 set it: store lots, compute whatever basis the rule pack asks for — French
`prix moyen pondéré d'acquisition` for securities held outside a wrapper, specific identification where
Australian rules permit it. Never store a blended cost; lots cannot be recovered from it.

#### 8.3.2 Per wrapper

| Wrapper | Must be recorded from day one |
|---|---|
| **Bourse Direct PEA** | **the date of the first payment**, not the contract signature date — the five-year clock and the plan's fiscal seniority run from the first *versement*, and a signed but unfunded PEA has no seniority at all. Plus the full **payment stream** with dates, since the €150,000 ceiling is on cumulative payments and withdrawals do not restore capacity |
| **Linxea assurance vie** | contract inception date (the eight-year clock); the **premium stream with dates**, split at **27 September 2017** (different regime either side) and per-premium so the €150,000 outstanding-premium threshold is computable across contracts; premiums paid after age 70 flagged separately (art. 757 B vs 990 I); and the **insurer's country of establishment** — see §8.3.4 |
| **Swiss Pillar 2** | annual *avoir de vieillesse* split **obligatoire / surobligatoire** (different withdrawal treatment), employee vs employer contributions, interest credited, and any *versement anticipé* for a property purchase — which carries a repayment obligation and a lien. The annual certificate itemises all of it, so §2.6's "prefer the document's own decomposition" applies directly |
| **Hostplus superannuation** | preserved / restricted non-preserved / unrestricted non-preserved components; taxable vs tax-free components; contribution type per contribution (concessional / non-concessional); and **whether a CGT event I1 election was made on ceasing Australian residency, and when residency ceased** |
| **The house** | acquisition date, acquisition price, notarial fees, and **capital-improvement expenditure with dates and invoices**; plus a per-year `is_residence_principale` flag |

Two of these deserve emphasis because they are single facts with permanent consequences.

**CGT event I1.** On ceasing Australian tax residency, non-Taxable-Australian-Property assets are treated
as disposed of at market value. The taxpayer may instead elect to disregard the event, which deems those
assets to be Taxable Australian Property and defers the liability — and the election is **all-or-nothing
across assets**. Separately, the 50% CGT discount is unavailable for the portion of an ownership period
spent as a foreign resident after **8 May 2012**, apportioned by days. So the Australian cost base and the
discount fraction of everything the household holds there depend on: the residency-cessation date, whether
the election was made, and the resident/non-resident day counts. Three facts, recorded once, at onboarding.
Not recorded, they are gone.

**The PEA's first payment.** A €10 transfer into the cash account on the day of opening is what starts the
clock. If the household ever opens a second wrapper *pour prendre date*, the app should be the thing that
remembers the date — and should prompt for it, because it is the cheapest tax optimisation available to
anyone and it is invisible on a statement.

#### 8.3.3 Per person, per year — the facts that change the rate on everything

These are not in any bank statement and cannot be derived. They must be onboarding questions with an
annual review.

| Fact | Why it is load-bearing |
|---|---|
| **Tax residence** | assert it per person per year; never infer it from an address |
| **Social-security affiliation**: Swiss LAMal vs French CMU-frontalier | see below — roughly a 10-point swing on every euro of investment income |
| **Telework days in France** (Partner A) | see below — the highest-consequence tax fact in the household |
| Marital / fiscal status and `parts` | drives the assurance-vie €9,200 couple abatement and the IFI household definition |
| Whether Partner A elects Geneva **taxation ordinaire ultérieure** | determines whether actual deductions are available and whether a Swiss return exists at all |
| Commune of residence | DVF comparables and taxe foncière are commune-level. Crédit Agricole *des Savoie* covers both Savoie and Haute-Savoie; a Geneva frontalier is almost certainly in Haute-Savoie or Ain. Store the commune; never infer it from the bank |

**Social-security affiliation, and boxes 8SH / 8SI.** A person affiliated to a social-security scheme
outside compulsory French social security is exempt from CSG and CRDS on capital income and pays only the
**7.5% prélèvement de solidarité** instead of the full social levy — declared by ticking box **8SH** for
declarant 1 and **8SI** for declarant 2. In practice the administration treats both frontalier options,
Swiss LAMal and the French CMU-frontalier scheme, as qualifying. On 2026 rates that is 7.5% against 18.6%
on dividends and securities gains: a difference larger than most of what D4 lists, on a fact that appears
in no statement the app will ever ingest.

Two cautions to carry in the rule pack rather than in code: the reduced rate applies to *capital* income,
not to everything; and whether the new 2026 *contribution financière pour l'autonomie* follows the CSG
exemption is precisely the sort of question that should ship with `confidence: low` and a citation, not as
a number.

**Telework days.** The avenant to the France–Switzerland double-taxation convention **entered into force on
24 July 2025 and applies from 1 January 2026**, making permanent the rule that a French-resident employee of
a Swiss employer may telework up to **40% of annual working time** (including up to 10 days of temporary
assignment abroad) without changing where the salary is taxed. Swiss employers must report each
French-resident employee's telework rate to the federal tax administration, first for 2026 data in January
2027. Crossing 40% changes the taxation of the excess and cannot be undone within the year.

This is a **counter, not an estimate.** It needs a monthly one-tap input, a running year-to-date percentage,
a projected year-end percentage, and an alert well before the threshold. It is the cheapest high-value
feature in the tax area after the declaration checklist, it requires no tax modelling whatsoever, and the
app is the only place in the household's life where it will be tracked.

#### 8.3.4 Declaration support — what to record, and the trap

Forms **3916 / 3916-bis** require a French tax resident to declare every account **opened, held, used or
closed** abroad during the year — including payment accounts and neobank accounts — and foreign life
insurance or capitalisation contracts. The penalty is **€1,500 per account per undeclared year**, rising to
**€10,000** for an account in a non-cooperative jurisdiction, with an extended assessment period.

For this household that is the Julius Baer accounts (CH), the CBA accounts and cards (AU), Wise (BE), and
Revolut Bank UAB (LT). What the ledger must therefore hold per account: **institution legal name, country of
establishment, account type, account number, opening date, closing date**, and a computed
**`used_during_year`** flag.

Two notes:

- **The `used` criterion is computable and nothing else in this app comes close in value per unit of effort.**
  A Wise account with one transaction in the year is declarable, and the household will not remember. The
  posting stream already knows.
- **The trap is the insurer's country, not the broker's.** Linxea is a French distributor; its contracts are
  underwritten by French insurers, so the assurance vie is **not** a foreign contract and must **not** appear
  on the checklist. Record the *insurer's* country of establishment, never the broker's, or the app will
  manufacture a declaration obligation that does not exist — which is exactly as damaging to trust as missing
  one that does.
- The account number is required by the form and is precisely what the redaction pipeline is built to remove
  (INV8, `11-data-security.md` §3.2). It must be captured once, deliberately, by the account's owner at
  account-creation time, and stored as an account attribute in that account's perimeter — never recovered
  from a document after the fact.

### 8.4 The PEA, and the liquidation consequence

Referenced from §4.1's T2 band, which requires a wrapper consequence attached to the "available if we
liquidate" figure.

- A withdrawal **before five years** (counted from the first payment) generally **closes the plan**, with
  gains taxed under the PFU. Limited statutory exceptions exist (redundancy, invalidity, business creation).
- **After five years**, partial withdrawals no longer close the plan and further payments remain possible —
  the *loi PACTE* of 24 May 2019 replaced the old eight-year threshold with five.
- Gains are exempt from income tax after five years; **social levies remain due on the gain portion** of each
  withdrawal, and which 2026 rate applies to a PEA gain is a rule-pack question flagged low-confidence rather
  than asserted here.
- The payment ceiling is **€150,000**.

So the cash view's T2 band carries, per account, a `liquidation_consequence` string. For the PEA before five
years that string is *"closes the plan"* — which is not a cost in euros and must not be rendered as one, but
is the single most important thing to know before selling into it.

### 8.5 IFI, and the value of a zero

Net taxable real-estate assets valued at **1 January**, threshold **€1,300,000**, a décote smoothing the band
between €1,300,000 and €1,400,000, and a **30% abatement** on the résidence principale.

Ledger consequences:

- The house needs a **1 January valuation every year**, which is R1's worked case: 1 January is always a
  TARGET closing day, so a 1 January valuation always uses the preceding business day's rate, and the figure
  must state its rate date (§3.3).
- The mortgage's `capital restant dû` at 1 January is deductible, and the ledger holds it exactly (§5.7).
- On present holdings this household is far below the threshold, so the honest output is **"not applicable,
  and here is your margin"** — a number, a distance, and a note that the primary residence carries a 30%
  abatement. A tax lens that can confidently report a zero is more useful than one that reports nothing, and
  far more useful than one that implies the question is hard.

### 8.6 Withholding, and why gross must be booked

The chart of accounts in §2.2 already separates `INCOME:DIVIDEND:GROSS` from
`EXPENSE:WITHHOLDING:<jurisdiction>`. The rule behind it:

> **Always book the gross and the withholding separately. Never book only the net, and never impute a gross
> by grossing up at an assumed rate.**

Two reasons. First, a foreign tax credit is computed on the gross and capped at the treaty rate; a net figure
has destroyed the information the credit needs. Second, over-withholding is reclaimable — Swiss *impôt
anticipé* is levied at 35% and a French resident reclaims down to the treaty rate; Australian dividend
withholding is capped by treaty and franked dividends are exempt; interest withholding has its own rate — and
none of that is visible if only the net was ever recorded. Grossing up at an assumed rate is worse than
recording nothing, because the assumed rate is the answer we were trying to compute.

Where a statement gives only the net, book the net and set `gross_unknown`. That flag is a permanent, honest
gap; an imputed gross is a permanent, invisible fabrication.

### 8.7 Assurance vie

Eight-year clock from contract inception. After eight years, an annual abatement on the **gain portion** of
withdrawals of **€4,600** (single) or **€9,200** (married or PACSed couple). Gains above the abatement are
taxed at **7.5%** on the share attributable to outstanding premiums up to **€150,000** and **12.8%** above,
plus social levies — which the 2026 reform left at **17.2%** for life-insurance products.

The arithmetic the ledger must support:

```
gain_fraction(withdrawal) = withdrawal × ( 1 − premiums_remaining / contract_value )
```

which requires the **premium stream**, not a running total — hence §8.3.2. Succession treatment (art. 990 I
for premiums paid before age 70, art. 757 B after) requires the same stream keyed on the insured's age at
each payment.

D5's mark-to-market applies to *valuation*, not to *tax*: the insurer computes the taxable gain and issues
the IFU, and our marked figure is a decision aid that must be reconciled to the insurer's statement per
§5.6. Where they differ, the insurer is right about tax and we are right about nothing.

### 8.8 Pillar 2

Untaxed while it accumulates. Swiss wealth tax does not reach a French resident except on Swiss-situated real
property, so the Pillar 2 balance is a French-side asset with no Swiss annual charge.

On withdrawal as a **lump sum**: Swiss source tax is levied by the canton of the paying institution and is
**refundable on proof of French tax residence and of declaration in France**. On the French side, article
163 bis CGI permits — on **express and irrevocable election** — a **7.5% liberatory levy applied after a 10%
deduction**, an effective **6.75%**, provided the capital is paid in a **single instalment**; deliberate
fractionation defeats it, and the contributions must have been deductible.

The app's job here is not to answer the capital-versus-annuity question. It is to **hold the inputs** — the
obligatoire/surobligatoire split, the certificate's own decomposition, any early withdrawal for property and
its repayment obligation — for a decision that is decades away and is the largest single tax decision this
household will make. The line is `HOLD` between annual certificates (§6.1) and T3 throughout (§4.1).

### 8.9 Australia

- **Super access.** Preservation age is **60** for anyone born after 30 June 1964, and benefits taken at 60
  or over from a taxed fund are not assessable income in Australia. Ceasing Australian residency does **not**
  unlock super early, and the departing-Australia payment route is unavailable to Australian citizens and
  permanent residents. So Hostplus is T3 for decades and `HOLD` between annual statements.
- **CGT.** A foreign resident is generally taxable only on Taxable Australian Property; the 50% discount is
  unavailable for foreign-residence periods after 8 May 2012; and CGT event I1 governs what happened on
  departure (§8.3.2).
- **Withholding.** Book gross plus withholding, per §8.6.
- **The French side is genuinely unsettled.** How French law characterises an Australian superannuation
  interest — and therefore whether and how it is reportable and taxable in France — does not have a settled
  answer I am willing to encode. The rule pack must carry it as `confidence: low` with a citation to the
  open question, and the lens must render it as an explicit unknown rather than choosing. This is exactly the
  case the feasibility assessment had in mind: a wrong tax number presented confidently is worse than none.

### 8.10 D4, verified — what holds and what is missing

**Holds.**

- Geneva is **not** a party to the 1983 Franco-Swiss frontalier accord. Geneva-source employment income of a
  French resident is taxed **at source in Geneva**, unlike the eight cantons that tax in the state of
  residence. D4's central claim is correct and it is the correct thing to have singled out.
- France taxes worldwide income and relieves double taxation by a **credit equal to the French tax**
  (convention of 9 September 1966, art. 25 A), declared on forms 2047 and 2042.
- Swiss wealth tax does not reach a French resident except on Swiss-situated real property.
- Pillar 2 is untaxed until withdrawal, and its withdrawal treatment is a genuine cross-border question
  (§8.8).
- Australian assets are held by a non-resident for Australian purposes, which changes withholding and CGT
  (§8.9).
- Foreign account and foreign life-insurance declaration obligations apply (§8.3.4).

**One clarification with a direct ledger consequence.** Because relief is by *credit equal to the French
tax*, the Swiss salary is effectively not taxed twice **but still enters the French return** and therefore
still affects the household's French effective rate and every threshold keyed to *revenu fiscal de
référence*. So the salary must be carried at **gross, in CHF**, with Swiss tax withheld as a separate figure —
which §8.6 requires anyway, and which means the ledger needs the payslip decomposition, not just the net
credit that lands in the bank.

**Three things D4 does not mention, each with a material effect.**

1. **The CSG/CRDS position (§8.3.3).** 7.5% instead of the full social levy on capital income, via boxes
   8SH/8SI, contingent on a per-person affiliation fact. Larger in euro terms, for this household, than most
   of what D4 does list.
2. **Telework (§8.3.3).** The 40% rule became permanent from 1 January 2026 and is now reported by the
   employer to the Swiss federal administration. It is a ledger input from day one.
3. **The 2026 social-levy split (§8.1).** Any tax figure conceived before December 2025 is now stale in a way
   the app must be structurally able to absorb — which is the argument for §8.1's rule packs, made by events
   rather than by me.

### 8.11 Seven things the tax lens must never do

1. Present an estimate without the persistent estimation frame (presentation §15).
2. Give an estimated tax figure a hero-sized rendering. Ever.
3. Default a missing input. `Unknown` is a valid output; a default is not.
4. Emit a figure without its rule-pack version and citation.
5. Compute over anything outside the audience-closed slice (INV14), or include the custodial perimeter (FC3).
6. Use the mark-to-market valuation where the insurer's or custodian's own tax figure exists (§8.7).
7. Use the word "advice", or any construction that implies it, anywhere in its copy (D4).

---

## 9. Invariants: one exception, two rulings, two additions

### 9.1 INV1 — the one declared exception

**The exception:** `Lot.cost_basis_{ccy}` is a converted value, and it is persisted.

**Why it is not the thing INV1 forbids.** INV1 exists because a persisted conversion corrupts history:
the same historical figure renders differently depending on when the conversion happened, and a rate
revision (R6) silently rewrites the past. A cost basis has neither property. It is **a legally fixed fact
created at the moment of acquisition**, in the jurisdiction's own currency, at that day's rate, and it
never changes again for the life of the lot. Recomputing it at display time from the current ECB series
would not merely be expensive — it would be **wrong**, and it would change every time the ECB revised a
1997 rate.

**What keeps it honest.** Four conditions, all enforceable:

1. Stored **with** `rate` and `rate_version`, so it is reproducible and auditable rather than asserted.
2. Frozen at acquisition. Any write after the lot's creation is a defect, and there is no code path to
   update it other than a corporate-action lineage event that creates a *new* lot.
3. **Used only by the tax lens.** Any read of `cost_basis_*` from the valuation, cash-position,
   decomposition, projection or goals code is a defect and should be caught by a lint rule on the module
   boundary, not by review.
4. **Never displayed as a current value.** Where it appears it is labelled as an acquisition-date basis
   with its date.

**Generalised from §2.3**, per §8.3.1: freeze in every jurisdiction currency the household is exposed to,
not only EUR. Australian CGT computes in AUD and that basis is as unrecoverable as the French one.

### 9.2 Ruling on presentation's requested INV1 exception (its §12)

Presentation asks to persist a rendered display snapshot — the last-viewed state's figures — solely so
the PWA paints instantly on cold start; never readable by any arithmetic, purged on reconnect and on
rate-set change, expired after seven days. It offers to drop the idea if I judge the distinction will not
survive contact with a codebase.

**Ruling: the distinction will not survive as written. The need is real. Granted in a different form,
which turns out not to be an INV1 exception at all.**

**Why "never readable by any arithmetic" fails as a rule.** It is a rule about *intent*, and intent is
not typed. The snapshot is, by construction, the only thing available before the ledger hydrates — so the
first feature that wants a number early will read it, and not maliciously: a "since you last looked"
delta, an offline goal ring, a sparkline that needs a previous point. Each of those is a reasonable
half-hour change, each passes review, and the third one turns a screenshot into a data source. Six months
later a figure on screen has no provenance and INV3 has quietly become false. The invariant would be
enforced by everyone remembering it, which is the enforcement model INV1 exists to replace.

**Why the need is nevertheless real.** INV5 makes offline usability a requirement rather than a polish
item, the delivery target is an Android PWA opened on trains and aeroplanes, and presentation is right
that a projection's fan bands are expensive enough that a cold start would be visibly slow. Taking the
200–400ms is the wrong trade for a household app that must feel worth opening.

**Granted, on three conditions — and the first one is the whole ruling.**

1. **The snapshot stores glyphs, not numbers.** It holds the **formatted strings** — `"CHF 412,600"`,
   `"57.0% – 62.3%"`, `"+ up to CHF 12,400 unknown"` — never the minor units they were rendered from.
   A string is not arithmetic-shaped: there is no `Money` to add, no `Decimal` to multiply. Anyone who
   wants to compute from it must first write a parser, which is a **visible, reviewable act** rather than
   an invisible one. This costs nothing, because painting is the snapshot's entire purpose, and it
   converts an unenforceable rule about intent into a structural property of the data.
2. **It lives in a physically separate store, behind a module boundary that a linter checks.** A distinct
   IndexedDB database with its own name; an accessor module exporting exactly one function returning
   `{ strings, renderedAt, rateSetVersion, ledgerVersion }`; that module may not import the money or
   decimal packages, and nothing under the ledger, valuation, decomposition, projection or goals
   namespaces may import it. Two lint rules, both trivially checkable in CI.
3. **It is visibly a photograph, and it is purged on more triggers than proposed.** While the snapshot is
   what is on screen: the header states its age, figures render in the secondary/`MODELLED` treatment, and
   every affordance that would compute something is disabled until hydration completes. Purge on
   reconnect, on rate-set change, and at 7 days — as proposed — **plus** on ledger version change, on
   device revocation (security §7.2), and **on any perimeter-policy change**. That last trigger is mine
   and it is not cosmetic: a snapshot rendered before an account moved into a private perimeter would
   repaint a figure the viewer is no longer entitled to see, which turns a rendering cache into an
   authorisation bypass. The snapshot must also be encrypted under the same perimeter key as its content,
   since a rendered household summary is the most concentrated disclosure in the system.

**Under those conditions no converted value is persisted — only its typography.** So I am recording this
not as an exception but as a **gloss on INV1**, which I would like added to `02` §4:

> *INV1 gloss:* INV1 governs **values**. A value is something arithmetic can be performed on. Rendered
> text is not a value, provided it is stored as text, kept out of every arithmetic module by construction,
> and marked with the moment and rate set it was rendered from.

To presentation: don't drop it, and don't take the cold-start penalty. Take the strings.

### 9.3 Two invariants I want to add

Offered in `02` §4's style, for adoption alongside security's INV12–INV14.

> **INV15 — A derived figure is exactly as fresh as its stalest input.** Freshness, confidence and the
> Unknown state propagate through every derivation — conversion, aggregation, decomposition, projection,
> goal progress and the tax lens — and no derived figure may report a freshness, confidence or as-of date
> better than the worst of its inputs. A position priced from this morning's NAV against a four-month-old
> unit count is a four-month-old figure.

*Why it needs to be an invariant rather than a rule in §6:* it is the one violation that is invisible in
the output. Everything about a stale-quantity, fresh-price figure looks live, and there is no screen on
which the error announces itself.

> **INV16 — Observations are append-only; derivations are disposable and reproducible.** A correction is a
> new observation, never an edit to an old one. Every stored derivation — reconciliation postings, plugs,
> model outputs, cached balances, converted figures, snapshots — is a cache identified by the
> observation-set version and rate version it was computed from. Deleting every derived row and rebuilding
> must produce bit-identical state.

*Why:* this single property is what makes F3, INV7 and INV3 hold **simultaneously** under late and
out-of-order arrival, and it is the specific defence against the failure this whole design is arranged to
prevent — a reconciliation plug that was honest when it was created quietly becoming permanent because
something downstream started depending on it.

### 9.4 On security's INV12, INV13 and INV14

**INV12 — read-only by construction: adopt, without reservation.** It carries a financial-correctness
benefit the security document does not claim: a ledger with no capability to move money never has to
reconcile **its own effects**. Every discrepancy in §5 therefore has an external cause, which removes an
entire category of ambiguity from reconciliation — the one where you cannot tell whether the system caused
the difference it is reporting.

**INV13 — no perimeter administrator: adopt, and I own one consequence.** S10 accepts that a four-eyes
escalation on a private account may sit unresolved forever, because no one outside that perimeter may
resolve it. The financial-correctness consequence is that an unresolved escalation must leave the figure
**Unknown** — not "pending, showing last known" — in every audience that contains the account. Pleasingly,
this costs the household view nothing: an account in a private perimeter is outside the `{A,B}` slice
already, so the household total is unaffected either way. The cost falls entirely on the owner's own view,
which is where it belongs, and where it will actually be fixed.

**INV14 — every figure declares its audience: adopt, and it is load-bearing for me.** Without it, §3.5's
decomposition silently means different things in different views. With it, the identity becomes
audience-relative, which is what forces the perimeter term in §10.2. One adjudication to record so that
`10` and `15` stop appearing to disagree: **a count over the audience-closed slice is a permitted figure;
a count that spans a perimeter boundary is not** (§6.4).

---

## 10. Interfaces: what the other agents asked for

### 10.1 Architecture's eight questions (`12` §10.5)

| # | Question | Answer |
|---|---|---|
| 1 | Money representation | **Confirmed as assumed.** `BIGINT` minor units + ISO 4217 exponent; `Quantity` decimal(28,8); `Price` decimal(28,10); `Rate` decimal(28,10). Round half-away-from-zero, **once**, at display; largest-remainder apportionment for displayed component sums (§3.1). Minor units rather than NUMERIC for money, because addition must be exact and order-independent |
| 2 | Observation vs posting authority | **Both, per §2.6.** Truth is the fold over postings *including* derived reconciliation postings. A statement's closing balance is a **check** where posting coverage is complete and a **fact** where it is not — and `choose_contra` is the function that decides which. Per-class carry policy in §6.1 |
| 3 | Transaction identity | `external_ref` where present **and measured stable** (§5.3); otherwise the natural key. **Pendings never enter the ledger** and match to settled postings by the three-tier rule in §5.5 |
| 4 | Plausibility parameters | §5.8, with numbers and derivations. Two corrections to ease-of-use's proposal: self-scaling drifting bands, and property as its own class |
| 5 | Rate convention | §3.2, R1–R7. Valuation at the figure's own date; a flow at its own **booking** date; `rateFor()` must return `carried_forward` and the caller must render the rate date. Ask for the **value date** where it differs and the account is interest-bearing (Livret A quinzaines, §6.2) |
| 6 | Restatement semantics | **Append-only reversal**, never amendment in place (§5.6). Architecture's own preference, confirmed, with the reasons |
| 7 | Modelled figures | **Stored and dated**, agreeing with architecture's lean, with `model_version` on every row. Computed-on-read silently rewrites history whenever the model changes, which is the one thing a modelled figure must not do |
| 8 | Balance kind | `closingBooked` is the ledger balance; `interimAvailable` (or `expected`) is the **spendable** figure feeding T0. Store both, label both, **never compare across kinds** (§5.5) |

Plus one amendment architecture did not ask for: **`12` §7.3 layer L1's "return the existing document and
its state" is replaced by the constant response** required by S9, with duplicate detection moved into the
uploader's client (§5.2, condition 2).

And one confirmation: `12` §9.1 offers to drop its materialised-view exception to INV1 if I prefer no
exception at all. **I prefer no exception.** A materialised view of converted values is a cache under
INV16, which means it must carry the rate version and observation-set version it was built from and be
rebuildable and disposable — at which point it is not an exception to INV1, it is an ordinary derivation.
Keep it if it buys performance; drop it if it does not; either way it is not an invariant question.

### 10.2 `decompose()` — the contract, amended

Presentation's §11.1 asks for exactly three cause terms, `unknown` as a list rather than zero, and the
residual folded explicitly into `fx`. Security's FC1 asks for a fourth class for perimeter egress. Those
two requests appear to conflict, and §3.5 already carries a fourth term. Here is the resolution.

**Three *cause* terms — confirmed and preserved.** `contribution`, `market`, `fx`. The palette constraint
(R1: hue means cause) is respected, and a fourth hue is not introduced.

**But `unexplained` is not folded into `fx`, and it is not a cause.** Folding it would put honest ignorance
into the aqua cause hue and make FX movement look larger than it was — in a household where FX movement is
the exact thing the user is trying to distinguish from their own behaviour. `unexplained` is the *absence*
of a cause, and it renders in the **unknown vocabulary** (R4: hollow, outlined, no fill) rather than in any
cause hue. That satisfies presentation's real constraint, which is about the palette, while keeping §3.5's
"always displayed, never folded" property.

**And `perimeter` is a boundary term, not a cause either.** FC1 is right that without it a transfer out of
the shared perimeter reads as a loss in the household view. Adding a fifth `flow_class`:

```
flow_class = PERIMETER               # value left or entered this audience's visible universe
  PERIMETER:EGRESS
  PERIMETER:INGRESS
```

It is **audience-relative and zero by construction in a single-principal audience**: a transfer from the
joint account to B's private account is entirely inside B's own `{B}` slice and is an ordinary internal
transfer there. It is non-zero only in the `{A,B}` household view, where it says precisely *"this money
left the shared view; it did not leave the household"* — which is neither contribution nor loss.

*Disclosure check.* S7 already establishes that the visible side of a cross-perimeter transfer shows money
leaving the perimeter. `PERIMETER:EGRESS` discloses that same fact and nothing more: no destination, no
resulting balance, no counterparty. It must never be broken down by destination perimeter, and it must not
be reported at a finer time granularity than the visible posting already is. **One residual handed back to
security:** with exactly two partners, an egress the viewer did not initiate is necessarily the other
partner's. That is inherent in S7 rather than created here, but it should be named in `10` §11 rather than
left implicit.

```ts
decompose(input: {
  audience: PrincipalId[];        // INV14 — required, no default, no server-chosen value
  accounts: AccountId[];          // must be a subset of the audience-closed slice; violation is an error
  from: Date; to: Date;
  displayCcy: Ccy;                // caller's, per INV2
}): {
  opening: Minor; closing: Minor;

  // three cause terms — the only three, forever
  contribution: Minor;
  market:       Minor;
  fx:           Minor;

  // not causes: rendered in the unknown / boundary vocabularies, never in a cause hue
  unexplained:  Minor;
  perimeter:    Minor;            // audience-relative; 0 in a single-principal audience

  byPair: Array<{ from: Ccy; to: Ccy; effect: Minor;
                  rateOpen: Decimal; rateClose: Decimal }>;
  marketSplit: { observed: Minor; estimated: Minor };        // §3.5
  granularity: 'DAILY'|'MONTHLY'|'QUARTERLY'|'ANNUAL';       // the worst contributing account's
  rateSet: { date: Date; source: 'ECB'; version: string };

  unknown: Array<{ account: AccountId; lastKnown: Minor | null;
                   since: Date; band: [Minor, Minor] | null }>;
}
```

> **Identity, exact:** `contribution + market + fx + unexplained + perimeter === closing − opening`.
> No residual, no rounding slack, no interaction term. This is a test, not a hope — §3.6's worked example
> is the fixture.

`unknown` is a **list, never zero** — confirmed, and extended with `band`, so R4 can render an interval
rather than only an appendix. `granularity` is the addition presentation did not ask for and needs: it
carries the worst contributing account's observation cadence, which is what licenses or forbids drawing the
decomposition at monthly resolution (§3.5).

### 10.3 The other contracts presentation asked for

**`liquidityTier`.** Presentation's `'T0'|'T2'|'T30'|'locked'` does not match §4.1 and would produce two
different tier vocabularies in one app. Use §4.1's:

```ts
liquidityTier(account, asOf): {
  tier: 'T0' | 'T1' | 'T2' | 'T3';
  settlement: string;                 // "immediate", "≤3 business days", "T+2 (T+1 from 11 Oct 2027)"
  consequence?: string;               // "closes the plan" for a PEA before 5 years — §8.4
  unclassified?: true;                // a JB holding failing the §4.3 four-part test: defaults to T2
}
```

Mapping if presentation's names must survive in the view layer: `T0 → T0`, `T1 → T30` (available inside
the horizon), `T2 → T2`, `T3 → locked`. I would rather it did not; four tiers with one set of names is
cheaper than a translation table.

**`goalProgress`.** Per §7.5, with the double-count corrected:

```ts
goalProgress(goal, audience): {
  goalCcy: Ccy; target: Minor; valueNow: Interval; valueAtCreation: Minor;
  contributed: Minor; valuation: Minor; fx: Minor; unexplained: Minor;
     // contributed + valuation + fx + unexplained === valueNow − valueAtCreation
  ratio: Interval; ratioAtCreationRates: Interval;      // the ghost track
  requiredMonthly: Minor; actualMonthly: Minor;         // contribution only, never valuation
  projectedCompletion: Date | null;
  status: 'on-track' | 'behind' | 'at-risk' | 'unknown';
  perAccount: Array<{ account, native: Money, contributed, valuation, freshness }>;
}
```

The meter fill is `contributed` + `valuation`. The ghost gap is `fx`. Nothing is drawn twice.

**`projection`.** Per §7.2–§7.4:

```ts
projection(horizonMonths, audience, displayCcy): {
  central: Point[];  inner: Band[];  outer: Band[];
  contractual: Point[];             // the part that is not a forecast at all — draw it differently
  assumptions: Assumption[];        // presentation §8.5's table, as data
  fxHeldFlatAt: { date: Date; version: string };
  maxHorizonMonths: number;         // requests beyond are refused, not silently truncated
  basis: 'full' | 'contractual-only';
}
```

**`Observation.expectedCadenceDays`.** Confirmed as required per-line data, plus `cadence_source`
(§6.5). And confirmed with emphasis: `connection.state === 'expired'` forces `value: null` **at the
source**. Presentation must never be in a position to choose between a stale number and Unknown, and
neither must anything else — §6.3's trigger 1 is a step function precisely so that no consumer has a
choice to make.

### 10.4 Ease of use (`14` §1058)

| Request | Answer |
|---|---|
| Volatility class per account, plausibility bands, materiality thresholds | §5.8. Three classes confirmed, drifting band made self-scaling, property split into its own class, one materiality row added (cost-basis-bearing figures) |
| Propagation of Unknown and Held through aggregates | §6.4. A total containing Unknown is an **interval**; which end is the headline depends on whether the surface is a decision or information surface |
| Superseding corrections — append-only, old value retained, reason recorded | §5.6, and promoted to INV16 |
| **Predicted balance** for movement-triggered asks | Supported: `last observed + observed transfers + contractual accrual`, carrying carry-policy `ACCRETE` and a distinct provenance kind `MODEL`. It may seed a question to the household; it may **not** enter the cash position or any total (§6.1) |
| Card liability floor semantics, and excluding Ageing figures from the cash view | §4.4's L1–L4 is the floor plus the labelled accrual; §6.3's trigger 2 with liquidity thresholds halved (§5.8) is what keeps an ageing card balance out of the headline |
| Does *"same as last time"* record a new assertion at a new as-of date? | **Yes, and it must.** It is a new `BalanceAssertion` with `source.kind = MANUAL`, `confirmed_by = HUMAN`, and today's `as_of`. It is *not* an extension of the old figure's validity, because the two carry different information: one person looked today and says it has not moved. Under §6.3 it resets σ from zero, which is the correct and intended effect |

Ease-of-use's INV4 reframing — *INV4 applies to figures that have become unknowable, not to figures that
are simply old* — is adopted verbatim, and §6.3's three triggers are its enforcement.

### 10.5 Data security (`11`)

- **Market-data query shape (`11` §12.4).** Per-ISIN queries disclose the portfolio to the data provider,
  which is a real leak against the threat model's third-party row. **Prefer a bulk end-of-day file covering
  a whole exchange segment or fund universe** over per-ISIN calls: the household's holdings are then a
  subset of something we would have fetched anyway, and there is no query to observe. Euronext Paris EOD
  plus the relevant UCITS universe is a small daily file. If bulk is unobtainable at household price,
  per-ISIN queries must be padded with decoys from the same segment — a weak control that should be
  documented as weak, not relied on. **Fallback if market data is dropped entirely:** the custodian's and
  insurer's own statements are authoritative anyway (§5.6), so the app degrades to monthly marks with
  wider σ, which is fully INV5-compatible. Market data buys resolution, not correctness, and it should be
  the first thing cut if A1's budget binds.
- **Erasure vs INV7 (`11` §6).** The permanent hash tail that lets erasure and idempotency coexist must
  store the **perimeter-scoped HMAC**, not the bare SHA-256 (§5.2). A permanent bare-hash tombstone is a
  permanent oracle.
- **The son's Livret A (`11` §10.3).** It survives contact with the projection maths cleanly, because it
  never enters it: excluded from T0/T1, from the headline, from goals, from household net worth, and from
  every projection (§4.6, §7.7 rule 6, FC3). "Assets under our administration" is a separate total with its
  own audience and its own arithmetic, and nothing subtracts it from anything.

---

## 11. What I could not settle

Stated rather than papered over, in the order I would resolve them.

1. **Which 2026 social-levy rate applies to a PEA gain.** The December 2025 reform raised the aggregate
   levy on dividends, fixed-income products and securities gains to 18.6% while retaining 17.2% for
   life-insurance products, rental income, real-estate gains and certain regulated savings. Which side a
   PEA withdrawal's gain falls on, and whether the frontalier 7.5% exemption extends to the new autonomy
   contribution, are two questions I am not willing to answer from secondary sources. Both ship as
   `confidence: low` rule-pack entries with citations. **Verify with the household's accountant before the
   liability estimator ships**, not before the wrapper clocks do — the clocks do not depend on it.
2. **The French characterisation of Australian superannuation.** Genuinely unsettled. The lens renders it as
   an explicit unknown (§8.9) rather than choosing, and the declaration checklist should ask the accountant
   whether the Hostplus interest is reportable rather than asserting either answer.
3. **Whether Partner A's CGT event I1 election was made, and the residency-cessation date.** Not a design
   question — a fact the household holds and the app must capture at onboarding. It permanently determines
   the Australian cost base of everything they own there, and it becomes unrecoverable as records age.
4. **Which balance types each of the four ASPSPs actually populates.** §5.5's rule is right regardless, but
   the per-connection fact must be observed at link time in build phase 1, because if an ASPSP populates
   only `interimAvailable` then that account's ledger balance has no direct feed source and its statement
   becomes load-bearing.
5. **Whether the ECB SDMX parameter list in §3.2 is still current.** The portal's URL scheme has changed
   once. The plain XML files are the resilient path; the SDMX call is for gap-filling and revision
   detection, and should be verified at implementation rather than trusted from this document.
6. **The Livret A rate series before 1 August 2026.** The current rate is confirmed at 1.70%; the historical
   series must be backfilled from the published schedule before §6.2's accrual model can be checked against
   any statement older than this year.

---

*Rates, thresholds and wrapper rules cited here were verified in August 2026 and are stated as structural
facts about how the ledger must be shaped, never as advice. Every one of them belongs in a dated rule pack
(§8.1) rather than in this document, precisely because the 2026 social-levy split proved that a number
written into a design is a number that will be wrong before the design ships.*
