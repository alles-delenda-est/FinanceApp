# Graphical Presentation

**Status:** Design output. Binds the view layer.
**Owner:** Graphical presentation agent
**Date:** 27 August 2026
**Inputs:** `01-feasibility-assessment.md`, `02-decisions-and-constraints.md` (read in full)
**Method:** the house `dataviz` method — form chosen before colour, every palette run through
`scripts/validate_palette.js` rather than eyeballed. Validator output is pasted verbatim in §3.

---

## 0. What this document decides

Twelve calls, made rather than offered. Everything after this is the argument and the build detail.

| # | Call |
|---|---|
| C1 | **Hue means *cause*, never *currency*.** Blue = contribution, orange = market, aqua = FX. Currencies are identified by their ISO code as text and never by colour. |
| C2 | **Freshness is measured in cadence units, not days.** A three-week-old Julius Baer statement is *Current*; a three-day-old Revolut balance is *Stale*. The ratio `age ÷ expected_cadence` drives everything. |
| C3 | **Silence means fresh.** A figure inside its cadence carries no badge, no colour, no caveat. Only deviation is decorated. One coverage bar per view replaces fifty badges. |
| C4 | **Unknown turns a point into a range.** An expired connection never renders a stale number and never renders zero. The total becomes an interval and the hero figure grows an appendix. This is how INV4 survives contact with a chart. |
| C5 | **The bridge is the house form for "how did we get to this number."** Two of them: the liquidity bridge (View 1) and the change bridge (View 2/net worth). Same grammar, learned once. |
| C6 | **The compact FX decomposition is text, not a chart.** Three causes is a handful of numbers; a chart of three bars is worse than three labelled numbers. The chart form appears only on expansion. |
| C7 | **Goals are denominated in a goal currency and their progress ratio is computed in it**, regardless of the viewer's display currency. A second ghost track shows what progress would be at the rate on the day the goal was set; the gap between the tracks *is* the FX contamination, drawn. |
| C8 | **The app forecasts no exchange rate.** Projections hold the last rate flat and widen a separate, differently-shaded band for currency variability. A projection that predicts EUR/CHF would be a lie with a confidence interval on it. |
| C9 | **The table view is the provenance view.** Every chart's accessibility twin is the same table that carries as-of, source and derivation. INV3 is satisfied by one surface doing two jobs, not by printing provenance on every number. |
| C10 | **There is no "connected" chrome anywhere in the four views.** Feed, document, computed and manual are four values of one `source` field, visible only in the provenance drawer. INV5 is a rendering decision, not a fallback path. |
| C11 | **The partiality label on shared views is constant, never conditional.** A caption that appears only when something is hidden is itself a disclosure channel. |
| C12 | **Tax is a lens, not a tab.** It re-renders existing views inside a persistent estimation frame. The one thing that ships early under the tax banner is the foreign-account declaration checklist, which is a table and needs no modelling. |

One place where this design presses on an invariant is stated openly in §12.

---

## 1. The four rules the whole system hangs on

Everything below is an application of these. If a future screen contradicts one, the screen is wrong.

### R1 — Hue means cause, never currency

This household holds four currencies and will be tempted, on every screen, to colour by them.
Resist it. Currency is *nominal* identity with a perfect free channel already attached: a
three-letter code that every reader already knows. Spending the categorical hue budget on
EUR/CHF/AUD/USD leaves nothing for the thing that actually needs colour — **why** a number moved.

So the categorical budget is spent once, on three slots, forever:

| Slot | Hue | Means |
|---|---|---|
| 1 | blue | **Contribution** — money you moved. The only one you control. |
| 2 | orange | **Market movement** — instruments repricing in their own currency. |
| 3 | aqua | **FX** — pure exchange-rate effect. |

Three causes, three slots. That is not a coincidence chosen for tidiness: the house palette
validates its **first three slots on the all-pairs test** in both light and dark, which is the
harder gate and the one that governs scatter, small multiples and any layout where any two marks
can end up adjacent. A fourth cause would break the palette before it broke the arithmetic. If
financial correctness ever proposes a fourth term (fees, accruals), it folds into one of the
three or into "Other" grey — it does not get a colour.

Currency mix, when it must be shown, is a **list of single-hue bars** (all slot-1 blue) with the
ISO code direct-labelled — the correct treatment for nominal categories under the house method,
and immune to the "value ramp on nominal categories" anti-pattern.

### R2 — Silence means fresh

A financial app that annotates every figure with its age teaches the reader to stop reading
annotations. The information budget for freshness is small and must be spent on deviation only.

- A figure **within its cadence** gets nothing. No dot, no tint, no tooltip nag.
- A figure **past its cadence** gets one glyph.
- A figure **well past** loses its fill and its ink weight.
- A figure that **cannot be known** does not appear as a number at all.

Aggregate confidence is carried by exactly one object per view — the **coverage bar** (§4.3) —
which answers "how much of this total do you actually know?" in one 8px strip.

### R3 — The bridge is the house form for "how did we get to this number"

Both hard questions in this app are the same shape: a headline number that is the end of a chain
of additions and subtractions the reader cannot see. Liquidity is `cash + accessible − cards −
bills`. Net worth change is `opening + contribution + market + FX`. Both get a waterfall, with the
same geometry, the same anchor treatment, the same 2px surface gaps. Learn it on View 1, read it
on View 2.

### R4 — Unknown turns a point into a range

INV4 says an expired connection renders as *unknown*, never as a stale number. That is easy on a
row and hard on a total, and impossible on a line chart unless you decide what a total *is* when
a term is missing. The decision:

> A total with an unknown component is **not a number. It is an interval**: `[confirmed, confirmed
> + last-known-unknowns]`. The confirmed floor is rendered as the figure; the interval is rendered
> as an appendix on a stat tile, a shaded band on a time series, and a hollow segment on a
> coverage bar.

Zero is never substituted. Last-known is never silently substituted. The reader sees a floor and
a fog, which is the true epistemic state.

---

## 2. Information hierarchy

Every view is built from the same four altitudes. This is the whole layout system.

| Altitude | Content | Count per view | Spec |
|---|---|---|---|
| **1. Hero** | The one number the view exists to answer | Exactly 1 | ≥48px, system sans, proportional figures (never `tabular-nums`), display currency, with an unknown appendix if any |
| **1b. Coverage** | How much of the hero is actually known | Exactly 1, directly under the hero | 8px coverage bar + one sentence |
| **2. Stat row** | Supporting figures and their attribution | 3–4 tiles | stat-tile contract: label / value / delta / optional sparkline |
| **3. The argument** | The single chart that makes the view's case | 1 primary, ≤2 secondary | the bridge, the trajectory line, the goal meters |
| **4. The ledger** | Every contributing line, with as-of and source | 1 table, always present | the chart's table-view twin *and* the provenance surface |

Rules that follow:

- **One hero per view.** Not two. If a view seems to need two heroes, it is two views.
- **Altitude 4 is never collapsed away on desktop.** The table is the trust anchor; hiding it
  behind a toggle on a large screen signals that the numbers are for looking at rather than
  checking. On mobile it collapses to a "See all 26 lines" row.
- **Filters live in one row above altitude 1** and scope everything below. Never per-card.
  The only global filters are: period, display currency (§6), and household/mine scope.

### Shell

Five destinations. Tax is not one of them.

```
┌────────────────────────────────────────────────────────────────┐
│  Position   Trajectory   Goals   Accounts   Upkeep    [CHF ▾]  │  ← display-ccy chip, per user
├────────────────────────────────────────────────────────────────┤
│  [ 30 days ▾ ]  [ Household ▾ ]            ≈ Tax lens  ( off ) │  ← filter row
└────────────────────────────────────────────────────────────────┘
```

`Upkeep` is the ease-of-use agent's surface. Presentation supplies its input (§11.3) and its badge
count; it does not design it.

**Where the four requested views live:** Position → §7 · Trajectory → §8 · Goals → §9 · Tax → §15
(placed last because it is deferred, and because its answer is *where it lives*, not *what it
shows*).

---

## 3. Tokens

Substituted into the house method as the FinanceApp instance. Everything here was run through
`scripts/validate_palette.js`. **Do not add a colour to this table without re-running it.**

