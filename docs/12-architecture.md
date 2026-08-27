# Architecture — System Shape, the Aggregator Seam, Ingestion, Deployment and Resilience

**Owner:** Architecture agent
**Scope:** System shape; the aggregator adapter boundary; the ingestion pipeline end to end; deployment topology; resilience when every feed is down.
**Status:** Design. Binding on where things run and how data moves.
**Date:** 27 August 2026

**Inputs, in order of authority:** `02-decisions-and-constraints.md` (D1–D5, INV1–INV11) ·
`01-feasibility-assessment.md` · `10-security-by-design.md` (S1–S16, AR1–AR6) ·
`11-data-security.md` · `13-financial-correctness.md` §§0–4 (settled; treated as the storage
contract) · `15-presentation.md` §11 (the `Observation` envelope this document owes it).

> **Supersedes an earlier draft.** A previous version of this file was written in parallel with, and
> therefore without sight of, `10-security-by-design.md` and `11-data-security.md`. It placed a
> plaintext ledger on a managed VPS and a "vault node" mini-PC in the house. Both are withdrawn.
> §1 explains why the security work makes that topology not merely weaker but *more* expensive.

---

## 0. The sixteen calls

| # | Decision | § |
|---|---|---|
| **AD1** | The host is a **synchronising encrypted store, not an application server.** There is exactly one ledger implementation and one ingestion pipeline, and both run in the client, for every perimeter. This supersedes security §1.5's split, in which the node holds `K_H`/`K_S` and processes the shared perimeter server-side. | [1](#1-reconciling-s1-with-managed-hosting) |
| **AD2** | Deployment is **managed EU serverless plus managed Postgres**, no home server, no VPS to patch. ≈ **€12–14/month**, inside A1 with room to spare. | [3](#3-deployment-topology) |
| **AD3** | **The partner runner.** One deployable artefact — feed poller *and* extraction relay — deployed **once per partner**, in that partner's own cloud account, under that partner's own aggregator registration. This is the thing that makes a live feed compatible with a private perimeter, and it generalises S12 from a rule about credentials into a rule about deployment. | [3.3](#33-the-partner-runner-ad3) |
| **AD4** | Every source of numbers emits an **`Observation`**. Five producers — feed, document, manual, model, external estimate — one pipeline. Automation is a *cadence*, never a subsystem. This is what makes INV5 structural. | [2](#2-system-shape) |
| **AD5** | The aggregator seam is **two layers**: a raw `FeedAdapter` port and a per-provider `Normaliser`. Every raw provider payload is stored sealed and kept, so history can be **re-normalised after the provider is gone**. | [4](#4-the-aggregator-adapter-boundary) |
| **AD6** | **No provider identifier ever enters the ledger.** `feed_binding` is the only place they live. A provider migration is a binding swap, not a data migration. | [4.4](#44-what-must-never-cross-the-seam) |
| **AD7** | Ingestion is a **client-side state machine over an encrypted operation log.** Escalation routing (S10), staleness counts (L7), export scoping (L15) and push-payload emptiness (L13) stop being policies the server is trusted to follow and become properties of what the server physically holds. | [5](#5-the-ingestion-pipeline), [1.4](#14-what-the-topology-makes-structural) |
| **AD8** | Idempotency is **three-layered and perimeter-scoped**: HMAC'd content hash, HMAC'd logical period key, client-side multiset reconciliation. INV7's literal wording is amended, not broken. | [5.6](#56-idempotency-inv7--s8) |
| **AD9** | Freshness runs on a **fail-closed clock** computed by the client from observed poll success, never from a server flag. `connection.state === 'expired'` forces `value: null` at two independent points. | [6](#6-connections-that-expire-inv4) |
| **AD10** | The **Android share target terminates in the service worker.** Sealing needs only public keys, so a shared statement can be accepted, encrypted and queued **while the app is locked**. | [7](#7-the-android-share-target) |
| **AD11** | **The clients are the system of record.** The host is a rendezvous point and an off-device backup. Host loss is not data loss; key loss is. | [9.4](#94-backup-restore-and-what-the-recovery-kit-actually-is) |
| **AD12** | The presentation agent's display snapshot is admitted under **exactly the rule financial correctness applies to reconciliation postings (F3)**: derived, regenerable, never an input — enforced by a build-time import boundary rather than by discipline. | [10.2](#102-ruling-on-presentation-12--the-offline-display-snapshot) |
| **AD13** | Lock-on-background gets a **90-second grace window**. A small, argued departure from security §6.4. | [11.4](#114-ad13--a-90-second-grace-window-on-background-lock) |
| **AD14** | Market data (D5) is fetched as a **fixed universe file**, never as a per-household ISIN query. The query shape is a portfolio disclosure and there is no need to make it. | [8.3](#83-market-data-ad14) |
| **AD15** | **INV5 is a drill, not a claim.** `NullFeedAdapter` in CI: every feed dead, the whole test suite still passes and the four views still render. | [9.2](#92-the-zero-feed-drill-inv5-made-executable) |
| **AD16** | Stack: TypeScript end to end · Svelte 5 + Vite PWA · Fastify on Node 22 · Postgres 17 · Scaleway (Paris). Justified against A1 in §3.5. | [3](#3-deployment-topology) |

---

## 1. Reconciling S1 with managed hosting

This is the assignment's hard part, so it goes first and everything else is derived from it.

### 1.1 The security agent's argument, restated fairly

Partner A operates the host. Partner A is also the adversary in the threat-model row that D3 exists to
address. Therefore server-side authorisation cannot enforce D3: a policy that stops the curious partner
at the UI stops nobody who can run `psql`. Security §1.4 concludes that private-perimeter content must
be encrypted under keys the host cannot obtain, which drags client-side redaction (§5.5), client-side
four-eyes (§5.6) and client-side plausibility checking behind it.

**I agree with the conclusion.** I want to be precise about *why*, because the reason determines where
to stop spending.

The cryptography does not make disclosure impossible. Security says so itself: R1 concedes that Partner
A can modify the client to capture `K_B` at the next unlock, and offers detection rather than
prevention. So the guarantee is not "he cannot ever obtain it". Someone reading only the invariant list
might conclude the machinery is therefore theatre.

It is not, and the reason is behavioural rather than cryptographic. Without client-side encryption, the
private balance is one `SELECT` away — available in thirty seconds, at eleven at night, in a bad mood,
leaving no trace and requiring no decision. With it, obtaining the same number requires deliberately
shipping sabotaged code to your wife's phone, against a signed manifest she can check, in a system that
gossips journal heads between the two clients. The design's real product is **raising the act from
curiosity to burglary**: making the easy path impossible and leaving only a path that is slow,
deliberate, morally unambiguous, and detectable. For a domestic threat model that is the correct
objective, and it is achievable, which "cryptographically impossible" is not.

So: not over-rotated. The one place I think the security document *under*-rotated is the topology it
implies, and correcting that makes the system smaller.

### 1.2 Where I disagree: the split is more expensive than going all the way

Security §1.5 draws the boundary so that the household node holds `K_H` and `K_S`, processes HOUSEHOLD
and CUSTODIAL server-side, and holds only ciphertext for perimeters `A` and `B`. §3.5 accordingly
specifies Postgres RLS with server-side aggregate functions such as `figure_networth(audience uuid[])`.
The reasoning given is that HOUSEHOLD is visible to both partners anyway, so server-side processing
there "costs nothing in confidentiality and buys the entire D1 pipeline".

It costs nothing in confidentiality. It costs a great deal in engineering, and the bill lands on the
part of the system that must not have bugs.

Security §5.5, §5.6 and §5.7 already require, for private perimeters: a client-side redactor, a
client-side four-eyes cross-check, and client-side plausibility checking against account history. That
last one is decisive. **Plausibility-against-history needs the ledger.** So a full ledger — postings,
balance derivation, reconciliation postings, materiality thresholds — must exist in the client
regardless. Under the split, a *second* full ledger and a *second* full extraction pipeline then have to
exist on the server for the shared perimeter.

Two implementations of double-entry balance derivation, two implementations of the reconciliation
algorithm in financial correctness §2.6, two implementations of the four-eyes commit rule, two
implementations of the FX decomposition in §3.5 — in a system whose entire value proposition is that
the numbers are right, maintained by one person at weekends. They will diverge. The divergence will be
subtle, it will be in the arithmetic, and it will be discovered by a household total that disagrees with
itself depending on which view produced it.

That is a worse outcome than any confidentiality property the split buys, and the split buys none.

### 1.3 The resolution: go further, and it gets simpler and cheaper

**AD1. The host is a synchronising encrypted store. All ledger computation, all extraction
orchestration, all figure derivation runs in the client, for every perimeter.**

This is the position `11-data-security.md` takes independently (its §0.6: "the entire ledger is
client-side encrypted… the server never needs plaintext to do its job, so it never gets it") and it is
the right one. The two security documents are not in conflict about the threat; they are in conflict
about the topology, and data security has the better answer.

It works here specifically because of how small this system is. §3.4 does the sizing: a decade of this
household's ledger is single-digit megabytes, and a full fold over it is tens of milliseconds. A
client-side ledger is not a heroic engineering feat at this scale; it is the *default* thing to build,
and only habit makes a server feel necessary.

What we give up, stated plainly:

| Lost | Consequence | Replacement |
|---|---|---|
| Server-side aggregates (security §3.5's SQL functions) | No figure exists server-side | The closure rule is enforced in the client's query layer and its type system — security §3.4 layer 3, which it already calls the layer that will actually catch the real bug |
| Server-side extraction for HOUSEHOLD documents | Extraction is slower on a phone | One pipeline instead of two; the client one had to exist anyway |
| Server-visible due dates for reminder scheduling (AR2) | The scheduler cannot see what is due | Clients register **opaque wake tokens** at day granularity with jitter (§6.5) |
| Server-side search | No full-text search on the host | Search runs over the in-memory ledger. At 26 accounts this is not a feature that needed a server |
| A server-side answer to "was this document already ingested" | — | Was already forbidden by S9 |

And what we gain, beyond removing a duplicate implementation:

- **No home server.** Data security §2 makes this argument and it is a good one: an always-on machine in
  Savoie is cost, patching, NAT traversal, and a failure mode when the household is in Australia for
  three weeks. A1 tolerates modest spend; it does not tolerate a second box to babysit.
- **Managed hosting becomes uncontroversial.** A2 wanted EU residency and the brief wants managed, low
  maintenance. Once the host holds only ciphertext, the choice of host stops being a confidentiality
  decision and becomes an availability-and-jurisdiction decision, which is a much easier one. The
  provider is no longer in the threat model as a reader — only as a party that can lose the ciphertext
  or refuse to serve it, and §9 handles both.
- **INV8 by construction rather than by policy.** Data security §2 is right that a server-side redactor
  must first *receive* the unredacted document, after which the invariant depends on that server never
  logging it, never caching it, and never core-dumping with the buffer live. If redaction runs before
  the first network write there is no code path along which an unredacted byte can leave.

### 1.4 What the topology makes structural

The strongest argument for AD1 is not cost. It is that a set of requirements security had to state as
*policies the server must be trusted to follow* become *facts about what the server physically holds*.

| Security requirement | Under the split | Under AD1 |
|---|---|---|
| S10 — escalations never route across a perimeter | The queue service is trusted to route correctly | The escalation exists only inside that perimeter's ciphertext op log. The server cannot route it anywhere because it cannot read it |
| L7 — staleness and reminder counts are perimeter-scoped | The scheduler is trusted to filter | The scheduler has no figures and no due dates. It delivers opaque wakes |
| L13 — push payloads carry no figures | The push sender is trusted not to include one | The push sender has no figure to include |
| L15 — an export by A never contains a row from B | The export query is trusted to close over the audience | A's client cannot decrypt B's rows |
| AR2 — no service role may read a private-perimeter figure | Per-service DB roles and per-service RLS policies | No service can read *any* figure, private or shared |
| S5 — no cardinality crossing a perimeter | Every query reviewed for `count(*)` | The server can count ciphertext rows per perimeter (see R9 and §9.6) but has nothing to put a count in front of a partner |
| §3.4 layer 3 — the wrong query does not compile | A property of the server codebase | A property of the only codebase there is |

Every one of these was a place where a correct design depended on nobody making a mistake in the one
component the adversary administers. That is the wrong place to put a dependency.

### 1.5 Disposition of AR1–AR6

Security's explicit requirements on this document, answered.

| # | Requirement | Disposition |
|---|---|---|
| **AR1** | App connects to Postgres as non-owner, non-superuser; RLS `FORCE`d; `app.principal` transaction-local, from the session only | **Kept, demoted to defence in depth.** RLS now guards the *envelope* — which perimeter's ciphertext a session may fetch. It is no longer the confidentiality boundary; key possession is. Kept because it costs nothing and stops an application bug from serving B's blobs to A, which would be harmless but is still a bug worth failing on. Mechanics in §3.6 |
| **AR2** | Per-service DB roles; no service reads a private figure; scheduler reads due dates only | **Exceeded.** No service reads any figure. The scheduler does not read due dates either — §6.5 replaces them with opaque wakes |
| **AR3** | The BFF holds all tokens; no token reaches the browser | **Kept, and narrowed.** PSD2 tokens live in the partner runner (§3.3), which is a *smaller* confidential client than the main API and, for a private perimeter, is not administered by the other partner |
| **AR4** | Operations plane not on the public interface | **Kept.** Managed Postgres on a Scaleway Private Network with public access disabled; no bastion, no SSH, no admin listener; §3.6 |
| **AR5** | The seam must accommodate two independent aggregator registrations, one per partner | **Kept and raised.** AD3 makes it two independent *deployments*, not merely two credential sets — §3.3. Registration separation alone does not help if both registrations are polled by a process A administers |
| **AR6** | Upload is constant-work, constant-response; no server-side matching; the blind inbox is a first-class component | **Kept in full**, specified in §5.3, with three gaps closed that §5.3 of the security document leaves open: size bucketing, quota accounting, and TTL sweeping (§5.4) |

---

## 2. System shape

### 2.1 One ingress shape (AD4)

Four of twenty-six lines have an API. Seventeen are structured exports. Five are manual or modelled. A
design in which "connected accounts" are the main path and everything else is a fallback fails at that
ratio, and it fails in a specific way: the fallback path gets less attention, the manual accounts feel
second-class, and INV5 becomes a sentence in a document rather than a property of the software.

The fix is one artefact.

```
        ┌─────────────┐   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
        │  Feed       │   │  Document   │   │  Manual     │   │  Model      │   │  External   │
        │  adapter    │   │  extraction │   │  entry      │   │  (mortgage, │   │  estimate   │
        │  4 lines    │   │  17 lines   │   │  any line   │   │  Livret A)  │   │  (DVF)      │
        └──────┬──────┘   └──────┬──────┘   └──────┬──────┘   └──────┬──────┘   └──────┬──────┘
               └─────────────────┴─────────┬───────┴─────────────────┴─────────────────┘
                                           ▼
                                   ┌───────────────┐
                                   │  Observation  │   value · ccy · asOf · cadence
                                   │   + Provenance│   source · derivation · docRef
                                   └───────┬───────┘
                                           ▼
                  idempotency (§5.6) → plausibility (§5.7) → commit → ledger (13-fc §2)
                                           ▼
                        postings · balance assertions · reconciliation postings
                                           ▼
                                   figures, closed over an audience
```

An `Observation` is the *only* way a number enters the ledger. There is no privileged path. A
Julius Baer balance that arrived as a PDF and a Revolut balance that arrived over PSD2 are the same
kind of object, differing in one field (`source`) and one number (`expectedCadenceDays`).

The consequences are exactly the ones INV5 wants:

- Turning a feed off does not remove an account from the system. It changes its cadence from 1 day to
  30 and its `source` from `feed` to `document`. Nothing downstream notices.
- The most-exercised code path in the system is the document path, because 17 lines use it. Under a
  connection-first design the most-exercised path would be the least-tested one.
- Presentation §13's "no connected chrome" rule is not a discipline the view layer has to maintain. The
  view layer has no field that would let it draw a plug icon in the first place.

### 2.2 The `Observation` envelope

Presentation §11.2 asked for this. Delivered verbatim as the view contract:

```ts
type Observation = {
  value: Minor | null;                  // null ⇒ Unknown. Never 0-as-unknown.
  ccy: Ccy;
  asOf: Date;
  expectedCadenceDays: number;          // per account line, stored, never inferred
  source: 'feed' | 'document' | 'computed' | 'manual';
  derivation: string;                   // human-readable, for the provenance drawer
  connection?: { state: 'live' | 'expiring' | 'expired'; expiresAt?: Date };
  docRef?: { filename: string; sha256: string; page?: number };
};
```

Three things architecture adds, because a contract that only the caller enforces is not a contract.

**The `expired ⇒ null` rule is enforced at construction, not at the boundary.** Presentation's
requirement is that it "forces `value: null` at source" so the view layer is never in the position of
choosing between a stale number and unknown. The mechanism:

```ts
// The ONLY exported constructor. The Observation type's fields are readonly and its
// constructor is private to this module; `as Observation` is banned by lint rule.
export function observe(input: {
  value: Minor; ccy: Ccy; asOf: Date; cadenceDays: number;
  source: Source; derivation: string; connection?: ConnectionState; docRef?: DocRef;
}): Observation {
  if (input.connection?.state === 'expired') {
    return { ...input, value: null, derivation:
      `${input.derivation} — connection expired ${fmt(input.connection.expiresAt)}; last known value withheld` };
  }
  return { ...input, value: input.value };
}
```

The withheld figure is not deleted. It remains a `BalanceAssertion` in the ledger with its own `as_of`
(financial correctness §2.3), which is what keeps history intact and charts honest. What expires is its
right to be presented as *current*. Financial correctness F10 calls this a step function to unknown
rather than a decay curve, and this is where the step is taken.

**`expectedCadenceDays` is a stored column on the account line, seeded per institution, editable.**
Presentation is right that this field carries the whole freshness design, so it must not be inferred
from observation history — an account that has been neglected for four months would infer a 120-day
cadence and then declare itself fresh. Defaults:

| Line | Cadence (days) | Why |
|---|---|---|
| CA joint current, Revolut ×2, Wise | 1 | Live feed, daily poll |
| CBA transaction, CBA cards ×2 | 31 | NetBank export cycle |
| Julius Baer current ×5 | 31 | Monthly statement |
| Julius Baer investment | 31 | Monthly portfolio statement; quarterly for some line items — take the shorter |
| TopCard ×4 | 31 | Monthly statement, pre-LSV |
| Bourse Direct PEA — positions | 31 | Monthly export |
| Bourse Direct PEA / Linxea — prices | 1 | Marked daily (D5), §8.3 |
| Livret A ×3 | 183 | Twice-yearly interest posting |
| Linxea assurance vie — units | 92 | Quarterly statement (authoritative on unit count and fees) |
| Pillar 2 | 366 | Annual certificate, Q1 |
| Hostplus | 92 | Quarterly member statement |
| Mortgage | 1 | Modelled daily from contract terms |
| The house | 366 | DVF, revalued annually, never month to month |

**Companion envelopes.** Presentation also asked for a rate set and an offline snapshot envelope:

```ts
type RateSet = {
  date: Date;                    // the publication date these rates are FOR
  version: string;               // ECB revisions create a new version (13-fc R6)
  source: 'ECB';
  pairs: Record<Ccy, Decimal>;   // units of X per 1 EUR, as ECB quotes them
  fetchedAt: Date;
};

type OfflineSnapshot = {
  renderedAt: Date;
  rateSetVersion: string;
  displayCcy: Ccy;
  audience: PrincipalId[];       // INV14 — a snapshot is a figure and declares its audience
  payload: RenderedView;         // opaque to the ledger; see AD12 / §10.2
};
```

### 2.3 The five producers, and their shape

| Producer | Lines | Emits | Runs | Cadence source |
|---|---|---|---|---|
| Feed adapter | 4 | O1 postings + O2 balances | Partner runner (§3.3) | Daily poll |
| Document extraction | 17 | O1 and/or O2, per document family | Client (§5) | Upload event |
| Manual entry | any | O2, occasionally O1 | Client | User |
| Model | mortgage, Livret A accrual | O4, `estimated = true` | Client, on ledger open | Daily recompute |
| External estimate | the house | O5, `estimated = true`, never auto-committed | Client, prompted | Annual |

The model and external-estimate producers deserve a note, because they are the two that a
connection-centric architecture forgets. Both are **pure functions of stored terms plus a date**, so
they are recomputed on every ledger load rather than stored. That is the same discipline financial
correctness applies to reconciliation postings (F3): derived artefacts are regenerated, never
persisted as user data. Storing a computed mortgage balance would mean that fixing a bug in the
amortisation schedule requires a data migration; recomputing means it requires a deploy.

---

## 3. Deployment topology

### 3.1 The shape

```
   Partner A's devices                                      Partner B's devices
   ┌────────────────────────┐                              ┌────────────────────────┐
   │ installed PWA          │                              │ installed PWA          │
   │  keys K_A, K_H, K_S    │                              │  keys K_B, K_H, K_S    │
   │  in-memory ledger      │  ← the system of record →    │  in-memory ledger      │
   │  redactor · extractor  │                              │  redactor · extractor  │
   │  four-eyes · plausib.  │                              │  four-eyes · plausib.  │
   │  share target (SW)     │                              │  share target (SW)     │
   └───────────┬────────────┘                              └───────────┬────────────┘
               │  TLS · passkey session · signed ops                   │
               └──────────────────────┬───────────────────────────────┘
                                      ▼
        ┌───────────────────────────────────────────────────────────────┐
        │  CORE  —  Scaleway, Paris (fr-par)   [administered by A]      │
        │  · sync API (Fastify, Serverless Container, scale-to-zero)    │
        │  · Postgres 17 (Managed DB, private network only)             │
        │      op log ciphertext · policy journal · inbox envelopes     │
        │      · rate table (PLAINTEXT — public data) · feed_binding    │
        │  · Object Storage: sealed document blobs, sealed raw payloads │
        │  · Serverless Cron: ECB rate fetch · universe price fetch     │
        │  · Web Push (VAPID) — opaque wake tokens only                 │
        │  HOLDS NO KEY THAT OPENS ANY FIGURE                           │
        └───────────────────────────────────────────────────────────────┘
                     ▲                                    ▲
      sealed to K_H  │                    sealed to K_A   │  sealed to K_B
        ┌────────────┴───────────┐            ┌───────────┴────────────┐
        │ RUNNER-A               │            │ RUNNER-B               │
        │ Scaleway project A     │            │ Scaleway project B     │
        │ · Enable Banking reg A │            │ · Enable Banking reg B │
        │ · PSD2 tokens (A)      │            │ · PSD2 tokens (B)      │
        │ · extraction relay     │            │ · extraction relay     │
        │   (model API key)      │            │   (model API key)      │
        └────────────┬───────────┘            └───────────┬────────────┘
                     ▼                                    ▼
              Enable Banking · model provider · (same, independently)
```

Three tiers, and the important property is which of them Partner A can read.

| Tier | Administered by | Sees plaintext | Failure = |
|---|---|---|---|
| Clients | each partner | everything in their own perimeters | that partner is offline |
| Core | Partner A | the ECB rate table, instrument prices, envelopes (§3.7), the policy journal's public bodies | sync stops; both clients keep working offline |
| Runner-A | Partner A | A's and HOUSEHOLD's feed payloads; redacted extraction payloads A originated | A's feeds go stale → `unknown` |
| Runner-B | Partner B | B's feed payloads; redacted extraction payloads B originated | B's feeds go stale → `unknown` |

### 3.2 Why there is no home server

D1 requires that personal data be redacted "on infrastructure we control" before document content
leaves it, and A2 notes that this implies *something* runs on controlled infrastructure. The previous
draft of this document read "controlled infrastructure" as a machine in the house. Data security §2 reads
it as **the uploading device**, and that reading is both stronger and cheaper:

- Stronger, because the redactor runs before the first network write. INV8 stops depending on a server
  not logging, not caching and not crashing with the plaintext in a buffer.
- Cheaper, because there is no second machine. No patching, no UPS, no dynamic DNS, no port forwarding,
  no "the house lost power while we were in Sydney".
- More available, because the house's internet connection is removed from the critical path of every
  action taken from a phone.

The mini-PC bought nothing that the phone in the uploader's hand does not already have, and it added a
component whose failure mode is invisible until someone needs it.

### 3.3 The partner runner (AD3)

Two jobs genuinely cannot run in a browser:

1. **Polling PSD2.** The tokens must not reach the client (AR3), the aggregator's API is not
   CORS-reachable from a browser, and the poll must happen daily whether or not anyone opens the app.
2. **Calling the extraction model.** The API key cannot be shipped to a browser.

Both of these touch plaintext — the feed poller sees full transaction narratives and IBANs; the relay
sees the redacted payload. Under a single shared deployment, both run on infrastructure Partner A
administers, and for Partner B's private accounts that reopens the hole S1 spent an entire document
closing. Security's S12 saw half of this and answered it at the level of credentials: each partner
registers their own aggregator application. That is necessary and insufficient — a separate registration
polled by a process A controls leaks exactly as much as a shared one.

**AD3: the runner is one artefact, deployed once per partner, in that partner's own cloud account.**

```
partner-runner/            single container image, reproducible build, published digest
  ├── poller/              FeedAdapter + Normaliser, per §4
  │     in:  PSD2 tokens (this partner's registration only)
  │     out: RawPayload sealed to the target perimeter's public key → Core object storage
  │          + a ConnectionHeartbeat op sealed to the same perimeter
  └── relay/               stateless extraction relay, per 11-data-security §2
        in:  redacted synthesised payload, sealed in transit to this runner
        out: model response, sealed back to the requesting client
        never logs a request or response body; emits counters only
```

Properties that make this work rather than merely sound good:

- **Sealing needs only a public key.** Each perimeter has an X25519 keypair whose public half is
  published in the policy journal and whose private half exists only on that perimeter's devices. The
  runner can encrypt *to* a perimeter it can never read. Plaintext exists only in runner memory, only
  for the duration of one poll.
- **Deployment is not administration.** A Scaleway Serverless Container on a cron trigger has no host to
  patch, no disk, no SSH, and scales to zero between polls. Partner B "operating" hers means holding the
  credentials for one Scaleway project and clicking redeploy when the published image digest changes.
  This is not a home server; it is closer to a browser extension with a schedule.
- **It fits inside the free tier.** Two polls a day of four accounts plus ten relay calls a month is
  comfortably inside Scaleway's monthly Serverless Containers allowance. The marginal cost of Runner-B
  is approximately zero.
- **Failure is isolated and visible.** If Runner-B stops, B's feeds go `expiring` then `expired` and
  render as unknown (§6). A's do not. Nothing silently blends.

**The rule this replaces.** Security S12 says an account whose feed runs through the other partner's
registration cannot be private. The generalised rule is:

> **A private account may have a live feed only if it is polled by a runner its owner controls.**
> Otherwise it is `HOUSEHOLD`, or it is private and document-driven.

Stored as `account.feed_transit_exposure ∈ { none | own_runner | other_partner_runner }`, surfaced at
account creation alongside the other constraints UX3 already requires, and journalled. Default:
`none` — a private account is created with feeds disabled and the owner must deliberately enable one.

**If a partner declines to run a runner**, which is a legitimate choice, that partner's private accounts
are document-and-manual. Given that 22 of 26 lines are document-and-manual anyway, this is a much
smaller loss than it sounds, and it is precisely the trade the household should be allowed to make
consciously rather than have made for them by a deployment diagram.

**Why this is not just R1 again.** One could argue that since R1 already concedes A can sabotage the
client to steal `K_B`, a poller A operates is below the accepted residual line. I disagree, and the
distinction matters: R1 requires an *active, deliberate, detectable* act of tampering against a signed
manifest. A shared poller sees B's transactions **passively, by design, continuously, forever, leaving
no trace and requiring no decision.** Passive-by-design exposure is categorically worse than
active-tampering exposure, and a design that collapses the two has stopped taking its own threat model
seriously. The runner costs one extra deployment and closes it.

### 3.4 Sizing — why a client-side ledger is not heroic

The claim in §1.3 needs numbers, because it is the load-bearing claim.

| Quantity | Estimate | Basis |
|---|---|---|
| Transactions/month | ≈ 950 | 4 live accounts ≈ 600 · 6 cards ≈ 240 · 5 JB current ≈ 75 · CBA ≈ 60 |
| Postings/month | ≈ 2,400 | double entry, ~2.5 postings per transaction |
| Postings/decade | ≈ 290,000 | |
| Serialised, plaintext | ≈ 45 MB | ~150 B/posting |
| After compression + AEAD | ≈ 10–12 MB | ledger data compresses well |
| Cold-start fold (§2.4 of 13-fc) | 20–60 ms | one pass, integer addition, no allocation in the hot loop |
| Documents | ≈ 12/month × 250 KB | ≈ 3 MB/month → ≈ 360 MB/decade |
| Postgres op-log rows | ≈ 290k × ~400 B | ≈ 120 MB/decade, inside a 10 GB DEV-S instance |

The client holds the **whole ledger** and **recent documents**; older document blobs are fetched from
object storage on demand and cached with an LRU budget. Cold start after unlock is a decrypt-and-fold
of ~12 MB, which is where presentation §12's 200–400 ms concern comes from and why AD12 admits a
display snapshot.

The mobile constraint that actually binds is not the ledger. It is data security's ~135 MB quantised NER
model for redaction detector layer (c). Budget:

| Asset | Size | When |
|---|---|---|
| App shell (Svelte + views) | < 400 KB gz | install |
| `pdf.js` | ≈ 1 MB | first document |
| Redactor WASM (regex, checksums, layout) | ≈ 2 MB | first document |
| Template parsers (10 families) | ≈ 200 KB | first document |
| NER model, layer (c) | ≈ 135 MB | **opt-in, desktop-first, cached in Cache Storage** |

Layer (c) is a refinement, not a dependency (data security §3.5). Architecture's ruling: **it is
downloaded on explicit consent, on desktop by default, and an account whose documents require layer (c)
to redact safely is flagged desktop-only for ingestion.** Android is not blocked from ingesting; it is
blocked from ingesting the families that need layer (c), which per data security's own open question 7
should be measured on the household's actual phone before this is fixed.

### 3.5 The stack, and A1

| Concern | Choice | Why this and not the obvious alternative |
|---|---|---|
| Client | **Svelte 5 + Vite**, TypeScript strict | Compiles templates away, so `require-trusted-types-for 'script'` and `script-src 'self'` (security §7.5) hold without fighting the framework. No virtual DOM means the bundle budget goes to `pdf.js` and the redactor instead of to the framework. React would work; it costs ~40 KB more and more CSP friction |
| PWA | `vite-plugin-pwa` (Workbox) | The share target must be handled *inside* the service worker (§7). Workbox has a first-class recipe for exactly this |
| Client crypto | WebCrypto (AES-256-GCM, HKDF, ECDSA P-256) + `@noble/curves`, `@noble/ciphers` for X25519 | WebCrypto lacks X25519 uniformly; noble is small, audited, dependency-free, and matches security §7.5's bias toward fewer dependencies |
| Client ledger | **In-memory, TypeScript**; IndexedDB stores only ciphertext chunks | SQLite-WASM over OPFS is tempting and wrong here: OPFS would hold *plaintext* at rest, contradicting security §7.2's "lock drops the decrypted snapshot". In-memory makes zeroise-on-lock trivially true, and §3.4 shows the fold is fast enough |
| Server | **Fastify on Node 22**, TypeScript | One language across client, runner and server. The `Observation` envelope, the adapter DTOs and the op schema are *shared source*, not two hand-maintained copies. For a household maintained at weekends this matters more than Go's smaller binary |
| Database | **Postgres 17**, Scaleway Managed DB (fr-par) | RLS for AR1; managed removes the maintenance that actually hurts (major upgrades, PITR, backups) |
| Blobs | Scaleway Object Storage (S3 API), versioning + object lock on the document bucket | Ciphertext only; object lock defends against a ransomware-style delete |
| Compute | Scaleway Serverless Containers, scale to zero | Nothing runs between requests. No instance to patch |
| Schedules | Scaleway Serverless Jobs / cron triggers | ECB fetch, universe price fetch, inbox TTL sweep, wake dispatch |
| Auth | `@simplewebauthn/server`, `__Host-` cookie, session binding | Security §6, implemented not redesigned |
| Region | **fr-par** | A2. French company, French jurisdiction, no CLOUD Act exposure. Hetzner (DE) is the equally good alternative on jurisdiction |
| CI | GitHub Actions → reproducible container build, published image digest; PWA build produces a **signed release manifest** (data security §4.4, security R1) | The manifest hash is what makes R1 detectable rather than merely regrettable |

**Cost, against A1.**

| Line | €/month |
|---|---|
| Managed Postgres DEV-S (1 vCPU / 2 GB / 10 GB, automated backups) | ≈ 11 |
| Object Storage, < 5 GB | ≈ 0.1 |
| Serverless Containers, sync API + 2 runners | ≈ 0–2 (free-tier allowance) |
| Serverless Jobs (cron) | ≈ 0 |
| Domain | ≈ 1 |
| Extraction model calls, ≈ 12 documents/month | ≈ 1–3 |
| **Total** | **≈ €13–17/month** |

Below a streaming subscription; A1 satisfied with headroom. Two items to watch:

- **Managed Postgres is the only real line item**, and DEV-S is explicitly a non-HA development
  instance. That is the right call here: R10 and security §1.1 both rank availability as the least
  valuable property, both clients hold a full replica, and paying €80/month for HA on a two-person
  household's rendezvous point would be the one place this design failed A1's spirit. If the DB is
  down, both partners keep working offline and sync catches up.
- **Market data (D5)** is the line item that can escape. §8.3 keeps it near zero by design.

**The honest alternative.** A single Hetzner CX22 in Falkenstein with Docker Compose, Caddy, Postgres in
a container and `restic` to a Storage Box costs ≈ €7/month and is entirely defensible now that the
server is this dumb. The difference is roughly an hour a month of patching and one nervous evening per
Postgres major version. I recommend Scaleway because the brief asked for managed and low maintenance,
and because €6/month is a fair price for never doing a `pg_upgrade` at 11pm. If the household would
rather own the box, nothing else in this document changes — the topology is identical, and that is the
point of AD1.

### 3.6 Envelope, RLS and the operations plane

The server-visible schema is deliberately tiny. Everything in it must be safe for Partner A to read
about Partner B (data security DS2, envelope minimisation).

```sql
-- Server-visible. Nothing here may be sensitive.
create table op (
  seq            bigserial primary key,          -- server-assigned, monotonic
  op_id          uuid not null unique,           -- client-generated
  perimeter      perimeter not null,             -- routing + RLS
  author_device  uuid not null references device(id),
  hlc            bigint not null,                -- hybrid logical clock, for ordering
  size_bucket    smallint not null,              -- padded; see §5.4
  ciphertext     bytea not null,                 -- AEAD under K_perimeter
                                                 -- AAD = (op_id, perimeter, author_device, hlc)
  device_sig     bytea not null,                 -- over the AAD ‖ H(ciphertext)
  created_at     timestamptz not null default now()
);
create index on op (perimeter, seq);

alter table op enable row level security;
alter table op force  row level security;
create policy op_visible on op for select using (
  perimeter = any (visible_perimeters(current_setting('app.principal')::uuid))
);
create policy op_insert on op for insert with check (
  perimeter = any (visible_perimeters(current_setting('app.principal')::uuid))
  and author_device in (select id from device where principal = current_setting('app.principal')::uuid)
);
```

Note what is *absent* and deliberately so: no `account_id`, no `as_of`, no `institution_id`, no amount,
no document type, no state flag. Security §3.5 put `account_id` on every figure-bearing table because
RLS needed it to find the perimeter. Here the perimeter is on the op directly, so the account
identifier — which is a correlatable handle across time — never has to be server-visible at all. This is
a genuine strengthening of DS2 that falls out of the topology.

Retained plaintext tables, and the justification for each:

| Table | Plaintext because |
|---|---|
| `rate` (ECB reference rates, versioned per 13-fc R6) | Public data. Shared by both perimeters. Caching it server-side saves every client a fetch |
| `instrument_price` (fixed universe, §8.3) | Public data, and by construction not household-specific |
| `journal_entry` (`public_body` only) | Security §3.7 requires both partners to read every policy event, and head gossip requires the chain be verifiable without a perimeter key |
| `device`, `principal`, `perimeter_membership` | Needed to authenticate and to enforce RLS |
| `feed_binding` (provider, provider_account_id, perimeter, **not** account_id) | The runner needs it to know where to seal a payload. §4.4 |
| `wake` (principal, deliver_after, opaque token) | §6.5 |

**Operations plane (AR4).** Managed Postgres has public access disabled and lives on a Scaleway Private
Network reachable only by the sync API container. There is no bastion and no SSH — with serverless
compute there is no host to SSH to. Migrations run from CI as a separate role that the running
application never uses; the application role owns nothing and is not superuser. Backup restore is a
console operation, journalled by its effects (a restored op log with a lower head is a fork, which §9.5
detects).

### 3.7 What Partner A can still infer from the Core

Honest accounting, extending security's R9.

Visible to the operator: the number of ops per perimeter, their timing and their padded size buckets;
which perimeters exist; when each device syncs; when an inbox item is created and when it changes
state; the volume of document blobs and their padded sizes; every policy-journal public body (which is
by design).

That is a **traffic and activity profile**, not a figure. It supports inferences like "B ingested
something on the 3rd of every month" — which, combined with knowing she banks somewhere with monthly
statements, is weak but non-zero. Mitigations, in ascending cost:

1. **Padding** (already required by S9's constant-work rule): all op ciphertexts and all document blobs
   are padded to power-of-two buckets before write. Cheap, mandatory, in §5.4.
2. **Day-granularity timing**: clients batch op pushes to a randomised time within a jittered window
   rather than pushing on every keystroke. Cheap, mandatory.
3. **Cover traffic**: each client emits a fixed budget of no-op ops per sync regardless of activity, so
   op counts stop tracking activity. Costs storage and complexity. **Offered, not mandated** — I would
   ship without it and add it only if the household asks, because it is the first control in this
   document whose benefit is hard to explain to the people paying for it.

---

## 4. The aggregator adapter boundary

### 4.1 The premise: it will be discontinued

This is not a hypothetical. GoCardless Bank Account Data — the rebranded Nordigen free tier, and the
obvious choice when this project started — closed to new signups mid-project. Free open-banking tiers
are contracting industry-wide. Enable Banking's restricted-production mode is the current route: you
operate under their AISP licence, whitelisted to accounts you personally authenticate to, JWT/RS256
auth with your application id as the `kid`, and endpoints for sessions, accounts, balances and
transactions.

It is the right choice today and it is a component with a shelf life. So the seam is designed on the
assumption that in year three we will be porting to something whose DTOs we cannot currently see, and
possibly whose *concepts* differ.

### 4.2 Two layers, not one

The mistake to avoid is a single adapter that returns ledger objects. It looks tidy and it fuses the two
things that change at different rates: the provider's wire format (changes when the provider changes)
and the ledger's semantics (must never change).

```
┌───────────────────┐    RawPayload     ┌───────────────────┐   Observation[]   ┌──────────┐
│  FeedAdapter      │ ────────────────► │  Normaliser       │ ────────────────► │  Ledger  │
│  provider-shaped  │  stored, sealed,  │  provider-shaped  │  provider-free    │          │
│  network + auth   │  kept forever     │  pure function    │                   │          │
└───────────────────┘                   └───────────────────┘                   └──────────┘
        runs in the partner runner            runs in the CLIENT
```

The `FeedAdapter` port — deliberately the *union* of what AISPs offer, not the intersection of what one
offers, so that a richer successor is not squeezed into a poorer predecessor's shape:

```ts
interface FeedAdapter {
  readonly provider: ProviderId;            // 'enablebanking' | 'gocardless' | 'tink' | ...
  readonly capabilities: AdapterCapabilities;

  listInstitutions(country: ISO3166_1): Promise<InstitutionRef[]>;

  // Linking is a two-phase flow because every PSD2 provider redirects to the bank for SCA.
  beginLink(req: { institution: InstitutionRef; redirectUri: URL; psuContext?: PsuContext })
    : Promise<{ authUrl: URL; linkId: LinkId; expiresAt: Date }>;
  completeLink(linkId: LinkId, callback: Record<string, string>): Promise<LinkHandle>;

  listAccounts(link: LinkHandle): Promise<RawAccountRef[]>;
  fetchBalances(link: LinkHandle, acct: RawAccountRef): Promise<RawPayload>;
  fetchTransactions(link: LinkHandle, acct: RawAccountRef, window: DateWindow, cursor?: string)
    : Promise<{ payload: RawPayload; nextCursor?: string }>;

  linkState(link: LinkHandle): Promise<ConnectionState>;
  unlink(link: LinkHandle): Promise<void>;
}

type AdapterCapabilities = {
  supportsBalances: boolean;
  supportsTransactions: boolean;
  supportsPendingTransactions: boolean;    // matters for card L3 (13-fc §4.4)
  maxHistoryDays: number;
  consentValidityDays: number;             // 180 under current PSD2 RTS
  requiresSCAPerFetch: boolean;
  paginationStyle: 'cursor' | 'dateWindow' | 'none';
  sandboxAvailable: boolean;
};
```

`RawPayload` is the provider's response, verbatim, plus a small header:

```ts
type RawPayload = {
  provider: ProviderId;
  adapterVersion: string;      // semver of the adapter that fetched it
  fetchedAt: Date;
  requestDescriptor: string;   // endpoint + parameters, for reproducibility. NEVER a token
  body: Uint8Array;            // the provider's bytes, untouched
  sha256: Hex;
};
```

### 4.3 The rule that makes the seam survivable

**AD5: every raw payload is stored, sealed to the target perimeter, and kept.**

This is the single most important resilience property in this section and it is the one most often
skipped, because at the moment you write the adapter it looks like storing the same data twice.

What it buys:

- **A normaliser bug is repairable.** Discover in month nine that you have been reading Revolut's
  `bookingDate` where you wanted `valueDate` — which matters for financial correctness's Livret A
  *quinzaine* handling and for month-boundary attribution — and the fix is: bump the normaliser, replay
  every stored payload, regenerate. Without stored payloads the fix is: hope the provider still serves
  history that far back. Enable Banking's restricted production, and PSD2 generally, will not.
- **A provider death is survivable.** When Enable Banking goes the way of Nordigen, the account's
  *history* does not live in Enable Banking's shape any more — it lives in the ledger, and the raw
  payloads that produced it are still on disk for audit. The migration is forward-only.
- **INV3 gets its strongest possible form.** Provenance for a feed-sourced figure is not "Revolut, 3
  March" but a content hash resolving to the exact bytes the provider returned. Financial correctness's
  `Provenance.feed_connection` points at the payload, not at the connection.
- **The four-eyes analogue exists for feeds.** A re-normalisation that produces different postings from
  the same bytes is a detectable defect. Documents get a cross-check by design (D1); feeds get one for
  free from this rule.

Storage cost: four accounts × daily poll × ~40 KB = ~6 MB/month, ~700 MB/decade. Roughly €0.01/month.
There is no argument against it.

### 4.4 What must never cross the seam

**AD6.** These rules are enforced by the type system — `RawAccountRef` and `LinkHandle` are opaque
branded types that the ledger package does not import, and a dependency-cruiser rule fails the build if
it ever does.

| Never in the ledger | Where it lives instead |
|---|---|
| Provider account id | `feed_binding(provider, provider_account_id) → perimeter + sealed account ref` |
| Provider transaction id | Normalised into `Transaction.natural_key` (13-fc §5.3) alongside date, amount, counterparty — so the key survives a provider change |
| Consent / link id | `feed_binding` |
| Access or refresh token | Runner secret store only (AR3, DS4) |
| Institution id in the provider's namespace | Mapped to our own `institution` row at binding time |

The consequence worth spelling out: **a provider migration is a re-link, not a re-import.** The household
authenticates to the new provider, the new bindings point at the same accounts, and the ledger does not
change. Transactions already ingested are not re-ingested because `natural_key` matching (financial
correctness F9) is provider-independent by construction. That is the whole reason `natural_key` must not
be the provider's id, and it is the mistake that makes most aggregator migrations into data migrations.

### 4.5 Adapters that exist on day one

| Adapter | Status | Notes |
|---|---|---|
| `EnableBankingAdapter` | Primary | Restricted production. Confirm at signup: that a natural person may register **two** applications (one per partner, S12/AD3); the exact consent validity; whether Crédit Agricole des Savoie exposes Livret A alongside the payment account |
| `NullFeedAdapter` | **Ships in v1** | Every method returns `expired`. Used by the §9.2 drill and by CI. Not a test double — a supported production configuration |
| `ManualLinkAdapter` | Ships in v1 | For a "feed" that is a human on a schedule. Emits only `ConnectionState` and a cadence, so an account can carry a re-link-style reminder without any provider at all |
| `GoCardlessAdapter` | Not built | Written down only to record that the port shape was checked against a second provider's concepts before being frozen |

The `NullFeedAdapter` deserves the emphasis. Shipping it in v1, and running the whole suite against it,
is what stops "usable with zero feeds" from decaying. It is INV5 as an executable artefact.

### 4.6 The discontinuation drill

Run once before phase 5 ships, then annually. Not a document — a checklist someone actually performs.

1. Switch the runner's adapter to `NullFeedAdapter` for all four lines. Confirm: no crash, four accounts
   go `expiring` → `expired` on schedule, four `Observation`s carry `value: null`, the four views render,
   the coverage bar grows four hollow segments, no total is wrong, no total is silently smaller.
2. Re-point one account at `ManualLinkAdapter`. Confirm it behaves exactly like the seventeen tier-B
   lines — same review flow, same cadence badge, same provenance drawer.
3. Replay all stored raw payloads through the current normaliser into a scratch ledger. Confirm the
   result is byte-identical to the live ledger's postings for those accounts. **This is the test that
   proves AD5 works**, and if it fails the seam is already broken.
4. Write down the time it took. If step 3 takes longer than an evening, the seam has acquired a
   dependency it should not have.

---

## 5. The ingestion pipeline

Seventeen of twenty-six lines arrive this way, plus five more by hand. This is the main path.

### 5.1 End to end

```
 ┌──── either partner's device (D2) ────────────────────────────────────────────────┐
 │                                                                                  │
 │  1  ACQUIRE      file picker · drag-drop · Android share target (§7) · paste     │
 │        │         nothing has left the device; nothing has been decrypted yet     │
 │        ▼                                                                         │
 │  2  SEAL         H = SHA-256(plaintext)                                          │
 │                  K_doc ← random; blob ← AEAD(K_doc, plaintext); pad to bucket    │
 │                  wrap K_doc to BOTH partners' perimeter public keys              │
 │        │         ── this step needs NO private key: works while LOCKED ──        │
 │        ▼                                                                         │
 │  3  POST /inbox  { ciphertext, wrap_A, wrap_B, size_bucket }                     │
 │        │              server answers 201 { inbox_id, status:"received" }         │
 │        │              ALWAYS. Byte-identical. No verdict of any kind. (S9)       │
 │        ▼                                                                         │
 │  4  ROUTE        [requires unlock] client opens it locally, classifies it,       │
 │        │         offers accounts in (own perimeter ∪ HOUSEHOLD) + "not mine"     │
 │        │              ├─ routed    → step 5                                      │
 │        │              └─ not mine  → dual-sealed in the blind inbox; §5.5        │
 │        ▼                                                                         │
 │  5  IDEMPOTENCY  idem = HMAC(K_perimeter, H); then period key; §5.6              │
 │        │              already present → stop, tell THIS user, tell nobody else   │
 │        ▼                                                                         │
 │  6  REDACT       client-side, deny-by-default synthesis (11-ds §3.6)             │
 │        │         out: a synthesised payload, never the original with boxes on it │
 │        ▼                                                                         │
 │  7  EXTRACT ×2   pass 1: model, via THIS partner's runner relay (§3.3)           │
 │        │         pass 2: local deterministic template parse                      │
 │        ▼         + arithmetic closure over the document's own totals             │
 │  8  CROSS-CHECK  agree? · closure holds? · confidence? · materiality?            │
 │        │              └─ any failure → ESCALATE, in-perimeter only (S10)         │
 │        ▼                                                                         │
 │  9  PLAUSIBILITY against this account's own history, in the ledger, locally      │
 │        │              └─ implausible → ESCALATE                                  │
 │        ▼                                                                         │
 │ 10  COMMIT       Observations + Provenance, one transaction, INV3 enforced       │
 │        │                                                                         │
 │ 11  SYNC         ops sealed to the perimeter → Core → the other device           │
 └──────────────────────────────────────────────────────────────────────────────────┘
```

Steps 4 through 10 run entirely on the device. The server's involvement is steps 3 and 11: accept an
opaque blob, hand back a constant, and later hand out ciphertext to a device that can open it.

### 5.2 Why the constant response costs nothing in usability

Security S9 requires a byte-identical upload response with no dedup, match or format verdict. Read
naively this sounds like the user uploads a statement and is told nothing, which would be a miserable
experience for the 17 lines that depend on it.

**It is not, and the reason is AD1.** The constant response is a property of the *server's* API. All
the feedback the uploader deserves is produced by the *client*, locally, in steps 4–10, about documents
that uploader can already see. Concretely, for a TopCard statement Partner A uploads to a HOUSEHOLD
account, A sees: "Recognised: TopCard CHF, period 26 Jul – 25 Aug. Already ingested" or "Extracted 34
transactions, closing balance CHF 1,204.55 — confirmed by both passes". Immediately, offline, in full.

What A does *not* see is anything about documents he cannot route — and for those he genuinely knows
nothing, because his own device cannot classify them either.

So S9's cost is not a loss of feedback. It is a loss of feedback **about other people's accounts**,
which is the entire point. This is worth stating explicitly because a reviewer reading S9 alone would
reasonably conclude the ingestion UX had been sacrificed, and it has not.

### 5.3 The blind inbox (AR6)

```
POST /inbox
  Content-Type: application/octet-stream
  Body: { ciphertext[padded], wrap_A, wrap_B, size_bucket }

  201 { "inbox_id": "<uuid>", "status": "received" }      ← the only possible response
```

Invariants the server implementation must hold, each stated as something to test rather than intend:

| # | Invariant | Test |
|---|---|---|
| I1 | The handler performs identical work for every request: verify session, verify device signature, check quota, write blob, write row, respond | Code review plus a timing histogram: p50/p95/p99 must be indistinguishable across 1,000 uploads of distinct, duplicate, well-formed and malformed content |
| I2 | No branch inspects ciphertext content | The handler has no access to any key. Structurally true |
| I3 | Errors are indistinguishable | The only non-201 responses are `401` (no session) and `429` (quota). Malformed bodies, oversize bodies and duplicate content all return `201`; the client discovers the problem locally |
| I4 | Latency floor | Every response is held to a fixed floor of 250 ms (padded, not slept-then-worked, to avoid leaking via variance above the floor) |
| I5 | Blob-level dedup by content hash is permitted (DS5) and never observable | Dedup happens in object storage on the *ciphertext* hash. Two partners sealing the same plaintext produce different ciphertext, so it will rarely fire — which is fine; it is an optimisation, not a feature |

### 5.4 Three gaps in S9 that architecture has to close

Security §5.3 specifies the protocol but leaves three implementation questions that are each an oracle
if answered carelessly.

**Size.** Ciphertext length is a strong fingerprint: a 40-page Julius Baer portfolio statement and a
2-page Livret A statement are not confusable. Rule: pad every blob to the next bucket in
`{64 KB, 256 KB, 1 MB, 4 MB, 16 MB}` before sealing, and store `size_bucket` rather than a length.
Uploads over 16 MB are rejected client-side with local guidance; the server never sees them.

**Quota.** A shared storage quota is a counter that moves when the other partner ingests (security
§5.2 channel 6). Rule: quotas are **per principal**, never household-wide, and a `429` reports only
*your own* remaining budget. Budget is generous — 200 uploads and 2 GB per principal per month — because
its purpose is to bound a compromised device (§7.4 exfiltration limiting), not to manage cost.

**TTL and sweep.** Security §5.3 step 5 says unrouted items disappear from the uploader's view after 30
days unconditionally. Two details it does not settle:

- The sweep must be **unconditional and scheduled**, never triggered by a claim. A sweep that ran early
  because something was claimed is a claim notification. Implementation: a nightly cron deletes
  `wrap_uploader` for every inbox row older than 30 days, in one statement, regardless of state.
- Claiming (security §5.3 step 4) deletes the *other* partner's wrap. That is a row update the operator
  can see. Under R9 this is an activity signal, not a figure, and the mitigations are §3.7's. The
  cheapest real improvement: the client performs its claim as part of a batched sync containing a fixed
  number of op writes, so a claim is not a solitary write at a distinctive time.

### 5.5 Documents that cross a perimeter, and documents that straddle one

**Cross-perimeter (the normal case).** A uploads B's Revolut statement — perhaps it arrived in the
household email. He cannot classify it; his client offers "not mine / not sure"; it stays dual-sealed
in the inbox. B's client lists unrouted items, opens them locally, and may claim one. On claim, B
re-wraps `K_doc` to `K_B` alone and deletes `wrap_A`. From that moment A cannot open a document he
himself uploaded. A's view says "handed to the household" for 30 days and then says nothing, whether
it was claimed, ignored or discarded.

**Straddling (S11, the hard case).** A consolidated Julius Baer statement covering five current
accounts across both partners is one document containing lines from more than one perimeter. Security's
rule — a document takes the sensitivity of its most-restricted line — is right, and its consequence is
that **an institution issuing only consolidated statements cannot host a private account**.

Architecture's addition is where the split happens. Splitting must occur on the owner's device *before*
anything is sealed to a wider audience, which means step 2 of §5.1 has a branch: if the client
recognises a document family as `multi_perimeter_capable` (a property of the template, known before
extraction), it seals to the **uploader's perimeter only** and never dual-wraps. The document then
cannot be handed to the household at all, and the uploader is told so at upload time — locally, which
is permitted, because it is a fact about a template rather than about anyone's accounts.

This means the household must answer security's open question 4 (is Julius Baer statemented per account
or consolidated?) before phase 3, because it determines a template flag that changes the upload path.

### 5.6 Idempotency (INV7 + S8)

INV7 says: key on institution, account, period and document hash. S8 says the bare hash is a membership
oracle and the key must be a perimeter-scoped HMAC. Financial correctness F9 says: content hash, then
logical period key, then transaction-level multiset reconciliation. All three are satisfiable together,
at three layers.

**Layer 1 — document identity.**
```
idem_doc = HMAC-SHA256(K_perimeter, "doc:" ‖ H)        where H = SHA-256(plaintext)
```
Held in a unique index the client maintains inside the perimeter's encrypted state. Not server-visible.
A re-upload into the same perimeter deduplicates exactly as INV7 requires. Partner A cannot compute
`HMAC(K_B, ·)` for any `H`, so he cannot test membership — the oracle is closed by not possessing a key
rather than by the server declining to answer.

**Layer 2 — logical period.**
```
idem_period = HMAC-SHA256(K_perimeter, "period:" ‖ institution_id ‖ account_id ‖ period_start ‖ period_end)
```
Catches the same statement re-issued with a different byte layout — a corrected PDF, a re-download, a
different export encoding. Financial correctness F9 is explicit that periods are **replaced wholesale,
never merged**: a second document for the same period supersedes the first, its postings are withdrawn,
and the reconciliation postings for the interval are regenerated (F3). The superseded document is
retained with `superseded_by` provenance so INV3 still resolves.

**Layer 3 — transaction multiset.** Financial correctness owns this and it is the one that catches the
genuinely hard case: an overlapping CBA export, or a transaction that arrives both by feed and on the
following statement. Matching is by `natural_key` as a **multiset**, not a set, so two identical €4.20
coffees on the same day survive as two. Architecture's only contribution is where it runs — in the
client, over the ledger, which is the only place the ledger exists.

**Erasure and the hash tail.** Data security §6 keeps a permanent hash tail so Article 17 erasure and
INV7 can coexist. The tail is `idem_doc` and `idem_period`, not `H`: after erasure the perimeter still
recognises a re-upload of the erased document, and the surviving artefact is a keyed MAC that discloses
nothing to anyone without the perimeter key — including to the person running the database.

**Feeds, briefly.** The same three layers apply: `idem_doc` over the `RawPayload` hash, no period key
(feeds are windows, not periods), and the same multiset reconciliation. A repeated poll returning
overlapping days is the normal case, not an error, and must be silent.

### 5.7 Four-eyes, cross-check and escalation

D1 asks for two independent extraction passes that must agree, with disagreement, low confidence or a
materiality breach escalating to a human, and forbids silent auto-commit at any confidence level. Data
security §3.7 argues — persuasively — that two samples from the same model are not independent, and
proposes instead: **one model pass, one deterministic local template parse, and mandatory arithmetic
closure.** Architecture adopts that reading and owns the orchestration.

**The four gates.** All four must pass for auto-commit. Any failure escalates.

| Gate | Check | Failure means |
|---|---|---|
| G1 Agreement | Every extracted field matches between the model pass and the template parse, exactly for identifiers and dates, to the minor unit for amounts | The document has drifted from its template, or the model hallucinated |
| G2 Closure | The document's own arithmetic holds: opening + Σ movements = closing; Σ statement lines = statement total; card L1 + L2 = billed | The extraction dropped or duplicated a line — the failure that a confidence score will not catch |
| G3 Confidence | Both passes above threshold; no field marked ambiguous by the redactor | Low-quality scan, unusual layout |
| G4 Plausibility | Against this account's own history: is this delta possible? (thresholds are financial correctness's — data security §10 asks them for exactly this) | Correct extraction of the wrong document, or a real event that deserves a human's eye |

**Document families with no arithmetic to close.** Data security flags Pillar 2, Hostplus and Linxea as
having no closure property. Rule: **a family with no G2 never auto-commits.** It always escalates to a
confirmation, which for an annual document is the right cost — one confirmation a year for a Pillar 2
certificate is not a burden, and these are precisely the documents whose figures are largest and least
frequently checked.

**Escalation routing is structural, not policed.** An escalation is an op sealed to the perimeter that
owns the account. It is in the perimeter's encrypted op log. There is no queue service that could route
it wrongly, no admin view that could enumerate it, and no timeout that widens its audience. Security S10
requires that "an unresolved escalation is a correct state, not an incident"; under AD1 the system is
incapable of treating it as anything else, because no component outside the perimeter knows it exists.

**Extraction state machine.** Every transition is an op; the state is the fold of the ops.

```
received ─routed→ redacted ─relay→ extracted ─parse→ crosschecked ─┬─pass→ plausible ─commit→ committed
    │                                                              │
    └─unrouted→ (blind inbox, TTL 30d) ──claim→ routed             └─fail(G1..G4)→ escalated(reason)
                                                                             │
                                                        ┌────────────────────┼──────────────┐
                                                    resolved(edits)     abandoned      (stays open forever)
                                                        │
                                                        └─→ committed, provenance.confirmed_by = HUMAN
```

**Provenance (INV3).** Committed in the same op as the figure, never after. The `Provenance` record is
financial correctness §2.3's, with architecture supplying `document_id` (content-addressed),
`page`, `region` (so the review UI can show the figure beside the pixels it came from — a client-side
render of a locally-decrypted blob), `model_version`, `adapterVersion` for feed-sourced figures, and
`confirmed_by ∈ {HUMAN(user), FOUR-EYES, AUTO}`. There is no code path that constructs a posting
without one: the posting constructor takes a `Provenance` as a required argument and the type has no
default.

### 5.8 Manual entry and the models

Manual entry is not a lesser path — it is the path for 5 lines permanently and for all 26 whenever
anything else is broken. It emits the same `Observation` with `source: 'manual'` and
`confirmed_by: HUMAN`. The only difference from an extracted figure is that G1–G4 do not run, because
there is nothing to cross-check a human against.

The two models — mortgage amortisation and Livret A interest accrual — are pure functions of stored
contract terms and a date, recomputed on every ledger load, never stored (§2.3). Their `Observation`s
carry `source: 'computed'`, `estimated: true`, and a `derivation` string naming the contract terms used.
When the annual statement arrives, the reconciliation posting against `RECON:MODEL-DRIFT` (financial
correctness §2.6) measures how wrong the model was, which is a genuinely useful number: persistent
drift means the contract terms on file are wrong.

---

## 6. Connections that expire (INV4)

### 6.1 The problem is not expiry, it is silence

PSD2 access requires the AISP to apply SCA when the user accesses retrieved account data at least every
180 days; security §6.4 traces this to the RTS as amended and the PSD3/PSR framework. Operating under
an aggregator's licence means inheriting the obligation. The link will therefore expire, twice a year,
per connection, and the household will not be told by anyone.

But scheduled expiry is the *easy* failure. The dangerous ones are silent:

| Failure | Signal available |
|---|---|
| Consent reaches 180 days | `expiresAt`, known in advance |
| User revoked consent in the bank's app | None until a fetch fails |
| Bank revoked it (fraud rules, password change, card reissue) | None until a fetch fails |
| ASPSP outage | Fetch fails, transiently |
| Aggregator outage, or aggregator discontinued | Fetch fails, permanently |
| **The runner itself stopped** | **None at all — no fetch, no failure, no news** |

The last row is the one that kills trust, and it is the one a design based on a server-set `is_stale`
flag gets wrong: if the component that would set the flag is the component that died, the flag never
changes and yesterday's balance quietly becomes last quarter's.

### 6.2 The fail-closed clock (AD9)

**Freshness is computed by the client from evidence of success, never read from a flag.**

```ts
function connectionState(a: AccountLine, now: Date): ConnectionState {
  const lastOk = a.lastSuccessfulPoll;              // an op, sealed, written by the runner
  const consentEnds = a.consentExpiresAt;
  const grace = 3 * a.expectedCadenceDays;          // tolerate a weekend and a bank holiday

  if (!lastOk)                                       return { state: 'expired' };
  if (consentEnds && now >= consentEnds)             return { state: 'expired', expiresAt: consentEnds };
  if (daysBetween(lastOk, now) > grace)              return { state: 'expired' };   // silence == expiry
  if (consentEnds && daysUntil(consentEnds) <= 30)   return { state: 'expiring', expiresAt: consentEnds };
  if (a.consecutiveFailures >= 2)                    return { state: 'expiring', expiresAt: consentEnds };
  return { state: 'live', expiresAt: consentEnds };
}
```

Two properties matter more than the thresholds:

- **Silence is expiry.** No heartbeat within three cadences is `expired`, whatever any stored flag says.
  A dead runner, a deleted cron, a lapsed cloud account and a revoked consent all converge on the same
  honest answer.
- **The runner writes a heartbeat on failure too.** Every poll attempt, successful or not, emits a
  `ConnectionHeartbeat` op sealed to the perimeter, carrying `{ attemptedAt, outcome, providerError? }`.
  This distinguishes "the runner is alive and the bank said no" from "the runner is gone", which are
  different problems with different fixes and must not present identically.

`expired` forces `value: null` at two independent points: the runner stops emitting balance
observations once it observes expiry, and the `observe()` constructor of §2.2 refuses to carry a value
alongside an expired connection even if one is passed. Presentation asked for enforcement "at source";
this is two sources, and they do not share code.

### 6.3 Re-linking

The re-link prompt is a scheduled event addressed to the account's owner (UX4), not an error discovered
when a figure disappears. Timeline per connection: notice at T−30, T−14, T−3, and on expiry. Each is
generated **client-side** from the state above and delivered as a §6.5 wake.

Re-linking is an SCA ceremony only the account holder can complete, so the notice goes to the owner and
to nobody else — which also means a `HOUSEHOLD` account with two owners nominates one at link time. The
flow is `beginLink` → bank redirect → SCA → `completeLink` → new `LinkHandle` → rotate the stored token
(DS4) → resume. The account's history is untouched throughout; only the connection object is replaced.

**During the gap.** The account is not removed and its history is not hidden. It shows `Unknown` with
its last-known figure available in the provenance drawer and dated. Financial correctness F10's step
function; presentation §4.2's ladder. Crucially the household total does not silently shrink: an
account that went unknown appears in the unknown list and the total states that it is incomplete
(presentation R4 turns the point into a range).

### 6.4 The 180-day calendar is a household fact, not an incident log

Four connections, each expiring on its own 180-day cycle, is roughly eight re-authentications a year,
split between two people. That is a real recurring chore and the design should treat it as a known
periodic obligation — like a tax deadline — rather than as a series of surprises. Architecture's
contribution: `consentExpiresAt` is a first-class field, the schedule is derivable months ahead, and
ease-of-use can render it as a calendar rather than as an alert. It is also a reason to keep the
tier-A accounts' *document* path warm: a Revolut statement download works whether or not the consent
does, and the household should never be in a position where a lapsed consent means no data at all.

### 6.5 Reminders without a server that can read due dates

AR2 permitted the reminder scheduler to read due dates. Under AD1 it cannot, so:

```sql
create table wake (
  id             uuid primary key,
  principal      uuid not null references principal(id),
  deliver_after  date not null,          -- DAY granularity only
  jitter_minutes smallint not null,      -- 0..1439, chosen by the client
  token          bytea not null,         -- opaque; meaningless without the perimeter key
  delivered_at   timestamptz
);
```

The client computes what is due — re-links, expected statements, escalations, custodial milestones —
and registers wakes. The cron dispatches a **content-free** web push at the jittered time (security L13:
the payload is a wake token; the notification text is generic). The client fetches and renders the
actual reminder after unlock, from its own state.

The server learns: *this principal wants to be woken on this day*. That is a weak activity signal of the
same class as §3.7's and is bounded by day granularity plus jitter. It learns nothing about which
account, which institution, which amount, or even how many reminders that wake represents — one wake per
principal per day carries all of them.

---

## 7. The Android share target

### 7.1 Why it is worth building early

The feasibility assessment puts the share target after the thing it feeds, on the reasoning that a
native wrapper is a later concern. That reasoning no longer applies, because **the share target does not
need a native wrapper.** An installed PWA is a share target on Android through the Web Share Target API
in the manifest, and the shared file can be handled entirely inside the service worker. Chrome for
Android has supported this for years; Workbox has a recipe for a share target that "lives entirely
inside your service worker" so that a static app can receive shares without any server endpoint at all.

That last property is not a convenience. It is what makes the share target **compatible with INV8**: the
POST that the Android share sheet generates is intercepted by our own service worker before it reaches
the network, so the raw PDF never leaves the device. A share target implemented the ordinary way — POST
to a server route which then stores and processes the file — would be a direct INV8 violation, and it is
the obvious way to build it.

Given 17 of 26 lines arrive as documents, mostly as PDFs opened in a bank app on a phone, this removes
the single largest friction point in the primary data path. It should ship in phase 3, with ingestion,
not in a later phase.

### 7.2 Manifest and interception

```json
{
  "share_target": {
    "action": "/_share",
    "method": "POST",
    "enctype": "multipart/form-data",
    "params": {
      "title": "title",
      "text": "text",
      "url": "url",
      "files": [{
        "name": "documents",
        "accept": ["application/pdf", ".pdf",
                   "text/csv", ".csv",
                   "application/vnd.ms-excel", ".ofx", ".qif",
                   "image/jpeg", ".jpg", ".jpeg",
                   "image/png", ".png"]
      }]
    }
  }
}
```

Both the MIME type **and** the extension are listed for every format. Omitting the extension is a known
Chrome-for-Android failure mode in which the PWA appears in the share sheet but cannot actually receive
the file — a bug that manifests as an empty form and is miserable to diagnose.

```js
// service worker
self.addEventListener('fetch', (event) => {
  const url = new URL(event.request.url);
  if (event.request.method !== 'POST' || url.pathname !== '/_share') return;

  event.respondWith(Response.redirect('/inbox?shared=1', 303));   // respond immediately;
                                                                  // Android's share sheet is waiting
  event.waitUntil((async () => {
    const form  = await event.request.formData();
    const files = form.getAll('documents');
    for (const f of files) await stageSealed(f);                  // §7.3 — never touches the network
  })());
});
```

The `respondWith` returns a redirect immediately so the share sheet dismisses; the actual work happens
in `waitUntil` and the app opens on the inbox. `/_share` has no server route. If the service worker is
somehow not controlling the page, the POST would reach the network, so the server MUST define `/_share`
as an endpoint that returns `410 Gone` and never reads the body — a belt-and-braces guard against the
one path by which a raw statement could escape. This is a required test.

### 7.3 Sharing while locked — and why it works

The share sheet does not care whether the app is unlocked, and requiring a passkey ceremony before a
share can be accepted would defeat the purpose. It is not required, because of an asymmetry worth
naming:

> **Sealing needs only public keys. Only opening needs a private one.**

Each perimeter's X25519 public key is cached in plaintext in IndexedDB (it is published in the policy
journal anyway; it is not a secret). So `stageSealed()` can, with the app fully locked and no key
material in memory:

1. read the file into memory,
2. compute `H = SHA-256(plaintext)`,
3. generate `K_doc`, encrypt, pad to a size bucket,
4. wrap `K_doc` to both partners' perimeter public keys,
5. zeroise the plaintext,
6. queue the sealed blob in an IndexedDB outbox.

On next unlock the outbox flushes to `POST /inbox` and routing (step 4 of §5.1) begins. The user's
experience is: share from the bank app, get a notification, open the app when convenient, confirm.
Nothing was decrypted, nothing was lost, and the plaintext existed in memory for a few hundred
milliseconds inside our own service worker.

The outbox is also the general offline upload path: a share taken on a train with no signal behaves
identically.

### 7.4 The limits, stated

- **iOS is out of scope.** Web Share Target is not supported by Safari. The household targets Android
  and desktop (`02` §1), so this is not a gap for them; it is a gap for the design's portability and
  should not be papered over.
- **Screenshots cannot be blocked** on the web platform (security R4). If a Trusted Web Activity wrapper
  is ever built for `FLAG_SECURE`, its share target must feed this same service-worker path — the
  wrapper must not gain a privileged route to the server. That is a review gate on any future wrapper.
- **Desktop share** is the ordinary file picker and drag-and-drop into the same outbox. The Windows
  share sheet also supports PWA share targets, which costs nothing extra since the manifest is shared.

---

## 8. Scheduled work

Four scheduled jobs, and they are the only things the Core does on its own.

### 8.1 ECB rate fetch

Financial correctness §3.2 owns the semantics; architecture owns the plumbing. Poll after 16:15 CET on
TARGET open days, from **both** the static XML files (`eurofxref-daily.xml`, `eurofxref-hist-90d.xml`)
and the SDMX API, because they fail differently — a static file is very hard to break, and the API is
what detects revisions and fills gaps. Write to the plaintext `rate` table, append-only, versioned
(R6). Alarm if no new row for two consecutive TARGET business days.

This is public data, shared by both perimeters, identical for everyone, and therefore the one thing the
Core computes that needs no encryption at all. Clients pull the table and convert locally, which is what
keeps INV1 and INV2 honest: no conversion is ever performed on the server, so no converted value can
ever be persisted there.

### 8.2 Inbox TTL sweep

Nightly, unconditional, one statement (§5.4).

### 8.3 Market data (AD14)

D5 upgrades Linxea to mark-to-market by ISIN, and the PEA is marked from prices too. Data security's
open question 4 is the right one: **per-ISIN queries disclose the portfolio** — to the data provider,
and to whoever operates the fetcher.

Rule: **never query the household's ISINs.** Fetch a fixed universe file — a few hundred EUR-denominated
UCITS ETFs and the household's plausible instrument space, chosen once and stored in the repository —
on a daily schedule, and store all of it in the plaintext `instrument_price` table. Clients look up what
they need locally. The provider and the operator learn which universe file we fetch, which is the same
for every household that uses this software and reveals nothing about this one.

Cost: a few hundred instruments daily is inside the free tier of several EOD price providers, and a
paid EOD-only feed is single-digit euros a month if needed. Note that this is the only line item in §3.5
with a plausible path to real money; keeping it a universe fetch keeps it flat rather than proportional
to portfolio size. If a household instrument is genuinely not in any universe file, the honest fallback
is the insurer's or custodian's own statement valuation, which the feasibility assessment already
identifies as authoritative.

### 8.4 Wake dispatch

Per §6.5. Reads `wake`, sends a content-free push, marks delivered. It has no other capability.

---

## 9. Resilience

### 9.1 Failure taxonomy

Every row is a thing that will happen, with what the user sees and what does not break.

| # | Failure | User-visible | Blast radius |
|---|---|---|---|
| F1 | One PSD2 consent expires | That line reads `Unknown`; a re-link task appears for its owner | 1 of 26 lines |
| F2 | All four consents expire | Four lines `Unknown`; four re-link tasks | 4 of 26. The app is *unchanged* in structure — this is the §9.2 drill |
| F3 | Enable Banking is discontinued | Same as F2, plus a provider migration (§4.6) | 4 of 26, plus an evening's work |
| F4 | A runner stops (deleted cron, expired card, lapsed account) | Its owner's fed lines go `Unknown` within 3 cadences (§6.2) | That partner's fed lines |
| F5 | Core is down | Both clients keep working from their local ledgers. Uploads queue in the outbox. Sync resumes | Cross-device sync only |
| F6 | Postgres lost, no backup | Ciphertext gone from the host; **both clients still hold the full ledger.** Re-seed the Core from a client | Nothing, if either client is intact |
| F7 | A partner loses every device | Recovery kit restores the key; the ciphertext is on the host | That partner's access, until the kit is used |
| F8 | A partner loses every device **and** the kit | That perimeter is unrecoverable (security R5). `HOUSEHOLD` and `CUSTODIAL` survive via the other partner | One private perimeter |
| F9 | Model provider down or refusing | Extraction escalates to manual confirmation; the local template parse still runs and still closes arithmetically | Ingestion slows; nothing is wrong |
| F10 | A template drifts (bank redesign) | G1 fails → escalation with both passes shown side by side | One document family, loudly |
| F11 | ECB feed gaps > 7 days | Converted figures become `Unknown` (13-fc R7), not stale | Display currency conversion only; native figures unaffected |
| F12 | A normaliser bug ships | Replay stored raw payloads (§4.3), regenerate | Repairable without the provider |
| F13 | Clock skew between devices | HLC ordering tolerates it; a device more than 5 minutes off warns | Ordering of same-second ops |
| F14 | Both partners edit the same account setting offline | LWW by `(hlc, device_id)`; the loser is journalled and shown | One field |

The shape to notice: **no row in this table produces a wrong number.** Failures produce `Unknown`,
queued work, or an escalation. That is the property the whole system is arranged around, and it is
worth more than uptime.

### 9.2 The zero-feed drill (INV5 made executable)

INV5 says the app is fully usable with zero live feeds. AD15 makes that testable rather than assertable.

```
CI matrix:
  feeds=live      →  full suite, against the Enable Banking sandbox
  feeds=null      →  full suite, NullFeedAdapter, ALL FOUR lines expired
```

The `feeds=null` job must pass with **zero test exclusions**. No test may be skipped, marked pending, or
guarded by `if (hasFeeds)`. The moment one is, INV5 has quietly become conditional, and the review rule
is that a PR adding such a guard is rejected.

What the drill asserts specifically:

1. All four views render, with content, in both display currencies.
2. The four fed lines show `Unknown`, not zero, not last-known-as-current.
3. Household net worth is reported with an explicit unknown list (presentation R4), and the number that
   *is* shown is exactly true over the accounts it covers.
4. Upload, redaction, extraction, four-eyes, escalation and commit all work.
5. Manual entry works for every account, including the four that normally have feeds.
6. The upkeep list contains four `relink` items addressed to the right owners.
7. No banner, no error state, no degraded chrome. Presentation §13: a document-sourced balance and a
   feed-sourced balance render identically because they are equally true.

### 9.3 Degradation ladder

The system has four rungs and it should be possible to say which one you are on at any moment.

| Rung | Condition | Capability |
|---|---|---|
| **Full** | Core reachable, runners healthy, unlocked | Everything |
| **Offline** | No network, unlocked | Read everything, enter figures, upload to the outbox, extract with the local template parser. No model pass, no sync |
| **Locked** | Any network state, locked | **Accept and seal shares** (§7.3). Nothing readable |
| **Cold** | New device, no key | Authenticate; then either an enrolled device's wrap or the recovery kit |

The Locked rung is the one that usually does not exist in designs like this, and it is the one that
makes the Android share target usable in practice.

### 9.4 Backup, restore, and what the recovery kit actually is

Under AD1 the topology inverts the usual backup story, and it is worth being explicit because it changes
what a household must protect.

- **The host is not the system of record.** Each client holds a full plaintext replica of the perimeters
  it can read. The host holds ciphertext and is a rendezvous point plus an off-device copy.
- **Host backups are therefore cheap and safe.** Managed Postgres automated backups plus versioned
  object storage with object lock. There is no scenario in which a backup discloses a figure, so backups
  can be numerous, long-retained and stored wherever is convenient. This is a real and underrated
  benefit of encrypting everything: the backup problem becomes an availability problem only.
- **The recovery kit is the disaster-recovery plan.** Restoring data requires ciphertext (host) *and* a
  key (device or kit). Security §6.3 forbids partner-assisted recovery of a private perimeter, so the
  printed kit is not a nicety — it is one of exactly two copies of the ability to read one partner's
  own financial history.

Three architectural requirements follow, all cheap:

1. **Kit generation is blocking at enrolment** and includes a **verified round trip**: the app prints the
   kit, then requires the user to re-enter a check-word from the printed sheet, so a kit that printed
   incorrectly is discovered in the first five minutes rather than in year three.
2. **An annual kit drill.** Once a year, a non-dismissible task: "confirm you can still find your
   recovery kit." Not a test decryption — just confirmation that the paper exists and where. This costs
   one interaction a year and addresses the most likely cause of F8, which is not loss but forgetting
   where it went.
3. **Restore is rehearsed once**, in phase 2, on real hardware: wipe a device, restore from kit, confirm
   the ledger reconstitutes. A recovery path that has never been executed is a hypothesis.

### 9.5 Detecting a rewritten history

Security §3.7 makes the policy journal a two-party transparency log: each client verifies the hash chain
from its last-verified head and publishes that head back as a `journal.head_gossip` entry. Two honest
clients that have seen different histories produce heads that do not appear on each other's chain.

Architecture's contributions are the two operational cases that would otherwise produce false alarms and
teach the household to dismiss the warning:

- **A legitimate database restore rolls the op log back.** After a restore, the head is lower than a
  client's last-verified head, which is indistinguishable from tampering. Procedure: a restore writes a
  `journal.restore_performed` entry, co-signed by both partners' devices before sync resumes. A restore
  neither partner signed is a fork, and is treated as one.
- **Op-log compaction prunes history.** Compaction (§10.1) must never touch the journal, only the op
  log. The journal is small — a few thousand entries over a decade — and is kept whole forever.

A fork alert must be **loud, non-dismissable and identical on both devices**. It is the only alert in
the system with that treatment, and it earns it by being the one signal that the guarantee described in
R1 has been violated.

---

## 10. Sync, offline and the client data model

### 10.1 The encrypted operation log

Sync is not a database replication problem here; it is a small append-mostly log with two writers per
perimeter. Building a general CRDT would be over-engineering. The model:

| Class | Example | Conflict rule |
|---|---|---|
| **Immutable facts** | postings, balance assertions, provenance, raw payloads, heartbeats | None possible. Append; order by HLC for display only |
| **Mutable metadata** | account label, cadence, category assignment, goal target, extraction state | LWW by `(hlc, device_id)`; the losing value is journalled and surfaced once (F14) |
| **Derived artefacts** | reconciliation postings (13-fc F3), model output, display snapshot | **Never synced.** Recomputed on every client from the facts |

The third row is a direct consequence of financial correctness's F3, and it is what keeps sync small
and keeps two devices from disagreeing about a plug figure.

**Compaction.** The op log grows forever; a decade is ~290k ops (§3.4), which is fine, but a client
should not replay it from genesis on every cold start. Periodically a client writes a **sealed
snapshot** of the perimeter's state at `seq = N`; once both of that perimeter's devices acknowledge the
snapshot, ops `≤ N` may be pruned. The server prunes on instruction and cannot verify the instruction,
which is acceptable: a malicious prune is a fork, and §9.5 detects forks. The policy journal is never
pruned.

### 10.2 Ruling on presentation §12 — the offline display snapshot

Presentation §12 asks whether caching a rendered view — which necessarily contains converted values —
breaches INV1's "a converted value is never persisted", and explicitly asks financial correctness or
architecture to rule on whether the distinction survives contact with a codebase.

**Ruling: accepted, under the same discipline financial correctness already applies to reconciliation
postings (F3), and enforced by a build-time boundary rather than by care.**

The distinction presentation draws — between persisting a converted value *as a fact*, which corrupts
history, and caching a *rendering*, which is a screenshot with a timestamp — is real. INV1 exists
because a persisted conversion silently becomes an input to later arithmetic. So the property to enforce
is not "no converted number is ever written to disk"; it is **"no converted number is ever read back as
an input."**

That is enforceable structurally:

1. The snapshot lives in a physically separate IndexedDB object store, `display_cache`, with its own
   module, `@app/display-cache`.
2. `@app/ledger`, `@app/fx`, `@app/projection` and `@app/goals` **do not import** `@app/display-cache`.
   A `dependency-cruiser` rule fails the build if any of them ever does. This is the concrete answer to
   presentation's "will it survive a codebase" — it survives because a violation is a red build, not a
   code review someone might skim.
3. The snapshot carries `renderedAt` and `rateSetVersion`, is purged on reconnect, on rate-set change
   and after 7 days, and its age is shown in the header whenever it is what is on screen.
4. It is `audience`-tagged like every other figure (INV14), so a locked or switched principal cannot see
   another audience's cached render.

The fallback presentation offered — native amounts plus a rate table only, with a visible 200–400 ms
cold start — remains available behind a flag. If the import boundary is ever removed, that flag becomes
the default. But with the boundary in place I do not think INV1 is blurred: the snapshot is a derived
artefact of exactly the kind F3 already establishes, and F3 has to be got right anyway.

### 10.3 Client storage layout

| Store | Contents | Cleared on lock |
|---|---|---|
| Cache Storage: `shell` | app shell, WASM, templates | No |
| Cache Storage: `models` | NER model, if downloaded | No |
| IndexedDB: `ops` | perimeter op-log ciphertext | No (it is ciphertext) |
| IndexedDB: `blobs` | recent document ciphertext, LRU | No |
| IndexedDB: `outbox` | sealed uploads awaiting POST | No |
| IndexedDB: `display_cache` | display snapshot (§10.2), encrypted | **Yes** |
| Memory | decrypted ledger, perimeter keys | **Yes, zeroised** |

Security §7.6 requires that API responses containing figures never reach Cache Storage. Under AD1 no API
response contains a figure, so the rule holds by construction — another policy that becomes a fact.

---

## 11. Where this document departs from its inputs

Five places, argued rather than worked around.

### 11.1 AD1 supersedes security §1.5's perimeter split

Security places `K_H` and `K_S` on the household node and processes the shared perimeter server-side.
This document does not. The argument is §1.2: the split requires two implementations of the ledger and
the extraction pipeline, and the second one lands on the most correctness-critical code in the system.
The security *requirement* — that private perimeters be confidential from the host — is not weakened;
it is extended to the shared perimeter, where it costs nothing because the client had to do the work
anyway. `11-data-security.md` reached the same conclusion independently and its §0.6 and §2 are adopted
as the topology of record. Security's §3.5 SQL is retained as the *specification of the closure rule*,
implemented in the client's query layer and type system, with RLS kept on the envelope as defence in
depth (§1.5, AR1).

### 11.2 AR2 is exceeded, and the reminder scheduler loses a capability it was granted

AR2 permits the reminder scheduler to read due dates. It cannot, because there are none to read. §6.5
replaces them with client-registered opaque wakes at day granularity with jitter. This is stricter than
required and slightly worse operationally: the server cannot batch intelligently and cannot tell a
household that something is overdue if nobody has opened the app in a month. Accepted: a reminder
system that only works when someone opens the app is a real limitation, mitigated by push wakes being
registered *ahead of time*, so a wake fires whether or not the app has been opened since.

### 11.3 INV7's literal key is amended

INV7 says "key on institution, account, period and document hash". Under S8 the key must be a
perimeter-scoped HMAC or it is a membership oracle. §5.6 keys on
`HMAC(K_perimeter, institution ‖ account ‖ period)` and `HMAC(K_perimeter, H)`. The *property* INV7
asks for — re-uploading a statement never double-counts — is preserved exactly. The wording is not, and
a reader implementing INV7 literally would build the oracle. Proposed replacement wording in §11.6.

### 11.4 AD13 — a 90-second grace window on background lock

Security §6.4 specifies "immediate on background for private perimeters", with keys wiped from memory
and a fresh PRF ceremony to resume. On Android a PWA backgrounds constantly: a notification, a glance
at a message, a phone call, switching to the bank app to fetch the statement you are about to share.
Immediate lock means a biometric prompt several times per session, and the observable consequence of
that is not better security — it is that the app stops being opened, which for INV5's "fully usable"
is a real failure.

**Proposal: keys are retained for 90 seconds after backgrounding, then zeroised.** Immediate zeroise on
explicit lock, on screen-off beyond the window, and on any app-switch that lasts longer.

The argument for accepting this: the system is read-only by construction (security's proposed INV12),
so the marginal harm of a 90-second window is bounded by what an attacker with physical possession of an
*unlocked, recently-used* phone could read — and such an attacker could have read it 90 seconds earlier
anyway without any window. The threat this protects against, a phone picked up minutes later, is
untouched. This is squarely in the availability-versus-confidentiality trade that architecture owns, the
cost of getting it wrong in the strict direction is an unused app, and the value is measured in whether
the household actually keeps the ledger current.

If security-by-design disagrees, the fallback is a per-perimeter setting defaulting to 0 seconds for
private perimeters and 90 for `HOUSEHOLD` — which is more complex and, I think, worse, but it is not
unreasonable.

### 11.5 A narrowing D2 does not suffer, and one it does

Security §12.4 preserves D2's upload right and removes the uploader's feedback. §5.2 shows most of that
cost is recovered by AD1 — the client tells the uploader everything about documents they can route.

The narrowing that survives is §5.5's: a document family flagged `multi_perimeter_capable` seals to the
uploader's perimeter only and cannot be handed to the household at all. So Partner A physically holding
Partner B's consolidated Julius Baer statement cannot get it into the system for her. He must hand her
the paper or the file. That is a genuine friction and it is the correct one — the alternative is a
document containing her private lines sealed to a key he holds.

### 11.6 Invariant dispositions

| Invariant | Disposition |
|---|---|
| INV1 | Held. Strained once, at the display snapshot; ruled acceptable in §10.2 under a build-enforced boundary. No conversion occurs on any server, so no converted value can be persisted as a fact |
| INV2 | Held, and structurally: conversion happens only in the client, per viewer |
| INV3 | Held, strengthened. Feed provenance resolves to the raw payload's content hash (§4.3), not to a connection |
| INV4 | Held, strengthened by the fail-closed clock (§6.2). Silence is expiry |
| INV5 | Held, and made executable (§9.2). No test exclusions permitted |
| INV6 | Held. Also endorsing security's INV12: no RPA, no stored bank passwords, no automated login to any bank's web front end, ever — including for the seventeen tier-B lines, where it will be tempting in month six |
| INV7 | **Property held, wording amended.** Proposed: *"Ingestion is idempotent. Re-uploading a statement must never double-count. The key is derived from institution, account, period and document hash, and is scoped to the perimeter so that it cannot be computed by anyone outside it."* |
| INV8 | Tightened, not broken: controlled infrastructure means the uploading device (§3.2). Redaction precedes the *first network write*, not merely egress from a datacentre |
| INV9 | Held. Exact types throughout, including in adapters and normalisers — a `float` in a normaliser is as much a defect as one in a balance |
| INV10 | Held. Requires security's FC1 fourth class (perimeter egress/ingress), which financial correctness §2.2's `flow_class` accommodates |
| INV11 | Held. AD1 is its direct consequence |
| INV12/13/14 (proposed by security §13) | **Endorsed.** INV12 in particular is the highest-leverage control in the system and is free: the adapter port has no payment-initiation method and no write method, so the capability does not exist to be misused. INV13 is structural under AD1. INV14 extends naturally to the offline snapshot (§2.2) |

---

## 12. What other agents can rely on

### 12.1 To presentation (`15-presentation.md`)

| # | Provided |
|---|---|
| P1 | The `Observation` envelope, exactly as §11.2 specified (§2.2), with `expectedCadenceDays` stored per account line and defaults tabled |
| P2 | `connection.state === 'expired' ⇒ value: null`, enforced at two independent points that share no code (§6.2). Presentation is never asked to choose between a stale number and unknown |
| P3 | `RateSet { date, version, source, pairs, fetchedAt }` for §6.3's rate-age banding |
| P4 | `OfflineSnapshot { renderedAt, rateSetVersion, displayCcy, audience, payload }` (§2.2), with the INV1 ruling in §10.2: **accepted, with a build-enforced import boundary** |
| P5 | `UpkeepItem` is produced client-side from the fail-closed clock; `owner` is the account owner who can actually complete the SCA (§6.3) |
| P6 | Loading states cannot transiently contain out-of-scope figures, because out-of-scope figures are ciphertext the client cannot decrypt (§1.4) |
| P7 | `partial: true` is a constant, not a computed flag — under AD1 there is nothing to compute it from |

### 12.2 To financial correctness (`13-financial-correctness.md`, resuming at §5)

| # | Provided or asked |
|---|---|
| C1 | **Provided:** the ledger runs in the client, in memory, over ~290k postings per decade with a 20–60 ms full fold (§3.4). §2.4's "no stored balance field anywhere" is affordable; do not compromise it for performance |
| C2 | **Provided:** three idempotency layers (§5.6). Layers 1 and 2 are perimeter-scoped HMACs; layer 3 — the multiset reconciliation of F9 — is yours and runs client-side |
| C3 | **Provided:** raw payloads are stored forever and replayable (§4.3), so a normaliser fix is a replay, not a data-loss event |
| C4 | **Asked:** the G4 materiality and plausibility thresholds per account type (§5.7). Data security §10 asks you the same thing; one answer serves both |
| C5 | **Asked:** confirmation that document families without arithmetic closure (Pillar 2, Hostplus, Linxea) may never auto-commit (§5.7) |
| C6 | **Provided:** `flow_class = UNOBSERVED` reconciliation postings are never synced — they are recomputed per client from facts (§10.1), which is F3's rule extended to the transport |
| C7 | **Asked:** the `natural_key` construction must be provider-independent (§4.4), or a provider migration becomes a re-import. This is a constraint on §5.3 when you write it |

### 12.3 To ease of use (`14-ease-of-use.md`)

| # | Provided |
|---|---|
| E1 | The Locked rung (§9.3): a shared statement can be accepted while locked and confirmed later. Design the flow around "share now, confirm later" |
| E2 | The extraction state machine (§5.7) is the vocabulary for the review queue. `escalated(reason)` carries which of G1–G4 failed, and the review UI shows both passes side by side |
| E3 | Re-link is a scheduled calendar of eight events a year across two people (§6.4), not an error state |
| E4 | Upload feedback is rich for routable documents and silent for others (§5.2). The silence is not a bug to design around |
| E5 | Three blocking rituals: second passkey (UX1), recovery kit with a verified round trip (§9.4), and the annual kit drill |

### 12.4 To data security (`11-data-security.md`)

| # | Confirmed or asked |
|---|---|
| S1 | **Confirmed:** the controlled component for INV8 is the uploading device's PWA. Your §2 stands and this document is built on it |
| S2 | **Confirmed:** the server surface is a content-addressed blob store, an opaque row store and a stateless relay. No server-side content processing, no search index, no server-side document rendering |
| S3 | **Changed:** the relay is deployed **per partner** (§3.3), not once, so that a private perimeter's relay traffic does not pass through infrastructure the other partner administers. Same artefact, two deployments |
| S4 | **Confirmed:** reproducible builds with a published manifest hash. CI publishes both the container digest and the PWA release manifest; clients verify the manifest on update, which is what makes R1 detectable |
| S5 | **Asked:** which document families fail client-side redaction (your DS6). §5.5 turns your answer into a template flag that changes the upload path, and §3.4 turns it into an Android-vs-desktop capability split |
| S6 | **Asked:** confirm that padding to `{64 KB, 256 KB, 1 MB, 4 MB, 16 MB}` before sealing is compatible with your blob format (§5.4) |

---

## 13. Build order

Mapped onto the six phases of `01` §"Build order". Each row is a thing that stops being possible to get
wrong later.

| Phase | Architecture deliverables | The thing this phase de-risks |
|---|---|---|
| **0. Before phase 1** | Enable Banking signup. Confirm: restricted-production terms; that a natural person may hold **two** applications (S12/AD3); consent validity; what Crédit Agricole exposes for a Livret A | If two registrations are impossible, private-and-fed is impossible and the household must know before D3 is confirmed with Partner B |
| **1. Prove the connection** | `FeedAdapter` port frozen against **two** providers' concepts. `EnableBankingAdapter` + `NullFeedAdapter`. Raw-payload storage (AD5) from the first fetch | The seam's shape, decided while a second provider is still hypothetical rather than urgent |
| **2. Ledger and two logins** | Core deployed. Op log, RLS envelope, policy journal, two-signature genesis. Passkeys + PRF. Client ledger, in-memory fold, encrypted IndexedDB. **Restore-from-kit rehearsed on real hardware** | Retrofitting perimeters into a live ledger is a migration through the boundary it protects. Security §14 is right that this is the one phase where nothing may be deferred |
| **3. Document ingestion** | Blind inbox with all five §5.3 invariants. Client redaction. Per-partner runner + relay. Four gates. Escalation as a perimeter op. **The Android share target ships here** (§7.1) | An ingestion path without the inbox is an oracle in production. The share target is the difference between a document pipeline that gets used and one that does not |
| **4. The four views** | `Observation` everywhere. Display snapshot with the dependency-cruiser boundary (§10.2). Offline rungs | Presentation's contract, honoured at the point figures first reach pixels |
| **5. Live feeds and staleness** | Fail-closed clock. Heartbeats. Wake scheduling. Re-link flow. **The §4.6 discontinuation drill, run for the first time** | Feeds arrive last deliberately: by the time they land, 22 lines have been working without them for months, which is the only honest test of INV5 |
| **6. Tax lens, property, refinement** | Universe price fetch (§8.3). Custodial milestone scheduling | — |

Two things that must not slip, because they are the ones that get dropped when a schedule does:

- **The raw-payload store (AD5)** looks redundant on the day you write it and is the only thing that
  makes phase 5's provider migration survivable.
- **`NullFeedAdapter` in CI with zero exclusions (AD15).** The first `if (hasFeeds)` guard is the moment
  INV5 becomes a claim again.

---

## 14. Open questions

**For the household:**

1. **Will Partner B operate her own runner?** (§3.3) One Scaleway project, one Enable Banking
   registration, one redeploy when the image digest changes. If not, her fed accounts are `HOUSEHOLD`,
   or private and document-driven. This is a five-minute decision with a large structural consequence
   and it should be made explicitly, not by default.
2. **Scaleway managed, or a Hetzner box you own?** (§3.5) ≈€14/month with nothing to patch, versus
   ≈€7/month and an hour a month. The topology is identical either way.
3. **Is Julius Baer statemented per account or consolidated?** (§5.5) Determines a template flag that
   changes the upload path, and — per security S11 — whether any Swiss line can be private. Answerable
   by looking at last month's statement.
4. **Cover traffic (§3.7): worth it?** It hides activity levels from the host operator and costs storage
   and complexity. I would ship without it.

**For other agents:**

5. **Financial correctness:** the G4 thresholds (§12.2 C4), and confirmation that `natural_key` is
   provider-independent (C7).
6. **Data security:** which document families fail client-side redaction (§12.4 S5), and whether size
   bucketing fits the blob format (S6).
7. **Security-by-design:** AD13's 90-second grace window (§11.4). This is the one place I have relaxed
   something you specified, and it is a judgement call about whether the app gets used.

**To verify before depending on it:**

8. Enable Banking's exact endpoint shapes, pagination and consent validity — the feasibility assessment
   already says to confirm restricted-production terms at signup, and this document adds: confirm the
   two-application question at the same time.
9. Android in-browser NER performance on the household's actual phone (data security's open question 7),
   which decides whether §3.4's desktop-only ingestion flag applies to one family or several.
10. Scaleway Serverless Containers' free-tier allowance against two runners plus the sync API. The
    numbers in §3.5 are current as of August 2026 and the tier has changed before.

---

*Topology, seam design and deployment as of 27 August 2026. External facts used: GoCardless Bank Account
Data (formerly Nordigen) is closed to new signups and Enable Banking's restricted-production tier —
whitelisted to accounts you authenticate to yourself, JWT/RS256 with the application id as `kid` — is
the current self-serve replacement in the EU; the Web Share Target API supports POST/multipart file
shares to installed PWAs on Android, handled entirely within the service worker; Scaleway Managed
PostgreSQL DEV-S is ≈€11/month with serverless container and object-storage costs near zero at this
volume. All three change frequently and each should be reconfirmed before the architecture depends on
it. The PSD2 180-day re-authentication cadence is taken from `10-security-by-design.md` §6.4 and is
regulatory rather than a design choice.*
