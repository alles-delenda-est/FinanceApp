# Ease of Use — Onboarding, Ingestion & Review, Staleness & Reminders

**Agent:** Ease of use
**Status:** Design proposal. Binds on the scope in `02-decisions-and-constraints.md` §6.
**Date:** 27 August 2026
**Reads:** `01-feasibility-assessment.md`, `02-decisions-and-constraints.md` — in full, as settled input.

---

## 0. The problem this document is actually solving

The feasibility assessment's most important sentence is not about APIs. It is this: *designing for full
automation stalls in month three*. Four of twenty-six account lines refresh themselves. The other
twenty-two are somebody walking to a portal with a password manager open.

So the real risk is not extraction accuracy or FX arithmetic. It is **month four**: the month where
Partner A is travelling, Partner B never agreed to be an operator, the TopCard portal has redesigned
its download button, three PSD2 consents expired silently in the same fortnight, and nobody notices
that the net worth figure on the home screen has been quietly wrong for six weeks. The app does not
fail loudly at that point. It degrades into a confident lie, and confident lies are how financial
tools lose households permanently. You do not get a second attempt at trust.

Every decision below is chosen against that failure, not against a feature list. Three consequences
run through the whole document:

1. **The ledger must be allowed to get smaller rather than get wrong.** An app that helps you track
   twelve lines truthfully beats one that nags you about twenty-six and holds nine stale ones.
2. **The consequence must replace the nag.** After a small number of unheeded asks, the app stops
   asking and starts visibly showing the gap. Silence plus a hole in the picture is more motivating,
   and far less corrosive, than a fourth notification.
3. **The chore must pay out.** Ten documents is tolerable if, at the end of the ten, something
   arrives that you actually wanted and could not have had otherwise. If the payoff is diffuse, the
   habit dies.

---

## 1. Design principles

Seven, in priority order. Where they conflict, the earlier wins.

**P1 — Honest before helpful.** No number is ever shown as more certain than it is. "Unknown" is a
legitimate, well-designed, non-embarrassing state that appears constantly and is never treated as a
failure of the user.

**P2 — One visit, one chore.** Load is perceived per *trip to a source system*, not per account line.
Twenty-two manual lines collapse to roughly eight recurring visits. Model, schedule, remind and
measure at the visit level. A single Julius Baer download that yields six figures must appear as one
item, never six.

**P3 — Ask for the least, at the last moment that is still useful.** Never ask for a statement before
it exists. Never ask for a re-link before it is nearly expired. Never ask a question the app could
have inferred from a feed it already has.

**P4 — Every ask is renegotiable in the ask itself.** Every reminder carries "make this quarterly",
"hand this to my partner", "stop tracking this". A demand you can amend on the spot is not a nag.

**P5 — Two people, two relationships to this app.** Partner A is the operator and wants the whole
picture. Partner B has not signed up to be a bookkeeper. Her viable participation floor is roughly
six interactions a year, and the design must make that a respectable outcome rather than a
delinquency. No comparative statistics between partners, ever — no per-person streaks, no "Partner B
is 3 tasks behind". That is a domestic-friction generator dressed as gamification.

**P6 — The fix is one tap from the problem.** Tapping an Unknown balance opens the specific action for
that account — re-link, upload, or a number pad — never a settings screen, never a generic upload
page.

**P7 — Reversible everywhere, destructive nowhere.** Any committed figure can be superseded; any
answer to a review question can be revisited; "I don't know" is a first-class answer on every screen
that asks something. People tap carelessly when there is no exit, and careless taps are how bad data
gets in.

---

## 2. What the household is actually being asked to do

### 2.1 The visit model

The unit of work — and therefore the unit of scheduling, reminding, and measurement — is an
**Obligation**: one visit to one source system, in one window, by one named person, producing one or
more figures.

```
Obligation {
  id
  institution                 # "Julius Baer", "TopCard — Visa CHF (A)"
  produces: [account_line…]   # 1..n figures from one visit
  owner: person               # who is asked. Defaults to account owner; editable
  credentials_holder: person  # who *can* do it. May differ from owner; constrains delegation
  cadence: RRULE-like         # monthly / quarterly / annual / on-event
  earliest_useful: rule       # the date the document can first exist
  due: rule                   # when we start asking
  grace_days: int             # after which the produced figures degrade
  channel: mobile | desktop | paper | auto
  est_minutes: int            # seeded from a playbook, then learned from actuals
  delegable: bool
  playbook_id                 # the household's own written steps
  state: idle | open | snoozed | done | overdue | dormant
}
```

A **Task** is one instance of an Obligation in one window. A **Check** is an escalated figure needing
a human answer (§6). "Needs you" is the single user-facing list containing both.

### 2.2 The household's real year

Derived from the inventory in `01`. Owner assignments marked *(confirm)* are not stated in the source
documents and must be settled during onboarding, not guessed by the software.