```css
.fa-viz {
  color-scheme: light;

  /* surfaces & ink */
  --surface-1:      #fcfcfb;   /* card / chart surface */
  --surface-0:      #f9f9f7;   /* page plane */
  --text-primary:   #0b0b0b;
  --text-secondary: #52514e;
  --text-muted:     #898781;
  --grid:           #e1e0d9;   /* hairline, solid, never dashed */
  --axis:           #c3c2b7;
  --hairline:       rgba(11,11,11,0.10);

  /* CAUSE — the only categorical namespace in the app (R1) */
  --cause-contribution: #2a78d6;   /* slot 1 blue  */
  --cause-market:       #eb6834;   /* slot 2 orange */
  --cause-fx:           #1baf7a;   /* slot 3 aqua   */

  /* FRESHNESS — ordinal grey, confidence reads as ink weight */
  --fresh-current: #3f3e3b;
  --fresh-due:     #6f6d67;
  --fresh-stale:   #a3a19a;
  /* unknown has no fill: surface + 1px --fresh-stale ring */

  /* GOALS — ordinal blue, one hue, three steps */
  --goal-contributed: #184f95;   /* step 600 — you put this in      */
  --goal-valuation:   #3987e5;   /* step 400 — growth + FX lent it  */
  --goal-track:       #86b6ef;   /* step 250 — still to go          */

  /* POLARITY — surplus / deficit, diverging with a neutral midpoint */
  --pos: #2a78d6;
  --neg: #e34948;
  --zero:#f0efec;

  /* STATUS — reserved, always with icon + label, never a series */
  --status-good:     #0ca30c;
  --status-warning:  #fab219;
  --status-serious:  #ec835a;
  --status-critical: #d03b3b;
}

@media (prefers-color-scheme: dark) {
  :root:where(:not([data-theme="light"])) .fa-viz { /* …dark block below… */ }
}
:root[data-theme="dark"] .fa-viz {
  color-scheme: dark;
  --surface-1: #1a1a19; --surface-0: #0d0d0d;
  --text-primary: #ffffff; --text-secondary: #c3c2b7; --text-muted: #898781;
  --grid: #2c2c2a; --axis: #383835; --hairline: rgba(255,255,255,0.10);

  --cause-contribution: #3987e5;
  --cause-market:       #d95926;
  --cause-fx:           #199e70;

  --fresh-current: #d6d4cc;   /* inverted: confidence is still "most ink vs surface" */
  --fresh-due:     #9c9a93;
  --fresh-stale:   #6a6862;

  --goal-contributed: #86b6ef;
  --goal-valuation:   #3987e5;
  --goal-track:       #184f95;

  --pos: #3987e5; --neg: #e66767; --zero: #383835;
}
```

Declare the dark values under **both** scopes exactly as the house method specifies — the media
query for the OS setting, the `data-theme` scope so the in-app toggle wins in both directions.

### Validator results (run, not reasoned)

```
$ node scripts/validate_palette.js "#2a78d6,#eb6834,#1baf7a" --mode light --pairs all
  [PASS] Lightness band      all 3 inside L 0.43–0.77
  [PASS] Chroma floor        all 3 >= 0.1
  [PASS] CVD separation      worst all-pairs #1baf7a↔#eb6834 ΔE 9.2 (deutan) · tritan 9.6
  [PASS] Normal-vision floor worst all-pairs #1baf7a↔#2a78d6 ΔE 24.0
  [WARN] Contrast vs surface below 3:1 — relief required: [["#1baf7a",2.74]]
  → ALL CHECKS PASS

$ node scripts/validate_palette.js "#3987e5,#d95926,#199e70" --mode dark --surface "#1a1a19" --pairs all
  → ALL CHECKS PASS   (worst all-pairs CVD ΔE 9.4; contrast all ≥3:1)

$ node scripts/validate_palette.js "#3f3e3b,#6f6d67,#a3a19a" --mode light --ordinal      → ALL PASS (light end 2.52:1)
$ node scripts/validate_palette.js "#6a6862,#9c9a93,#d6d4cc" --mode dark  --ordinal      → ALL PASS (near-surface end 3.13:1)
$ node scripts/validate_palette.js "#184f95,#3987e5,#86b6ef" --mode light --ordinal      → ALL PASS (light end 2.06:1)
$ node scripts/validate_palette.js "#184f95,#3987e5,#86b6ef" --mode dark  --ordinal      → ALL PASS (near-surface end 2.15:1)
$ node scripts/validate_palette.js "#2a78d6,#e34948" --mode light                        → ALL PASS (ΔE 21.6 CVD)
```

**The one obligation this creates.** Aqua — the FX cause — sits at **2.74:1** on the light
surface, below the 3:1 mark threshold. Under the house method a contrast WARN is not
dismissable; it obligates a relief channel. Therefore:

> **Every FX mark in the app carries a visible direct label.** Not a tooltip — a rendered
> number or a legend key with the value beside it. This is a hard build rule, and it happens to
> be the right editorial choice anyway: the FX term is precisely the one the reader must be
> *told*, not left to infer from a bar length.

Typeface: `system-ui, -apple-system, "Segoe UI", sans-serif` everywhere, including the hero.
Proportional figures on hero and stat-tile values; `tabular-nums` only in table columns and axis
ticks.

---

## 4. Freshness

### 4.1 The problem, stated precisely

The naive design measures staleness in days and is wrong on this household's data in both
directions. Worked from the real inventory:

| Line | Age today | Cadence | Ratio | Naive "days old" verdict | Correct verdict |
|---|---|---|---|---|---|
| Julius Baer investment statement | 21 d | 31 d | **0.68** | "three weeks old — warn" | **Current.** It is not late. |
| Swiss Pillar 2 certificate (Feb) | 190 d | 365 d | **0.52** | "six months old — alarming" | **Current.** It arrives once a year. |
| Revolut balance (Partner A) | 3 d | 1 d | **3.0** | "three days old — fine" | **Stale.** The feed has stopped. |
| CBA credit card export | 40 d | 31 d | **1.3** | "a month old — fine" | **Due.** Statement cycle missed. |

A days-based badge cries wolf on the pension certificate — the single most stable figure in the
household — while staying silent on a broken daily feed. It would be actively harmful. So:

> **Staleness ratio `r = age_days ÷ expected_cadence_days`. Bands, not a gradient.**

### 4.2 The ladder

| State | Rule | Ink | Chart mark | Number |
|---|---|---|---|---|
| **Current** | `r ≤ 1.0` | `--fresh-current` | filled, series hue | full `--text-primary`, **no decoration at all** |
| **Due** | `1.0 < r ≤ 2.0` | `--fresh-due` | filled + 2px surface ring | full ink + a 5px hollow dot after the value |
| **Stale** | `r > 2.0` | `--fresh-stale` | **2px outline, no fill**, series hue | `--text-secondary`, hollow dot filled `--fresh-stale` |
| **Unknown** | connection expired · never observed · `r > 4.0` on a feed line | *(no fill)* | **absent**, region shaded (§4.5) | **no number** — `—  Unknown` + re-link affordance |
| **Modelled** | derived, not observed (mortgage schedule, DVF property estimate) | `--text-secondary` | filled, series hue, `ƒ` glyph in legend | value prefixed `ƒ` |

Two notes that matter.

**Freshness is ordinal, not status.** It never wears `--status-*`. A February pension certificate
is not a *warning*; it is a normal fact about pensions. Status colour is reserved for things a
human must act on — an expired PSD2 link, a goal that has fallen behind, a projected negative
balance — and always ships with an icon and a word. Colouring ordinary age red is how an app
trains its users to ignore red.

**Modelled is not on the ladder.** The mortgage balance is computed daily from the contract and is
never stale; it is *derived*, which is a different kind of uncertainty and gets a different glyph.
Do not let a modelled figure sit in the Current band pretending to be an observation, and do not
let it drag the coverage bar down either — it is its own segment.

### 4.3 The coverage bar — one caveat instead of fifty badges

Directly under the hero figure of every view. Eight pixels tall, full width of the hero card,
segments proportional to **share of |value| in the displayed set**, ordered Current → Due →
Stale → Modelled → Unknown, separated by 2px gaps in the surface colour (never a stroke).

