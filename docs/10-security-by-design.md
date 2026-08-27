# Security by Design

**Owner:** Security-by-design agent
**Scope:** Threat model; authentication and authorisation; the two-user privacy model and its enforcement; session and device security.
**Status:** Design proposal. Binds implementation unless overridden by a later settled decision.
**Date:** 27 August 2026

Reads with, and does not restate, `01-feasibility-assessment.md` and `02-decisions-and-constraints.md`.

---

## 0. The positions this document takes

Sixteen calls, each argued below. Where one contradicts the brief it is marked and the argument is
made in §12 rather than buried.

| # | Position |
|---|---|
| **S1** | Partner A operates the host **and** is the adversary in the threat model row that matters most to Partner B. Server-side authorisation therefore cannot enforce D3. Private data must be confidential **from the host**, which makes client-side encryption a requirement, not an upgrade. |
| **S2** | The unit of visibility is a **perimeter**, not a per-account ACL. Exactly four exist: `A`, `B`, `HOUSEHOLD`, `CUSTODIAL`. Every account sits in exactly one. |
| **S3** | Every derived figure carries an **audience** (a set of principals) and is computed only over accounts visible to *every* member of that audience. This is the closure rule; it is enforced by the database, the type system, and key possession — three layers, independently. |
| **S4** | The "this view is partial" label is **unconditional and permanent**, shown to both partners whether or not any private account exists. A conditional label announces existence. |
| **S5** | No figure may ever expose a count, cardinality, coverage ratio, or "n accounts excluded" computed over rows outside the caller's perimeter. |
| **S6** | Coarsened inclusion (feasibility option 2) is removed from the configurable set. It is not a privacy control; it is a slower leak. Argued in §4.3. The configurable axis becomes two positions, not three. |
| **S7** | Cross-perimeter transfers are recorded as **two unlinked half-entries**. The visible side shows money leaving the perimeter; the destination and the resulting balance are not derivable. |
| **S8** | Idempotency keys are **perimeter-scoped HMACs** of the document hash, not the bare hash. This closes the dedup oracle that D2 otherwise opens into D3. |
| **S9** | Upload returns a **constant response**. No dedup verdict, no match verdict, no error differentiation, no timing signal. Documents the uploader cannot route go to a dual-sealed **blind inbox** and leave it silently. |
| **S10** | Four-eyes escalation (D1) routes **only** to principals inside the target account's perimeter. There is no household administrator who can resolve another perimeter's queue. An item may legitimately sit unresolved forever. |
| **S11** | A document whose most-restricted line is private determines the sensitivity of the whole document. Consequence: **an institution that issues only consolidated statements covering both partners cannot host a private account.** |
| **S12** | An account whose PSD2 feed runs through the other partner's aggregator registration **cannot be private**, because the aggregator console is outside our authorisation boundary. This answers feasibility open question 4. |
| **S13** | Authentication is **passkeys only**. No passwords, no email reset, no SMS, no partner-assisted recovery of a private perimeter. Recovery is an offline printed kit per principal. |
| **S14** | Household genesis requires **both partners' devices to co-sign** the first policy-journal entry. Partner A cannot stand the system up alone. This turns D3's "warrants confirmation with Partner B" caveat from a good intention into a structural precondition. |
| **S15** | The son's Livret A: transaction-level ingestion **stops by default at 16** (when French law lets him operate it alone); the account **freezes at 18** pending his own decision; it **never enters household net worth** at any age. |
| **S16** | Three new invariants are proposed — INV12 read-only by construction, INV13 no perimeter administrator, INV14 every figure declares its audience. §13. |

---

## 1. Threat model

### 1.1 Assets, ranked by what their loss actually costs

| Asset | Where it lives | Loss consequence |
|---|---|---|
| The **map** — which institutions, which jurisdictions, how much, moving when | Ledger, every view | Permanent, irreversible disclosure. Cannot be rotated, cancelled or reissued. |
| **Identity kit inside statements** — full name, address, IBAN/BIC, account numbers, employer via salary credits, Swiss AVS number on the Pillar 2 certificate, possibly an Australian TFN on the Hostplus statement | Document store, extraction pipeline, model provider | Enough to pass telephone or web KYC at several of the eleven institutions. Account takeover at the *bank*, not at us. |
| **PSD2 refresh tokens** (4 accounts) | Aggregator + our token store | Up-to-180-day silent read access without further SCA. |
| **Partner B's private position** | Perimeter `B` | Domestic. The failure mode this whole document exists to prevent. |
| **The son's financial data** | Perimeter `CUSTODIAL` | A third party's data held without his consent, retained past the point where he can object. |
| **Behavioural schedule** — payday, mortgage debit date, travel spend | Ledger | Targeting information: when the house is empty, when a large balance is sitting. |
| Availability of the app | Everything | Low. INV5 already says the app must work with zero feeds; it can also survive being down. Availability is explicitly the least valuable property here. |

### 1.2 Why this is a higher-value target than any single bank login (INV11, argued)

INV11 asserts it. The assertion is correct and worth the argument, because it is what justifies the
cost of everything in §3 and §6.

| | One bank login | This app |
|---|---|---|
| Breadth | One institution, one jurisdiction | 26 lines, 11 institutions, 4 jurisdictions, correlated |
| History depth | Typically 2 years; CBA NetBank exports cap at 600 transactions | Unbounded, and growing, by design |
| Authentication boundaries an attacker must cross | Eleven, independently, each with its own SCA | **One** |
| Identity documents present | Your own statements | Everyone's statements, including a pension certificate and a super statement — the two documents most likely to carry government identifiers |
| Write capability | Payments, standing orders | **None, by construction** (INV12, §13) |
| Fraud recourse | Bank fraud team, chargeback, deposit guarantee, regulator | One operator, a home server, and a support queue of zero |
| Reversibility of compromise | Money is often recovered | Disclosure is never recovered |
| Attacker's next step | Move money and be detected | Sit quietly, read for months, and social-engineer eleven institutions with material that reads as insider knowledge |

The correct calibration: this is not "a household app". Its data profile is that of **a small private
bank's client portal**, operated by two people, with no fraud team. Two structural conclusions follow
and drive the rest of this document:

1. **Confidentiality must not depend on the host.** The host is a single point of total disclosure and
   is administered by one of the two people the privacy model must hold against.
2. **There must be no write path anywhere.** The most effective control available is to make the
   system incapable of moving money, so that compromise is a disclosure event and never a loss event.

### 1.3 Actors — the seed table, extended

The five rows from `02-decisions-and-constraints.md` §5 are carried unchanged and four are added. Added
rows are marked ✚.

| Actor | Capability | Primary concern | Treatment |
|---|---|---|---|
| External attacker | Network access, credential stuffing, stolen session | Full household map; identity data | §6 (passkeys close credential stuffing outright), §7.4, §5 CSP |
| Compromised partner device | Authenticated session, local storage | The *other* partner's private accounts | §7.3 — bounded by key possession, not by policy |
| Curious partner | Legitimate credentials, ordinary UI | Deriving a private balance by arithmetic | §4 in full |
| Hosting provider / insider | Storage and DB access at rest | Statement contents, identity, full position | §3.4 layer 1; DS1–DS4 |
| Extraction model provider | Whatever is sent to it | Residual identity surviving redaction | §5.7, DS6 |
| Future subject — the son at majority | Right of access and erasure | His own data, retained without consent | §8 |
| ✚ **Partner A as host operator** | Root on the machine, DB superuser, backups, logs, TLS termination | Partner B's private perimeter, without her knowing | **§1.4 — the central finding** |
| ✚ **Curious partner as uploader** (D2 × D3) | Legitimate upload right over any document | Turning upload into an oracle about the other perimeter | §5 |
| ✚ **The aggregator** (Enable Banking or successor) | Holds the PSD2 consent and its own console | Sees every linked account under one registration | §5.8, S12 |
| ✚ **Coercion / duress** | Either partner, physically present, demanding an unlock | Everything the coerced party can see | §11 accepted risk — stated, not solved |

### 1.4 The central finding: the operator asymmetry

Partner A is "the primary operator" (`02` §1). Whatever the deployment shape, D1's local-redaction
requirement means *some* component runs on infrastructure the household controls (`02` A2) — in
practice, a machine Partner A administers.

That gives Partner A, in his operator role, capabilities that no authorisation layer running on that
machine can withhold from him:

- read the database directly, bypassing every row-level security policy;
- read backups, WAL segments, and application logs;
- read plaintext at the redaction stage, before it is sealed or after it is opened;
- read TLS-terminated request bodies;
- modify the application to log what it sees, silently, and revert.

Therefore: **any design in which Partner B's private accounts are confidential only because the server
declines to serve them to Partner A is not an implementation of D3. It is a UI convention.** It stops
the curious partner at the ordinary UI and stops nobody who reads a file.

There are exactly three ways out.

1. **Move the host outside both partners' control** — a managed platform neither administers. Fails: it
   contradicts D1/INV8 (something must run on controlled infrastructure), and it substitutes a
   different untrusted party rather than removing one.
2. **Accept it and be honest** — tell Partner B that private means "private from the app, not from
   your husband". Defensible only if she is told in those words, which relegates D3 to a courtesy.
3. **Make the host structurally unable to read private perimeters** — private-perimeter content is
   encrypted under keys held only by the owning partner's devices; the server stores ciphertext and a
   minimal non-sensitive envelope.

**This document takes option 3.** It is the only reading under which D3 means what Partner B will
assume it means when she is asked to confirm it. It is not free — §12 sets out exactly what it costs
and which parts of D1 and D2 it constrains.