| Visit | Lines produced | Owner | Cadence | Earliest useful | Channel | Est. |
|---|---|---|---|---|---|---|
| Crédit Agricole feed | Joint current | A *(consent holder)* | continuous | — | auto | 0 |
| CA consent renewal | ↑ | A | ~180 d | T−14 d | mobile | 1 min |
| CA Livret A sweep | Livret A ×3 | A (his + son's), B (hers) | annual, early Jan | 2 Jan | mobile | 4 min |
| CA mortgage statement | Mortgage | A | annual | as issued | mobile | 3 min |
| Julius Baer documents | 5 current + investment | A, and B if separate login | monthly | ~day 3–5 | mobile | 5 min |
| CBA NetBank export | Txn account + 2 cards | A / B *(confirm)* | monthly | day 1 | **desktop** | 8 min |
| TopCard portal ×4 | 4 card lines | A ×2, B ×2 | monthly *(see 2.4)* | statement date | mobile | 3 min ea. |
| Revolut (A) feed | Multi-ccy | A | continuous | — | auto | 0 |
| Revolut (B) feed | EUR | **B** | continuous | — | auto | 0 |
| Wise (A) feed | Multi-ccy | A | continuous | — | auto | 0 |
| PSD2 renewals ×3 | ↑ | each consent holder | ~180 d | T−14 d | mobile | 1 min |
| Bourse Direct positions | PEA | A | monthly | month end | mobile | 4 min |
| Linxea statement | Assurance vie | **B** | quarterly | ~3 wks after qtr end | mobile | 4 min |
| Pillar 2 certificate | Pillar 2 | A | annual, Q1 | Jan | paper → camera | 3 min |
| Hostplus statement | Super | *(confirm)* | annual, **September** | Sept | mobile | 4 min |
| Property revaluation | House | A | annual | anniversary | desktop | 10 min |
| Livret A rate update | *(model input)* | A | 1 Feb, 1 Aug | on date | in-app | 1 min |
| Son's majority handover | Son's Livret A | A | once, at 18 | T−90 d | in-app | — |

**Steady-state monthly load: seven or eight visits, roughly 25 minutes, producing about thirteen
figures.** Of those, one is desktop-only, and up to four are the TopCard portal. That is the shape of
the thing. It is a Saturday-morning chore, not a daily habit, and it should be designed as one.

### 2.3 Two cadence corrections to the brief

The remit statement gives "Livret A interest twice a year" and "quarterly super" as cadence examples.
Both are slightly off, and the reminder schedule inherits the error if uncorrected.

- **Livret A interest is capitalised once a year, on 31 December**, and appears on the account in
  early January. What happens twice a year — 1 February and 1 August — is the *rate revision*. So
  there are two distinct obligations: an **annual balance check in early January**, and a
  **twice-yearly rate update** that is a one-tap model input, not a portal visit. Modelling them as
  one twice-yearly "go and check the balance" produces an August reminder for a balance that has not
  moved, which is exactly the kind of reminder that teaches people to ignore reminders.
- **Hostplus issues an annual member statement for the year ending 30 June, generally available in
  September**, with five years of statements retained in Member Online. Balance is viewable online at
  any time, so a quarterly *optional* balance snapshot is reasonable; a quarterly *document* reminder
  is asking for a document that does not exist three times a year. Default: annual document
  obligation in September, plus an optional quarterly snapshot the household can switch off in one
  tap without guilt.

### 2.4 The worst chore in the household, and what to do about it

Four TopCard logins, every month, for four card lines. It is the highest-friction, lowest-information
item in the inventory, and it is the one most likely to be the first thing abandoned — which matters,
because pending card liability feeds the *number one priority view* (current cash position).

Recommendation, offered to the household as an explicit trade rather than decided silently:

- Cards are settled by debit from an account we already see. The **settlement debit observed in the
  Crédit Agricole or Julius Baer feed is authoritative for the prior statement total.** Use it. That
  gives a correct, zero-effort figure for the closed period.
- Between statements, the outstanding liability is genuinely unknowable without the portal. Do not
  invent it. Represent it as a **floor plus an unknown accrual** (§7.4).
- Default: keep **monthly** downloads for the two highest-spend cards; drop the other two to
  **quarterly** reconciliation. Show the household the honest consequence of each choice in the cash
  view before they pick.
- Independently, offer the one-time setup task "turn on email delivery of TopCard statements", which
  converts a monthly portal login into a forwarded attachment (§5.1, path 4).

This halves the worst recurring chore. It is the single largest ease-of-use win available in the
inventory and it costs precision in exactly one number, honestly labelled.

---

## 3. Onboarding

Two people, two paths, one shared decision that neither may take alone.

### 3.1 Partner A's path (~25 minutes, resumable at every step)

**Act 1 — The honest bargain.** The first substantive screen states the cost before the benefit:

> This app cannot fetch most of your money.
> Four of your account lines update themselves. The other twenty-two need someone to go and get a
> file — about eight portal visits and ten documents a month, roughly 25 minutes.
> If that is more than you want, we can track fewer accounts accurately instead of all of them badly.
> You can change this later, in either direction.

Followed by a scope choice — **Everything (26 lines) / Core (the ~12 that move) / Cash only** — which
pre-sets which obligations get created. Putting scope reduction on screen at minute one is deliberate:
it makes shrinking the ledger a normal operating mode rather than an admission of failure in month
four, when it is most needed and hardest to propose.

**Act 2 — Add accounts by institution, not by account.** The mental model is trips, not lines (P2).
Each institution card asks, in this order: who banks here → which lines → who owns each → who may see
each → how the data arrives (auto / document / typed) → who is responsible → cadence, pre-filled from
the institution playbook. **No balances are requested in Act 2.** Asking for twenty-six numbers before
demonstrating any value is how onboarding dies at line nine.

**Act 3 — The visibility audit** (§3.2). Mandatory, and the most novel part of onboarding.

**Act 4 — Prove exactly one number.** Onboarding does not complete without a single true figure on
screen, sourced end to end. Preferred: link Revolut (fastest tier-A win, ~60 seconds). Fallback:
upload one real statement and watch it become a number with a tappable provenance trail. This is the
moment the app stops being a form and starts being a ledger, and it must happen inside the first
session.

**Act 5 — Build the calendar.** Show the derived year (§2.2) as an actual calendar, then ask three
questions and no more: when is **Statement Day** (a recurring slot, default first Saturday morning);
what are quiet hours; which currency does A think in (INV2). Notification permission is requested
*here* — in context, immediately after showing what will be sent and offering email and calendar-feed
alternatives — never on first launch. A permission prompt fired before the value is visible is a
permanent loss, because a denied Notification permission on Chrome is effectively unrecoverable
without the user digging through site settings.

**Act 6 — Invite Partner B.** A's onboarding reaches "set up, pending Partner B" and **cannot reach
"complete"** until B has answered §3.3. The household state is honestly displayed as provisional in
the interim.

### 3.2 The visibility audit, and the co-tenancy leak

The feasibility assessment identifies the arithmetic leak: a total containing a hidden number reveals
that number. D3 resolves it. But there is a **second leak of the same family that D3 does not touch**,
and it is arguably more likely to cause harm because it feels like a feature:

> If Partner A's Julius Baer e-banking login shows all five current accounts, then Partner A can read
> Partner B's balances any time he downloads the monthly statement — regardless of anything this app
> does. And D2 says he may upload it.

Call this the **co-tenancy leak**: privacy the app enforces but the *source system* does not. Any
privacy model that ignores it is theatre, and the household will discover this at the worst possible
moment. Candidates in this inventory: Julius Baer (one relationship, five accounts), CBA NetBank
(joint visibility of two personal cards), TopCard (possibly one portal per cardholder, possibly not),
Crédit Agricole (joint account plus three livrets, one of them the son's under A's PoA).

So onboarding runs a **visibility audit**, one question per institution:

> When you log in to Julius Baer, whose accounts can you see?
> ○ Only mine  ○ Mine and Partner B's  ○ Not sure — I'll check

From the answers the app builds a co-visibility graph and shows both partners one plain screen, split
in two:

- **Private, and the source agrees** — the app hides it and nobody with a legitimate login can see it.
- **Private in this app only** — the app hides it, but the named person can read it at the source. The
  app says so in those words, per account, and offers three real options: (a) accept and mark the line
  *co-visible* rather than private; (b) the owner fetches her own statements from her own login, and
  the co-tenant's obligation is removed; (c) keep it marked private, understanding this protects the
  *stored data, the extraction provider and the app's screens* but not the human who holds the login.

Option (c) is worth having and worth being blunt about. It genuinely narrows the threat model rows for
"hosting provider / insider" and "extraction model provider" in `02` §5. It does nothing at all for
"curious partner". Saying that out loud at setup is the difference between an informed household and
an ambushed one.

**How D2 and D3 coexist.** Upload stays unrestricted (D2). What is controlled is what an upload
*shows back*. When A uploads a combined Julius Baer statement:

- A sees a receipt: *"Julius Baer, August 2026. 6 accounts recognised. 4 updated. 2 routed to Partner
  B."* — institution, count, and nothing else. No values, no line names for B's private accounts.
- Figures for B's private lines commit or escalate on B's side. **Checks route to the account owner,
  never the uploader** (§6.4).
- The stored document is redacted of B's private lines before extraction egress (INV8), so the
  extraction provider and the database never hold them, even though A's own hard drive might.

### 3.3 Partner B's path (~8 minutes), and the privacy conversation

The caveat in D3 is explicit: *Partner A has accepted this on the household's behalf … the failure mode
is domestic rather than technical.* The design's job is to convert a fait accompli into a decision,
without staging a confrontation.

B arrives on her own invite link and creates **her own credentials on her own device**. There is no
shared login at any point, and the app must never suggest one.

Her first screen is not a feature tour. It is a disclosure:

> Partner A has set up a household ledger with 26 accounts across 11 institutions.
> He has proposed that 6 of them are yours, 4 of those private to you, and that the household total
> shows only shared accounts and says so.
> Nothing here is settled. The next three screens are yours to decide.

Then, in order:

**Screen 1 — What "private" means here, including the arithmetic.** All three options from `01` §"the
hard part", in plain language, with the leak stated honestly for each:

| Option | What the household total shows | What the other partner can work out |
|---|---|---|
| **Outside totals** (D3, proposed) | Only shared accounts. Labelled partial. | Nothing. He knows a private account exists, not its size or movement. |
| **Included but coarsened** | A true-ish total, rounded or banded | Roughly the size, and — over months — the direction of movement. Reduces the leak; does not remove it. |
| **Included, leak accepted** | Exact and complete | The balance, by subtraction, and its month-to-month movement. Fine *if mutual and informed*. |

**Screen 2 — Her choice.** Not a confirmation of A's. A real three-way selection, with the app's
recommendation marked and A's proposal marked, distinctly.

**Screen 3 — Her accounts, per line.** Which of her six are private, which shared. Default from A's
proposal, every one changeable. Plus her display currency (INV2 — she may think in EUR while he thinks
in CHF), her reminder channel, and which of her obligations she keeps versus delegates to A.

**Screen 4 — Her minimum.** Explicitly offered, because P5:

> The smallest useful version of this for you is: renew your Revolut link twice a year, and send your
> Linxea statement four times a year. Six things. Everything else can be Partner A's job.
> [ That's what I want ] [ I'd like to do more ] [ Let me choose per account ]

Offering the floor as a respectable, named option is what stops B from either over-committing in the
setup glow and defaulting in month three, or refusing outright.

### 3.4 Disagreement, decline, and dual consent

**If B chooses a different privacy option from A**, the app does not adopt hers, does not keep his, and
does not average them. It:

1. Sets the household to the **most protective of the two positions** as an interim state — private
   accounts outside aggregates — because that is the only position that can be reversed later without
   having already leaked.
2. Tells both of them, in the same words, that there is a difference and what each position implies.
3. Opens a shared, neutral screen showing both positions side by side. No messaging, no arbitration,
   no nudging. The app's job here is to be an accurate minute-taker, not a mediator.
4. Leaves the household provisional until they agree. Views work; they are labelled.

**If B declines entirely** — no login, no participation — that is a legitimate outcome and must be
handled without pretence. The household is recorded as **one-person-maintained**; B's account lines are
labelled *recorded by Partner A, not confirmed by Partner B*; the privacy model stays at its most
protective; and A's home screen carries a persistent, quiet, non-dismissable note saying so. The app
must never render an unconfirmed arrangement as an agreed one.

**Strengthening D3 (declared, argued):** D3 requires that a change to visibility policy be *visible to
both partners*. I am specifying **dual consent** instead: a change to the household privacy model takes
effect only when both partners have accepted it, logged tamper-evidently, with the previous policy and
the initiator recorded.

The argument: visibility-after-the-fact means one partner can unilaterally move the household from
"private outside totals" to "included, leak accepted" and the other learns about it once her balances
are already derivable. That is not reversible — a leaked six months of movement stays leaked. The
failure this guards is precisely the domestic one D3's own caveat flags. The cost is one extra
acceptance on a screen that will be visited perhaps twice in the product's life. Per-account visibility
changes by an account's *own owner* remain unilateral and instant; only the household-wide policy needs
both. Security-by-design owns the mechanism; this is the requirement.

### 3.5 The son

A third data subject with no login, no consent, and an expiry date on the current arrangement.

- His Livret A is tagged **held under power of attorney**, visually distinct from both partners'
  accounts everywhere it appears, and excluded from household net worth by default (it is his money,
  not theirs) with an override that must be chosen deliberately.
- Minimal retention by default: balance, interest credit, as-of date. Not transaction narratives.
- Onboarding creates a dated obligation at **T−90 days before his 18th birthday**: *"[Son] turns 18 on
  4 March 2031. Decide now: hand over his account data to him, export and delete it, or ask him
  whether he wants it kept here."* It appears in the calendar the day it is created, so it is visible
  for years before it fires.

---

## 4. The ingestion experience

### 4.1 Capture paths, in priority order

| # | Path | Platform | Why it matters |
|---|---|---|---|
| 1 | **Android share target** | Android, installed PWA | Julius Baer Mobile and Cornèrcard's card24 both expose statement PDFs in-app. Share → FinanceApp removes the download-find-upload loop entirely. Manifest `share_target` with `method: POST`, `enctype: multipart/form-data`, `params.files.accept` listing **both** `application/pdf` and `.pdf` — extension-only entries are a known cause of the app appearing in the share sheet but failing to receive the file. |
| 2 | **Multi-file picker** | All | Drop six PDFs at once. The app fans them out to accounts by matching content, not by asking. |
| 3 | **Camera** | Mobile | The Pillar 2 certificate arrives on paper in Q1; so may notary and property documents. Capture, confirm crop, done. |
| 4 | **Email forward** | All | Most of these institutions email "your statement is ready", and several will attach it. A per-household ingest address turns that email into a task or a document with zero portal visits. **Flagged for data security** — see §11. |
| 5 | **Desktop file association** | Chromium desktop | `file_handlers` for `application/pdf` plus `launch_handler: { client_mode: "focus-existing" }` so a downloaded statement can be opened straight into the app. Desktop-only capability, correctly matched to the desktop-only chores. |
| 6 | **Manual number entry** | All | §4.5. Never a form. A number pad. |

Every path lands in the same pipeline and produces the same provenance record. A document captured by
camera and one shared from a bank app differ only in a provenance flag.

### 4.2 Statement Day

**The batching claim:** ten documents scattered across a month is ten interruptions, each costing a
context switch worth more than the task. Ten documents in one sitting is one chore with a beginning
and an end. Batching is the single highest-leverage decision in this document.

The household picks a recurring slot at onboarding. Everything accumulates into it. **Only three
things are allowed to interrupt between sessions:**

1. A PSD2 consent within 14 days of expiry (missing it costs a re-authentication, not a document).
2. A Check older than seven days (a held figure is a hole in the ledger).
3. A card statement inside three days of its payment due date (this one has money consequences).

Everything else waits. Nothing else is urgent, and pretending otherwise is what burns the channel.

**The session — "Catch-up".** Structure:

```
┌──────────────────────────────────────────┐
│  Catch-up — September                    │
│  9 things · about 12 minutes             │
│                                          │
│  ON THIS PHONE                     4     │
│   Julius Baer — August documents   5 min │
│   TopCard Visa CHF (A)             3 min │
│   TopCard Visa EUR (A)             3 min │
│   Bourse Direct — positions        4 min │
│                                          │
│  JUST A NUMBER                     2     │
│   Livret A (his) — annual interest       │
│   Property — annual revaluation          │
│                                          │
│  NEEDS A COMPUTER                  2     │
│   CBA NetBank — Jul, Aug, Sep slices     │
│   [ Send to my computer → ]              │
│                                          │
│  PARTNER B'S                       1     │
│   Linxea — Q3  · reminded 2 days ago     │
│   [ Nudge her ]                          │
└──────────────────────────────────────────┘
```

Rules that make this work:

- **Grouped by channel, not by importance.** The user's constraint in the moment is what device they
  are holding. Within a group, shortest first — early wins carry a session.
- **Time estimates are learned from their own history**, not guessed. An estimate that is
  systematically wrong is worse than none.
- **Partial completion is a success state.** "4 of 9 done — the rest are still there, nothing was
  lost." There is no failure screen, no red, no percentage-complete shaming. The five undone items
  simply remain, and the affected figures age honestly.
- **Partner B's items are visible to A but not actionable by him** unless delegated — he gets one
  "nudge her" button, rate-limited to once per day per item, and the nudge is clearly attributed to
  him rather than to the app. Making the app the messenger for an interpersonal ask is how software
  starts domestic arguments.
- **The session ends with the payoff, always.** Not "Done!" — a screen that shows what the effort
  bought:

  > Everything is current as of today.
  > Household net worth **€ 847,210** — up € 4,180 since 2 August.
  > **€ 2,900 contribution · € 1,420 market · −€ 140 FX.**
  > Cash position **€ 18,340**, after € 3,100 of pending card liability.
  > 3 accounts still unknown → see them.

  The FX decomposition (INV10) is the thing you could not have gotten by logging into eleven portals,
  and it is the reason the chore was worth doing. The presentation agent owns how this renders; I own
  the requirement that **a session must never end without it**.

### 4.3 Playbooks

The quiet tax in month four is not the download. It is *remembering where the download button was*, in
a portal you visit twelve times a year, in your third language.

Every Obligation carries a **playbook**: the household's own numbered steps, shown inline in the task
card.

> **TopCard — Visa CHF (Partner A)**
> 1. card24 app → Cards → Visa CHF ····4471
> 2. Statements → most recent month
> 3. Share → FinanceApp
> *Last worked: 2 August. Usually takes 3 minutes.*
> [ These steps didn't work ]

Seeded from a built-in library at onboarding; **edited by the household**, because only they know their
own portals, their own MFA quirks, and which of the four TopCard logins is which. "These steps didn't
work" marks the playbook stale, keeps the task open without penalty, and offers to capture the corrected
steps right after they succeed — the only moment they will ever accurately remember them.

Portal redesigns are certain over a multi-year life. A playbook that the household maintains degrades
gracefully; a hard-coded flow does not.

### 4.4 The desktop handoff

CBA NetBank export is desktop-only, capped around 600 transactions and two years, and fails silently on
large ranges — three-month slices are the reliable unit. This is the worst-fitting chore for a phone in
the entire inventory, so it gets explicit support:

- Phone → **Send to my computer** → six-character code plus QR.
- On desktop, in an **already-authenticated session** (the code continues a session; it never
  substitutes for a login — security-by-design owns the mechanism), entering the code opens exactly the
  desktop-only items.
- The NetBank task pre-computes the exact slices still missing and shows them as **copy-buttons**:
  `01/07/2026 – 30/09/2026`, `01/04/2026 – 30/06/2026`. Dates in the format NetBank expects. The
  household pastes rather than calculates, and cannot accidentally request a range that fails silently.
- Reverse direction too: a desktop session leaves phone-able items queued for later rather than forcing
  them now.

### 4.5 Manual entry, and asks the app can infer

**Manual entry is a number pad, not a form.** Amount, then done. Currency is pre-set from the account.
As-of date defaults intelligently (statement date, quarter end, 31 December for Livret A interest) and
is one tap to change. An optional photo attaches for provenance. Two taps and a number is the target
for the five tier-C lines.

For genuinely static items, offer **"same as last time"** as a single-tap confirmation that records a
new as-of date without re-entering the value — this is honest (it is a fresh human assertion, logged as
one) and it makes the annual Pillar 2 or property confirmation a five-second job.

**Movement-triggered asks.** The four live feeds see money leave. Use them to pre-fill the manual world:

- A €500 transfer from the Crédit Agricole joint account to a Livret A generates:
  *"You moved €500 to Livret A (hers) on 12 September. New balance should be €12,840. Confirm?"* —
  with the arithmetic shown, one tap to accept, one to correct. The figure is provenance-tagged
  *inferred from observed transfer, confirmed by Partner B*, which is a different and weaker class than
  *extracted from statement*, and the app knows the difference.
- A settlement debit to TopCard establishes the prior statement total without a portal visit (§2.4).
- A standing order to Linxea updates a contribution total between quarterly statements.

This is P3 in its strongest form: the best ask is the one you replaced with arithmetic you already had.
It converts the most common cause of movement in tier-B/C accounts into a one-tap confirmation, and it
is available precisely because the household *does* have four live feeds — the small automated surface
earning its keep well beyond the four accounts it covers.

### 4.6 Offline

Statements get downloaded on trains. The capture path must not require connectivity:

- A shared or picked file is written to IndexedDB and queued immediately; the UI confirms receipt, not
  transmission: *"Saved. Will upload when you're back online."*
- One-shot **Background Sync** drains the queue (well supported in Chromium; a foreground retry on next
  launch is the fallback everywhere else).
- Do **not** depend on Periodic Background Sync for anything user-visible: Chromium-only, gated on site
  engagement, with a ~12-hour floor between events across all origins, and it stops entirely if
  engagement decays — which is exactly the month-four scenario. It is acceptable as an opportunistic
  refresh, never as a scheduling mechanism.
- The task list, playbooks, and all last-known figures are readable offline, correctly labelled by
  freshness. INV5 applies to connectivity as much as to feeds.

---

## 5. Review — earning back D1

D1 replaces the feasibility assessment's mandatory human confirmation with automated four-eyes, and
says explicitly that *the design must earn it back*. This section is that argument. It has four parts:
a hard gate, three well-designed escalations, a visible record of everything that skipped review, and
an empirical measurement of whether the gate is actually working.

### 5.1 Pipeline states

```
captured → identified → redacted → extracted×2 → adjudicated → { committed | check | held }
                                                                     ↓         ↓
                                                              digest+audit  Needs you
```

- **identified** — institution, account, period, document hash. Fails here → *"We don't recognise this
  document"* with a one-tap "tell us what it is". Never discard silently.
- **redacted** — locally, before egress (INV8, D1).
- **extracted×2** — two passes that must be **genuinely independent**, not the same prompt run twice. A
  model asked the same question twice agrees with its own mistakes, and four-eyes over correlated eyes
  is a comfort blanket, not a control. Minimum viable independence: **different model families**, or
  failing that, different input modality (text layer vs. rasterised page) *and* different prompt
  formulation *and* different field ordering. Architecture owns the implementation; ease-of-use owns
  the requirement, because a false sense of verification is worse for trust than none.
- **adjudicated** — the gate below.
- **held** — a figure a human declined to confirm. Not committed, not lost, not blocking.

### 5.2 The auto-commit gate

A figure commits without human review **only if all seven hold**:

1. **Agreement** — both passes agree exactly on every field of that figure (value, currency, as-of
   date, account identifier).
2. **Confidence** — each pass self-reports at or above the per-institution threshold. Self-reported
   confidence is weak evidence; it is a necessary condition, never a sufficient one.
3. **Identity** — institution, account last-4 and period parsed and matched to an *expected* obligation.
   An unexpected document never auto-commits.
4. **Internal consistency** — where the document contains it, opening + movements = closing, to the
   cent, in exact types (INV9).
5. **Plausibility** — the delta against the last known value passes the account's volatility band
   (§5.3, case C).
6. **Materiality** — the change does not exceed the material thresholds below.
7. **Standing** — the institution+document-type pair is past its **first three documents** and not on
   **probation** (§5.5).

Conditions 6 and 7 are additions to D1's stated gate, and both are deliberate.

**Materiality defaults** (financial correctness owns the final numbers; these are the ease-of-use
proposal). A figure escalates regardless of agreement if:

| Rule | Threshold |
|---|---|
| Absolute move | > €1,000 / CHF 1,000 / A$1,700 / $1,100 |
| Relative move | > 15% of the account's prior value |
| Household impact | moves household net worth by > 1% |
| Sign change | any balance crossing zero |
| Liquidity-feeding | any figure entering the cash position view: thresholds halved |
| First figure | the first value ever recorded for an account |

The last row matters more than it looks: the first figure for an account has no history to be checked
against, so conditions 5 and 6 are vacuous. First figures are always confirmed by a human.

**The first-three rule.** For each new institution+document-type pair, the first three documents are
fully human-confirmed even when everything agrees. This costs the household about thirty confirmations
across the whole inventory, once, during the months when they are most engaged. It buys a calibration
baseline, a chance to catch a systematic parsing error (a Swiss `1'234.56` read as `1.23`) before it is
committed forty times, and — not incidentally — it lets the household watch the extraction be right
three times before being asked to trust it. Trust is built by observed correctness, not by a settings
description.

### 5.3 The three checks

An escalation is not one screen. The *question being asked* differs by cause, and asking the wrong
question is how you get a careless tap.

**Case A — Disagreement.** Pass 1 says CHF 12,431.05; pass 2 says CHF 12,431.50.

> **Julius Baer · Current CHF · 31 August**
> [ source crop, number highlighted ]
> Which does this say?
> ┌──────────────┐ ┌──────────────┐
> │ 12,431.05    │ │ 12,431.50    │
> └──────────────┘ └──────────────┘
> [ Neither — I'll type it ]   [ I don't know ]

The user's entire job is reading one number off a picture. Two large tap targets, no keyboard, no
typing when a candidate is right. **Target: 3 seconds.**

**Case B — Low confidence.** Both agree, confidence below threshold, or a field is missing or the crop
is poor. Single candidate, highlighted in context: *"Does this say € 4,208.60?"* → Yes / No, correct
it / I don't know. If the crop is unreadable, offer a wider crop and a re-capture path rather than
demanding the user squint.

**Case C — Implausible.** This is the important one, and the one most designs get wrong. Both passes
agree, confidence is high, the document is internally consistent — and the number still looks odd. In
most cases **the number is right and the app is questioning reality**. The copy must never imply the
user made a mistake or that the extraction is broken; it must show the movement in context and let
them recognise it.

> **TopCard Visa CHF (A) · August**
> Balance **CHF 4,820** — that is **3.1× your usual month**.
> [ 12-month sparkline, the new point marked ]
> Biggest month in the last year was CHF 1,930 (December).
> ● Yes, that's right — big month
> ● That's not right — let me correct it
> ● I don't know — hold it for now

"Yes, that's right" commits **and widens the plausibility band for that account**, so the app learns
that this household's December-sized months happen in August too, and stops asking. An escalation rule
that never learns becomes noise within a year.

**Volatility classes** driving case C (proposed defaults; financial correctness owns the final model):

| Class | Accounts | Escalate when |
|---|---|---|
| **Stable** | Livret A, Pillar 2, super, mortgage, property | any change not explained by a known interest credit, payment schedule, or observed transfer |
| **Drifting** | PEA, Linxea, JB investment | account move exceeds a reference-index move over the same period by more than 8 percentage points |
| **Volatile** | current accounts, cards | outside the rolling 24-month range, or a sign change, or > 3× the trailing median absolute move |

The *drifting* rule is the one that catches real extraction errors in investment accounts, where naive
percentage bands are useless because 12% moves are ordinary.

### 5.4 Routing, "I don't know", and correction

**Routing.** A Check goes to the **owner of the account**, never to the uploader (§3.2). If A uploads a
combined statement, checks on B's private lines appear in B's "Needs you" and A sees only a count.

If an owner does not answer within seven days, the app offers the owner — not the other partner — a
**one-time grant**: *"Ask Partner A to answer this? He will see this one figure and its source page,
once. It will be recorded in Activity."* Consent stays with the owner, the grant is scoped to a single
figure, and it is logged where both can see it. This is the only route by which a private figure ever
reaches the other partner, and it requires an affirmative act by its owner.

**"I don't know" is on every check.** Choosing it holds the figure: not committed, account marked
*Needs attention*, the rest of the batch continues, no penalty, no repeat prompt for 48 hours. Forcing
a binary answer from someone who genuinely cannot tell is precisely how bad data enters a ledger, and
a held figure is more honest than a guessed one.

**Correction is never destructive** (P7). Correcting a committed figure creates a superseding version
carrying the old value, the new value, who changed it, when and why. Any view that showed the old value
can show that it was superseded. Financial correctness owns the ledger mechanics; the UX requirement is
that **a wrong number can always be fixed later, from the number itself, in two taps**, and that fixing
it is a normal act rather than an error state. If correction feels like an admission, people leave
wrong numbers in place.

**Batch ergonomics.** Checks queue and are swiped through, counted ("3 of 7"), with keyboard shortcuts
on desktop (`1`/`2` for candidates, `Y`/`N`, `?` for I-don't-know, arrows to move). Target: ten seconds
per check, and a whole month's checks in under two minutes.

### 5.5 The digest, the audit tap, and probation

Auto-commit that nobody ever sees is indistinguishable from an app making numbers up. Three mechanisms
make it visible and, more importantly, **measurable**:

**The digest.** At the end of every Catch-up session, and monthly regardless: *"23 figures were
committed automatically since 2 August."* One tappable row each — account, value, as-of date, agreement
and plausibility badges, one tap to the source crop. Not a notification, not a modal. A place that
exists and can be checked by anyone who wants to check.

**The audit tap.** Once a month, the app picks **one** auto-committed figure at random and asks a human
to eyeball it against its source:

> One spot check. Does this match?
> **Bourse Direct · PEA · 31 August · € 61,204.18**
> [ source crop ]  [ Yes ] [ No ] [ Skip ]

Ten seconds. Its purpose is not to catch that one error; it is to produce an **empirical auto-commit
error rate** the app can honestly report: *"In 7 months, 7 spot checks, 0 wrong."* That sentence is
what earns D1 back. A claim that two models agreed is an assertion; a measured record is evidence, and
after a year of it the household has a rational basis for trust rather than a vendor's promise.

**Probation.** If a spot check fails, or a committed figure is later corrected by a human, that
institution+document-type pair drops to **full human confirmation for the next three documents**, the
household is told plainly why, and the pair only returns to auto-commit after three clean confirmations.
Confidence thresholds are per-institution and adaptive within a floor. Automation is a privilege each
document format earns and can lose — the same posture INV11 takes toward aggregation.

**Escalation-rate monitoring.** In steady state, escalations should run at or below 15% of figures. If
any institution+doctype exceeds **30% for two consecutive periods, the pipeline is broken, not the
human**: the app says so, drops that pair to full confirmation, and raises an operator task. Otherwise
the four-eyes mechanism silently transfers its failure onto the household as review fatigue, and review
fatigue produces rubber-stamping, which is worse than no review at all.

### 5.6 Four locales, one number

A cross-border foot-gun serious enough to have its own rule. These four formats all appear in this
household's documents:

| Source | Written | Means |
|---|---|---|
| Swiss (JB, TopCard) | `1'234.56` | one thousand two hundred |
| French (CA, Linxea, Bourse Direct) | `1 234,56` | one thousand two hundred |
| Australian (CBA, Hostplus) | `1,234.56` | one thousand two hundred |
| Ambiguous | `1.234` | one thousand two hundred and thirty-four, or one and a bit |

**Rule: every check screen shows the raw string exactly as it appears in the document, next to the
parsed value in the reader's own locale.** `1'234.56` → `€ 1 234,56`. Without this, a French-formatting
reader confirming a Swiss statement will approve a wrong number *confidently*, and the human review step
becomes an accuracy hazard rather than a control.

The same rule applies to dates: `03/04/2026` is shown raw next to **3 April 2026**, spelled out. And
document language (de/fr/it/en) is displayed as-is in the crop, with the app's own labels in the
reader's language — never a translated crop, because the crop is evidence.

---

## 6. Staleness (INV4)

### 6.1 Six states

Freshness is a property of every figure. Six user-facing states, and the vocabulary must be used
identically everywhere:

| State | Meaning | In totals? | Shown as |
|---|---|---|---|
| **Live** | automated feed, refreshed within cadence | yes | value |
| **Current** | as-of date within the account's expected window | yes | value |
| **Ageing** | past the expected window, inside grace | yes, with the total's as-of range widened | value + prominent as-of date |
| **Unknown** | consent expired, or past grace on a volatile account, or provenance missing | **never** | `—`, with last-known available on tap |
| **Held** | a check answered "I don't know" | no | `—` + "a question is waiting" |
| **Dormant** | deliberately untracked | no, and the total says so | greyed, with the date it went dormant |

### 6.2 A declared refinement of INV4

INV4 says an expired connection renders as *unknown, never as a stale balance*. Read literally and
applied to every figure, the app would be almost entirely unknown almost all the time: a Julius Baer
statement is a month-end snapshot, so by the 20th it is twenty days old, and a strict reading makes it
unusable. That directly contradicts INV5, which requires the app to be **fully usable with zero live
feeds** — and the document-driven path is the primary experience, not a fallback.

I am therefore proposing a precise reading rather than a change, and stating it openly:

> **INV4 applies to figures that have become *unknowable*, not to figures that are simply *old*.** A
> figure is Unknown when (a) a live connection's consent is invalid or has failed twice consecutively,
> (b) a document is overdue past grace **and** the account's volatility class means the last value can
> no longer be relied on, or (c) provenance is missing. A last-known value with a clearly stated
> as-of date, in a Stable-class account, is not stale in INV4's sense — provided it is **never
> presented as current**.

Two consequences follow, and both are requirements on the other agents:

- **Grace is per volatility class, not global.** Volatile accounts (current accounts, cards) go Unknown
  quickly — 15 days past due. Drifting accounts get 45 days. Stable accounts (Livret A, Pillar 2,
  mortgage, property) can carry a last-known value for a year, because that is genuinely what is known
  about them, and marking a Livret A "unknown" in July when it provably cannot have moved is noise
  that trains people to ignore the state entirely.
- **The cash position view has a stricter rule than the net worth view.** Liquidity may only be
  computed from Live and Current figures. Anything Ageing is excluded from the cash number and shown
  separately as *"+ € X not counted, last seen 30 June"*. Priority-one decisions get made on the cash
  view; it must not silently include a six-week-old card balance.

### 6.3 Rendering rules (requirements on the presentation agent)

1. **Unknown never contributes to a total**, and never renders as zero.
2. **A total containing Unknown lines is labelled partial**, with a count — *"€ 812,400 · 3 accounts
   unknown"* — and that label is itself a tap target listing them with their fixes.
3. **Freshness is never conveyed by colour alone** (accessibility, and colour semantics differ across
   the household's languages).
4. **Ageing and Current must not share a visual weight.** If they look the same, the state does not
   exist.
5. **The fix is one tap from the figure** (P6): tapping an Unknown balance opens re-link, or the upload
   task, or the number pad — the specific one, never a menu.

### 6.4 The card-liability floor

Pending card liability between statements is genuinely unknowable without the portal. Representing it
as a single number is a lie in either direction. The cash position shows:

> Pending cards **at least € 3,100** — last statements, unpaid.
> **+ up to ~3 weeks of unseen spending.** [ Update cards ]

A floor plus an honestly-labelled unknown accrual. If the household chooses monthly TopCard downloads
(§2.4), the accrual window shrinks to days; if quarterly, it is wide and says so. The user can see the
consequence of their own cadence choice in the number they care most about — which is the most
effective possible argument for keeping the chore up, and it is made by the data rather than by a
notification.

### 6.5 Re-link, before it breaks

PSD2 consents expire roughly every 180 days, per connection, silently. Three consents held by two
people (A's Revolut, A's Wise, B's Revolut) plus Crédit Agricole means roughly eight re-authentications
a year scattered unpredictably. Handled properly this is trivial; handled by waiting for failure it is
the most likely cause of a silently wrong home screen.

- **T−14 days:** a quiet in-app card in Needs you. No push. *"Revolut connection expires 12 October —
  renew now, 30 seconds."*
- **T−3 days:** one push, to **the person who authenticated it**, not the household administrator.
- **T−0:** the figure goes Unknown *immediately*. No grace for consent expiry — the bank has told us we
  can no longer know.
- **Silent failure:** two consecutive refresh failures ⇒ treat as expired within 24 hours. Requirement
  on architecture: the aggregator adapter must distinguish **consent invalid** from **temporarily
  unavailable**, and the app must never render a cached balance as Live after two failures.
- **Renewal is one tap → bank SCA → back**, returning to where the user was.
- **Cross-partner:** if B's Revolut consent expires, **B** gets the task; A gets a read-only line in the
  household view — *"Revolut (B) — unknown since 12 October"* — and a single "nudge her" button, once
  per day maximum, attributed to him.

---

## 7. Reminders

### 7.1 Addressing

Reminders are addressed to the **owner** of an Obligation (§2.1), which defaults to the account owner
and is editable. Partner B is reminded about her Linxea; Partner A is not reminded about it at all
unless she has delegated it to him, in which case **she still receives the completion notice** — so she
knows her account was touched — unless she opts out of that too.

Delegation is per-obligation, reversible, and constrained by `credentials_holder`: an obligation
requiring her login cannot be delegated, and the app says so rather than silently accepting a
delegation that will fail.

### 7.2 Channels and mechanics

| Channel | Role |
|---|---|
| **In-app "Needs you"** | Canonical. Always complete. Everything else is a pointer to it. |
| **Web Push** | Opt-in, requested in context at onboarding Act 5. Deep-links to the specific task. |
| **Email** | Always available, never opt-out-only. The fallback when push is denied — and push denial is effectively permanent, so email is not optional infrastructure. |
| **Calendar feed (ICS)** | Opt-in per user, token'd URL. Obligations as events with alarms. Survives uninstalling the app, denied notification permission, and a new phone. Cheap and disproportionately effective for people who live in a calendar. |

Mechanics, settled against what the platform actually does:

- **Scheduling is server-side, delivered by Web Push (VAPID).** Do **not** use Notification Triggers /
  `TimestampTrigger`: it went through a Chrome origin trial and never shipped to stable, remaining
  behind an experimental flag. Do not use Periodic Background Sync for scheduling either, for the
  reasons in §4.6.
- **Badging is best-effort.** `navigator.setAppBadge` is unsupported or dot-only on Chrome for Android.
  Feature-detect, and always carry an in-app count as the real indicator.
- **Notification content never contains an amount.** Per INV11 and the compromised-device row of the
  threat model, a lock screen must never read "Julius Baer CHF 142,000". Reminders name the institution
  and the action; checks say *"a figure needs your eyes"*. The number lives behind authentication.
- **Quiet hours** are per user, in that user's local timezone, defaulting 21:00–08:00, and are hard —
  nothing is urgent enough here to wake anyone.

### 7.3 The anti-nag rules

Concrete and enforceable, in order of importance:

1. **One reminder per person per day, maximum.** Everything due is consolidated into one message:
   *"3 things need you — Julius Baer and 2 TopCards. About 10 minutes."*
2. **A budget of six pushes per person per month** in steady state. If the schedule would exceed it,
   the app consolidates further or drops the least material item to in-app only. A hard budget forces
   the design to prioritise instead of accumulating notification types over time.
3. **Never remind before the document exists** (P3): each obligation's `earliest_useful` gates it. No
   "check your Julius Baer statement" on the 1st when it lands on the 4th; no Hostplus reminder in
   March.
4. **The consequence replaces the nag.** The ladder is: **due date** → task appears silently in Needs
   you, badge only. **+3 days** → one push. **+10 days** → one softer push, leading with renegotiation
   rather than the task. **+grace** → *no further push ever for that instance*; the figure degrades to
   Ageing then Unknown, the views visibly change, the household total labels itself partial. The app
   stops asking and starts showing.
5. **Silence when nothing is needed.** If everything is current, the app sends nothing at all. An app
   that only ever speaks to demand things gets muted; one that is usually quiet is read.
6. **One positive message a month, maximum** — after a completed Catch-up, and never a streak, a score,
   or a comparison between partners (P5).

### 7.4 Renegotiation and self-repair

Every reminder and every task card carries the same set of honest exits:

```
[ Do it now ]  [ Snooze 3 days ]  [ Do it on Statement Day ]
[ This is too often → make it quarterly ]
[ Hand this to Partner A ]
[ I don't want to track this → mark dormant ]
```

These are not dismissals — they **change the obligation**. "Make it quarterly" rewrites the cadence.
"Mark dormant" removes the line from totals with a visible note and stops all reminders, permanently
and without guilt, until the household revives it.

**Self-repair.** The app volunteers to ask for less, before the household has to:

- Snoozed 3 times consecutively, or completed late 3 times, ⇒ *"This one seems to be a hassle. Make it
  quarterly, hand it to Partner A, or stop tracking it?"*
- An obligation never completed within grace over two cycles ⇒ automatically proposed for cadence
  reduction at the next Catch-up.
- An obligation whose figures have never once escalated over twelve documents ⇒ *"This one has been
  right twelve times running. Want to hear from us less about it?"*

An app that proposes reducing its own demands is the opposite of a nag, and it is the mechanism by which
the ledger shrinks toward the size the household can actually sustain — before it silently becomes
wrong at the size it cannot.

### 7.5 Pause

People go away for a month. An app that keeps notifying through a holiday gets its notifications
disabled permanently, which is unrecoverable.

**Pause** (per user or household, with an end date): all reminders stop; obligations continue to
accrue; figures age and go Unknown honestly; views label themselves. On return, one consolidated
Catch-up: *"You were away 3 weeks. 11 things accumulated, about 20 minutes. Start, or should we thin
this out first?"* — the second option offered first-class, because a 20-minute wall on the day you get
home is the other way this habit dies.

---

## 8. Detecting month four, and shrinking the ledger

### 8.1 Health signals

Tracked continuously, shown to the household plainly on request, never as a score:

| Signal | Healthy | Warning |
|---|---|---|
| **Coverage by value** — % of household value that is Live or Current | > 90% | < 80% for 3 weeks |
| Median days-late per obligation | < 3 | > 10 |
| Catch-up completion rate | > 70% of items | < 40% |
| Checks answered within 48h | > 80% | < 50% |
| Escalation rate | 5–15% | > 30% (pipeline fault — §5.5) |
| Per-partner participation | *not compared* | *not compared* |

Coverage **by value**, not by account count, is the right headline: losing the two TopCard EUR lines
matters far less than losing Julius Baer, and a count-based metric would send the household chasing
the cheapest items.

### 8.2 The reset conversation

When coverage by value stays below 80% for three consecutive weeks, the app opens a conversation —
once, and not again for ninety days:

> **This has got heavy.**
> 62% of what you own is currently up to date. Six months ago it was 94%.
> The four TopCard downloads take the most effort for the least movement. Hostplus and the property
> revaluation are annual anyway.
> **Suggestion:** drop the 4 cards to quarterly and pause Bourse Direct until January. That is
> 25 minutes a month down to 9, and coverage back above 90% — because a smaller ledger you keep true
> beats a bigger one you don't.
> [ Do that ]  [ Let me choose ]  [ Leave it — I'll catch up ]

This is the most important screen in the document. It has to be **offered by the app**, unprompted,
without judgement, and framed as a competent operational decision rather than a failure — because the
alternative is a household that quietly stops and a ledger that quietly lies. "Leave it, I'll catch up"
must be a genuinely respected answer, taken at face value with no follow-up nagging.

---

## 9. Mobile reality, IA, and the details

### 9.1 Information architecture

Five destinations. No more — every added tab dilutes the one that matters.

| Destination | Contains |
|---|---|
| **Now** | Cash position, and one "Needs you" strip. The default screen. |
| **Needs you** | Tasks and Checks, one list, grouped by channel. Empty state is a real, calm achievement. |
| **Accounts** | Grouped by institution (matching the visit model), each with its freshness state. |
| **Activity** | Append-only: what changed, what auto-committed, who did what, every privacy-policy event, every one-time grant. The transparency surface for both partners and for the digest. |
| **Settings** | People, privacy policy, currencies, reminders, playbooks, data and export. |

Account detail: value, freshness, history, provenance (INV3 — one tap to the source crop), its
obligation, its playbook, its volatility class and grace.

### 9.2 Mobile specifics

- **Installability is a real requirement, not a nicety**: the Android share target (§4.1) requires an
  installed PWA, and it is the largest single friction removal available. Onboarding Act 4 should
  prompt installation immediately after the first successful figure, when the value is freshest.
- Targets ≥ 44px; the whole review flow must be one-thumb operable; check candidate buttons are large
  and far apart, because a mis-tap here writes a wrong number into a ledger.
- Everything must work at 360px wide. The Catch-up list, the check screen and the number pad are the
  three screens to design mobile-first and adapt upward.
- **Desktop is not a phone stretched.** Multi-file drag-and-drop, keyboard shortcuts through the check
  queue, and the file-association path (§4.1 path 5) are all desktop-only wins that suit the
  desktop-only chores.

### 9.3 Language, locale, accessibility

- **UI language per user** (fr / en), independent of display currency (INV2) and of document language.
- Source documents appear in de / fr / it / en. Labels are translated; **crops never are** — a crop is
  evidence.
- Number and date formats per §5.6, with the raw source string always shown beside the parsed value in
  every confirmation context.
- Freshness never by colour alone; every state has a text label and a distinct shape or weight.
- The whole review flow keyboard-navigable and screen-reader labelled, including the source crop, which
  needs a text alternative stating the extracted value and its context.

---

## 10. Targets

Falsifiable, and instrumented from day one. If these are not met, the design is wrong, not the users.

| What | Target |
|---|---|
| Onboarding, Partner A | ≤ 25 min, resumable, ending with one true figure on screen |
| Onboarding, Partner B | ≤ 8 min, ending with a privacy decision she actually made |
| Monthly household effort | ≤ 15 min steady state (from ~25 at launch, via playbooks, §2.4 and inference) |
| Of which Partner B's | ≤ 5 min/month, ≤ 6 interactions/year at her stated floor |
| Human time per clean document | ≤ 20 seconds |
| Time per check | ≤ 10 s (case A ≤ 3 s) |
| All checks in a month | ≤ 2 min |
| Escalation rate | 5–15% |
| Auto-commit error rate | measured monthly by audit tap; any failure ⇒ probation |
| Pushes per person per month | ≤ 6 |
| Taps from an Unknown figure to its fix | 1 |
| Coverage by value at month 6 | > 90% |

---

## 11. Interfaces I need from other agents

Stated as requirements, not designs.

**Graphical presentation (`15`)**

- A **freshness token vocabulary** rendering all six states of §6.1 identically everywhere, distinguishable
  without colour, with Ageing and Current visually distinct.
- **Partial-total treatment**: any aggregate containing Unknown lines renders with a count and a tap
  target listing them. Unknown never renders as 0.
- **Provenance affordance**: one tap from any number to its source crop, as-of date and derivation
  (INV3).
- **Session payoff view** (§4.2): what changed since last session, decomposed into contribution / market
  / FX (INV10). A Catch-up must not end without it.
- **Cash position floor + accrual** rendering (§6.4).
- **Sparkline-in-context** for plausibility checks (§5.3 case C).
- Task and check cards legible at 360px.

**Architecture (`12`)**

- Obligation/Task scheduler with per-obligation `earliest_useful`, due, grace, and the escalation ladder.
- Server-side reminder scheduling + Web Push (VAPID) + email; **not** client-side triggers or Periodic
  Background Sync.
- **Two genuinely independent extraction passes** (§5.1) — different model families, or at minimum
  different modality *and* prompt *and* field ordering.
- Aggregator adapter must distinguish **consent invalid** from **temporarily unavailable**, and expose
  consent expiry dates so T−14 works.
- Share-target POST endpoint, offline upload queue (IndexedDB + one-shot Background Sync), desktop
  `file_handlers` + `launch_handler`.
- Document → account matching from content (institution, account tail, period) so multi-file drops fan
  out without user routing.
- Per-user token'd ICS feed endpoint; short-lived single-use desktop handoff codes.

**Financial correctness (`13`)**

- **Volatility class per account** and the plausibility bands of §5.3; materiality thresholds of §5.2.
- Propagation of Unknown and Held through aggregates: a total containing them is *partial*, never
  computed as zero.
- **Superseding corrections** — append-only, old value retained, reason recorded, never destructive.
- **Predicted balance** for movement-triggered asks (§4.5): last known + observed transfer + accrued
  interest where the terms are known, with its own weaker provenance class.
- **Card liability floor** semantics (§6.4) and the exclusion of Ageing figures from the cash view.
- Confirmation that `same as last time` records a new human assertion at a new as-of date rather than
  extending the old figure's validity.

**Security by design (`10`)**

- Separate identities, separate devices, no shared login. Invite flow for Partner B.
- **Dual consent** for household privacy-policy changes (§3.4) with a tamper-evident log both can read;
  per-account visibility changes remain unilateral for the account's own owner.
- Check routing by account owner, and the **scoped one-time grant** mechanism (§5.4).
- Desktop handoff code: continues an authenticated session only, never substitutes for login.
- Notification content policy: no amounts in any push or email subject.

**Data security (`11`)**

- **Decision needed on email ingest** (§4.1 path 4). It is the single largest friction reduction
  available for the seventeen tier-B lines, and it means a statement PDF transits a mail provider before
  reaching controlled infrastructure. INV8 governs *egress*, so this is not a literal breach — but it is
  the same risk wearing different clothes, and it is your call, not mine. If it needs a self-hosted
  receiver to be acceptable, say so and I will design around the constraint rather than assume it away.
- **Retention required for the provenance affordance**: every committed figure needs its redacted source
  crop retained for as long as the figure is displayed (proposal: 13 months of crops; full source
  documents discardable after 90 days). Tapping a number and reaching "source no longer available" is a
  trust failure, so this is a floor, not a preference.
- Redaction must be able to drop **specific account lines** from a combined statement before egress, per
  the visibility audit (§3.2) — not just personal identifiers.
- ICS feed tokens: revocable, and the feed reveals institution names and cadences even without amounts.
  Opt-in with that stated.
- Son's Livret A: minimal retention, and an export-and-erase path ready before his majority obligation
  fires (§3.5).

---

## 12. Where I depart from the brief

Declared openly, per `02` §4.

1. **INV4 — refined, not broken (§6.2).** Literal application makes almost every figure Unknown almost
   always and contradicts INV5. I propose: Unknown means *unknowable* — expired consent, missing
   provenance, or past grace on a volatility class where the last value can no longer be relied on —
   with grace set per class, and a stricter rule for the cash view than the net-worth view. A stated
   as-of date on a stable account is not staleness; presenting it as current would be.

2. **D3 — strengthened to dual consent (§3.4).** D3 requires policy changes be *visible* to both. I
   require both to *accept*. Visibility-after-the-fact allows one partner to start leaking the other's
   position and the other to find out afterwards, which is irreversible and is exactly the domestic
   failure D3's own caveat names. Cost: one extra acceptance, twice in the product's life.

3. **D1 — accepted, with three additions (§5.2, §5.5).** Auto-commit needs the first-three rule,
   materiality thresholds and probation on top of D1's stated gate; and auto-commits must be visible in
   a digest and measured by a monthly audit tap. D1 says the design must earn automated four-eyes back.
   Two models agreeing is an assertion; a measured error rate over months is evidence, and evidence is
   what a household can rationally trust.

4. **Cadence corrections (§2.3).** Livret A interest is annual (31 December), not twice-yearly; the
   twice-yearly event is the rate revision, which is a model input rather than a portal visit. Hostplus
   member statements are annual, available around September. Reminders built on the brief's phrasing
   would fire for documents that do not exist.

5. **A leak the brief does not name (§3.2).** The co-tenancy leak — privacy the app enforces but the
   source system does not — sits alongside the arithmetic leak and is arguably more likely to bite,
   because it feels like a working feature until the day it isn't. The visibility audit is my answer;
   it belongs in onboarding, and it is not optional.

6. **A recommendation with a real cost (§2.4).** Dropping two of four TopCard downloads to quarterly
   trades precision in the priority-one cash view for a halved worst chore. I recommend it as a default
   with the trade shown honestly in the number itself, rather than deciding it silently in either
   direction.

---

## 13. Build order

Mapped onto `01` §"Build order", with what this document contributes at each phase.

| Phase | Ease-of-use deliverable |
|---|---|
| 1 — Prove the connection | Nothing user-facing. Capture real consent-expiry dates for §6.5. |
| 2 — Ledger and two logins | Onboarding Acts 1–3 and 6, the **visibility audit**, Partner B's path and the privacy conversation, dual consent, manual number-pad entry, the six freshness states, Activity. **The privacy conversation ships before any automation.** |
| 3 — Document ingestion | Capture paths 1–3 and 5, playbooks, the three checks, the auto-commit gate with first-three and probation, the digest and audit tap, locale display rules. Start with Julius Baer and TopCard, per `01`. |
| 4 — The four views | Session payoff, partial totals, the card-liability floor, cash-view exclusion rules. |
| 5 — Live feeds and staleness | Obligation scheduler, reminders and the anti-nag budget, the escalation ladder, re-link flow, **Statement Day / Catch-up**, movement-triggered asks, desktop handoff, ICS feed. |
| 6 — Refinement | Health signals, the **reset conversation**, self-repair, pause mode, learned time estimates. |

One deliberate ordering note against the phase list: **Catch-up (§4.2) should be pulled forward into
phase 3** as soon as there are three or more document obligations. Batching is not a polish feature —
it is the mechanism that makes the chore survivable, and shipping unbatched ingestion first teaches the
household the wrong habit during exactly the months when the habit forms.

---

## References

Platform and institutional facts checked August 2026. Everything here changes; re-verify before the
architecture depends on it.

- [Web Share Target API — Chrome for Developers](https://developer.chrome.com/docs/capabilities/web-apis/web-share-target) and [`share_target` — MDN](https://developer.mozilla.org/en-US/docs/Web/Progressive_web_apps/Manifest/Reference/share_target) — POST + `multipart/form-data`; include MIME *and* extension in `accept`.
- [Notification Triggers API — Chrome for Developers](https://developer.chrome.com/docs/web-platform/notification-triggers) — origin trial through Chrome 83; never shipped to stable, experimental flag only. Do not schedule on the client.
- [Web Periodic Background Synchronization API — MDN](https://developer.mozilla.org/en-US/docs/Web/API/Web_Periodic_Background_Synchronization_API) and [Periodic Background Sync — Chrome](https://developer.chrome.com/docs/capabilities/periodic-background-sync) — Chromium-only, ~12 h floor, site-engagement gated, stops when engagement decays.
- [Badging API — Chrome for Developers](https://developer.chrome.com/docs/capabilities/web-apis/badging-api) and [w3c/badging#108](https://github.com/w3c/badging/issues/108) — unsupported or dot-only on Chrome for Android; feature-detect.
- [Associate files with your PWA — MDN](https://developer.mozilla.org/en-US/docs/Web/Progressive_web_apps/How_to/Associate_files_with_your_PWA), [LaunchQueue.setConsumer — MDN](https://developer.mozilla.org/en-US/docs/Web/API/LaunchQueue/setConsumer) — desktop Chromium only.
- [How to export CommBank statements as CSV](https://aussiebankstatements.com/blog/how-to-download-commbank-statement-csv) — desktop NetBank only, no mobile export, ~600 transactions / 2 years, three-month slices in practice.
- [Calcul des intérêts du Livret A](https://www.lesclesdelabanque.com/particulier/calcul-interets-epargne/), [Taux du Livret A : dates de révision](https://eco3min.fr/taux-livret-a-date-revision/) — interest capitalised once yearly on 31 December; rate revised 1 February and 1 August.
- [Hostplus member FAQs](https://hostplus.com.au/members/manage-your-super/forms-and-resources/faq) — annual statement to 30 June, generally available September; five years retained in Member Online.
- [Julius Baer Mobile](https://play.google.com/store/apps/details?id=com.juliusbaer.jbmobile) — in-app e-documents and document-availability push, so statements are shareable from the phone.
- [Cornèrcard card24](https://www.cornercard.ch/en/) — up to 24 months of statement history in-app.