```
CHF 412,600
+ up to CHF 12,400 unknown  (2 accounts)

████████████████████████████▏▏███▏▏██▏▏░░░░░░
Confirmed for 92% of value · 1 line due · 2 unknown            [ what's missing → ]
```

```html
<figure class="fa-viz coverage" role="img"
        aria-label="Confidence: 78% current, 9% due, 5% stale, 5% modelled, 3% unknown">
  <div class="cov-track">
    <span class="seg cov-current"  style="--w:78%"></span>
    <span class="seg cov-due"      style="--w:9%"></span>
    <span class="seg cov-stale"    style="--w:5%"></span>
    <span class="seg cov-modelled" style="--w:5%"></span>
    <span class="seg cov-unknown"  style="--w:3%"></span>
  </div>
  <figcaption>Confirmed for 92% of value · 1 line due · 2 unknown</figcaption>
</figure>
```

```css
.cov-track { display:flex; gap:2px; height:8px; background:var(--surface-1); }
.seg { width:var(--w); border-radius:2px; }        /* 4px rounded ends, capped small */
.cov-current  { background:var(--fresh-current); }
.cov-due      { background:var(--fresh-due); }
.cov-stale    { background:var(--fresh-stale); }
.cov-modelled { background:var(--fresh-stale);
                box-shadow: inset 0 0 0 1px var(--surface-1); opacity:.6; }
.cov-unknown  { background:transparent;
                box-shadow: inset 0 0 0 1px var(--fresh-stale); }  /* hollow = absent */
```

Rules:
- **The Unknown segment is always direct-labelled**, at any size, even at 1%. Every other segment
  is labelled only if it is the largest.
- A segment below 1.5% is rendered at a 1.5% minimum width so it cannot vanish; the caption
  carries the true percentage.
- **When everything is Current, the bar is a single solid strip and the caption reads
  "Everything current."** It does not disappear. Its presence is what makes its absence
  meaningful, and its presence-when-perfect is what makes a healthy app look healthy.

### 4.4 What a total's as-of date actually is

A composite has no single as-of date, and picking one is where apps start lying. The contract:

- **A total never displays a single date.** It displays coverage: `Confirmed for 92% of value`.
- The **oldest contributing observation** and the **value-weighted median age** are both in the
  provenance drawer, never on the face.
- **Sorting and comparison are by ratio `r`, not by date.** The Upkeep list is ordered by how
  overdue a line is relative to its own cadence, so the broken Revolut feed sits above the
  pension certificate.

### 4.5 Freshness in time series

Three line states, one vocabulary, used identically on every chart in the app:

| Segment | Stroke | Meaning |
|---|---|---|
| **Observed** | 2px solid, round cap/join, series hue | real observations at real dates |
| **Carried forward** | 2px dotted, same hue, ≥8px end-marker at the last real observation | last known value held between statement cycles |
| **Projected** | 2px dashed + fan band | future (§8.2) |

And the rule that keeps this honest:

> **Never interpolate a straight line between two statement observations.** A monthly-cadence
> account is drawn as a **step held at the last known value**, with a marker at each real
> observation. A rising diagonal between two statement points is invented data with a slope, and
> the slope is the part people read.

**Unknown in a time series (R4 in practice).** When a series has no valid observation for a
period, its own line **stops** — it does not continue and it does not fall to zero. The *total*
line continues, drawn at the confirmed floor, with a shaded band above it spanning
`[floor, floor + last-known-of-the-unknowns]`:

```
        ╱──────╲
   ────╱        ╲────░░░░░░░░░░░░   ← band: what the total could be
                     ────────────   ← solid: what is confirmed
                     │
                     └ Revolut feed expired 14 Aug
```

Band fill: the series hue at 10% opacity, no stroke on the upper edge (a stroke reads as data).
An always-visible annotation names the cause and date at the band's leading edge; it is a link
into the re-link flow.

### 4.6 Provenance without a caveat flood (INV3)

INV3 requires every figure be *traceable*, not that every figure be *annotated*. Two surfaces
carry it, and both are always reachable:

1. **The provenance drawer.** Any figure, anywhere, is a focusable element. Click / tap /
   `Enter` opens a right-side drawer (bottom sheet on mobile):

   ```
   CHF 8,412.30
   Julius Baer · current account (CHF) · yours
   ───────────────────────────────────────────
   As of        6 Aug 2026        (21d · cadence 31d · Current)
   Source       statement PDF     JB-2026-07.pdf · sha256 4f2a…c19
   Derived      extracted, two passes agreed, auto-committed
   Native       CHF 8,412.30      (no conversion — display ccy is CHF)
   ───────────────────────────────────────────
   [ Open source document ]   [ View extraction ]
   ```

   For a converted figure the block gains:
   `Rate  EUR→CHF 0.9412  ECB reference, 26 Aug 2026  ·  rate-set v2026-08-26`

2. **The table view (altitude 4).** Every chart's accessibility twin *is* the provenance table.
   One surface, two obligations, no duplicated work:

   | Account | Native | Display (CHF) | As of | r | State | Source |
   |---|---|---|---|---|---|---|
   | JB current (CHF) | 8,412.30 | 8,412.30 | 6 Aug | 0.68 | Current | statement |
   | CA joint current | € 3,104.55 | 2,921.99 | 27 Aug | 0.00 | Current | feed |
   | Revolut (A) | — | — | 14 Aug | 13.0 | **Unknown** | feed expired |
   | Mortgage | −€ 184,220.00 | −173,388.02 | 27 Aug | ƒ | Modelled | amortisation |

   Note the Unknown row shows **no figure in either column**, and keeps its as-of so the reader
   can see how long it has been dark.

### 4.7 Offline is not a special state

The PWA will be opened on a phone with no connectivity, and it must not look broken. It doesn't
need to: **offline is the freshness ladder applied to the whole app.** The last render is served
from cache, the coverage bar reflects the ages at cache time, and the header carries one line:

```
Offline · showing the position as of 27 Aug, 08:14
```

No modal, no greyed-out screen, no spinner. When data returns, charts **hold their previous
render at reduced opacity** during refetch and swap in place — never a skeleton flash, never a
layout jump.

---

## 5. FX decomposition

INV10 is the invariant most likely to be quietly defeated by a nice-looking dashboard. A EUR/CHF
move of 2% against a CHF-heavy household with EUR liabilities is worth thousands, arrives
unannounced, and looks exactly like a good month. The design's job is to make it impossible to
read it as one.

### 5.1 The compact form is text (C6)

Three components is a handful of numbers. A chart of three bars, at stat-tile size, is less
legible than three labelled numbers with a colour key. So the everyday device — the thing that
appears under **every change figure in the app** — is an attribution block:

```
Net worth · CHF                                     30 days
────────────────────────────────────────────────────────────
CHF 412,600                                  ▲ +4,210  (+1.0%)

▌ You put in                                          +3,900
▌ Markets moved                                       +1,150
▌ Exchange rates                                        −840   ← not your doing
────────────────────────────────────────────────────────────
                              [ ▸ show the bridge ]  [ table ]
```

```html
<dl class="fa-viz attrib">
  <div><dt><i class="key k-contribution"></i>You put in</dt>      <dd>+3,900</dd></div>
  <div><dt><i class="key k-market"></i>Markets moved</dt>          <dd>+1,150</dd></div>
  <div><dt><i class="key k-fx"></i>Exchange rates</dt>             <dd class="fx">−840</dd></div>
</dl>
```

```css
.attrib .key { display:inline-block; width:3px; height:1em; margin-right:8px;
               border-radius:1px; vertical-align:-2px; }
.k-contribution { background:var(--cause-contribution); }
.k-market       { background:var(--cause-market); }
.k-fx           { background:var(--cause-fx); }
.attrib dd      { font-variant-numeric: tabular-nums; color:var(--text-primary); }
```

- **Line keys, not filled boxes**, at this density.
- **Text wears text tokens.** The number `−840` is `--text-primary`, not aqua. The 3px key beside
  it carries identity. This also satisfies the aqua contrast obligation from §3 — the FX value is
  always a rendered, high-contrast number.