### 1.5 Trust boundaries

```
┌─ Partner A's devices ─┐            ┌─ Partner B's devices ─┐
│ keys: K_A, K_H, K_S   │            │ keys: K_B, K_H, K_S   │
│ plaintext: A + H + S  │            │ plaintext: B + H + S  │
└───────────┬───────────┘            └───────────┬───────────┘
            │  TLS + session binding             │
            ▼                                    ▼
      ┌──────────────────────────────────────────────┐
      │  Household node — ADMINISTERED BY PARTNER A  │   ← untrusted for
      │  API, RLS, journal, blob store, redactor     │      perimeters A and B
      │  keys: K_H, K_S.  NOT K_A, NOT K_B.          │      trusted for HOUSEHOLD
      └───────────┬──────────────────────┬───────────┘
                  │ redacted content     │ PSD2
                  ▼                      ▼
        ┌──────────────────┐    ┌──────────────────┐
        │ Extraction model │    │ Aggregator/AISP  │  ← both fully untrusted
        └──────────────────┘    └──────────────────┘
```

The node holds `K_H` and `K_S` because the HOUSEHOLD and CUSTODIAL perimeters are visible to both
partners anyway — server-side processing there costs nothing in confidentiality and buys the entire D1
pipeline. It never holds `K_A` or `K_B`. **This split — server-processed shared perimeter, client-only
private perimeters — is the single most important structural decision in this document.**

### 1.6 Attack trees for the three scenarios that decide the design

**T1 — External attacker wants the full map.**
```
Full household map
├── Steal a session            → passkey-only authn; session bound to a non-extractable
│                                device key; short idle timeout; §6.5
├── XSS in the PWA             → strict CSP + Trusted Types + no third-party script,
│                                no CDN, no analytics; §7.5. (Token binding does NOT
│                                help here — say so plainly.)
├── Compromise the host        → yields HOUSEHOLD + CUSTODIAL in full. Yields perimeters
│                                A and B as ciphertext only. §3.4 layer 1
├── Compromise the aggregator  → yields 4 account lines' transactions, no documents,
│                                no map. Bounded by S12 + INV12
└── Compromise the model       → yields redacted extracts of HOUSEHOLD documents only.
     provider                    Private documents never reach it in openable form. §5.7
```

**T2 — Compromised Partner A device wants Partner B's private balance.**
```
B's private balance from A's device
├── Read A's cached views      → contains HOUSEHOLD + A only. Nothing of B.
├── Call the API as A          → RLS refuses; and even if RLS is bypassed, the rows
│                                are ciphertext under K_B
├── Read A's keys and use them → K_A, K_H, K_S. None decrypt perimeter B.
├── Subtract from a total      → §4. Totals are closed over the audience; nothing to subtract
├── Upload-oracle B's docs     → §5. Constant response, perimeter-scoped idempotency
├── Escalation queue           → §5.6. Never routes across a perimeter
└── Persuade the server to     → §3.6. Only the OWNER can issue a grant; grants are
     re-key B's data to A         journalled and pushed to the owner's device
```
Every branch terminates. That is the test a design of this kind has to pass.

**T3 — Partner A, as operator, wants Partner B's private balance.**
```
├── Read the DB / backups / logs → ciphertext (DS2 requires envelope minimisation)
├── Read the redaction stage    → private documents are never redacted on the node (§5.7)
├── Modify the app to exfiltrate → NOT PREVENTED. §11 accepted risk, with detection:
│    B's key on next unlock         signed release manifests + client-side journal
│                                   verification (§3.7) make silent tampering detectable,
│                                   not impossible
└── Coerce                       → not a technical problem. §11
```
Branch three is the honest residual. It is disclosed in §11 and must be disclosed to Partner B in
those words. A design that pretended otherwise would be worse than one that admits it.

---

## 2. Principals

| Principal kind | Instances | Authenticates? | Notes |
|---|---|---|---|
| `partner` | A, B | Yes, passkey | Owns a private perimeter. Equal authority. Neither is an admin of the other. |
| `custodial_subject` | The son | Not yet; from 18, yes | A data subject with rights today and credentials later (§8) |
| `device` | ~2 per partner | Bound to a principal | Named, listed, revocable, holds a non-extractable session key |
| `service` | ingest worker, feed poller, FX fetcher, reminder scheduler | No interactive session | Each gets its own DB role; none may read perimeters A or B; §3.5 |
| `aggregator` | Enable Banking or successor | External | Holds consent, not credentials (INV6). Outside our authorisation boundary — S12 |

There is deliberately **no `admin` principal.** Operational actions (migrations, restore, key rotation)
are performed as a service role or at the OS level and are journalled by their effects, not granted a
seat in the authorisation model. Giving the model an admin seat would immediately become the shortest
path in tree T2.

---

## 3. The authorisation model

### 3.1 Perimeters (S2)

A **perimeter** is a visibility domain. Every account belongs to exactly one. Exactly four exist:

| Perimeter | Visible to | Contents |
|---|---|---|
| `A` | Partner A | Accounts A has chosen to keep private |
| `B` | Partner B | Accounts B has chosen to keep private |
| `HOUSEHOLD` | A and B | The joint current account, the mortgage, the house, and every account either partner has chosen to share — which will be most of them |
| `CUSTODIAL` | A and B jointly, until handover; the son thereafter | The son's Livret A, and nothing else |

Per-account ACLs were considered and rejected. With two partners, per-account ACLs produce 2^n
configurations, of which four are meaningful, and the other 2^n − 4 are places for a bug to hide. Four
named perimeters are auditable by a human in one screen, which matters more here than generality. The
schema keeps a general grant table (§3.6) so that a fifth perimeter — a shared account with an
accountant, a future second custodial subject — is a migration and not a rewrite.

**Ownership and perimeter changes.**

- `account.owners` is a set of one or two principals.
- An account's perimeter may be changed **only by unanimity of its owners.** Consequence: the joint
  current account can never be made private by one partner. Correct, and it removes an entire class of
  domestic failure.
- Moving an account *into* a private perimeter takes effect immediately and **prospectively**: history
  already seen by the other partner stays visible. There is a separate, explicit, loudly-journalled
  "also retract history" action. Rationale: silently rewriting what your partner saw last week is a
  worse property than the leak it prevents, and the other partner may have it in a screenshot anyway.
- Moving an account *out of* a private perimeter takes effect immediately (you may always reveal your
  own).
- Every change writes a journal entry visible to both partners (§3.7). This is D3's second rule,
  discharged.

### 3.2 Audience and the closure rule (S3)

Let `vis(a)` be the set of principals who can see account `a` (determined by its perimeter).
Let a **figure** be any derived number: net worth, liquidity, a projection, a goal percentage, a
category total, a tax estimate, a chart point.

> **Closure rule.** Every figure declares an **audience** `S ⊆ Principals`. It is computed over
> `{ a : S ⊆ vis(a) }` and over nothing else.

Consequences, which fall out rather than being bolted on:

| View | Audience | Computed over |
|---|---|---|
| A's personal position | `{A}` | `A ∪ HOUSEHOLD` |
| B's personal position | `{B}` | `B ∪ HOUSEHOLD` |
| Household position | `{A, B}` | `HOUSEHOLD` only |
| Custodial view | `{A, B}` | `CUSTODIAL` only, never mixed into either of the above |

This is D3, stated as one rule that a compiler and a query planner can both check. Note that A's
personal position is *complete and true* — that is exactly what D3 promises each partner — while the
household position is *true over a stated domain*. Neither is a lie. Neither contains a subtractable
term.

**The custodial perimeter is never summed into household net worth (S15).** The son's Livret A is not
the household's asset; it is a third party's money held under power of attorney. It renders as a
separate custodial line with its own total. This is a financial-correctness consequence of a security
decision — see FC3.

### 3.3 What "aggregate" must include

The closure rule applies to *every* derived figure, not to net worth alone. The channels that get
missed are always the second-order ones, so they are enumerated here and again in §4.2:

projections · savings-goal progress (per-account and aggregate) · liquidity including pending card
liabilities · category and merchant totals · the FX/contribution/market decomposition of INV10 ·
month-over-month deltas · chart axis ranges and scales · sparkline ranges · "you spent more than
usual" comparisons · the tax lens · reminder counts · staleness counts · search result counts ·
export row counts · **anything with a `count(*)` in it.**

### 3.4 Three independent enforcement layers

Defence in depth here is not decoration; each layer stops a different actor from §1.3.

| Layer | Mechanism | Stops | Does not stop |
|---|---|---|---|
| **1. Key possession** | Private-perimeter content encrypted under `K_A` / `K_B`, held only on that partner's devices | Host operator, hosting provider, backup theft, DB compromise, the other partner in every role | A malicious client build (§11) |
| **2. Database** | Postgres RLS, `FORCE ROW LEVEL SECURITY`, session principal set per request, non-owner app role | Application bugs, injected SQL, a service worker overreaching | Anyone with superuser or filesystem access |
| **3. Type system** | Figures parameterised by audience; no combining across audiences; the only constructor of a data slice is the RLS-backed repository | The overwhelmingly most likely real failure: a developer writing `sum(allAccounts)` in a hurry, eight months in | Nothing at runtime |

Layer 3 deserves a word. In practice the leak will not arrive as an attack. It will arrive as a
correct-looking dashboard query written on a Sunday. Making the wrong query *not compile* is worth more
than any of the cryptography.

### 3.5 Concrete schema and RLS