- **Order is fixed: contribution, market, FX — never sorted by magnitude.** Sorting by size makes
  the block re-order month to month and destroys the reader's muscle memory. Contribution is
  first because it is the only line the household controls.
- The trailing `← not your doing` on the FX row appears whenever `|fx| > 15%` of `|net change|`.
  Editorial, deliberate, and the single most valuable sentence in the app.

**Optional 6px signed strip** for a denser layout — a sparkline-class graphic, permitted by the
stat-tile contract. Positive causes stack rightward from a zero rule, negative leftward, and a
caret marks the net:

```
          ◄ negative │ positive ►
   ░░░░░░░░░░░[fx]───┼───[contribution]███████[market]██
                     ▲                            ▲ net +4,210
```

Total ink width = gross movement; caret position = net. "Wide bar, caret near zero" reads
instantly as *a lot happened and none of it stuck*, which is exactly the month this household
will have when EUR/CHF moves.

### 5.2 The expanded form is a bridge

On expansion, and permanently on the net-worth detail and View 2, the change bridge:

```
CHF
420k ┤                                        ┌───┐
     │                    ┌───┐               │   │
415k ┤        ┌───┐       │   │   ┌───┐       │   │
     │        │   │       │   │   └───┘       │   │
410k ┤  ┌───┐ │   │       │   │     ▼         │   │
     │  │   │ └───┘       └───┘   ┌───┐       │   │
405k ┤  │   │                     │   │       │   │
     └──┴───┴───┴───┴─────┴───┴───┴───┴───────┴───┴──
       Opening   You     Markets    FX      Closing
       408,390  +3,900   +1,150     −840    412,600
```

Build spec:

| Element | Spec |
|---|---|
| Anchor bars (opening, closing) | `--fresh-stale` at 40% opacity, ≤24px wide, square both ends, seated on the axis |
| Cause bars | slot hue, ≤24px wide, **4px rounded on the leading (data) end, square on the running-total end** |
| Connectors | 1px solid `--grid` hairline between the top of one bar and the base of the next |
| Direction | **geometry only** — bars grow up or down from the running total. Hue never encodes sign. |
| Labels | every cause bar direct-labelled with its signed value; anchors labelled with the level |
| Zero rule | not drawn — the bridge has no meaningful zero; the y-axis is truncated and **the truncation is stated in the caption** |
| Gaps | 2px surface gap between any touching marks |

Truncating the y-axis on a bridge is correct (a bridge from zero would render all four causes as
invisible slivers against a 400k baseline) and it is also the classic way to mislead, so the
caption is mandatory: *"Vertical axis starts at CHF 405,000 to show the movement."*

**Sign is geometry, not hue.** This is the one place the design deliberately declines a diverging
palette. Contribution can be negative (a month you drew down), market can be negative, FX can be
either. If hue carried sign, hue could no longer carry identity, and identity is the entire point
of INV10. Waterfall geometry states direction unambiguously and every bar is direct-labelled with
its sign, so nothing is lost.

### 5.3 Per-pair drill-down: small multiples, one hue

"FX cost you 840 francs" invites the next question: *which currency?* The answer is **not** a
stacked bar in four colours (R1). It is small multiples, all in aqua, all direct-labelled:

```
   FX effect, 30 days · CHF                                  −840 total

   EUR → CHF   ██████████████████████  −612    (rate 0.9520 → 0.9412, −1.13%)
   AUD → CHF   ███████████             −298    (rate 0.5240 → 0.5090, −2.86%)
   USD → CHF   ▌                        +70    (rate 0.8010 → 0.8055, +0.56%)
```

Bars from a shared zero rule, sorted by |effect|, each labelled with the effect **and** the rate
move that caused it. This is the surface where the household learns that its mortgage is a EUR
problem and its super is an AUD one — which is a thing the feasibility assessment says out loud
and which no other view says at all.

Because all-pairs CVD only validates to three slots, and because this is one series in one hue,
this form is unconstrained by series count — a fifth currency costs nothing.

### 5.4 The rule that keeps goals honest

Deferred to §9.2, because it is the most important application of INV10 and it belongs with the
goals view. In one line: **a goal's progress ratio is computed in the goal's own currency, and
the FX contamination is drawn as the gap between two tracks.**

### 5.5 The app forecasts no exchange rate (C8)

Projections hold the last observed rate flat for the whole horizon. There is no drift term, no
forward curve, no mean reversion. Instead the uncertainty is drawn:

- **Inner band** (`--cause-contribution` at 14%): income and spending variance, from the
  household's own history.
- **Outer band** (`--cause-fx` at 7%): what the same projection would be at the 10th and 90th
  percentile of the last five years of the relevant pairs' 12-month moves.

Legend, mandatory, both bands direct-labelled:

```
    ░░░  likely range (your income and spending vary)
    ░░░  plus exchange-rate variability — this is not a forecast of rates
```

Two bands of one and two hues at low opacity, with a legend, is legible; three would not be.
A third source of uncertainty folds into the outer band.

---

## 6. Display currency (INV2)

### 6.1 The control

A chip in the top bar, **per user, persisted per user**, never a household setting. Partner A's
chip says CHF; Partner B's says EUR; neither can see or change the other's.

**Per-view override** is supported and visibly *pinned*:

```
[ CHF ▾ ]              ← the user's base, on every view
[ EUR 📌 ]             ← pinned override on this view only, with a one-tap "back to CHF"
```

A pinned override is a deliberate, marked state — never sticky-by-accident. Views where an
override is genuinely useful and should be offered as a one-tap suggestion:

| View | Suggested override | Why |
|---|---|---|
| Mortgage detail | EUR | The debt is a EUR obligation; a CHF view of it invents movement |
| Superannuation | AUD | Preserved until an Australian retirement condition; the CHF value is noise |
| Pillar 2 | CHF | Untaxed until withdrawal, a Swiss instrument |
| Cash position | user's base | This is the one number that must be in the currency you think in |

### 6.2 Rendering rules

1. **A converted figure never stands alone on a detail row.** The native amount appears beside it
   in `--text-secondary` at 0.85em: `CHF 2,921.99  · € 3,104.55`. On aggregate figures (hero,
   totals) only the display currency appears — a sum has no native currency.
2. **Every converted figure's provenance carries the rate, the rate date, and the rate-set
   version.** Two views that disagree must be diagnosable in one click.
3. **The rate convention is stated on the face, once per view, in the caption**, not buried:
   *"Converted at ECB reference rates for 26 Aug 2026."* The feasibility assessment is right that
   the convention is arbitrary but must be consistent and stated; stating it is a presentation
   job, and it is cheap.
4. **Never render a currency symbol without its code where two currencies share one.** `$` is
   ambiguous between AUD and USD in this household. Always `A$ 4,120` / `US$ 4,120`, or
   `AUD 4,120`. `€` and `CHF` are unambiguous and may stand alone.
5. **Rounding is to minor units for account lines, to whole units for heroes and axes.** Hero
   figures never show cents; a 48px `412,600.42` reads as false precision on an aggregate whose
   coverage bar says 92%.

### 6.3 The rate itself has a freshness state

ECB publishes on business days. A Monday-morning view converts at Friday's rate, which is correct
and needs no comment. So the rate joins the ladder with `cadence = 1 business day`:

| Rate age | State | Behaviour |
|---|---|---|
| ≤ 1 business day | Current | silent |
| 2–3 business days | Due | rate date shown in the view caption, no other change |
| > 3 business days | Stale | a single view-level banner: *"Exchange rates last updated 19 Aug. Converted figures may be off."* All converted figures drop to `--text-secondary`. |
| no rate for a required pair | Unknown | affected lines render in **native currency only** and are excluded from the display-currency total, which grows an unknown appendix (R4) |

The last row is the important one: a missing rate does not produce a wrong conversion, and it does
not blank the app. It falls back to the truth — the native amount — and tells the total it is
incomplete.

---

## 7. View 1 — Position

**The question:** *how much can we actually spend, right now, without breaking anything?*

This is the household's first priority and it is the view that must be right when everything else
is down. It is the landing view.

### 7.1 Layout

```
┌──────────────────────────────────────────────────────────────────────┐
│  Available now                                                        │
│  CHF 21,480                                          ▲ +1,240 (7d)   │  ← hero, ≥48px
│  + up to CHF 2,100 unknown (1 account)                                │  ← R4 appendix
│  ████████████████████████████████▏▏████▏▏░░                            │  ← coverage bar
│  Confirmed for 91% of value · 1 unknown            [ what's missing ] │
├──────────────────────────────────────────────────────────────────────┤
│  Instant (T0)      Settles ≤2d      Cards pending      Bills ≤30d     │  ← stat row
│  CHF 14,900        CHF 8,680        −CHF 1,540         −CHF 560       │
│  ▁▂▃▃▄▅▆           ▁▁▂▂▃▃▃           ▃▅▂▁▄▆▇             ▂▂▂▄▄▄▄       │
├──────────────────────────────────────────────────────────────────────┤
│  How that number is made                                             │
│  ──────────────────────────  the liquidity bridge  ──────────────────│
├──────────────────────────────────────────────────────────────────────┤
│  Next 30 days                                                        │
│  ──────────────────────────  the cash calendar  ─────────────────────│
├──────────────────────────────────────────────────────────────────────┤
│  Contributing lines (14)   ▸ table view / provenance                 │
└──────────────────────────────────────────────────────────────────────┘
```

### 7.2 The liquidity bridge

```
CHF
28k ┤ ┌────┐
    │ │    │  ┌────┐
24k ┤ │    │  │    │
    │ │    │  │    │              ┌────┐
20k ┤ │    │  │    │   ┌────┐     │    │
    │ │    │  │    │   │    │     │    │
16k ┤ │    │  │    │   │    │     │    │
    └─┴────┴──┴────┴───┴────┴─────┴────┴───────────────
       Cash    + Money    − Cards   − Bills    = Available
       in       market      pending   due ≤30d
       accounts +8,680     −1,540    −560       21,480
       14,900
```

- Same bridge grammar as §5.2: anchors grey, deductions grow downward from the running total,
  every bar direct-labelled, connectors as `--grid` hairlines, 2px surface gaps.
- **Deductions are not red.** They are the same neutral treatment as additions, distinguished by
  geometry. A pending card balance is not a problem; it is a fact. Red is reserved for the
  projected-negative case in §7.3.
- Tapping any segment expands it into its contributing lines *in place* (the bar splits into
  stacked sub-bars with a 2px gap, direct-labelled), rather than navigating away.

### 7.3 Liquidity tiers — an ordinal ramp, and one hard exclusion

"Money-market-type holdings in accessible investment accounts" is the requirement's subtlest
phrase and it needs an explicit classification, supplied per account line by financial
correctness (§11.1):

| Tier | Meaning | Examples in this household | Ramp step |
|---|---|---|---|
| **T0** | Spendable today | CA joint current, Revolut ×2, Wise, JB current ×5, CBA transaction, Livret A ×3 | `--goal-contributed` (darkest = most available) |
| **T2** | Settles within 2 business days | money-market and cash-equivalent positions inside JB investment; liquid ETF positions in Bourse Direct PEA | `--goal-valuation` |
| **T30** | Reachable within 30 days with notice or redemption | assurance vie partial surrender; longer-settlement positions | `--goal-track` |

One hue, three ordered steps, **darker = more available now**. Validated as an ordinal ramp (§3).

> **Locked assets do not appear in View 1 at all.** Not greyed, not in a collapsed section, not
> as a footnote total — absent. Swiss Pillar 2, Hostplus superannuation, and the house are not
> liquidity, and a "total including illiquid" line on the cash view is an invitation to
> mis-plan. They are on the net-worth and goals surfaces where they belong. The mortgage appears
> only as its next scheduled payment inside "Bills ≤30d"; the principal is not a liquidity event.

The PEA carries a five-year rule (D4) that makes withdrawal costly rather than impossible. It is
**T30 with a cost annotation**, never T0: `PEA · T30 · withdrawal before Mar 2028 closes the
wrapper`. Presentation shows the annotation; the tax lens quantifies it later.

### 7.4 The cash calendar

The actionable half of the view: a 30-day column chart of projected available balance, one series,
no legend (the title names it).

```
CHF
25k ┤ ▏▏▏▏▏▏▏
    │ ███████▏▏
20k ┤ ██████████▏▏▏▏
    │ ███████████████▏▏
15k ┤ ██████████████████▏▏▏▏▏▏▏▏
    │ ███████████████████████████
10k ┤ ███████████████████████████
    └─┬────────┬─────────┬────────┬──
      28 Aug   4 Sep     11 Sep   18 Sep
                          ▲ low point CHF 11,240 · 12 Sep
                            (TopCard ×2 + CBA card fall in one week)
```

- One series, slot-1 blue; **the trough column is the only direct-labelled one**, with a
  one-sentence explanation of what causes it. Everything else is carried by the axis and hover.
- Columns projected below zero switch to `--neg` and the view gains a status row:
  `⚠ Projected short by CHF 1,180 on 12 Sep` — icon + label + colour, never colour alone.
- Known-dated obligations (card due dates, mortgage direct debit, standing orders) are drawn as
  1px `--axis` tick marks on the x-axis with a small glyph, so the reader sees *why* the shape
  is the shape.
- Columns beyond the last confirmed obligation are drawn with the projected treatment (§4.5).

Because these are pending card liabilities in AUD and CHF against a CHF or EUR display currency,
the calendar caption states the rate convention (§6.2.3) and the FX band from §5.5 applies to the
tail of the horizon.

---

## 8. View 2 — Trajectory

**The question:** *are we going to be all right, and is it because of us?*

### 8.1 Layout

```
┌──────────────────────────────────────────────────────────────────────┐
│  Projected in 12 months                                              │
│  CHF 468,000                                    ▲ +55,400 (13.4%)    │  ← hero
│  ████████████████████████████▏▏███▏▏░░                                 │
│  Built from 92% confirmed value · rates held flat at 26 Aug           │
├──────────────────────────────────────────────────────────────────────┤
│  ▌ You will put in   +48,600    ▌ Markets  +7,640    ▌ FX  0 (held)  │  ← attribution block
├──────────────────────────────────────────────────────────────────────┤
│                       net position, 12 back / 12 forward             │
├──────────────────────────────────────────────────────────────────────┤
│  Monthly surplus / deficit          │  CHF earned, EUR spent          │
│  (diverging columns)                │  (the frontalier corridor)      │
├──────────────────────────────────────────────────────────────────────┤
│  Assumptions (7)  ▸                 │  Contributing lines  ▸          │
└──────────────────────────────────────────────────────────────────────┘
```

Note the hero's attribution block reads `FX 0 (held)` — the projection's honesty made structural.
The reader is told, at the top of the forecast, that no rate movement is being predicted.

### 8.2 The trajectory line

One series (net position), observed and projected:

- **Observed:** 2px solid, slot-1 blue, ≥8px markers at real observations with 2px surface rings.
- **Projected:** 2px dashed, same hue, plus the two fan bands from §5.5.
- **Today** is a 1px solid `--axis` vertical rule, labelled `today` at the top — not dashed
  (dashing is reserved for projection).
- Direct labels: the last observed value, the 12-month projected value, and nothing else.
- Under `Household` scope the caption is the constant partiality line (§10).

### 8.3 Monthly surplus / deficit

The genuine diverging case: above/below a zero baseline.

- Columns above zero `--pos`, below zero `--neg`, zero rule in `--zero` at 1px.
- ≤24px columns, 4px rounded on the data end, square at the baseline, 2px gaps.
- **A 3-month trailing mean as a 2px `--text-muted` line over the columns** — the columns are
  noisy (annual insurance, quarterly super) and the mean is the thing to read.
- Direct-label the worst deficit month and the current month; the axis carries the rest.
- Months containing an unknown component are drawn as an **outlined column** (§4.2) rather than
  filled, so a deficit that might be an artefact of a dead feed is visibly distinguishable from
  one that happened.

### 8.4 The frontalier corridor

This household has a structural, recurring FX exposure that no generic app would show: Partner A
earns CHF in Geneva and the household spends EUR in Savoie (D4). Every month, a fixed-ish CHF
amount becomes a variable EUR amount. That is a first-class fact about their finances, and it
deserves its own small chart.