```sql
create type perimeter as enum ('A', 'B', 'HOUSEHOLD', 'CUSTODIAL');

create table principal (
  id            uuid primary key,
  kind          text not null check (kind in ('partner','custodial_subject','service')),
  display_name  text not null,
  created_at    timestamptz not null default now()
);

-- Envelope only. Nothing here may be sensitive: the host can read this table.
create table account (
  id             uuid primary key,
  institution_id uuid not null references institution(id),
  perimeter      perimeter not null,
  owners         uuid[] not null check (cardinality(owners) between 1 and 2),
  native_ccy     char(3) not null,          -- INV1: never a display currency
  -- label is ciphertext for private perimeters; plaintext for HOUSEHOLD/CUSTODIAL
  label_ct       bytea not null,
  created_at     timestamptz not null default now()
);

-- Which principals can see which perimeter. Rows are created only by journalled
-- policy events (§3.7); there is no UPDATE path and no admin override.
create table perimeter_membership (
  perimeter   perimeter not null,
  principal   uuid not null references principal(id),
  journal_seq bigint not null references journal_entry(seq),
  valid_from  timestamptz not null,
  valid_to    timestamptz,                  -- null = current
  primary key (perimeter, principal, valid_from)
);

create or replace function visible_perimeters(p uuid) returns perimeter[]
language sql stable as $$
  select coalesce(array_agg(m.perimeter), '{}')
  from perimeter_membership m
  where m.principal = p
    and m.valid_from <= now()
    and (m.valid_to is null or m.valid_to > now())
$$;
```

Every table carrying a figure — `balance`, `transaction`, `position`, `document`, `goal`,
`extraction_task` — carries `account_id` and gets the same policy:

```sql
alter table balance enable row level security;
alter table balance force  row level security;   -- the table owner is bound too

create policy balance_visible on balance for select using (
  (select a.perimeter from account a where a.id = balance.account_id)
  = any (visible_perimeters(current_setting('app.principal')::uuid))
);
```

Rules that make this hold in practice:

- The application connects as `app_rw`, which **owns nothing** and is not superuser. Migrations run as
  a separate role that the running application never uses.
- `app.principal` is set with `set_config('app.principal', $1, true)` — transaction-local — as the
  first statement of every request transaction, from the authenticated session, never from a request
  parameter. A request that fails to set it gets a null principal and therefore sees zero rows. **Fail
  closed by default, not by policy.**
- Service roles get their own explicit policies. The ingest worker and FX fetcher have **no read
  policy at all** on private-perimeter rows — they operate on ciphertext blobs keyed by id. The
  reminder scheduler may read `account.id` and a due date, never a figure (§4.2, row "reminders").

**Aggregates are functions with an explicit audience, and the audience must contain the caller:**

```sql
create or replace function figure_networth(audience uuid[])
returns table (native_ccy char(3), minor_units bigint)   -- INV9: exact types only
language sql stable security invoker as $$
  select b.native_ccy, sum(b.minor_units)
  from balance b
  join account a on a.id = b.account_id
  where b.as_of = (select max(as_of) from balance b2 where b2.account_id = b.account_id)
    and (select array_agg(m.principal)
         from perimeter_membership m
         where m.perimeter = a.perimeter and m.valid_to is null) @> audience
  group by b.native_ccy
$$;

-- caller ∈ audience, enforced, not assumed
create or replace function assert_audience(audience uuid[]) returns void
language plpgsql stable as $$
begin
  if not (current_setting('app.principal')::uuid = any(audience)) then
    raise exception 'audience does not contain caller';
  end if;
end $$;
```

Note the aggregate returns **per-native-currency subtotals** and never a converted total: INV1 is
preserved and conversion stays a display-time function. Note also what it does *not* return — no row
count, no excluded count, no coverage ratio (S5).

### 3.6 Grants: the configurable part of D3

D3 requires the model be configurable. Under S1 the switch cannot be a server setting, because the
server cannot decrypt what it would need to include. It is instead an **owner-issued capability**:

```
grant {
  id, journal_seq,
  grantor:   principal,       -- must be an owner of the account
  grantee:   principal | audience,
  account:   uuid,
  right:     'read_full' | 'contribute_to_aggregate',
  expires:   timestamptz | null,
  wrapped_key: bytea          -- K_account wrapped to the grantee's public key
}
```

- `read_full` moves the account, in effect, into the grantee's view: it re-wraps the account key. This
  is how a partner shares a previously-private account without moving perimeter permanently.
- `contribute_to_aggregate` is **feasibility option 3, made explicit**: it wraps only the key for the
  account's *aggregate-relevant projection* (its current balance in native currency), not its
  transactions or documents. The owner is shown, at the moment of granting, a plain-language statement
  that the balance becomes derivable by subtraction and that this is the point of the grant. It is
  informed consent by construction rather than a checkbox.
- Revocation removes future access. It does **not** un-disclose. The UI must say so.
- Every grant and revocation is a journal entry (§3.7), so D3's "must be visible to both partners"
  holds for the configuration mechanism itself, not just for perimeter moves.

The migration path to option 3 is therefore: the owner issues `contribute_to_aggregate` per account.
There is no global switch, and there should not be one — a global switch is a single mis-click that
discloses everything.

### 3.7 The policy journal

A hash-chained, per-entry-signed, append-only log. It is the mechanism by which D3's rule *"the privacy
policy must not be changeable by one person without the other seeing that it changed"* becomes true
rather than aspirational — and, in its gossip form, the only defence available against the operator.

```
journal_entry {
  seq          bigserial primary key,
  prev_hash    bytea not null,
  actor        uuid not null,       -- principal
  device       uuid not null,       -- which device
  event_type   text not null,
  public_body  jsonb not null,      -- ALWAYS readable by every partner principal
  private_ref  uuid null,           -- optional perimeter-scoped detail
  signature    bytea not null,      -- actor's device key over
                                    -- (seq ‖ prev_hash ‖ canonical_json(public_body))
  hash         bytea not null       -- H(seq ‖ prev_hash ‖ actor ‖ event_type ‖ public_body ‖ signature)
}
```

**Journalled event types** (public body readable by both, always):

`principal.enrolled` · `device.registered` · `device.revoked` · `account.created` ·
`account.perimeter_changed` · `account.history_retracted` · `grant.issued` · `grant.revoked` ·
`feed.linked` · `feed.unlinked` · `export.performed` · `custodial.milestone` ·
`custodial.handover` · `recovery.kit_used` · `journal.head_gossip`

Properties, and how each is obtained:

1. **Both partners see every policy event.** `public_body` has no RLS restriction for `partner`
   principals. It carries the *fact* and the *actor*, never a figure. Example body for a perimeter
   change: `{account: <uuid>, label_hint: "Julius Baer ••••4471", from: "HOUSEHOLD", to: "B"}`.
   Deliberately: the label hint is included. Hiding *which* account went private while showing that
   *an* account went private is worse than either extreme — it manufactures suspicion without
   information.
2. **Tamper-evidence against the operator.** Each client independently verifies the chain from its last
   verified head on every sync, and stores that head locally. Both clients publish their last-seen head
   back into the journal (`journal.head_gossip`). Two honest clients that have seen different histories
   will produce heads that do not appear on each other's chain — a fork, surfaced to both as a loud,
   non-dismissable alert. This is a two-party transparency log. It does not prevent Partner A from
   rewriting the journal; it prevents him from doing so *undetectably*, which is the achievable goal.
3. **Signed by a device key, not a server session.** A server that forges an entry cannot produce the
   signature. `journal.head_gossip` entries and their signatures make the whole chain non-repudiable.
4. **Notice windows.** Tightening (making your own account private) and revealing your own account are
   immediate — you should never have to wait to obtain privacy. Anything that *reduces what the other
   partner can see about a jointly-owned account* requires unanimity and therefore has no notice
   window to speak of. `export.performed` and `device.registered` are pushed to the other partner
   within the sync interval.

---

## 4. The arithmetic leak, exhaustively

### 4.1 The direct leak, and why the closure rule ends it

The feasibility assessment states the problem correctly: `Total − Σ(visible) = hidden`. Under §3.2 no
figure containing a term the audience cannot see is ever computed, so there is no `Total` from which
to subtract. The leak does not need to be mitigated; the quantity that leaks is never constructed.

Two properties make this hold rather than merely appear to hold:

- **The label is unconditional (S4).** The household view says *"Covers accounts visible to both of
  you"* permanently, from day one, whether or not either partner has ever created a private account.
  A label that appears when a private account is created is itself a one-bit leak — and a one-bit leak
  that is loudest at the exact moment it matters most.
- **No cardinality anywhere (S5).** "Showing 14 of 26 accounts" is the same leak with extra steps.
  Coverage is expressed as a *definition* ("accounts visible to both of you"), never as a *count*.

### 4.2 Secondary channels

Each of these is a real path from a shared figure to a private one. Each has a stated treatment. This
table is the checklist for review; anything added to the app later gets a row here first.

| # | Channel | The leak | Treatment |
|---|---|---|---|
| L1 | Household net worth | Direct subtraction | §3.2 closure. Not computed over private accounts at all |
| L2 | The "partial" label | Existence of a private account | S4: unconditional, permanent, identical for both partners |
| L3 | Account counts, "14 of 26", pagination totals | Existence and number | S5: no cardinality crossing a perimeter, ever |
| L4 | **Cross-perimeter transfers** | €5,000 leaves the joint account "to B's private account" | §4.4 — two unlinked half-entries |
| L5 | Projections and forecasts | A forecast fed by income landing in a private account | Projections are figures; closure applies. A's forecast uses `A ∪ HOUSEHOLD`; the household forecast uses `HOUSEHOLD` |
| L6 | Savings-goal aggregate | A goal spanning a private and a shared account | Goals are perimeter-scoped. A cross-perimeter goal is expressible only in the owner's personal view |
| L7 | Staleness / reminder counts | "3 accounts need updating" counted across perimeters | Reminders are per-recipient and perimeter-scoped. The scheduler service may read due dates, never figures (§3.5) |
| L8 | **Idempotency / dedup response** | Upload a guessed document; "already ingested" confirms the account exists | S8 — perimeter-scoped HMAC idempotency key. §5.3 |
| L9 | Document matcher | "This statement matched an existing account" | S9 — constant upload response |
| L10 | Four-eyes escalation queue | An escalation about B's private account surfacing to A | S10 — perimeter-scoped queues, no cross-perimeter fallback. §5.6 |
| L11 | Chart axes and scales | A y-axis auto-scaled to include a hidden series | Axis ranges are figures. Computed from the audience-closed slice only. **UX2** |
| L12 | Search and filter results | Empty-vs-non-empty result differentiation | Search runs over the closed slice. Zero results and filtered-out results are indistinguishable because the latter never existed for that caller |
| L13 | Push notification payloads | FCM (a third party) seeing "Balance updated: CHF 42,310" | §7.6 — pushes carry an opaque wake token only. No figures, no account names, no institution names |
| L14 | Timing side channels | A slower response when a hidden account is touched | §5.3 — upload and lookup paths are constant-work on the visible path; where that is impractical, responses are delayed to a fixed floor |
| L15 | Backups and exports | An export by A containing a row from B's perimeter | Export runs the same audience-closed query as the UI. Encrypted backups contain perimeter-B ciphertext that A cannot open (layer 1) |
| L16 | The tax lens | A household tax estimate needing worldwide income (D4) | §9 — the lens runs per-principal on that principal's closed slice; the joint return is handled by minimal disclosure, not by pooling figures |
| L17 | FX decomposition (INV10) | "Pure FX movement of CHF 800" implying CHF exposure size | Decomposition is a figure. Same closure. **FC2** |
| L18 | The aggregator console | Outside our boundary entirely | S12 / §5.8 |

### 4.3 Why coarsening is rejected (S6, contradicts the feasibility menu)

Feasibility option 2 — private accounts included in totals, but banded or rounded — should be removed
from the configurable set. It sounds like a compromise and is not one.

- **Repeated queries destroy it.** The household total is observed daily, for years. Dinur and Nissim
  (2003) showed that answering enough queries with error below Ω(√n) permits reconstruction of the
  underlying values; with `n ≈ 26` accounts and an effectively unbounded number of daily observations,
  the noise required to actually resist reconstruction is far larger than the noise that leaves the
  total useful. This is the same result that forced the US Census to abandon the equivalent approach.
- **The other terms are not secret.** The classic statistical-database defence assumes the attacker
  does not know the visible terms. Here the attacker is the other partner, who knows every visible
  term to the cent, in real time. This is the easiest possible instance of the problem.
- **Differencing beats banding.** Band the total to the nearest €5,000 and the band boundary crossings
  themselves are a signal: the date on which the band moves, combined with known visible movements,
  bounds the hidden value tightly within a few months.
- **It produces a number that is neither true nor honest.** D3's design virtue is that each figure is
  exactly true over a stated domain. A banded total is approximately true over an unstated domain —
  worse on both axes, and it will be quoted in conversation as though it were exact.

The configurable axis therefore has **two** positions, not three: *outside aggregates* (default) and
*inside aggregates with informed, per-account, owner-issued consent* (§3.6, `contribute_to_aggregate`).
If the household later wants a middle position, the honest one is not coarsening — it is disclosing
*existence and institution* while withholding amounts, which is precisely the minimal disclosure set
of §9 and is already built.

### 4.4 Cross-perimeter transfers (S7, L4)

Money moving from the joint current account to Partner B's private Revolut is the hardest ordinary
case, because the outflow is visible by right — it left an account A co-owns — while its destination
and the resulting private balance must not be.

**Representation.** A cross-perimeter transfer is stored as two independent half-entries:

```
half_entry { account: <joint CA>, amount: -5000_00 EUR, class: 'PERIMETER_EGRESS',
             counterparty: null, link_id: null }          -- visible to A and B

half_entry { account: <B's Revolut>, amount: +5000_00 EUR, class: 'PERIMETER_INGRESS',
             counterparty: <joint CA>, link_id: <uuid> }  -- visible to B only
```

The link between them exists **only in the private perimeter**, encrypted under `K_B`. From A's side
the outflow is an egress with no destination; from B's side it is a normal, fully reconciled transfer.

**What still leaks, and why that is acceptable.** A sees that €5,000 left an account he co-owns. He
cannot avoid seeing that; it is his account. What he cannot derive is B's balance, because he does not
know the starting point, the other inflows, or the outflows. A single transfer discloses a *delta with
no base*. This is the correct boundary: joint money is jointly visible, private balances are not.

**Consequence for financial correctness (FC1).** INV10's decomposition needs a fourth class alongside
contribution, market and FX: **perimeter egress/ingress**. Without it, a transfer out of the shared
perimeter reads as a loss in the household view, and the decomposition silently lies — the exact
failure INV10 exists to prevent. This is a requirement on the financial-correctness agent, not a
suggestion.

**Reconciliation without linkage.** The matcher may propose that an egress and an ingress are the same
movement *only inside a single perimeter*. Cross-perimeter auto-matching is forbidden: it would require
a component that can see both sides, which is exactly the component that must not exist.

---

## 5. D2 × D3: upload must not become an oracle

D2 says either partner may upload any document. `02` §2 flags the interaction with D3 and leaves it to
this document. It is the sharpest problem in the brief, sharper than the totals leak, because the
totals leak is solved by not computing something whereas this one is solved only by very deliberately
declining to be helpful.

### 5.1 What Partner A already knows, and what he must not learn

If A is physically holding B's Julius Baer statement, the app cannot un-know it. Chasing that is
pointless. The threat is **amplification** — the app turning a document A already has into structured,
queryable, ongoing knowledge, or turning the *upload endpoint* into an oracle A can query about
documents he does *not* have.

| A already knows | A must not learn from the app |
|---|---|
| The contents of a document he physically possesses | Whether B has an account at that institution |
| That he uploaded something | Whether a document he guessed at has already been ingested |
| That the joint account exists | Whether his upload matched an existing private account |
| His own perimeter and HOUSEHOLD in full | Any extracted figure from a private-perimeter document |
| | Whether an upload was ever claimed, by whom, or when |

### 5.2 The amplification channels

1. **The dedup oracle.** INV7 requires idempotency keyed on document hash. If the API answers
   "already ingested" for a hash, A can synthesise or obtain a plausible statement, upload it, and read
   the answer as a yes/no about B's accounts. This is the single most exploitable channel in the
   system and it is created by an invariant, not by a bug.
2. **The matcher oracle.** "We matched this to an existing account" versus "this looks like a new
   account" is the same bit.
3. **The escalation queue.** D1's four-eyes escalates disagreements "to a human". If that human can be
   A for a document belonging to B's private account, the leak is not a bit — it is the full figure,
   presented in a review UI, with the source page beside it.
4. **Error differentiation.** "Unsupported format" vs "duplicate period" vs "account not found" vs
   "ambiguous match" — four distinguishable errors are two bits.
5. **Timing.** Extraction of a matched document takes longer than rejection of an unmatched one.
6. **Storage-quota and job-counter side effects.** Any shared counter that moves when B ingests.

### 5.3 The blind inbox protocol (S8, S9)

```
1. UPLOAD
   A's client computes H = SHA-256(plaintext) locally.
   A's client generates K_doc, encrypts the blob, and wraps K_doc to BOTH partners'
   public keys (nobody yet knows whose document it is).
   POST /inbox  { ciphertext, wrap_A, wrap_B, size_bucket }

   Server response, ALWAYS, byte-identical:
       201 { inbox_id, status: "received" }

   No dedup verdict. No match verdict. No format verdict. No account name.
   Format problems surface later, in the uploader's own review queue, only if the
   document turns out to be routable by the uploader.

2. IDEMPOTENCY (INV7 preserved, oracle closed)
   The idempotency key is NOT H. It is  idem = HMAC(K_perimeter, H).
   K_A, K_B and K_H are distinct and A does not hold K_B.
   → Re-uploading the same statement into the same perimeter still deduplicates
     exactly as INV7 requires.
   → A cannot compute B's idem for any H, so he cannot test membership.
   → If A and B upload the same document, it is stored twice as ciphertext. Blob-level
     dedup by content hash is permissible at the storage layer (DS5) provided the result
     never surfaces in any response.

3. ROUTING
   A's client shows: "Which account is this?" — a list of accounts in A ∪ HOUSEHOLD,
   plus "Not mine / not sure".
     → picks a visible account: normal path. Redaction, extraction, four-eyes, commit.
     → picks "not sure": the item stays in the household inbox, dual-sealed, unrouted.
   The client never proposes an account the uploader cannot see, and the server never
   attempts a match the uploader could not have made.

4. CLAIM
   B's client lists dual-sealed unrouted items, opens them locally, and may claim one.
   On claim: B's client re-wraps K_doc to K_B alone, writes state='claimed' into a
   column under B's RLS, and DELETES wrap_A. From that moment A cannot decrypt the
   document he uploaded.

5. THE UPLOADER'S VIEW
   A's inbox shows unrouted items as "handed to the household" for a fixed TTL
   (30 days), then they disappear FROM A'S VIEW — whether claimed, unclaimed,
   or discarded. A learns nothing from disappearance because disappearance is
   unconditional.

6. CONSTANT WORK
   The upload path performs identical work regardless of outcome: encrypt, store,
   respond. Classification is client-side. There is no server-side match at upload
   time, so there is no timing signal to equalise. Where a later endpoint must
   differ (e.g. quota checks), responses are padded to a fixed latency floor.
```