```
   CHF earned → EUR spent                             last 12 months

   EUR value of a constant CHF 7,000 salary
   6,700 ┤     ╭─╮
         │  ╭──╯ ╰──╮      ╭───╮
   6,550 ┤──╯       ╰──────╯   ╰───╮
         │                         ╰───
   6,400 ┤
         └──┬───┬───┬───┬───┬───┬───┬──
           Sep         Jan         May

   Range over the year: EUR 6,412 – 6,704 · a spread of EUR 292/month
```

One series, aqua (the FX cause hue — consistent with R1), direct-labelled at both extremes and at
the end, with the spread stated in words. It answers "how much does the rate actually cost us"
in a unit the household transacts in.

### 8.5 Assumptions are a first-class, visible object

A projection is only as trustworthy as its inputs, so the assumption set is not buried in
settings. It is a labelled row under the chart, showing count and last-edited, expanding to a
plain table:

| Assumption | Value | Basis | Last changed |
|---|---|---|---|
| Salary A (CHF, net) | 7,000 /mo | payslip, 12-mo mean | 2 Jun |
| Salary B (EUR, net) | 2,850 /mo | payslip, 12-mo mean | 2 Jun |
| Household spend | 4,900 /mo | ledger, 6-mo mean | auto |
| Market growth | 0% real | **deliberately none** | — |
| FX | held flat | **no forecast** (C8) | — |
| Mortgage | contract schedule | amortisation | 14 Jan |
| Livret A rate | 2.4% | published | 1 Feb |

Setting market growth to zero by default is a design position, not an oversight: a household
projection that assumes 6% equity growth is mostly displaying its own assumption. Growth is
configurable; the default is honest. The row is labelled **"deliberately none"** so it reads as
a choice rather than a gap.

---

## 9. View 3 — Goals

**The question:** *are we getting where we said we'd get, and is the progress real?*

### 9.1 Layout

```
┌──────────────────────────────────────────────────────────────────────┐
│  Toward all goals                                                    │
│  47%                                              4 of 6 on track    │  ← hero (a %, not a sum)
│  ██████████████████████▏▏███████░░░░░░░░░░░░░░░░░░░░                   │
│  Shared household view — jointly-visible accounts only               │  ← constant (C11)
├──────────────────────────────────────────────────────────────────────┤
│  goal meters, sorted by months behind                                │
├──────────────────────────────────────────────────────────────────────┤
│  Required vs actual monthly contribution   (dumbbell)                │
└──────────────────────────────────────────────────────────────────────┘
```

The aggregate hero is a **percentage, not a currency sum**, because summing goals denominated in
EUR, CHF and AUD into one display currency reintroduces exactly the FX contamination the goal
design exists to remove. Each goal's own ratio is FX-clean; the mean of clean ratios, weighted by
target, is clean. A converted currency sum would not be.

### 9.2 The goal meter — and the FX contamination made visible (C7)

This is the most important single control in the document, because a savings-goal bar is the
place where an FX swing most convincingly impersonates virtue.

```
House deposit                                    ✓ on track      EUR
target EUR 60,000                                                ≈ CHF 56,470
┌────────────────────────────────────────────────────────────────────┐
│████████████████████████████▏▏███░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░│
└────────────────────────────────────────────────────────────────────┘
 ╎                          ╎    ╎
 ╎  you contributed 32,800  ╎    ╎ growth + FX  1,400
 ╎          (54.7%)         ╎    ╎    (2.3%)
 └───────────── EUR 34,200 · 57.0% ───────────┘

 ▏▏▏▏▏▏▏▏▏▏▏▏▏▏▏▏▏▏▏▏▏▏▏▏▏▏▏▏▏▏▏▏▏▏▏▏▏▏  55.4% at the rate on 4 Mar 2025
 └ the gap is what exchange rates lent you: EUR 960

 Needs EUR 1,290/mo to finish by Mar 2028 · you're putting in EUR 1,400/mo
```

Three moves, each load-bearing:

1. **The fill is split.** `--goal-contributed` for money the household actually moved,
   `--goal-valuation` for everything valuation did on top, `--goal-track` for what's left. One
   hue, three ordinal steps, 2px surface gaps, validated in §3. A reader who glances at the bar
   sees immediately how much of their progress they earned.
2. **The goal currency governs.** The goal is denominated in EUR; the ratio 57.0% is computed in
   EUR; the display currency appears only as the small `≈ CHF 56,470` annotation. Partner A
   viewing in CHF and Partner B viewing in EUR see **the same percentage**, which is essential —
   two people looking at the same goal must not disagree about whether it is on track.
3. **The ghost track.** A 3px secondary track below the main one shows progress recomputed at the
   exchange rates in force on the day the goal was created. The gap between the two tracks is the
   FX contribution, drawn to scale, and captioned in words with its value. When the two tracks
   coincide (a EUR goal funded from EUR accounts), the ghost track is suppressed and the caption
   reads *"no currency exposure — funded in EUR."*

The ghost track is the answer to "make the decomposition a first-class visual, not a footnote."
It puts INV10 inside the progress bar itself, which is the one graphic a household actually looks
at every week.

**Status.** `on track` / `behind` / `at risk` wears `--status-good` / `--status-warning` /
`--status-critical`, always with a glyph and the word, never colour alone. This is a genuine
status — it demands action — and is the correct place for status tokens, unlike age (§4.2).

**Sorting: by months behind, descending.** Not by size, not by percentage. The goal that needs
attention is at the top.

### 9.3 Required vs actual — a dumbbell

One row per goal, two markers on a shared axis in the display currency:

```
                          EUR/month
  House deposit    ●───────○      1,290 required → 1,400 actual   ✓
  Son's fund       ○───●          200 required   → 120 actual     ⚠ 80 short
  Emergency (6mo)  ●○             410 required   → 420 actual     ✓
  Car replacement  ○──────────●   150 required   → 480 actual     ✓ ahead
```

Filled marker = actual, hollow = required, 2px `--text-muted` connector, both ≥8px with 2px
surface rings. One hue, two treatments — the dumbbell form's intended use. Direct-labelled
shortfall on any row that is short.

### 9.4 Per-account, as well as aggregate

The requirement asks for goal progress "at individual account level and on aggregate." A goal may
be funded from several accounts (a EUR house deposit fed from Livret A and the CA joint account).
So each meter expands into its funding accounts, each with its own contributed/valuation split,
its own freshness state, and its own native currency:

```
House deposit · EUR 34,200 of 60,000
├─ Livret A (hers)      € 22,600  ██████▏▏▏   contributed 22,300 · interest 300   6 Aug · Current
├─ CA joint current     €  8,400  ███▏         contributed  8,400 · —              27 Aug · Current
└─ Revolut (A) EUR      €  3,200  ▢▢▢          — Unknown, feed expired 14 Aug     [ re-link ]
                                               ↑ goal total shows a range while this is dark
```

The Unknown funding account propagates R4 upward: the goal's percentage becomes
`57.0% – 62.3%` and the meter grows a hollow extension segment. A goal is not permitted to
silently exclude a dark account, because that would make progress look *worse*, and a household
that fixes its feeds to make a number go up has learned the wrong lesson from the app.

---

## 10. Privacy-aware presentation (D3)

Presentation does not own the privacy model; security-by-design does. But presentation owns the
place where it leaks, so three rules bind the view layer.

**The partiality caption is constant (C11).** Every shared/household-scoped aggregate carries the
same caption at all times:

> *Shared household view — jointly-visible accounts only.*

It appears when there are zero private accounts and when there are five. A caption that appears
*because* something is hidden tells the other partner that something is hidden, and its
appearance and disappearance over time tells them when a private account was opened or closed.
The caption is chrome, not a conditional state.

**Never render a placeholder where a private account would be.** No greyed row, no "1 account not
shown", no count. The shared view simply enumerates the jointly-visible lines. Counts are an
inference channel.

**Scope is a filter, not a merge.** The `Household ▾` control offers `Household (shared)` /
`Mine` / a specific person's own view (for that person only). Switching to `Mine` re-renders
every altitude — hero, coverage bar, charts, table — against the personal set, and the caption
changes to *"Your complete position, including private accounts."* Presentation must never
compose a view that mixes a private figure into a shared total, including transiently during
loading, because a 200ms flash of the true total is a full disclosure.