The protocol's whole content is: **the server does no work at upload time that depends on data the
uploader cannot see.** Every oracle in §5.2 is a violation of that one sentence.

### 5.4 Mixed-tier documents (S11)

A consolidated Julius Baer portfolio statement covering five current accounts across both partners
plus the investment account is a single document containing lines from more than one perimeter. It
cannot be sealed to one perimeter without destroying data, and it cannot be sealed to both without
disclosing the private lines.

**Rule: a document takes the sensitivity of its most-restricted line.** Two consequences, both hard:

1. Splitting a document by line must happen on the **owner's device**, before anything is sealed to a
   wider audience — which means the owner must be the uploader for any document containing private
   lines. That is a real narrowing of D2 in practice, even though the *right* to upload is untouched.
2. **An institution that issues only consolidated statements covering both partners cannot host a
   private account.** For this household that most likely means Julius Baer accounts are all
   `HOUSEHOLD`, and privacy, if wanted, is exercised at Revolut, Wise, or a separately-statemented
   line. This is a household decision the design must surface at account-creation time, not a
   surprise discovered in month four. **UX3.**

### 5.5 Redaction and extraction for private perimeters (tightens INV8)

INV8 says redaction precedes egress and no unredacted content leaves controlled infrastructure. Under
S1, the household node is *not* controlled infrastructure with respect to perimeters A and B, because
Partner A administers it.

**Requirement: for private-perimeter documents, redaction runs on the owning partner's device.** The
node may relay only ciphertext it cannot open. In practice: the redactor ships as WASM inside the PWA,
runs against the decrypted blob in the client, and emits a redacted extract which the client then
seals for transmission to the extraction model. The node sees an opaque request and an opaque
response. HOUSEHOLD and CUSTODIAL documents keep the full server-side D1 pipeline unchanged, since the
node legitimately holds `K_H` and `K_S`.

Cost, stated plainly: client-side redaction is slower, harder to update, and worse on a mid-range
Android phone with a 40-page portfolio PDF. If the client-side redactor cannot handle a given format,
the correct answer is that **that account cannot be private**, not that the document takes the fast
path. **DS6** carries the mechanism requirement.

### 5.6 Four-eyes escalation routing (S10)

D1's automated four-eyes is two independent extraction passes that must agree, with disagreement, low
confidence or a materiality breach escalating to a human. The routing rule:

> An escalation is visible only to principals in the target account's perimeter. There is no fallback,
> no admin queue, and no timeout that widens the audience.

Consequences to accept deliberately:

- A document A uploaded, that B claimed into her private perimeter, that then failed its cross-check,
  escalates **to B only**. A is told nothing, including that it failed.
- If B never opens the app, the item waits. Indefinitely. **An unresolved escalation is a correct
  state, not an incident.** No SLA, no auto-commit at a lower bar, no "assign to household". Auto-commit
  on timeout would breach D1's own rule that silent auto-commit of an unverified number is never
  acceptable, *and* would be the only remaining cross-perimeter path.
- The cross-check itself must be perimeter-local. Two extraction passes over a private document both
  run against ciphertext the node cannot open; the comparison happens in the owner's client. This
  means the four-eyes comparison is client-side for private perimeters — a real engineering cost, and
  the price of S1. **DS7.**