**The son's Livret A** (a third data subject, feasibility §"the hard part") renders with a
distinct treatment on the Accounts view: its own section, a `held under power of attorney` label,
and — because he will one day be an adult with a view on this — it is **excluded from the
household net-worth hero by default**, appearing as a separate, clearly-labelled figure. A
minor's assets are not household net worth. This is a presentation call with a data-retention
consequence; data security owns the retention side.

---

## 11. Interfaces this design needs

Stated as contracts rather than designed on the other agents' behalf.

### 11.1 From financial correctness

```ts
// One call answers every attribution block, bridge and per-pair drill-down in the app.
decompose(input: {
  accounts: AccountId[];
  from: Date; to: Date;
  displayCcy: Ccy;              // caller's, per INV2 — never a server default
}): {
  opening: Minor; closing: Minor;              // in displayCcy, at the stated convention
  contribution: Minor;                         // cash the household moved
  market: Minor;                               // instrument reprice, in native ccy, then converted
  fx: Minor;                                   // pure rate effect; residual is folded here explicitly
  byPair: Array<{ from: Ccy; to: Ccy; effect: Minor;
                  rateOpen: Decimal; rateClose: Decimal }>;
  rateSet: { date: Date; source: 'ECB'; version: string };
  unknown: Array<{ account: AccountId; lastKnown: Minor | null; since: Date }>;
}
```

Requirements presentation depends on:
- **`contribution + market + fx` must equal `closing − opening` exactly.** Any residual is folded
  into `fx` and that folding is stated, not silently distributed. A bridge whose bars do not reach
  the closing anchor is a visible defect, and it should be — presentation will render the gap
  rather than hide it.
- **Exactly three cause terms, forever.** A fourth breaks R1 and the validated palette.
- **`unknown` is returned as a list, never as zero.** R4 depends on it.
- Also needed: `liquidityTier(account) → 'T0'|'T2'|'T30'|'locked'` with a cost annotation string
  for wrapper penalties (PEA five-year rule); `goalProgress(goal) → { contributed, valuation,
  target, ratioAtCreationRates }` — the last field is what draws the ghost track (§9.2); and
  `projection(horizon) → { central[], innerBand[], outerBand[] }` with FX held flat.

### 11.2 From architecture

Every figure the view layer receives carries an observation envelope:

```ts
type Observation = {
  value: Minor | null;                  // null ⇒ Unknown. Never 0-as-unknown.
  ccy: Ccy;
  asOf: Date;
  expectedCadenceDays: number;          // drives r = age / cadence — the whole of §4
  source: 'feed' | 'document' | 'computed' | 'manual';
  derivation: string;                   // human-readable, for the provenance drawer
  connection?: { state: 'live' | 'expiring' | 'expired'; expiresAt?: Date };
  docRef?: { filename: string; sha256: string; page?: number };
};
```

`expectedCadenceDays` is the single field this entire design leans on. It is per account line,
not per institution, and it must be stored as data (defaults in §4.1) rather than inferred.
`connection.state === 'expired'` forces `value: null` at the source — presentation must never be
in a position to choose between a stale number and unknown.

Also needed: a rate-set with `{ date, version, pairs }` so §6.3 can band the rate's own age; and
an offline snapshot envelope carrying `renderedAt` and `rateSetVersion` (see §12).

### 11.3 From ease of use

The boundary: **presentation detects and displays; ease-of-use resolves.**

Presentation emits, and does not design, this list:

```ts
type UpkeepItem = {
  account: AccountId;
  state: 'due' | 'stale' | 'unknown';
  r: number;                            // sort key — overdue relative to own cadence
  owner: UserId;                        // reminders go to the person who can fix it
  action: 'relink' | 'upload' | 'enter' | 'confirm';
  valueAtRisk: Minor;                   // display-ccy share of the total this line represents
};
```

Presentation owns: the coverage bar, the ladder tokens, the `[ what's missing → ]` affordance and
its badge count. Ease-of-use owns: the Upkeep screen itself, reminder scheduling and channel, the
re-link flow, and the review-and-confirm ingestion UI.

**One shared dependency:** the review-and-confirm screen must use the same freshness ladder tokens
and the same provenance-drawer layout as the four views, or the app will look like two apps. The
tokens in §3 are the contract; the drawer markup in §4.6 is the reference implementation.

### 11.4 From security by design

- `visibility: 'shared' | 'private'` per account line, resolved server-side per requesting user.
  Presentation must receive a set it is *allowed* to render; it must never filter client-side.
- `partial: true` as a **constant** on every household-scoped response (C11), not a computed flag.
- Loading states must not transiently contain out-of-scope figures (§10).

---

## 12. Where this design presses on an invariant

One place, stated openly rather than worked around.

**INV1 — "a converted value is never persisted" — versus the offline cache.**

INV5 requires the app be fully usable with zero live feeds, and the delivery target is an
installable PWA on Android that will be opened on trains, in Savoie, and on aeroplanes. Usable
offline means the last render is available offline. The last render contains converted values.
Storing it therefore persists converted values, which INV1 forbids.

The workaround of storing only native amounts plus a rate table and converting on device is
tempting and is *mostly* what should happen — but it does not fully escape the problem, because
the cached rate table is itself a persisted input to a conversion, and a projection's fan bands
are expensive enough that recomputing them on a cold phone start is a visible cost.

**What I propose, and why I think it is a narrow enough exception to accept:**

1. The offline store holds **native amounts and a rate set**, and converts on device. This is the
   default path and it does not violate INV1 in substance.
2. It additionally holds a **display snapshot** — the rendered figures for the last-viewed state —
   solely so the app paints instantly on cold start.
3. The snapshot is marked, at the schema level, as a **display artefact**: it carries
   `renderedAt` and `rateSetVersion`, it is **never readable by the ledger, the projection engine
   or any arithmetic**, it is purged on reconnect and on rate-set change, and it is invalidated
   after 7 days.
4. The header states its age whenever the snapshot is what is on screen (§4.7).

The distinction I am asking for is between *persisting a converted value as a fact* — which
corrupts history and is what INV1 exists to prevent — and *caching a rendering*, which is a
screenshot with a timestamp. If financial correctness or architecture judges that the distinction
will not survive contact with a codebase, the fallback is (1) alone plus a visible 200–400ms cold
start, and I would accept that trade rather than blur INV1. **Flagging it rather than shipping it
quietly is the point.**

Two smaller notes, neither a break:

- **INV3** is satisfied by traceability (drawer + provenance table), not by annotation. If the
  intended reading of INV3 is that provenance must be *visible on the face of every figure*, this
  design does not do that, and I would argue against that reading: an app where every number
  wears its source becomes an app where nobody reads any of them.
- **INV2** is honoured for all monetary displays. Goal *ratios* are deliberately computed in the
  goal's currency (C7/§9.2) and are therefore identical for both partners. That is not a display
  currency at all — it is a unit of account for a target — but it is close enough to the boundary
  to be worth naming.

---

## 13. Zero-feed, empty and error states (INV5)

The app must look complete and trustworthy with every live feed down. Four of twenty-six lines
have an API; the manual path is the primary experience.

**No connected chrome.** There is no plug icon, no "Connect your bank" call to action, no
institution logo grid in the four views. `source` is a field in the provenance drawer and a
column in the table. A document-sourced Julius Baer balance and a feed-sourced Revolut balance
render **identically**, because they are equally true.

**Empty states are affirmative, not blank:**

| Situation | What is shown |
|---|---|
| All feeds down, documents current | Nothing changes. The coverage bar is solid, the caption reads *"Everything current."* No banner. |
| A feed expired | That line is Unknown (§4.2); the coverage bar grows a hollow segment; the hero grows an appendix; `[ what's missing → ]` gains a badge. Nothing else changes. |
| No goals defined yet | A single card: *"No savings objectives yet. Set one and this view will track how much of the progress is yours."* — not an illustration, not a blank screen. |
| Not enough history to project | The trajectory chart renders the observed portion at full width with a caption: *"Projection needs three months of history. Two so far."* The card keeps its size; nothing collapses. |
| First run, nothing entered | The four views are present and empty-but-composed; the coverage bar reads *"Nothing recorded yet."* |

**Never a skeleton flash on refetch.** Charts hold their previous render at reduced opacity
(`opacity: .55`, no transform, no reflow) and swap in place.

**Reduced motion.** All chart entrance animation is opt-out via `prefers-reduced-motion`, and the
only animation permitted anywhere is a ≤200ms opacity fade. Bars do not grow, lines do not draw
themselves; on a financial figure, animation reads as the number still deciding what it is.

---

## 14. Accessibility

- **Every chart has a table-view twin**, which is the same table as altitude 4 and the same table
  that carries provenance (C9). Toggle sits in the chart's `<figure>` chrome.
- **Identity is never colour-alone.** ≥2 series ⇒ a legend is always present; the three causes are
  additionally direct-labelled everywhere; status always ships icon + word.
- **The aqua contrast obligation from §3 is a hard build rule** — every FX mark carries a visible
  rendered value.
- **Hit targets ≥24px**, including the 2px surface gap. Keyboard focus shows exactly what hover
  shows; the provenance drawer opens on `Enter` from any figure.
- **Untrusted labels.** Account names, goal names and extracted document fields originate from
  bank statements and user input. Insert them into tooltip, legend and table DOM with
  `textContent` / `createTextNode` — never `innerHTML` concatenation. This is a real exposure
  here: statement-derived strings pass through an extraction model before reaching the DOM.
- **Texture** is available for full CVD, print and `forced-colors` — one hand-drawn Lines fill at
  45° and its 135° mirror, ordered on value scales, off by default and never decorative.
  It is **not** used to mean "estimated" or "stale"; those use ink weight and fill/outline.
- **`forced-colors`**: fills fall back to `ButtonFace`/`Canvas` with `CanvasText` strokes; the
  fill-vs-outline freshness distinction survives, which is why that distinction is a geometry
  choice and not a colour one.
- Coverage bar carries an `aria-label` enumerating each band's percentage (§4.3).

---

## 15. View 4 — Tax, and where it lives

Deferred to a later phase (D-brief §1.4), but its home is designed now so the shell does not need
re-cutting later.

**It is a lens, not a tab (C12).** A toggle in the filter row re-renders the existing views with
an estimation layer: net worth gains a post-tax figure, the trajectory gains an after-tax line,
goals gain a "net of estimated tax" ghost. Making it a fifth destination would make the estimates
look like facts sitting alongside the ledger; making it a lens keeps them visibly *on top of*
the ledger.

**The estimation frame is persistent and non-dismissable while the lens is on:**

```
┌────────────────────────────────────────────────────────────────┐
│ ≈  Estimation lens on — figures below are modelled, not advice │  ← --status-warning, icon+label
└────────────────────────────────────────────────────────────────┘
```

Every estimated figure renders as `≈ CHF 4,120` in `--text-secondary` with a 1px dotted underline
carrying the assumption set on hover. Typography and a prefix, deliberately **not** texture and
**not** a colour — texture is reserved for accessibility, and a colour for "estimated" would
collide with the cause namespace.

**What ships first under the tax banner is not a number.** It is the foreign-account declaration
checklist — the feasibility assessment's observation that a French tax resident holding Swiss,
Australian and Belgian accounts plus foreign life insurance must declare them annually, and that
the app already holds exactly that list. It is a table, it needs no modelling, it is the highest
value-per-unit-risk feature in the tax area, and it can ship in phase 4 rather than phase 6:

| Institution | Country | Account | Opened | Closed | Declared 2025 | Declared 2026 |
|---|---|---|---|---|---|---|
| Bank Julius Baer | CH | current ×5, investment | 2019 | — | ✓ | ☐ |
| Commonwealth Bank | AU | transaction, cards ×2 | 2016 | — | ✓ | ☐ |
| Wise | BE | multi-currency | 2021 | — | ✓ | ☐ |
| Revolut Bank UAB | LT | multi-currency | 2022 | — | ✓ | ☐ |
| Linxea | FR | assurance vie | 2020 | — | n/a | n/a |

With a standing caption: *"A list of accounts, not a filing. Confirm your obligations with an
accountant."* Never presented as advice (D4).

**A liability estimate does not get a hero figure.** The tax lens is the one surface where the
information hierarchy is deliberately broken: no ≥48px number, ever. The largest an estimated tax
figure may render is stat-tile size. A wrong number presented confidently is worse than none,
and type size is how confidence is presented.

---

## 16. Anti-pattern self-check

Run against the house catalogue before any of this is built.

| Anti-pattern | Status here |
|---|---|
| Dual-axis chart | **None.** The frontalier corridor was the temptation (CHF salary vs EUR value) and is instead drawn as one series: the EUR value of a constant CHF amount, on one axis. |
| Recolour-on-filter | Impossible by construction — hue is bound to cause, and there are exactly three causes that never change. |
| Cycling hues past 8 | Only three categorical slots are ever used. Currency, the thing that would have demanded more, is text (R1). |
| Eyeballed CVD safety | Every ramp validated; output pasted in §3. |
| Value ramp on nominal categories | Avoided: currency mix is a list of same-hue bars. Ordinal ramps are used only where order is real (liquidity tiers, freshness, goal fill). |
| Rainbow / hue at diverging midpoint | Surplus/deficit uses blue↔red with a neutral `--zero` midpoint. |
| Status colour as a series | Status is reserved for on-track/behind, expired connections, projected shortfall. Age is ordinal grey, never status. |
| Eight hues when the story is one number | The attribution block is text (C6); the cash calendar is one series with one direct label. |
| One-bar bar chart / 2-slice pie | None. Heroes and stat tiles carry single numbers. |
| Donut for close values | No donut anywhere. |
| Thick blocks, heavy grid | ≤24px marks, hairline solid grid, 2px surface gaps throughout. |
| Dashed gridlines | Grid and the `today` rule are **solid** hairlines. Dashing is reserved for projection, which is its legitimate use. |
| A number on every point | Direct labels are selective: trough, extremes, endpoints, every cause bar, and every FX mark (the §3 obligation). |
| Border around marks | Never — 2px surface gaps and 2px surface rings only. |
| Clipped in-mark label | Goal-meter segment labels are placed **below** the bar with leader ticks, never inside, because a 2.3% valuation segment cannot hold text. |
| Fixed height excluding the axis band | Chart containers size to plot + axis band; cards grow with content. |
| Serif/display hero, `tabular-nums` on hero | System sans everywhere; proportional figures on heroes and tiles; `tabular-nums` only in table columns and axis ticks. |
| Tooltip as the only route to a value | Every value is in the table view; the table view is always present at altitude 4. |
| Pinpoint hover targets | ≥24px hit areas; crosshair snaps to the nearest date on line and column charts. |
| Per-chart filters | One filter row above everything (§2). |
| Skeleton flash on refetch | Previous render held at reduced opacity (§13). |
| Colour-only encoding | Legend always present ≥2 series; causes direct-labelled; status icon + word; table twin everywhere. |

---

## 17. Build order for this layer

Aligned to the feasibility assessment's phases; nothing here needs a live feed.

1. **Tokens, the freshness ladder, the coverage bar, the provenance drawer.** Ship with the manual
   ledger (phase 2). Everything else depends on these four.
2. **The stat tile, the attribution block, the table-view twin.** Still phase 2. At this point the
   app can tell the truth about a hand-entered number, which is the whole trust foundation.
3. **View 1 — Position**, with the liquidity bridge and cash calendar (phase 4).
4. **View 3 — Goals**, with the split meter and ghost track (phase 4). Before View 2, because
   goals are where FX contamination does the most damage and the fix is self-contained.
5. **View 2 — Trajectory**, with the change bridge, the fan bands and the frontalier corridor
   (phase 4/5). It needs the most history.
6. **The declaration checklist** (phase 4, cheap) — then the tax lens proper (phase 6).

The freshness engine (phase 5 in the feasibility build order) does not gate any of this: the
ladder works identically on a hand-entered figure with a cadence of 31 days as on a feed with a
cadence of 1.