- Plausibility-against-history (D1's third condition for auto-commit) needs account history, which for
  a private account lives in the owner's client. Same answer: client-side.

### 5.7 The extraction model provider

Unchanged from the seed's treatment for HOUSEHOLD documents. For private perimeters, the provider
receives content redacted by the *owner's* device, which is strictly better than the baseline. The
authorisation-relevant point: the provider must never receive an identifier that links an extract to a
principal or an account — no account UUIDs, no institution names in the prompt metadata, no stable
request identifiers across documents. **DS8.**

### 5.8 The aggregator registration constraint (S12) — answering feasibility open question 4

Enable Banking's restricted-production tier is scoped to accounts *you* link yourself, under one
registered application belonging to one natural person. If Partner A registers the application and
Partner B links her Revolut FR account through it, then:

- the consent, the tokens and the account data flow through a registration A controls;
- A can read B's transactions in the aggregator's own console, or via API calls with the application's
  credentials, entirely outside our app and therefore outside every control in this document.

**Rule: an account whose feed runs through the other partner's aggregator registration cannot be in a
private perimeter.** It may be `HOUSEHOLD`, or it may be private *and manual*, or Partner B registers
her own application and holds her own credentials.

The recommendation is the third: **each partner operates their own connections and holds their own
aggregator credentials.** It costs a second signup and removes a whole trust boundary. It also
distributes the 180-day re-authentication chore to the person who can actually complete the SCA, which
is the only person who can complete it anyway.

---

## 6. Authentication

### 6.1 Passkeys only (S13)

Registration: `residentKey: "required"`, `userVerification: "required"`, `attestation: "none"`,
platform authenticators preferred, `algorithms: [-7, -257]`. Authentication uses conditional mediation
for a clean start, falling back to a modal `get()`.

- **No passwords.** Credential stuffing — a named actor in the seed table — is closed outright rather
  than mitigated. There is nothing to stuff.
- **No email-based reset.** Email is the weakest credential in the household and, for Partner A, is
  likely the same identity that administers the host. An email reset path would make every control in
  §3 conditional on a Gmail session.
- **No SMS, no TOTP fallback.** A fallback factor is a downgrade attack with a support ticket attached.
- **Minimum two passkeys per principal** before the principal is allowed to hold a private perimeter:
  phone plus desktop. Enrolment of the second is a blocking onboarding step, not a nag. **UX1.**
- **No user enumeration.** The app has no public sign-up. Enrolment is by out-of-band invitation code,
  single-use, time-boxed, delivered in person. There are exactly three principals; a self-service
  registration flow is pure attack surface.

Synced passkeys (iCloud Keychain, Google Password Manager) are accepted rather than requiring
device-bound credentials. The alternative — device-bound only — makes household lockout the most
likely incident in the system's life, and the sync provider is already trusted transitively by every
device in the household. This is a deliberate trade, recorded here so it is not re-litigated silently.

### 6.2 Binding encryption keys to the passkey: the PRF extension

Layer 1 of §3.4 needs `K_A` and `K_B` to exist on the partner's devices, be strong, and never touch the
server. The WebAuthn `prf` extension is the right primitive: it yields a stable, per-credential,
authenticator-held secret that is only produced after successful user verification.

```
prf_out   = webauthn.get({ extensions: { prf: { eval: { first: salt_perimeter } } } })
K_unlock  = HKDF-SHA256(prf_out, salt = principal_id, info = "financeapp/perimeter/v1")
K_A       = AES-KW-unwrap(K_unlock, wrapped_K_A_from_server)
```

- The server stores only `wrapped_K_A`. It cannot unwrap it: `prf_out` never leaves the client, and is
  produced only by the authenticator after a biometric or device-PIN check.
- Each of the principal's passkeys wraps the same `K_A` under its own PRF output, so either device can
  unlock. Adding a device requires an already-unlocked device to produce a new wrap — the server can
  never mint one.
- **PRF support is not universal.** As of 2026 Chrome and Firefox support it on the platforms this
  household uses, and Safari supports it for iCloud Keychain passkeys, but roaming-authenticator
  support on Apple platforms is still absent. Since the targets are Android and desktop, this is
  workable, but the design must degrade explicitly: **if a device cannot do PRF, that device may
  authenticate and use HOUSEHOLD and CUSTODIAL, and cannot unlock a private perimeter.** It is told so
  in those words. Silently falling back to a password-derived key would reintroduce the password this
  section just removed. **DS3** carries the KDF and wrapping mechanism.

### 6.3 Recovery

| Perimeter | Recoverable by | Mechanism |
|---|---|---|
| `A` | Partner A alone | Offline recovery kit |
| `B` | Partner B alone | Offline recovery kit |
| `HOUSEHOLD` | Either partner alone | Either kit, or any enrolled device |
| `CUSTODIAL` | Either parent alone, until handover; the son alone thereafter | Kit or device |

**No partner-assisted recovery of a private perimeter.** If A could help B recover `K_B`, then A can
obtain `K_B`, and §3.4 layer 1 collapses. The consequence is real and must be said out loud at
enrolment: **if Partner B loses every device and her recovery kit, her private accounts' history is
unrecoverable.** That is the price of the guarantee. The mitigating design is to make the kit easy to
keep: a single printed sheet, generated at enrolment, with the perimeter name and a checksum, intended
for a physical safe or a bank deposit box — the household already has relationships with eleven of
them.

The kit is a high-entropy secret that wraps the perimeter key directly. Using it writes a
`recovery.kit_used` journal entry visible to both partners, because a recovery is exactly the event a
partner would want to know about.

### 6.4 Re-authentication cadence

Two clocks, and they are different clocks:

| Clock | Interval | Source | Effect |
|---|---|---|---|
| App session idle | 15 minutes foreground, immediate on background for private perimeters | This design | Keys wiped from memory; UV required to resume |
| App session absolute | 7 days | This design | Full passkey `get()` |
| Private perimeter unlock | Every session | This design | PRF ceremony; never cached across sessions |
| **AISP re-authentication** | **180 days per linked account** | PSD2 RTS as amended, and PSD3/PSR, which requires the AISP to apply its own SCA when the user accesses retrieved account data at least every 180 days | Feed goes stale; INV4 says it renders as *unknown* |
| Step-up (§6.5) | Per action | This design | Fresh UV within 120 seconds |

The 180-day clock is worth calling out: it is not our choice, it is regulatory, and PSD3/PSR places the
SCA obligation on the AISP for user access to retrieved data. Operating under an aggregator's licence
means inheriting that obligation. Design accordingly: the re-link prompt is a first-class scheduled
event addressed to the account's owner (INV4, **UX4**), not an error state discovered when a number
goes missing.

### 6.5 Step-up: which actions require fresh user verification

Fresh UV means a WebAuthn `get()` with `userVerification: "required"` completed within the last 120
seconds, for this specific action, with the action's parameters included in the challenge.

- Registering or revoking a device
- Changing an account's perimeter, or retracting history
- Issuing or revoking a grant (§3.6)
- Linking or unlinking a PSD2 feed
- Any export (§7.4)
- Any custodial milestone action (§8)
- Using a recovery kit

Everything else — reading, uploading, categorising, confirming an extraction inside your own perimeter
— does not. A step-up prompt that appears constantly is a step-up prompt that gets clicked through.

### 6.6 Sessions and tokens

**Backend-for-frontend.** The PWA never holds an OAuth token, an aggregator token, or a bearer
credential. The node is the confidential client; the browser gets a cookie.

```
Set-Cookie: __Host-sid=<opaque 256-bit>; Secure; HttpOnly; SameSite=Strict; Path=/
```

`__Host-` prefix (no `Domain`, `Path=/`, `Secure` — enforced by the browser), `SameSite=Strict`
(nothing enters this app cross-site; there is no OAuth redirect landing in the browser under BFF),
`HttpOnly` so script cannot read it.

**Session binding.** At first authentication the client generates a non-extractable ECDSA P-256
keypair via WebCrypto, stores the `CryptoKey` handle in IndexedDB, and signs a per-request proof over
`(method, path, body-hash, timestamp, nonce)`. The server binds the session to the public key.

Be precise about what this buys: it defeats a cookie stolen *from outside the origin* — a leaked log, a
proxy, a backup, a misdirected header. It does **not** defeat XSS, because script running on the origin
can use the key handle without extracting it. Against XSS the controls are §7.5's CSP and Trusted
Types, and nothing else. Pretending token binding is an XSS control is how XSS controls get skipped.

**PSD2 tokens** live server-side only, encrypted at rest under a key the node holds (DS4), never
proxied to the client, and are the one credential-like artefact in the system. INV6 holds: they are
tokens, not bank credentials, and no bank password is ever stored — including for the seventeen
tier-B accounts where a headless-browser login would be tempting in month six. **That temptation is
foreseeable and is refused here in advance: no RPA, no stored bank passwords, no session replay
against a bank's web front end. If a document must be downloaded by hand, it is downloaded by hand.**

---

## 7. Devices

### 7.1 Registry

Every device is registered, named by its owner at enrolment ("A — Pixel", "B — MacBook"), and appears
in a list **both partners can see** — device registration is a `device.registered` journal event with a
public body. A new device on either partner's account is exactly the kind of thing the other partner
should notice, and it costs nothing to surface.

Stored per device: principal, display name, public key of the session-binding keypair, credential ID
of the passkey, PRF capability flag, first-seen, last-seen, revoked-at.

### 7.2 Revocation and lock

- Any principal may revoke any of *their own* devices from any other of their devices. Step-up
  required. Revocation invalidates the session immediately server-side and removes that device's PRF
  wrap of the perimeter key, so the device cannot unlock the perimeter again even offline with a stale
  cache.
- Neither partner may revoke the other's device. There is no admin (INV13).
- **Lock** (manual, or on 15 minutes idle, or on backgrounding) zeroises perimeter keys from memory,
  drops the decrypted offline snapshot, and clears the data portion of Cache Storage. The app shell
  stays cached; figures do not.
- Revoking the *last* device of a principal is permitted but requires the recovery kit to be
  acknowledged first, with an explicit confirmation naming the consequence in §6.3.

### 7.3 What a compromised partner device actually reaches

The seed table's second actor row, answered concretely. Assume A's phone is fully compromised while
unlocked and authenticated.

| Reachable | Requires fresh UV first | Unreachable at any price |
|---|---|---|
| Perimeter `A` in full | Registering a new device | Perimeter `B` — no key exists on the device |
| `HOUSEHOLD` in full | Changing any perimeter | Any bank credential — none exist (INV6) |
| `CUSTODIAL` in full | Issuing a grant | Moving money — no write path exists (INV12) |
| Cached views and the offline snapshot | Linking/unlinking a feed | The other partner's escalation queue |
| Documents in `A`, `HOUSEHOLD`, `CUSTODIAL` | Bulk export | Documents sealed to `B` |
| PSD2-fed data for A's linked accounts | Using a recovery kit | The aggregator console, if S12 is followed |

The important row is the first one on the right. **Perimeter `B` is unreachable not because the server
refuses but because the key is not present.** That is the difference between an authorisation model
and a UI convention, and it is why §1.4 chose option 3.

### 7.4 Exfiltration limiting

A compromised device that can read is a device that can copy. Bound the rate and make it noisy:

- Export is a step-up action, rate-limited to one full export per principal per 24 hours.
- Every export writes `export.performed` to the journal with a public body — **the other partner is
  notified that an export happened**, though not of its contents. For `HOUSEHOLD` data this is
  straightforwardly their business; for a private perimeter the entry's public body names the actor
  and the fact, not the perimeter.
- Bulk read through the API is rate-limited per session, with a per-session cap on distinct documents
  retrieved. Crossing it locks the session and requires a fresh passkey ceremony.
- Document downloads are individually journalled within their perimeter.

None of this stops a patient attacker. All of it converts "silently take everything in ninety seconds"
into "take it over days, leaving a trail" — which for a two-person household with two devices each is
a meaningful difference, because the trail will be read.

### 7.5 Browser-side hardening

```
Content-Security-Policy:
  default-src 'none';
  script-src 'self';
  style-src 'self';
  img-src 'self' blob: data:;
  font-src 'self';
  connect-src 'self';
  manifest-src 'self';
  worker-src 'self';
  frame-ancestors 'none';
  base-uri 'none';
  form-action 'self';
  require-trusted-types-for 'script';
  trusted-types default;
Strict-Transport-Security: max-age=63072000; includeSubDomains; preload
Referrer-Policy: no-referrer
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
Cross-Origin-Resource-Policy: same-origin
X-Content-Type-Options: nosniff
Permissions-Policy: geolocation=(), camera=(), microphone=(), payment=(), usb=()
```

- **No third-party script, no CDN, no analytics, no error-reporting SDK, no web fonts from a remote
  origin.** Everything is served from the app origin. Each of these would be a party with script
  execution in the origin that holds the household's entire financial map. There is no version of this
  app in which a Sentry DSN is a good trade.
- **Trusted Types** is required, not advisory. It is the control that turns "we were careful about
  `innerHTML`" into something the browser enforces.
- Dependency policy: pinned lockfile, provenance-checked installs, and a deliberate bias toward fewer
  dependencies. Supply-chain compromise of an npm package is the most likely route to XSS in an app
  with no user-generated content and no third-party embeds.

### 7.6 Service worker, offline cache, and push (L13)

- The service worker caches **the app shell only**. API responses containing figures are never placed
  in Cache Storage.
- The offline snapshot lives in IndexedDB, **encrypted under the perimeter key**, and is decryptable
  only after a PRF ceremony. Lock drops the plaintext. INV5's "fully usable with zero live feeds" is
  about feeds, not about surviving a lock — offline reading requires an unlock, and that is correct.
- **Push notifications carry no content.** FCM is a third party in the delivery path. A push payload is
  an opaque wake token; the client fetches the actual reminder after unlock. Notification text on the
  lock screen is generic: *"FinanceApp needs your attention"* — never a figure, an account name, or an
  institution name. The institution name alone discloses the household's banking relationships to
  anyone glancing at a phone on a table.
- Screenshots cannot be blocked on the web platform. Accepted (§11). If the Android share-target
  wrapper from the feasibility document is ever built, it should set `FLAG_SECURE`, and its share
  target feeds the §5.3 blind inbox on identical terms.

### 7.7 Network exposure

The app must be publicly reachable over HTTPS for installability and push. But:

- **The operations plane is not.** Database, admin tooling, backups, the redaction service, and SSH are
  reachable only over a private network path (WireGuard or Tailscale). Nothing administrative listens
  on the public interface.
- Aggressive rate limiting and no user enumeration on the one public surface (§6.1).
- Certificate transparency monitoring for the app's domain, alerting both partners. A mis-issued
  certificate is one of the few external attacks that would otherwise be invisible.

---

## 8. The son

The feasibility assessment is right that this needs its own story, and `02` §5 names him as a future
subject with rights of access and erasure. Concretely:

### 8.1 The legal timeline, which the system must track

| Age | Fact | System behaviour |
|---|---|---|
| now (minor) | Livret A held under power of attorney by his legal representatives | `CUSTODIAL` perimeter. Data minimisation from day one (§8.2) |
| 15 | French age of digital consent; he can meaningfully exercise his own data rights | `custodial.milestone` journal entry. The parents are prompted to **inform him** that the account is in the ledger and what it holds. Not optional, not silent |
| 16 | Under French rules he may operate his own Livret A — withdraw without his representatives' involvement, absent their objection | **Transaction-level ingestion stops by default.** Balance and interest only. Rationale: from here the transactions are *his* activity, not the parents' administration of an asset |
| 18 | Banking majority. The bank withdraws the parents' access. Any continued access requires a *procuration he grants and can revoke* | **The account freezes.** No further ingestion. A decision is required (§8.3) |

The 16 rule is the one most likely to be missed and is the one that matters most: it is the point at
which continuing to ingest a full transaction feed changes from administering a child's savings to
monitoring a teenager's spending, without his knowledge. Making that a default rather than a setting
is the whole point.

### 8.2 Data minimisation for `CUSTODIAL`

- Balance, interest credits, and the statutory rate. Everything needed for the four views; nothing more.
- Transaction narratives are truncated to amount and date, with no counterparty text, from the outset.
  A Livret A's transactions are deposits, withdrawals and interest — the narrative adds nothing and
  carries counterparty data about a third party.
- Retention: document originals for the current plus previous tax year only, then the extracted figures
  survive and the source PDF is deleted. Shorter than the household's own retention, deliberately.
  **DS5** carries the mechanism.
- The custodial total is displayed separately and **never enters household net worth** (S15, FC3).

### 8.3 The majority handover

The system computes the son's 18th birthday from a stored date of birth and schedules the freeze. It
must not depend on anyone remembering.

At 18, ingestion halts and the account enters `CUSTODIAL_EXPIRED`, read-only, with three exits — and
this is a *required* decision, surfaced persistently to both parents:

1. **Transfer.** He enrols as a `partner`-class principal with his own passkeys and recovery kit. The
   `CUSTODIAL` perimeter becomes his private perimeter. He can then make it invisible to both parents.
   His parents' continued view, if any, is a grant *he* issues and can revoke — mirroring the bank's
   own procuration model exactly.
2. **Consult-only procuration.** He grants a time-boxed `read_full` to one or both parents. Journalled,
   expiring, revocable by him at any time from his own device.
3. **Erasure.** The account and its history are deleted, satisfying his GDPR Article 17 right at the
   moment he is first able to exercise it. This is the default the system offers first.

**If nobody acts within 90 days of the freeze**, the system executes a partial default: transaction-level
data is erased, and only year-end balances are retained, for as long as the parents have a documented
tax or legal reason to hold them. Doing nothing must not mean keeping everything.

### 8.4 Rights, ahead of time

The design should assume he will one day ask. Before then:

- A `custodial.milestone` entry at 15 requiring the parents to acknowledge that he has been told.
- A one-page export of everything the system holds about him, producible at any time by either parent
  in under a minute — his Article 15 subject access request, pre-built.
- No profiling, no categorisation, no spending analysis, no projections applied to `CUSTODIAL`. The
  system holds a balance history for a minor, and that is all it holds.

---

## 9. Where the privacy model collides with joint legal liability

A collision worth surfacing, because it is not a bug in the design and will otherwise be discovered by
an accountant.

Partner A and B are French tax residents filing as one *foyer fiscal*. French rules require the annual
declaration of foreign accounts *held* — form 3916/3916-bis — with penalties of €1,500 per undeclared
account per year (€10,000 where no assistance treaty applies), plus exposure on assessment. The
household holds Swiss, Australian and Belgian accounts and foreign life insurance. Both spouses sign
the return.

So: **a private foreign account that A does not know exists creates a legal exposure for A**, who is
signing a declaration he cannot make truthfully.

This is resolvable, and elegantly, because of what the form actually asks for. Form 3916 requires the
designation and address of the managing institution, the account title and number, its nature, and the
opening/closing dates. **It does not require the balance** — a zero balance does not exempt, and the
amount is in principle irrelevant to the obligation.

**The minimal disclosure set.** For any account in a private perimeter held **outside France**, the
system discloses to the other partner, and to the joint declaration view, exactly:

```
institution name · country · account nature (current / securities / life insurance)
· account or contract number · opened-on · closed-on
```

and **no balance, no transactions, no history, no movement, ever.**

This is precisely the statutory minimum and precisely nothing more. Amount privacy — which is what D3
is actually protecting — survives entirely. It is also, incidentally, the honest version of the
"middle position" that coarsening was reaching for in §4.3, and it is the one to offer if the household
ever wants one.

Implementation: `account.declarable_facts` is a distinct column set, in the envelope (server-readable),
visible to both partners for any non-French private account, and the account-creation flow states this
plainly at the moment a foreign account is made private. The tax lens (D4, deferred) consumes this set
and never the figures. Not advice, and the app must not present it as such (`02` §2 D4) — but the list
is the list, and the app already holds it.

---

## 10. Interfaces to the other agents

Stated as requirements, not designs. Where the boundary says the other agent owns the mechanism, only
the property is specified here.

### To Data Security (`11-data-security.md`)

| # | Requirement |
|---|---|
| **DS1** | A key hierarchy with four perimeter keys `K_A`, `K_B`, `K_H`, `K_S`. The household node may hold `K_H` and `K_S`. It must be structurally incapable of obtaining `K_A` or `K_B` — including from backups, memory, logs, or a restore procedure. |
| **DS2** | **Envelope minimisation.** Everything the server must read to route, schedule and enforce RLS — account id, perimeter, owner, native currency, institution id, as-of date — is plaintext and is therefore visible to the operator. Everything else in a private perimeter is ciphertext. Draw that line explicitly and defend it; an envelope field that turns out to be sensitive is a silent hole in §3.4 layer 1. In particular: **account labels and institution names for private perimeters must be ciphertext**, since "B has a Julius Baer account" is itself the disclosure. |
| **DS3** | Mechanism for deriving and wrapping perimeter keys from WebAuthn PRF output (§6.2): KDF choice, wrap algorithm, per-device wrap storage, rotation on device revocation, and the explicit no-PRF degradation path. |
| **DS4** | PSD2 token storage: server-side, encrypted, never proxied to a client, rotated on re-link, destroyed on unlink. |
| **DS5** | Blob storage may deduplicate by content hash **provided the result never surfaces in any response** (§5.3 step 2). Retention: `CUSTODIAL` documents at current + previous tax year (§8.2); everything else per your policy. |
| **DS6** | A redaction implementation that runs **client-side** for private perimeters (§5.5), and its format coverage. Where coverage fails, the account cannot be private — please state which formats fail. |
| **DS7** | The four-eyes cross-check for private perimeters runs in the owner's client (§5.6). Specify how two independent extraction passes are obtained without the node seeing plaintext. |
| **DS8** | No stable identifier links an extraction request to a principal, account, or prior request at the model provider (§5.7). |

### To Architecture (`12-architecture.md`)

| # | Requirement |
|---|---|
| **AR1** | The application connects to Postgres as a non-owner, non-superuser role. RLS is `FORCE`d. `app.principal` is transaction-local and set from the authenticated session only. |
| **AR2** | Every background service gets its own DB role with its own explicit policies. No service role may read a private-perimeter figure. The reminder scheduler reads due dates only. |
| **AR3** | The BFF holds all tokens (§6.6). No token, PSD2 or otherwise, is ever sent to the browser. |
| **AR4** | The operations plane (DB, admin, backups, redactor, SSH) does not listen on the public interface (§7.7). |
| **AR5** | The aggregator adapter boundary must accommodate **two independent aggregator registrations**, one per partner (§5.8), not a single household credential. |
| **AR6** | Upload is constant-work and constant-response (§5.3). No server-side matching at upload time. The blind inbox is a first-class component, not a queue table with a flag. |

### To Financial Correctness (`13-financial-correctness.md`)

| # | Requirement |
|---|---|
| **FC1** | INV10's decomposition needs a fourth class: **perimeter egress/ingress** (§4.4). Without it, a transfer out of the shared perimeter reads as a loss in the household view. |
| **FC2** | Every derived figure — including the FX/contribution/market decomposition, projections, goal progress and liquidity — carries an audience and is computed over the audience-closed slice (§3.2, §3.3). No exceptions for "it's just a chart". |
| **FC3** | The `CUSTODIAL` total never enters household net worth (S15). It is a separate line: money held for a third party, not an asset of the household. |
| **FC4** | Cross-perimeter matching and reconciliation is forbidden (§4.4). The matcher operates within one perimeter. |
| **FC5** | Aggregate functions return per-native-currency subtotals (INV1) and never a row count, excluded count, or coverage ratio (S5). |

### To Ease of Use and Presentation (`14`, `15`)

| # | Requirement |
|---|---|
| **UX1** | Second-passkey enrolment is a blocking onboarding step for any principal holding a private perimeter (§6.1). |
| **UX2** | Chart axes, scales and sparkline ranges are figures. They are computed from the closed slice, never from a wider one for visual convenience (L11). |
| **UX3** | Account creation must surface, at the moment of choosing a perimeter: the consolidated-statement constraint (S11), the aggregator-registration constraint (S12), the minimal-disclosure set for foreign accounts (§9), and the no-partner-assisted-recovery consequence (§6.3). These are decisions, not fine print. |
| **UX4** | The 180-day re-link prompt is a scheduled event addressed to the account's owner, not an error discovered when a figure disappears (§6.4, INV4). |
| **UX5** | The household view's "covers accounts visible to both of you" statement is permanent and identical for both partners, present from first launch (S4). Never render it conditionally, never render a count. |

---

## 11. Residual risks, accepted explicitly

Each of these is a real hole. Each is listed because a design that hides its residuals is worse than one
that names them. Items marked **(disclose)** must be stated to both partners in plain language before
launch.

| # | Residual | Why accepted |
|---|---|---|
| R1 | **(disclose)** Partner A, as operator, can modify the application to capture Partner B's key at next unlock. | Unsolvable without reproducible builds and independent verification the household will not perform. Mitigated by detection: signed release manifests, client-side journal verification and head gossip (§3.7) make persistent tampering discoverable. Partner B must be told that the guarantee is *"he cannot read it, and he cannot change the rules without you seeing"* — not *"he cannot ever obtain it"*. |
| R2 | **(disclose)** Coercion. | Not a technical problem. No duress unlock, no decoy perimeter — such features fail badly in a domestic setting and can make things worse. |
| R3 | A partner physically holding the other's paper statement learns its contents. | Outside the system. §5 prevents amplification, not possession. |
| R4 | Screenshots cannot be prevented on the web platform. | Platform limitation. `FLAG_SECURE` if a native wrapper is ever built (§7.6). |
| R5 | **(disclose)** Losing every device *and* the recovery kit loses a private perimeter permanently. | Direct consequence of no partner-assisted recovery (§6.3). The alternative is worse. |
| R6 | A total host compromise yields `HOUSEHOLD` and `CUSTODIAL` in full. | Deliberate: those perimeters are visible to both partners anyway, and server-side processing there is what makes the whole D1 pipeline possible. |
| R7 | Synced passkeys inherit the sync provider's security. | Chosen over device-bound credentials to avoid lockout as the most likely incident (§6.1). |
| R8 | Movement of money *out of* a shared account is visible even when the destination is private. | Unavoidable and correct — it is a jointly-owned account (§4.4). |
| R9 | Timing and traffic analysis by the host operator (upload volume, sync frequency, request sizes). | Padding to a fixed floor covers gross signals; a determined operator with packet captures can infer activity levels. Activity is not a figure. |
| R10 | Availability. | Explicitly the least valuable property (§1.1). INV5 already requires the app to work with no feeds; it can also be down. |

---

## 12. Where this document contradicts the brief

Five places, argued rather than worked around, as `02` §4 requires.

**12.1 — D3's "configurable" is narrowed from three options to two (S6).** Coarsened inclusion is
removed. §4.3 makes the argument: with the other terms known exactly, unbounded repeated observation,
and n ≈ 26, banding is a slower leak wearing a privacy control's clothing, and it produces a number
that is neither exact nor honestly bounded. The configurable axis becomes *outside aggregates* and
*inside aggregates by explicit per-account owner consent*, plus the minimal-disclosure position of §9,
which is the honest middle the household was probably reaching for.

**12.2 — D3 cannot be implemented at the authorisation layer alone; it requires client-side
cryptography (S1).** This is the largest departure. The brief's boundary statement says
security-by-design sets the requirement for the privacy model's cryptographic enforcement and data
security specifies the mechanism — so this is within remit — but it is worth being explicit that the
requirement is *strong*, and that it is driven by a fact about *this* household rather than by
cryptographic maximalism: the person who administers the server is one of the two people the privacy
model must hold against. If Partner A did not operate the host, server-side RLS would be sufficient and
this document would be half its length. The cost lands on D1 (§12.3), on complexity, and on the A1
budget: client-side redaction, client-side four-eyes and client-side plausibility checking are
materially more engineering than the server-side pipeline, though they add no recurring cost.

**12.3 — INV8 is tightened for private perimeters (§5.5).** "Controlled infrastructure" must be read as
*the owning partner's device*, not the household node, for perimeters A and B. This is a tightening
rather than a break, but it changes the deployment shape D1 implies and constrains which document
formats can belong to a private account (S11, DS6).

**12.4 — D2's upload right is preserved; the uploader's feedback is not.** Either partner may still
upload any document, exactly as D2 says. But the uploader receives a constant response, no matching, no
dedup verdict, no extraction result, and no confirmation of what became of a document they could not
route (§5.3). D2's spirit — "onboarding need not treat ingestion as a privileged operation" — is kept:
uploading is unprivileged and frictionless. What changes is that uploading a document about someone
else's private account is *usefully* unprivileged only for the person who can see the account.
Additionally, S11 makes some accounts ineligible for private status outright, which is a genuine
narrowing of the household's options and must be surfaced at account creation (**UX3**).

**12.5 — D3's Partner B caveat is escalated from a note to a launch gate (S14).** `02` records that
Partner A accepted D3 on the household's behalf and that this warrants explicit confirmation with
Partner B. Leaving that as an intention guarantees it gets skipped in the week before launch.

**Requirement: household genesis requires both partners' devices to co-sign journal entry #1.** The
first entry establishes the four perimeters and both partners' enrolment; it is invalid without two
signatures from two separately-enrolled devices. Partner A therefore *cannot* stand up a working system
alone, and Partner B's first act is necessarily an informed one — she enrols, is shown what a perimeter
is, what the recovery consequence is (§6.3), and what R1 and R2 mean, and then signs. The technical
mechanism costs a day. It makes the failure mode this whole document exists to prevent structurally
unreachable.

---

## 13. Proposed invariants

Offered as additions to `02` §4, in that document's style, for adoption.

> **INV12 — Read-only by construction.** The system holds no capability to move money. No payment
> initiation scope is requested, no write API to any institution is implemented, no bank credential is
> stored, and no automated login to any institution's web front end is performed. Every compromise of
> this system is a disclosure event and never a loss event. *(Strengthens INV6 and is the single
> highest-leverage control available, per §1.2.)*

> **INV13 — No principal administers another principal's perimeter.** There is no admin role, no
> break-glass, no support account, no cross-perimeter escalation, and no partner-assisted recovery of
> private data. Where an operational need appears to require one, the correct resolution is that the
> account in question leaves the private perimeter — never that an override is added.

> **INV14 — Every figure declares its audience.** Alongside INV3's provenance, every derived number
> carries the set of principals it was computed for, and is computed only over accounts visible to all
> of them. A figure without an audience does not exist. Counts, cardinalities and coverage ratios
> spanning a perimeter boundary are figures too, and are never emitted.

---

## 14. Security gates on the build order

Mapped onto the six phases of `01` §"Build order", so security is a gate rather than a phase.

| Phase | Gate |
|---|---|
| 1. Prove the connection | Confirm the aggregator's restricted-production terms permit **two separate registrations, one per partner** (S12). If they do not, D3's private perimeters become manual-only, and the household must know that before phase 2. |
| 2. Ledger and two logins | **The whole of §3 and §6 lands here, not later.** Perimeters, RLS, the closure rule, the type-level audience, passkeys, PRF key binding, the journal, and the two-signature genesis (S14). Retrofitting a privacy perimeter into a live ledger is a data migration through the exact boundary it is meant to protect. This is the one phase where security cannot be deferred. |
| 3. Document ingestion | The blind inbox (§5.3), perimeter-scoped idempotency (S8), escalation routing (S10), client-side redaction for private perimeters (§5.5). Ingestion must not ship before these; a "we'll add the inbox later" ingestion path is an oracle in production. |
| 4. The four views | The §4.2 channel table becomes the review checklist. Every new figure gets a row before it gets a pixel (UX2, UX5). |
| 5. Live feeds and staleness | Per-partner aggregator registrations, 180-day re-link scheduling addressed to owners (UX4), token handling per DS4, staleness counts perimeter-scoped (L7). |
| 6. Tax lens and refinement | The minimal-disclosure set of §9, per-principal lens computation (L16), and the custodial milestones of §8 wired to the calendar. |

Additionally, before any real data enters the system:

- Both partners enrolled, two passkeys each, both recovery kits printed and stored offline.
- R1, R2 and R5 disclosed to both partners in plain language, and the disclosure acknowledged in the
  journal.
- The son's date of birth entered and the milestone schedule (§8.1) verified against it.

---

## 15. Open questions for the household

Not for another agent — these need answers from Partner A and Partner B.

1. **Does Partner B want a private perimeter at all?** The design must support it either way, and the
   unconditional label (S4) means the answer is not observable from the app. But knowing changes what
   phase 2 must prove out.
2. **Will Partner B register her own aggregator application?** If not, her Revolut FR feed cannot be
   private (S12) and her accounts are `HOUSEHOLD` by construction.
3. **Where do the recovery kits live?** Two separate physical locations, ideally not both in the house.
4. **Is Julius Baer statemented per account or consolidated?** Determines whether any Swiss line can be
   private at all (S11). Answerable in five minutes by looking at last month's statement, and it should
   be answered before phase 2.
5. **The son's date of birth**, to schedule §8.1. And a decision, now rather than at 16, about what the
   household intends to do at each milestone.
6. **Does the household accept R1?** Partner B is being asked to accept that her private perimeter is
   confidential from her husband's *reading* but not from his *rewriting the software*. If that is not
   acceptable, the only remedy is that Partner A stops operating the host — which contradicts D1's
   local-redaction requirement and would need to be reopened as a decision.

---

*Threat model, invariant proposals and enforcement design as of 27 August 2026. Regulatory anchors
used: PSD2 RTS as amended and the PSD3/PSR framework for the 180-day AISP re-authentication cadence;
French rules on the banking majority at 18 and independent operation of a Livret A from 16; the French
age of digital consent at 15; and form 3916's information requirements, which do not include account
balances. All are structural observations, not advice, and each should be reconfirmed before the design
depends on it.*
