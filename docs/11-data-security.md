# Data Security

**Agent:** Data security
**Status:** Design. Binds implementation within this remit.
**Date:** 27 August 2026
**Inputs:** `01-feasibility-assessment.md`, `02-decisions-and-constraints.md` (D1–D5, INV1–INV11, threat model seed)

**Scope owned here:** encryption at rest and in transit, key management, document storage and
retention, the redaction pipeline, third-party data exposure, data-subject rights.

**Scope explicitly not owned here:** who may see what and how that is enforced (security-by-design);
system shape and deployment topology (architecture); what the numbers mean (financial correctness).
Where those meet this document, section 10 states the interface rather than designing their side.

---

## 0. What this document decides

1. **Redaction runs in the uploading client, not on a server.** The controlled component required by
   D1/INV8 is the PWA on the partner's own device. No unredacted document byte is ever transmitted
   anywhere. This is stronger than D1 requires and removes the need for a home server (§2).
2. **Nothing is filtered outbound; the outbound payload is synthesised.** Deny-by-default
   reconstruction from an allow-list of spans, never the original document with boxes drawn on it.
   Images and PDFs never leave (§3.6).
3. **Account identifiers never leave, and routing does not need them to.** Routing is a local lookup
   performed *before* egress. The dual role of the account number is resolved by ordering, not by
   compromise (§3.4).
4. **Transaction narratives are default-deny.** The numeric grid and the narrative column are
   separated; the model extracts the grid, and narratives are handled locally with a per-label
   consent escalation for the residue (§3.5).
5. **D1's "two independent extraction passes" is implemented as one model pass, one deterministic
   local parse, and mandatory arithmetic closure.** This contradicts the literal reading of D1 and is
   argued openly in §3.7.
6. **The entire ledger is client-side encrypted.** At two users and 26 account lines the dataset is a
   few megabytes; the server never needs plaintext to do its job, so it never gets it (§4).
7. **The host holds no key material of any kind.** Reading one partner's private accounts requires an
   identity private key that exists only inside that partner's devices (§5).
8. **Retention is tiered, with a permanent hash tail.** Erasure deletes the document and keeps the
   hash, so GDPR Article 17 and INV7 idempotency coexist without either being weakened (§6).
9. **The son's documents never go to a model at all**, and his key domain is separate from day one so
   that handover at 18 is a key operation rather than a data migration (§8.4).
10. **The GDPR household exemption is not relied on.** Reasons in §8.1.

---

## 1. Position

Two framing statements that the rest of this document follows from.

**Assume every outbound payload persists for two years.** Anthropic's published policy is that API
inputs and outputs are not retained by default and are never used for training without express
permission; a zero-data-retention arrangement is available but is enabled per organisation by an
account team through a sales contract, which is not obtainable at household scale. And *regardless of
arrangement*, content flagged by automated trust-and-safety systems may be retained for up to two
years. A statement full of transactions is not obviously benign to a classifier. The correct
engineering posture is therefore not "the provider deletes it in seven days"; it is "whatever we send
may sit in someone else's storage until 2028." Every redaction decision in §3 is made against that
assumption. This is the strongest available argument for D1 and it should be stated to the household
in those terms.

**The effort is asymmetric, and the household should know it.** This system will spend real
engineering effort redacting roughly ten documents a month before they reach a model, while a licensed
PSD2 aggregator receives complete transaction narratives, IBANs and account-holder names for four
accounts continuously, by design, as the price of the tier-A feeds. That is not an argument for
abandoning redaction — the aggregator is a regulated entity under a DPA and the model provider is
reached over a general-purpose API — but a security design that redacts fanatically on one path and
never mentions the other is misleading its owners. §7 inventories both honestly.

---

## 2. The trust boundary

```
┌──────────────────────────── CONTROLLED (plaintext lives here) ────────────────────────────┐
│                                                                                            │
│  Partner device (installed PWA)                                                            │
│   • original document bytes            • redaction map (pseudonym → real value)            │
│   • identity private key               • household key, partner key, son's key             │
│   • decrypted ledger                   • Known Identifiers Table                           │
│   • redaction pipeline §3              • local template parser                             │
│                                                                                            │
└──────────────────┬─────────────────────────────────────────────┬───────────────────────────┘
                   │ ciphertext only                             │ synthesised redacted payload
                   ▼                                             ▼
        ┌──────────────────────┐                    ┌─────────────────────────────┐
        │ Managed host (EU)    │                    │ Extraction relay            │
        │ • opaque blobs       │                    │ • stateless, log-free       │
        │ • wrapped-key stanzas│                    │ • holds the API key         │
        │ • no keys, no plaintxt│                   │ • structural egress checks  │
        └──────────────────────┘                    └──────────────┬──────────────┘
                                                                   ▼
                                                        ┌─────────────────────┐
                                                        │ Model provider      │
                                                        └─────────────────────┘
```

**The controlled component of D1 is the uploading device.** Not a home server. Reasons:

- It satisfies INV8 *by construction* rather than by policy. If the redaction runs before the first
  network write, there is no code path along which an unredacted byte can escape. A server-side
  redactor has to receive the unredacted document first, and then the invariant depends on that
  server never logging, never caching, never crashing with the buffer in a core dump.
- It removes an always-on home box: its cost, its patching, its NAT traversal, its failure mode when
  the household is in Australia for three weeks. A1 tolerates modest spend; it does not tolerate a
  second machine to maintain.
- Statements from Julius Baer, TopCard, Bourse Direct and Crédit Agricole are digitally generated
  PDFs with a real text layer. `pdf.js` yields glyph runs with positions — better input for both
  redaction and extraction than OCR, and it runs in a browser.

The costs are real and stated: a ~135 MB quantised NER model must be downloaded and cached on each
device; Android does this work more slowly than a desktop; and the pipeline is JavaScript/WASM rather
than Python, so Presidio and spaCy are not directly usable and their detector logic has to be
reimplemented (§3.3 specifies exactly what, and most of it is regex plus checksums).

**Interface required from architecture:** the server must expose only a content-addressed blob store
and a row store for opaque ciphertext, plus the stateless relay. No server-side content processing, no
server-side search index, no server-side rendering of document content. If architecture places the
redactor elsewhere, everything in §3 still applies — but §2's "by construction" claim downgrades to a
policy claim and §4's client-side-everything decision has to be revisited.

**The relay** holds the model-provider API key (a browser cannot). It therefore sees the redacted
payload. This is accepted rather than mitigated: the payload is by construction non-identifying, and
if we do not trust that property against our own host we have no business trusting it against the
model provider. Requirements on the relay: stateless; request and response bodies never logged, never
sent to an APM, never written to disk; it is the *only* server endpoint that ever receives
document-derived content, so it is a single auditable choke point; and it re-enforces the structural
half of the egress gate server-side (§3.6) as defence against a compromised client.

---

## 3. The redaction pipeline

### 3.1 Why bank statements are hostile

The general de-identification problem assumes the identifying content is incidental to the useful
content — strip the patient's name, keep the clinical narrative. Bank statements invert this.

- **The account number is the routing key.** INV7 keys idempotency on institution, account, period and
  document hash. Three of those four are printed in the identity block we want to destroy.
- **The transaction narrative is simultaneously the categorisation signal and the densest
  concentration of third-party personal data in the document.** `VIR SEPA M DUPONT JEAN REF SALAIRE
  JUIN` contains a name, an employment relationship and a payment purpose. `PRLV CABINET DR MARTIN`
  is Article 9 special-category data — health — printed in a bank statement.
- **The amounts cannot be redacted at all.** They are the entire payload. And a small set of exact
  amounts is a strong quasi-identifier: a CHF salary of a specific centime value arriving on the 25th
  alongside a EUR mortgage debit of a specific centime value is close to unique in Europe.
- **Layout is meaning.** Column position distinguishes debit from credit, value date from booking
  date, balance from amount. Naive redaction that deletes spans shifts the geometry and produces
  extraction errors — which then get "fixed" by weakening the redaction. This is the classic failure
  spiral and the design has to make it structurally impossible (§3.9, metric 8).
- **The most common real-world redaction failure is a black rectangle drawn over text that is still
  in the PDF content stream.** Any design that ships the original PDF with annotations is one
  `pdftotext` away from a full disclosure.

The design's response to all six: never ship the document, ship a synthesised reconstruction; do the
routing locally before egress; split the numeric grid from the narrative; preserve geometry through
format-preserving pseudonyms rather than deletion.

### 3.2 Stages

```
 0 intake ──► 1 template match ──► 2 detect ──► 3 pseudonymise ──► 4 EGRESS GATE ──► 5 extract
     │              │                                                   │              │
     │              └─ no match ──► QUARANTINE (never egresses)         │              │
     └─ hash, strip metadata                          identifier hit ──►│              │
                                                       = P1 incident    │              ▼
                                                                        │        6 four-eyes
                                                                        │              │
                                                                        ▼              ▼
                                                                    relay ──► model ──► 7 rehydrate
                                                                                          │
                                                                                          ▼
                                                                                    commit / escalate
```

**Stage 0 — Intake and canonicalisation.** Accept PDF, CSV, OFX, QIF, JPEG/PNG.

- `doc_hash = SHA-256(raw bytes)` — the INV7 key component. Stored forever (§6).
- `content_hash = SHA-256(canonical text layer)` — because PDF producers embed a generation timestamp
  and a document ID, the same statement downloaded twice has different bytes. Without this, INV7
  silently fails to detect the duplicate. Both hashes are stored; a match on *either* is a duplicate.
- Strip: XMP and Info metadata, embedded files, JavaScript, attached fonts' name tables, EXIF for
  images. These carry the operator's OS username and the bank's internal file paths.
- Extract positioned glyph runs. If the text layer covers less than 60% of the page's ink, route to
  the OCR lane (Tesseract WASM, `fra+deu+ita+eng`). OCR output is treated as lower confidence and
  raises the fuzzy-match tolerance in §3.3(b).

**Stage 1 — Template match.** A registry of declarative templates, one per (institution, document
family, layout generation). Approximately fourteen at launch for the eleven institutions.

```yaml
id: jb-current-2024a
institution_family: CH_PRIVATE_BANK        # deliberately coarse; see §3.8
document_family: current_account_statement
languages: [de, en, fr]
fingerprint:                                # all must match, or no match
  page_size: [595, 842]
  producer_re: '^(Adobe LiveCycle|OpenText Exstream)'
  anchors_any:
    - 'Kontoauszug'
    - 'Statement of Account'
    - 'Extrait de compte'
  anchor_positions: { top_band: [0, 140] }
zones:                                      # y-bands in page units
  identity:  { pages: [1],    y: [140, 250], policy: DROP }
  summary:   { pages: [1],    y: [250, 320], policy: ALLOW_NUMERIC }
  ledger:    { pages: '*',    y: [320, 760], policy: ALLOW_GRID }
  footer:    { pages: '*',    y: [760, 842], policy: DROP }
columns:                                    # x-ranges within `ledger`
  booking_date: [40, 95]
  value_date:   [95, 150]
  narrative:    [150, 380]   # policy: DEFAULT_DENY (§3.5)
  debit:        [380, 450]
  credit:       [450, 520]
  balance:      [520, 570]
locale: { decimal: "'", thousands: ' ', date: 'DD.MM.YYYY' }
```

**A document with no matching template does not go to the model.** It goes to quarantine and the
household is asked to confirm the new layout. This is the single most important safety property in
the pipeline, because the failure mode the feasibility assessment worried about — banks silently
redesigning statements — becomes a *refusal* rather than a *leak*. The `template_match_rate` metric
(§3.9) is the early warning: it falls before anything goes wrong, not after.

Template authoring is the household's ongoing maintenance cost. Budget one to two hours per new
layout, roughly two or three times a year across eleven institutions.

**Stage 2 — Detection.** Four layers, unioned. Details in §3.3.

**Stage 3 — Pseudonymisation, not deletion.** Every detected span is replaced by a
**format-preserving pseudonym** of comparable length, so column geometry survives:

| Class | Real | Pseudonym |
|---|---|---|
| Person | `M. Jean DUPONT` | `‹P1›` padded to width |
| Organisation | `CARREFOUR ANNECY` | `‹O3›` |
| IBAN | `CH56 0483 5012 3456 7800 9` | *not emitted at all* (§3.4) |
| Customer number | `4711-882-03` | *not emitted at all* |
| Address | `12 rue des Alpes, 74000 Annecy` | *not emitted at all* |
| Date of birth | `14.03.2011` | *not emitted at all* |
| Amount, booking date, value date, currency, balance | — | **never redacted** |

Pseudonym indices are **stable within a document and random across documents**. Cross-document
stability would let a recipient link twelve months of statements into one profile; there is no
extraction benefit that justifies it.

**Stage 4 — Egress gate.** §3.6. **Stage 5–6 — Extraction and four-eyes.** §3.7.

**Stage 7 — Rehydration.** The model returns tokens; the local layer substitutes real values from the
redaction map. The map is encrypted under the same key as the document (§5) and never transmitted.

### 3.3 Detector inventory — four languages, ten formats

The question "how do you detect personal data across four languages" has a much better answer than
"a multilingual model." The highest-risk identifiers in this corpus are **a closed set the household
already knows**, and identity is not language-dependent.

**(a) Structural and checksummed — language-independent, high precision.** Every one of these is
validated, not merely pattern-matched, which is what keeps false positives from eating product codes
and reference numbers.

| Entity | Pattern | Validation | Where it appears |
|---|---|---|---|
| IBAN | `[A-Z]{2}\d{2}[A-Z0-9 ]{11,34}` | ISO 7064 mod-97-10; length by country: FR 27, CH 21, BE 16, LT 20, DE 22, LU 20 | CA, JB, Revolut (LT), Wise (BE), Linxea |
| BIC | `[A-Z]{6}[A-Z0-9]{2}([A-Z0-9]{3})?` | ISO 3166 country at pos 5–6 | transfer lines, headers |
| French RIB | `\d{5}\s?\d{5}\s?[A-Z0-9]{11}\s?\d{2}` | clé = 97 − (banque·89 + guichet·15 + compte·3) mod 97 | Crédit Agricole |
| PAN | `\d{13,19}` and masked forms `\*{4,12}\d{4}`, `\d{4}[ X]{8,}\d{4}` | Luhn on unmasked; masked forms matched literally | TopCard, CBA cards |
| AU BSB + account | `\d{3}-?\d{3}` followed by `\d{6,9}` | BSB institution prefix in a small allow-list (06x = CBA) | CBA NetBank exports |
| AVS/AHV (CH) | `756\.\d{4}\.\d{4}\.\d{2}` | EAN-13 check digit | JB onboarding docs, Pillar 2 certificate |
| NIR (FR) | `[12]\d{2}(0[1-9]\|1[0-2])(\d{2}\|2[AB])\d{8}` | clé = 97 − (nir mod 97) | payslips, some CA documents |
| TFN / ABN (AU) | `\d{9}` / `\d{11}` | TFN weights 1,4,3,7,5,8,6,9,10 mod 11; ABN mod 89 | Hostplus statements |
| Swiss QR reference | `\d{27}` (QRR) or `RF\d{2}[A-Z0-9]{1,21}` | mod-10 recursive / ISO 11649 mod-97 | Swiss payment slips |
| SEPA mandate (RUM) | `(?i)\b(RUM|ICS|MANDAT)\b.{0,40}` | contextual | Crédit Agricole direct debits |
| Email, URL, IP | standard | — | e-banking footers |
| Phone | E.164 plus national forms for +33 / +41 / +61 / +32 | libphonenumber parse | headers |
| Date of birth | date near `né(e)`, `geboren`, `data di nascita`, `date of birth`, `Geburtsdatum` | within 60pt of the trigger | JB, Livret A |

Note the four-language handling here: it is done with **trigger-word lists per language**, not with
language detection. A document containing any of `né(e)|geboren|nato|born` near a date gets that date
suppressed. Multiplying a 5-word list by 4 languages is 20 strings, and it is exhaustive in a way a
model is not.

**(b) Known Identifiers Table — the highest-value layer, zero machine learning.**

Seeded at onboarding and maintained thereafter: both partners' full names and every printed variant
(given-surname and surname-given order, `M. DUPONT Jean`, `DUPONT-MARTIN`, all-caps as banks print
them, accent-stripped forms), the son's name, home address components including street, number,
postcode and commune, all 26 account identifiers, all card last-fours, all customer and contract
numbers, all phone numbers and e-mail addresses, both dates of birth and the son's, employer name,
and both partners' Swiss and French tax identifiers.

Roughly 150 strings. Matched after NFKD normalisation, accent stripping, case folding and whitespace
collapse; token-level fuzzy match at Levenshtein ≤ 1 (≤ 2 in the OCR lane) to absorb scan noise.

This layer is the answer to "four languages, ten formats": **recall on it should be 1.0 by
construction**, and any miss is a bug with a specific reproducible cause, not a tuning problem. It
also catches the case a header-zone-only design misses — the household's own name appearing *inside a
transaction narrative* on the other partner's statement, which happens on every internal transfer.

**(c) NER — the open set only.** Counterparty persons, organisations and locations appearing in
narratives that are not in (a) or (b). A quantised multilingual token classifier (`distilbert-base-
multilingual-cased` NER class, int8 ONNX, ~135 MB) via ONNX Runtime Web with WASM SIMD, cached in
Cache Storage after first download.

**Be honest about this layer: it is the weakest one.** Generic multilingual NER on French banking
narratives — abbreviated, uppercase, no diacritics, full of merchant strings that are neither proper
prose nor clean entities — has mediocre recall. Published de-identification work consistently shows
domain shift hurting exactly this case. That is precisely why the design does not depend on it, which
is layer (d).

**(d) Narrative default-deny — the blunt instrument that actually works.** §3.5.

### 3.4 The account identifier: routable and identifying at the same time

This is the hard case the brief names, and the resolution is ordering rather than compromise.

The document is routable because the account number is on it. But **we** need to route it; the model
does not. So:

1. Detect the IBAN / RIB / BSB+account / card last-four locally (layer (a)).
2. Look it up in the Known Identifiers Table → resolve to an internal `account_id`, a random UUID
   with no relationship to the real identifier.
3. **If the lookup fails, the document does not egress.** It goes to quarantine with reason
   `UNKNOWN_ACCOUNT` and the household is asked which account it is — which also updates the table.
   Guessing an account from a partially-redacted document is exactly the mistake that produces
   double-counting and cross-account contamination, and it is not necessary.
4. Emit **nothing** derived from the identifier. Not the last four digits, not a hash, not a masked
   form.

Why not a hash or a stable HMAC? Because an IBAN's residual entropy is small — country, bank and
branch are structural and often guessable from context, leaving on the order of 10^9–10^11
candidates, which is a few seconds of brute force. A stable pseudonym is also a linkage key across
every document ever sent. Neither is worth anything, because:

The model does not need to know *which* account. It needs to know **what kind** of account, to select
the right schema and interpret the columns. So the payload carries a household-controlled descriptor
instead:

```json
"account": { "institution_family": "CH_PRIVATE_BANK", "product": "current", "currency": "CHF" }
```

Nine `institution_family` values cover all eleven institutions
(`FR_RETAIL_BANK`, `CH_PRIVATE_BANK`, `AU_RETAIL_BANK`, `EEA_NEOBANK`, `FR_BROKER`, `FR_INSURER`,
`CH_CARD_ISSUER`, `CH_PENSION`, `AU_SUPER`). The family is chosen to be *useful for parsing and
uninformative for identification*: knowing a document came from a Swiss private bank narrows the
world to a few dozen institutions, not to one.

**The dual role is therefore not resolved by sending a degraded identifier. It is resolved by
performing the routing before the egress, so the identifier is never needed downstream at all.**

### 3.5 The narrative: default-deny, then per-item consent

The transaction narrative is where the residual risk actually lives. The design's central move:

**Pass one does not send narratives at all.** The template gives the column geometry, so the ledger
grid can be serialised as dates, amounts, signs and currency with each narrative replaced by a
positional token:

```
 2026-07-03  2026-07-03  ‹N01›                    -1 847.22        12 004.31
 2026-07-25  2026-07-25  ‹N02›                    +12 438.55       24 442.86
```

The model's job in pass one is the numeric grid — column alignment, sign conventions, Swiss `'`
thousands separators, French comma decimals, credit/debit column vs. signed column. That is where
extraction errors actually hurt (INV9, and financial correctness's whole remit), and it needs no
narrative whatsoever.

**Categorisation, which does need the narrative, is handled locally.** A household merchant lexicon
maps normalised narrative strings to categories. It starts empty and is filled by the household in
the review flow. Bank statements are extraordinarily repetitive: the same twenty to forty merchant
strings recur every month. After two or three cycles the lexicon covers the large majority of lines.

**Only unmatched narratives escalate**, one at a time, as an explicit consented action: *"Send this
label to the extraction model to suggest a category? `CB PHARMACIE DES ALPES 12/03`"* with a
one-click *Categorise manually instead*. Each escalation is logged in the audit trail with a
timestamp and the exact string sent.

Consequences worth stating plainly:

- The household can run this system with **zero narrative egress, forever**, at the cost of typing a
  category for new merchants. That is a real option, not a theoretical one, and it should be the
  default setting at first launch.
- Article 9 inference — a pharmacy, a doctor, a union, a place of worship, a lawyer, a political
  party — is removed from the automatic path entirely. It can only leave through a deliberate act.
- Counterparty third parties (the person you paid, the employer who paid you) are protected without
  relying on layer (c)'s mediocre recall.

Narratives *are* stored locally in full, encrypted, and shown in the UI. Nothing about the
household's own view of its own data is degraded.

### 3.6 The egress gate

**The outbound payload is synthesised, not filtered.** This distinction is the difference between a
design that fails safe and one that fails open. A filter emits everything it did not recognise; a
synthesiser emits only what it was told to. Concretely, the payload is constructed from scratch out
of: zone-allowed spans, pseudonym tokens, and the household-controlled account descriptor. Content in
`DROP` zones is not redacted — it is never read into the builder.

A complete outbound payload for a Julius Baer CHF current account:

```json
{
  "schema": "statement.extract.v1",
  "doc":     { "ref": "8c41", "pages": 2, "template": "jb-current-2024a" },
  "account": { "institution_family": "CH_PRIVATE_BANK", "product": "current", "currency": "CHF" },
  "period":  { "from": "2026-07-01", "to": "2026-07-31" },
  "locale":  { "decimal": "'", "thousands": " ", "date": "DD.MM.YYYY" },
  "grid": [
    { "row": 1, "cells": { "booking_date": "03.07.2026", "value_date": "03.07.2026",
                           "narrative": "‹N01›", "debit": "1'847.22", "balance": "12'004.31" } },
    { "row": 2, "cells": { "booking_date": "25.07.2026", "value_date": "25.07.2026",
                           "narrative": "‹N02›", "credit": "12'438.55", "balance": "24'442.86" } }
  ],
  "summary_zone_text": "Saldovortrag 01.07.2026  13'851.53\nSaldo 31.07.2026  24'442.86"
}
```

Nothing in that payload identifies the household. That is the property the whole pipeline exists to
establish, and it is checkable by inspection — which is itself an argument for the synthesised form.

**Gate checks, all of which must pass before a single byte is written to the network:**

| # | Check | On failure |
|---|---|---|
| 1 | Payload validates against `statement.extract.v1` strictly, `additionalProperties: false` | abort, quarantine |
| 2 | Detector suite from §3.3(a) re-run **on the synthesised payload** | abort, P1 incident |
| 3 | Known Identifiers Table scanned against the payload (normalised, fuzzy) — **zero hits required** | abort, P1 incident |
| 4 | No content-block type other than text; no base64; no `document` or `image` blocks | abort, P1 incident |
| 5 | Payload size under 64 KiB (a whole PDF cannot fit through) | abort, quarantine |
| 6 | Egress destination is the relay host and nothing else (CSP `connect-src` allow-list) | blocked by CSP |
| 7 | Household kill switch is off | abort, local-only mode |

Checks 2 and 3 are also **production monitoring**, not just tests. `identifier_hits_at_gate` must be
zero for the lifetime of the system; a single non-zero reading is an incident with a root cause.

**Server-side re-enforcement at the relay.** The relay independently applies the *structural* half of
the gate: strict schema validation, mod-97 IBAN check, Luhn check, e-mail/phone/AVS/NIR patterns,
size cap, text-only. It deliberately does **not** hold a copy of the Known Identifiers Table — that
would put the household's identity, in a brute-forceable form, on the host we are trying to keep it
away from. The structural checks need no household secrets and catch the case that matters here: a
compromised partner device attempting to exfiltrate through the one endpoint that is allowed to send.

**Never sent, under any circumstance:** the original PDF, any rendered image of any page, any OCR
image crop, the redaction map, `doc_hash`, `account_id`, the Known Identifiers Table, filenames.

### 3.7 Four-eyes — contradicting the literal reading of D1

**D1 says:** "two independent extraction passes that must agree before a figure is committed."

**I am implementing:** one model pass, one deterministic local parse, and a mandatory arithmetic
closure check, with the same escalation semantics D1 specifies. I think the literal reading is worse,
and here is the argument.

*Two model passes are not independent.* Two calls to the same model over the same redacted payload
share a prompt, a tokenisation, a training distribution and a failure mode. They will agree on the
same misread of a Swiss apostrophe thousands separator, the same confusion of a value date with a
booking date, the same dropped continuation line. Agreement between correlated estimators is weak
evidence, and D1's whole purpose is to buy back the human confirmation the feasibility assessment
recommended. Correlated agreement does not buy it back; it produces confident wrong numbers, which
the assessment identifies as the worst outcome in the system.

*A deterministic parser is genuinely independent.* The template already carries the column geometry.
A parser that reads glyph runs by x-range, applies the locale's decimal and thousands rules, and
emits integer minor units shares nothing with a language model — different failure modes entirely.
When the two agree, that agreement is informative. It also runs offline, costs nothing per document,
and is a required component anyway for INV5 (fully usable with zero feeds) and for the kill switch.

*Arithmetic closure is stronger than either.* Opening balance + Σ(signed transactions) = closing
balance, in exact integer minor units (INV9). If a statement's own arithmetic closes, essentially
every class of extraction error is caught: a dropped line, a transposed digit, a sign inversion, a
misparsed separator, a merged continuation row. This check is free, deterministic and available on
every current-account and card statement in the corpus.

**Commit rule:**

```
commit  iff  model_pass == parser_pass                     (field-exact, integer minor units)
        AND  arithmetic closure holds exactly
        AND  every figure passes the materiality gate vs. account history
        AND  the account is not private to the other partner   (see below)
else    escalate to a human, showing both extractions side by side with the differing fields marked
```

Where no template parser exists yet (a new institution before its template is written), fall back to
two model passes with **materially different serialisations** — one gets the `grid` form above, one
gets a reconstructed plain-text page — and require arithmetic closure regardless. Never auto-commit
on model-vs-model agreement alone when closure fails.

Statement types with no closing arithmetic — a Pillar 2 certificate, a Hostplus member statement, a
Linxea valuation — get no free check, so **they never auto-commit.** They always escalate. There are
perhaps six such documents a year; a human confirmation on each is not a burden.

**The D2 × D3 trap, and how the four-eyes step resolves it.** D2 lets either partner upload any
document. If Partner B uploads a statement for one of Partner A's private accounts, the plausibility
check "is this figure consistent with account history?" requires reading A's private history — on B's
device. That would make upload a route to seeing a private account's contents, which D2 explicitly
forbids.

**Resolution:** when the local router resolves the document to an account whose visibility class is
private to the other partner, extraction proceeds on the uploader's device (she has the PDF anyway;
nothing can change that), but the result is sealed to the owner's key and enters a `pending_owner`
state. The plausibility and materiality gates run on the **owner's** device at next unlock, and only
then does the figure commit. B sees "1 document delivered to A" and nothing else — not the amount,
not the account, not the institution.

This is a real constraint on latency, not a cosmetic one: a figure can sit pending for days. It is
the correct trade.

### 3.8 Residual identity risk

Redaction as specified removes **direct identifiers** with high assurance and **indirect
identifiers** with moderate assurance. It does not produce anonymous data in the sense of GDPR
Recital 26. What leaves is pseudonymised personal data under Article 4(5), and pseudonymised data
remains personal data. Anyone describing this pipeline as "anonymising" statements is wrong, and the
household should not be told that.

What survives, in rough order of how much it actually gives away:

1. **The amounts.** Irreducible. A recurring CHF credit of an exact centime value on the 25th, a
   recurring EUR debit of an exact centime value on the 3rd. Two or three such figures are close to
   unique. No mitigation exists that does not destroy the product.
2. **The household's shape.** CHF salary + EUR mortgage + AUD superannuation + a Belgian EUR account
   + a Lithuanian-IBAN neobank. The *combination* is rare — a Geneva frontalier resident in
   Haute-Savoie with Australian ties. Redaction removes the name, not the silhouette. Partial
   mitigation: coarse `institution_family` values (§3.4) and never emitting jurisdiction beyond what
   currency already implies. Honest assessment: an adversary with the full year's payloads and an
   interest in this household could narrow it to a small population.
3. **Institution family plus product plus currency.** Narrows to a class, not an entity. Accepted.
4. **Escalated narrative labels.** The deliberate, consented leak. Each is one merchant string.
   Bounded by the household's own clicks and logged.
5. **Traffic metadata.** Request count, timing and size, visible to the relay host and the provider
   regardless of payload. Roughly ten requests clustered in the first week of each month. Mitigation:
   jitter dispatch across a 0–72 h window and pad payloads to 4 KiB buckets. This is a weak
   mitigation and is presented as such.
6. **Special-category inference.** Removed from the automatic path by §3.5; present in escalations.
7. **The son.** A third party who cannot consent. §8.4 removes him from the model path entirely.
8. **Retention at the provider.** §1. Assume two years.

**What redaction does buy, stated precisely:** the recipient cannot link the payload to this
household without the mapping table, and the mapping table never leaves the household's devices. That
is the actual security claim. It is worth having and it is not the same as anonymity.

### 3.9 Measuring whether it works

A redaction pipeline that is not measured is a redaction pipeline that fails silently, which is worse
than none because it purchases confidence. Eight measurements, in the order they should be built.

**1. A labelled corpus, built once.** Take three statements per format from the household's own
history — about thirty-five documents across fourteen templates — and hand-label every identifier
span. Roughly a day of work, once. Stored encrypted on the controlled tier, referenced by hash in the
repository, **never committed to git**. This is the ground truth everything else is scored against.

**2. Recall per entity class, not aggregate F1.** In redaction, recall is the safety metric and
precision is only a utility metric; report them separately and never average them into a single
number that lets a precision gain hide a recall loss. Use F2 if a single figure is needed. Targets:

| Class | Recall target | Rationale |
|---|---|---|
| Known Identifiers (layer b) | **1.000** | closed set; any miss is a reproducible bug |
| Checksummed structural (layer a) | **1.000** | deterministic; any miss is a pattern bug |
| Open-set PERSON / ORG / LOC (layer c) | ≥ 0.95, tracked | best-effort; not load-bearing because of §3.5 |
| Article 9 trigger terms | ≥ 0.99 | health/religion/union/political lexicon per language |

Headline number: **leak rate per 1,000 documents** — the fraction of documents where at least one
ground-truth identifier appears verbatim in the synthesised payload. Target zero; report the actual
count, never a rounded percentage.

**3. Canary tokens, run nightly, no new documents needed.** Inject synthetic identifiers into a copy
of one document per template at known positions: a fake IBAN with a valid mod-97 check, a fake
holder name in the identity block, the same fake name *inside a transaction narrative*, a fake AVS
number in the footer. Run the whole pipeline. A canary reaching the gate is a P1. This tests the
synthesiser and the zone policies, not just the detectors — which pure detector benchmarks never do.

**4. Mutation testing against the failure modes that actually occur.** Programmatically mutate the
labelled corpus and require recall to hold:

- name split across a line break (`DUPONT-` / `MARTIN`) — *this is the failure that happens*
- name inside a narrative rather than the header (`VIR DE M DUPONT JEAN`) — *this is the other one*
- accents stripped, or doubled (`DUPONT` / `Dupónt`)
- all-caps vs. mixed case; surname-first vs. given-first
- IBAN with non-standard grouping: `FR76 3000 6000 01…`, `FR7630006000 01…`, hyphens
- OCR confusions: `rn→m`, `0→O`, `l→1`, `5→S`, `8→B`
- a second holder name added (joint account with the other partner)
- narrative overflowing into the balance column
- a page rotated 90°, a page inserted, a duplicated page

**5. Production gate assertion as a live metric.** `identifier_hits_at_gate` and
`structural_hits_at_relay`, both expected permanently zero, both surfaced in the app. A control that
is also a metric is the only kind that stays honest over years.

**6. Coverage and drift.** `template_match_rate` (target ≥ 0.98) and `quarantine_rate` by reason. A
falling match rate is the early warning of a bank redesign, and it arrives *before* a leak rather
than after one, because an unmatched template cannot egress at all.

**7. An annual re-identification exercise.** Once before launch and once a year: hand one month's
actual outbound payloads to a person who does not know the household, give them the open internet,
and ask them to say anything true about the family. Record what they found. This is the only
measurement that tests residual *identity* risk rather than *identifier* recall, and it costs an
afternoon. Everything in §3.8 is a hypothesis until someone has tried.

**8. Extraction accuracy with redaction on — the anti-spiral control.** Score field-level extraction
accuracy on the labelled corpus *with redaction enabled*. If redaction costs accuracy, the fix is
always on the redaction side (pseudonymise instead of delete, preserve width, keep the zone) and
never on the loosening side. Making this a tracked metric with an owner is what stops the slow
weakening that kills every real-world redaction system.

**Release gate:** no template ships to the household until it passes 2, 3 and 4 on its own corpus
slice. No exceptions, including for the household's most-used format.

### 3.10 Failure handling

**Quarantine** — the document is stored encrypted, no egress occurs, an item appears in the review
queue. Enumerated reasons: `NO_TEMPLATE_MATCH`, `TEMPLATE_CONFIDENCE_LOW`, `UNKNOWN_ACCOUNT`,
`GATE_SCHEMA_FAIL`, `GATE_SIZE_FAIL`, `OCR_QUALITY_LOW`, `ARITHMETIC_NOT_CLOSED`, `EXTRACT_DISAGREE`,
`MATERIALITY_BREACH`, `IDENTIFIER_AT_GATE` (P1). Quarantined documents are auto-deleted after 30 days
unless the household acts (§6).

**Kill switch** — one household-visible toggle stops all egress. The app continues on the local
template parser plus manual entry. INV5 already requires the app to work with zero feeds, so this
costs nothing extra to build and gives the household a real lever the day a provider changes its
terms.

**P1 incident response** — an identifier at the gate halts *all* egress for every document until a
human clears it, records the document, the detector state and the offending span in a local incident
log, and adds a regression case to the corpus. Not a warning toast.

---

## 4. Encryption at rest and in transit

### 4.1 The decision that makes everything else easy

Two users, 26 account lines, a decade of daily balances and maybe 40,000 transaction rows. The whole
ledger is a few megabytes. **The server therefore never needs plaintext to do its job, so it never
gets it.** All computation — aggregation, FX conversion at display time (INV1/INV2), projections,
goal progress, the four views — runs on the client over a fully decrypted local copy.

This single decision removes most of the "hosting provider / insider" row of the threat model. It is
available here only because the dataset is small; it would not be available for a product with ten
thousand households, and it should be taken while it is on offer.

What it costs: no server-side search, no server-side reporting, no server-side reminder logic over
content. The last one needs an accommodation — see 4.3.

### 4.2 At rest

Three data classes.

**Class 1 — ledger records.** Encrypted client-side, per record group, before any network write. The
server stores `(row_id, visibility_class, ciphertext, wrapped_key_stanzas, version)`. `row_id` is a
random UUID. `visibility_class` is in clear because the server has to route sync — it is an
enumerated value (`joint | private_a | private_b | trust_son`) and carries no content.

**Class 2 — source documents and their redaction maps.** Encrypted client-side under a per-document
key. Stored in the content-addressed blob store keyed by a random blob id, *not* by `doc_hash` — a
content-addressed store keyed on the document hash lets anyone holding a candidate document confirm
its presence.

**Class 3 — operational metadata that must be legible to the server.** Sync cursors, blob sizes,
device registrations, `visibility_class`, `next_due_date`. Nothing else. Enumerated in a single
schema file so that adding a field is a visible, reviewable act.

**Algorithm.** XChaCha20-Poly1305 (libsodium-wasm) for everything the client encrypts. One algorithm,
no negotiation, no downgrade path. Chosen over AES-256-GCM because the 192-bit nonce makes random
nonces safe without a shared counter, and this is a multi-device client with offline writes where a
counter would eventually repeat. Every ciphertext carries a 1-byte version prefix for future
migration.

**Blob padding.** Ciphertext padded to the next 64 KiB boundary up to 1 MiB, then the next 1 MiB. A
compromised host still learns document counts and approximate sizes; padding removes the fingerprint
that distinguishes a two-page TopCard statement from a twenty-page portfolio report.

**Provider-side encryption.** Enable managed volume encryption and encrypted backups, but claim
nothing for them. They protect against a stolen disk, not against the host. **Backup encryption keys
are held by the household, not the provider** — otherwise the backup is the weakest link and the
whole client-side scheme is decorative.

### 4.3 In transit

- TLS 1.3 only. HSTS with `preload`, `includeSubDomains`, `max-age` 63072000.
- Certificate pinning is **not** used. It cannot be enforced from a PWA, it breaks on rotation, and
  claiming it would be false comfort. Said explicitly so nobody adds it to a checklist later.
- CSP: `default-src 'none'`, explicit `script-src 'self'` with per-build SRI hashes, no
  `unsafe-inline`, no `unsafe-eval` (note: this constrains the WASM loader — use
  `wasm-unsafe-eval`, which is the narrow form), `connect-src` limited to the app origin and the
  relay origin, `object-src 'none'`, `frame-ancestors 'none'`, `base-uri 'none'`.
  `Cross-Origin-Opener-Policy: same-origin`, `Cross-Origin-Embedder-Policy: require-corp`.
- `Referrer-Policy: no-referrer`. `Permissions-Policy` denying everything not used.
- Outbound to the relay: TLS 1.3, and the relay to the provider: TLS 1.3. No corporate proxy, no TLS
  interception anywhere in the path.
- **Push notifications carry no content.** The push service (FCM/APNs) sees existence and timing.
  Payload is a bare `{"n": 1}`; the app fetches and decrypts locally to render "1 item needs
  attention". Never an amount, never an institution, never an account name.

### 4.4 The limit of browser-delivered end-to-end encryption

**A compromised host can serve malicious JavaScript that exfiltrates keys.** No client-side
encryption scheme delivered by the same host it defends against can fully escape this. Stating it
plainly is part of the design, not a caveat to bury.

What the design does about it:

- **Installed PWA**, not a browsing session. The service worker is the update boundary and a
  household setting can hold a build for a review period rather than auto-updating.
- **Reproducible builds with a published manifest hash**, and the app displays its own running build
  hash in Settings. The hash is published to a location the app host does not control (a git tag).
  A targeted attack now requires either compromising two places or serving a build whose hash the
  household can check.
- **Strict CSP with SRI on every script.** No third-party origins at all — no CDN, no font host, no
  analytics, no error reporting.
- **Choose a host whose only job is blob storage.** The less application logic the host runs, the
  smaller the surface that can be quietly changed.

**Honest framing for the household:** this reduces the host threat from *passive reading of the
database and backups* — which it eliminates — to *an active, targeted, detectable attack on the
served code*. That is a large and worthwhile reduction. It is not a guarantee, and INV11's premise
(this aggregation is a more attractive target than any single bank login) means the reduction should
not be oversold.

---

## 5. Key management

**Requirement from security-by-design, mechanism specified here:** a compromised host must not be
able to read one partner's documents or the other's private accounts.

### 5.1 Hierarchy

```
Partner A                                        Partner B
─────────                                        ─────────
Platform authenticator (passkey, PRF)            Platform authenticator (passkey, PRF)
        │ HKDF-SHA-512(prf_output, "fa/dwk/v1")          │
        ▼                                                ▼
Device Wrapping Key  (per device, never leaves) ◄─── same construction
        │ wraps                                          │
        ▼                                                ▼
Identity Key Pair A  (X25519 + Ed25519)          Identity Key Pair B
        │                                                │
        ├──────────────┬──────────────┬──────────────────┤
        ▼              ▼              ▼                  ▼
   Household Key   Partner Key A   Son Subject Key   Partner Key B
   HK (sym)        PPK-A (sym)     SSK (sym)         PPK-B (sym)
   wrapped to      wrapped to      wrapped to        wrapped to
   A and B         A only          A and B (now)     B only
        │              │              │                  │
        └──────────────┴──────────────┴──────────────────┘
                       │ each wraps the per-item content keys
                       ▼
        Document Content Key (DCK, random 256-bit, one per document)
        Record Group Key (RGK, one per visibility class per epoch)
```

Every encrypted object carries a header of **wrapped-key stanzas**, one per recipient key, in the
style of `age`. Adding a recipient is adding a stanza; removing one is dropping a stanza and rotating.

**Visibility class → wrapping key**, and this is the whole enforcement mechanism:

| Visibility class | Wrapped to | Consequence |
|---|---|---|
| `joint` | HK | both partners read; host cannot |
| `private_a` | PPK-A | **Partner B cannot decrypt, and neither can the host** |
| `private_b` | PPK-B | Partner A cannot decrypt |
| `trust_son` | SSK | separate domain from day one (§8.4) |

D3's privacy model is therefore enforced by cryptography rather than by an access-control check the
server could get wrong or an insider could bypass. A compromised host holding the entire database
gets ciphertext and stanza headers. Reading `private_a` requires Identity Key A, which exists only
inside Partner A's devices, wrapped by a key that only that device's authenticator can produce.

### 5.2 Unlock

**Primary: WebAuthn passkey with the PRF extension.** On unlock the app calls
`navigator.credentials.get()` with `extensions: { prf: { eval: { first: <salt> } } }`, takes the
32-byte output, and derives the Device Wrapping Key via HKDF-SHA-512 with a per-device salt and a
fixed info string. The identity private key is stored on the device encrypted under that key, and is
imported into a **non-extractable** `CryptoKey` where the operation allows.

PRF is available in Chrome/Android and Safari 18+; Windows Hello gained the underlying `hmac-secret`
capability in the February 2026 servicing update. It is not universal, so:

**Fallback: passphrase.** Argon2id with m = 64 MiB, t = 3, p = 1 — above the OWASP floor and
survivable on a mid-range Android browser; measure on the household's actual phone and raise `m` to
the largest value that keeps unlock under 1.5 s. The passphrase is never transmitted and never
verified server-side; the wrong passphrase produces an AEAD authentication failure, not a login
error, which means there is no server-side oracle to attack.

**Never:** a password sent to the server; a server-side "reset my password and recover my data" flow;
key escrow with the host. Any of these silently converts the design in §4 back into "trust the host".

### 5.3 Recovery — the part that decides whether this is usable

E2EE household software that loses a decade of financial history when a phone falls in a lake is
worse than no software. Recovery is a first-class feature, and it is deliberately physical.

- **Recovery kit, one per partner.** At enrolment, the identity key is wrapped under a key encoded as
  a 24-word BIP-39-style mnemonic and printed. The household is walked through printing it and told,
  in the flow, to store the two kits in different physical places. The kit is generated exactly once
  and never shown again after the enrolment session closes.
- **Household escrow of HK.** HK is wrapped to Identity A, Identity B, *and* a recovery key derived
  from a third mnemonic held with the household's other important papers. This means one lost phone
  plus one lost recovery kit is survivable.
- **Explicit and stated limit.** If both partners lose every device *and* both recovery kits, the
  data is unrecoverable. That is the correct trade for a design whose premise is that the host cannot
  read the data, and it must be said in the onboarding flow in one plain sentence, not in a tooltip.
- **The sanctioned escape hatch.** A quarterly reminder to export a full plaintext snapshot (JSON +
  documents) to the household's own storage. This is a deliberate, visible, consented downgrade to
  "as safe as your own laptop", chosen by the household, rather than a hidden weakness engineered in
  by us. Ease-of-use owns making this feel like housekeeping rather than an alarm.

### 5.4 Rotation and revocation

- **Device loss.** Revoke that device's registration; rotate HK and the affected PPK; re-wrap DCK and
  RGK stanzas. At a few thousand documents this is a background job of minutes.
- **What rotation does and does not do.** It prevents future reads with the old key. It does **not**
  retroactively protect data a host already copied. Forward secrecy at rest against a host that took
  a snapshot is not achievable and is not claimed.
- **Scheduled rotation.** HK annually; DCKs never (they are per-document and single-purpose); identity
  keys only on compromise or device loss. Rotation for its own sake adds failure modes.
- **Algorithm agility.** The version byte plus the stanza format allow re-wrapping to a new
  primitive without re-encrypting payloads. Do not build negotiation; build migration.

---

## 6. Document storage and retention

INV3 requires source documents be kept for provenance. INV7 requires them hashed for idempotency.
GDPR data minimisation and the son's future erasure right pull the other way. The resolution is
tiered retention with a permanent, non-personal tail.

| Tier | Contents | Retention | Purpose |
|---|---|---|---|
| **D0** | Full encrypted original + redaction map | **24 months** rolling | "Why is this number what it is" for any live question; covers the French declaration cycle for year N in year N+1 |
| **D1** | Redacted page extract — the specific page region(s) the committed figures came from, plus the full extraction record and both passes' outputs | **10 years**, configurable | Satisfies INV3 literally: the figure is traceable to *the page it came from*. Far smaller and far less identifying than D0 |
| **D2** | `doc_hash`, `content_hash`, `institution_family`, `account_id`, `period`, ingest timestamp, extraction-record hash, redaction-pipeline version | **Permanent** | INV7 |

**D2 is how erasure and idempotency coexist.** A SHA-256 of a document is not reversible and, standing
alone against a random `account_id`, is not personal data in any practical sense. So an erasure
request deletes D0 and D1 and keeps D2 — and re-uploading that same document afterwards is still
correctly recognised as a duplicate and still does not double-count. **INV7 survives an Article 17
erasure without weakening either.** This is worth calling out because the naive design — delete
everything — silently breaks idempotency the first time a household re-downloads a statement they had
previously erased.

The ten-year D1 horizon is the household's chosen figure, informed by the fact that French tax
reassessment reaches back three years ordinarily and materially longer where foreign accounts have
not been declared — a live consideration for this household given Swiss, Australian and Belgian
holdings (D4). It is a setting, not a constant, and it is not legal advice.

**Other rules:**

- **Quarantined documents**: deleted after 30 days if the household does not act. They were never
  routed, so there is no provenance obligation attached to them.
- **Deletion is real deletion.** The blob is removed and the wrapped-key stanzas are destroyed; even
  a host that retained the ciphertext holds an object no key opens.
- **Backups.** Encrypted, 90-day retention, household-held key. A deletion writes a **tombstone** that
  is re-applied on any restore. The honest statement, which belongs in the privacy notice: *a deleted
  document may persist in encrypted backups for up to 90 days; it is unreadable to the host and is
  re-deleted automatically on restore.* Do not claim instant deletion; nobody's backups work that way.
- **No document is stored on the server before redaction has run**, because redaction runs before the
  first network write (§2). There is no "pending redaction" state on the host to leak.
- **Local caches.** Decrypted documents held in memory only; any on-disk cache is in Origin Private
  File System, encrypted, and cleared on lock. Browser print/preview paths are avoided for document
  display since they can spill to temp files — render in-canvas from the decrypted buffer.

---

## 7. Third-party exposure

| Third party | Sees | Volume | Control |
|---|---|---|---|
| **Model provider** | Redacted synthesised payloads: amounts, dates, currency, coarse product family, pseudonym tokens. Plus consented narrative labels. | ~10 docs/month | §3. Assume 2-year retention (§1). No-training is the API default. ZDR unobtainable at this scale — do not design as if it were. |
| **PSD2 aggregator** (Enable Banking or successor) | **Everything the bank exposes** for 4 tier-A accounts: full narratives, counterparty names, IBANs, holder name, balances, continuously. | continuous, 4 accounts | Choose an EU-established AISP; DPA; request the minimum scope (balances + transactions, no PII enrichment); keep the adapter thin so the aggregator is replaceable (the feasibility assessment's warning about GoCardless). **This is the largest single disclosure in the system** and the household should decide on it as deliberately as they decided D1. |
| **Managed host** | Ciphertext, blob sizes, timing, IP addresses, and the JavaScript it serves. | continuous | §4. EU region. DPA. Build-hash verification. |
| **Extraction relay** | Redacted payloads only. | ~10/month | Stateless, log-free, single choke point, structural gate (§3.6). |
| **ECB rates** | Which currencies you care about — **if you query per pair.** | daily | **Fetch the whole `eurofxref-daily.xml` file, never a per-pair query.** One request, no signal, and it is the correct engineering choice anyway. |
| **Market data (ISINs, D5)** | **Your exact portfolio composition**, if queried per instrument. This is a real leak that is easy to miss. | daily | Fetch by fund family or a superset list rather than exactly-held ISINs; cache aggressively; batch. If a provider only supports per-ISIN queries, note the exposure explicitly rather than pretending the request is anonymous. |
| **DVF (property)** | The commune you query — which is where the house is. | annual | Download the whole commune or département dataset. |
| **Push service (FCM/APNs)** | Notification existence and timing. | occasional | Content-free payloads (§4.3). |
| **Error/telemetry** | **Nothing. There is none.** | — | No Sentry, no analytics, no session replay, no third-party fonts. A local error log the household can export by hand. This is not austerity; a crash report from a financial app routinely contains the state that caused the crash. |

---

## 8. Data-subject rights

### 8.1 The household exemption, honestly

GDPR Article 2(2)(c) exempts processing by a natural person in the course of a purely personal or
household activity. Two partners keeping their own accounts looks like the paradigm case.

Four reasons the design does not rely on it:

1. **The CJEU construes it narrowly.** In *Ryneš* (C-212/13) a home security camera lost the
   exemption because it captured a fragment of public space — the requirement is that the activity be
   *purely* personal, and the court read "purely" strictly. A system holding a third party's account
   data and transmitting derived content to two commercial processors is further from the paradigm
   than a doorbell camera.
2. **Recital 18 points the other way for our suppliers.** The recital indicates the Regulation still
   applies to controllers and processors that provide the means for personal or household processing.
   Even on the most favourable reading, the exemption relieves the *household*, not the host, the
   relay, the aggregator or the model provider. It does not shrink the actual exposure; it shrinks the
   paperwork about it.
3. **The son is the weakest point.** He is a separate data subject whose data is processed by his
   parents. Whether the exemption covers processing *about a third party, including a minor, held
   under a power of attorney, using commercial cloud processors* is exactly the question nobody should
   want to litigate — and he will be an adult with his own view during this system's life.
4. **The exemption evaporates on any sharing.** Give an accountant read access, or extend the app to
   a parent's accounts, and it is gone — at which point the rights machinery has to exist anyway,
   retrofitted onto a decade of data that was not designed for it.

**Position: build the rights machinery, do not claim the exemption.** At two-and-a-bit data subjects
the machinery is cheap, and the exemption is a defence to enforcement rather than a design principle.
Where the design deliberately spends less than a commercial controller would — no DPO, no records of
processing, no DPIA as a formal artefact — say so as a proportionality judgement rather than as an
exemption claim. The honest version is "this is proportionate for a household of two", not "GDPR does
not apply to us".

### 8.2 Mechanisms

All of these run **on-device**, because the server cannot decrypt anything (§4).

**Access and portability (Art. 15, 20)** — *Export subject bundle*, one action, produces a signed ZIP:
the ledger records for accounts where the subject is a party, as JSON; all retained source documents
for those accounts, decrypted; the extraction records and both four-eyes passes; the redaction maps;
and every audit-log entry naming the subject. Machine-readable and human-readable.

**Scope rule:** a subject bundle is limited to accounts the requester is *a subject of*. Joint
accounts appear in both partners' bundles. A subject-access request is not a route around D3 — this
matters, because "GDPR says I can have all the data" is exactly the argument the privacy model has to
survive, and the software should answer it structurally rather than socially.

**Rectification (Art. 16)** — the ledger is append-only for auditability, so rectification is a
correcting entry plus a visible amendment marker, never a silent overwrite. The original figure and
the correction are both retained, both attributed. Financial correctness owns the semantics; this
document requires only that the original remain retrievable and marked.

**Erasure (Art. 17)** — deletes D0 and D1, retains D2 (§6), writes a tombstone that survives backup
restore. Erasure of a subject's ledger entries is a per-account operation: for the son, clean, because
his key domain is separate; for a partner, effectively account closure.

**Objection and restriction (Art. 18, 21)** — the kill switch (§3.10) is, operationally, a
restriction-of-processing control: it stops all third-party egress while leaving the ledger usable.

**Transparency** — a plain-language notice inside the app listing every third party from §7, what
each receives, and for how long. Version-controlled with the app; a change to it requires a change to
the table in §7.

### 8.3 Article 9 special categories

Transaction narratives can reveal health, religion, trade-union membership and political opinion.
This is not a hypothetical: a monthly direct debit to a medical practice, a pharmacy card payment, a
standing order to a place of worship. The control is §3.5's default-deny on narratives, which removes
these from the automatic egress path entirely. Locally they are stored and displayed, which is
appropriate — it is the household's own record of its own life.

### 8.4 The son

He has a Livret A held under power of attorney, he is a minor now, and he reaches majority during
this system's life. Five commitments, each concrete.

**1. His documents never go to a model.** Not redacted, not summarised, not at all. A Livret A moves
once or twice a year, on interest. Local template parsing plus manual entry covers it completely. The
convenience being purchased by sending a third party's statement — a third party who cannot consent —
to a commercial processor is worth about four minutes a year. That is not a close call.

*This narrows D1's scope. D1 says redact-then-extract-remotely; for this subject the answer is
extract locally. I am stating it as a deliberate carve-out rather than an exception.*

**2. His key domain is separate from day one.** SSK, not HK. Everything about him — ledger records,
documents, extraction records, audit entries — is encrypted under a key that is not the household key.
Handover at 18 is then a key operation taking seconds, not a data migration through a decade of
commingled records that nobody will have the appetite to do properly.

**3. His money is not counted as the household's.** The Livret A is displayed as a distinct *held in
trust* section: included in "assets under our administration", excluded from household net worth. This
is honest accounting and honest data protection at the same time. *Interface to financial correctness:
they own whether this survives contact with the projection maths; I own that it is a separate key
domain regardless of how it is presented.*

**4. A notice he can actually read.** A one-page, plain-language description of what the app holds
about him and why, written for a twelve-year-old, kept current, shown to him when his parents judge
it appropriate. At 15 — France's digital-majority age — he is offered read access to his own section
under his own credential.

**5. Majority is a trigger, not a reminder.** The power of attorney ends at his eighteenth birthday
and the account leaves the household's administration as a matter of law, so the data event is tied to
the legal event rather than to a calendar note somebody can snooze. Note also that from 16 he can
already withdraw from the Livret A himself absent his representatives' objection, which is a natural
prompt for step 4's access offer.

On his eighteenth birthday the app raises a **non-dismissible** task:

- Export his subject bundle and hand it to him, with the SSK.
- He chooses: **(a)** continue with his own access under his own credential and his own copy of the
  key; **(b)** full erasure — D0 and D1 destroyed, D2 hash tail retained, account record zeroed and
  closed.
- **Default if he does not respond within 90 days: (b).** The default has to be erasure, because the
  alternative default is that a decade of a young adult's financial records quietly stays in his
  parents' software because nobody got round to the conversation.

**Consequence, stated openly:** option (b) breaks INV3 for his historical figures. They keep their
provenance metadata — institution, period, extraction record hash — but the source document is gone
and cannot be re-derived. **Rights win over the invariant.** This is a deliberate, argued breach and
it is the right outcome: an invariant written to make the household's own numbers trustworthy should
not be read as a claim on a third party's data after he has asked for it back.

---

## 9. Invariants: held, strained, broken

| Invariant | Status | Note |
|---|---|---|
| INV1 native currency | held | client-side encryption is currency-blind; nothing here persists a conversion |
| INV2 per-user display currency | held | per-user setting is client-side state |
| INV3 provenance | **strained, and broken once by design** | tiered retention keeps the page extract for 10 years rather than the full document for ever (§6); an Article 17 erasure destroys the source and keeps only provenance metadata (§8.4) |
| INV4 freshness | held | staleness is orthogonal |
| INV5 usable with zero feeds | **held and relied upon** | the local template parser and manual entry are load-bearing for both the four-eyes design (§3.7) and the kill switch (§3.10) |
| INV6 no bank credentials | held | nothing here stores or wants them |
| INV7 idempotent ingestion | **held, including through erasure** | dual hashing (§3.2 stage 0) plus the permanent D2 tail (§6) |
| INV8 redaction precedes egress | **held by construction** | redaction runs before the first network write (§2) |
| INV9 exact monetary types | held | integer minor units through the parser, the closure check and the commit rule |
| INV10 decomposable change | held | not touched here |
| INV11 aggregation is a privilege | **held, with a stated limit** | client-side E2EE plus per-partner key domains; the limit is host-served JavaScript (§4.4), stated rather than hidden |

**Decisions contradicted, with the argument in the body:**

- **D1, literal reading — "two independent extraction passes."** Implemented as one model pass, one
  deterministic local parse, and mandatory arithmetic closure. Argued in §3.7: two model passes are
  correlated, which is the failure mode this check exists to prevent; a deterministic parser is
  genuinely independent, free per document, and needed for INV5 anyway.
- **D1, scope — for the son's documents nothing goes remote at all.** §8.4.
- **D2 × D3 interaction** is not contradicted but is constrained: extraction for the other partner's
  private account completes only on the owner's device, so a figure may sit pending for days (§3.7).

---

## 10. Interfaces required from other agents

**Security-by-design.**
1. A stable, enumerated `visibility_class` on every account (`joint | private_a | private_b |
   trust_son`), assigned at account creation, mandatory. §5.1 maps each class to a wrapping key; if
   the enumeration is open-ended or mutable, the key hierarchy has to change.
2. Authentication with a platform authenticator that supports a key-derivation path — WebAuthn PRF
   preferred, with a passphrase fallback. If authentication is designed around a server-verified
   password, §5.2's "no server-side oracle" property is lost.
3. The rule for authorising a visibility-class change, and for making it visible to both partners
   (D3 requires the latter). §5.4 supplies the mechanism — a change is a re-wrap, so it is
   cryptographically observable — but the *policy* (both signatures? one plus a mandatory
   acknowledgement?) is theirs.
4. Device revocation semantics, since revocation drives key rotation.

**Architecture.**
1. The controlled component for INV8 is the uploading device's PWA (§2). If it moves, tell me.
2. Server surface: content-addressed blob store, opaque row store, stateless relay. No server-side
   content processing, no search index, no server-side document rendering.
3. The relay is the only endpoint that receives document-derived content, holds the provider API key,
   logs no bodies, and enforces the structural gate.
4. Reproducible builds with a published manifest hash (§4.4).

**Financial correctness.**
1. The materiality gate: what constitutes an implausible figure against account history, per account
   type. §3.7 requires the check; the threshold is theirs.
2. Confirmation that arithmetic closure is the right invariant per document family, and which
   families have none (Pillar 2, Hostplus, Linxea) and therefore never auto-commit.
3. Append-only ledger with amendment markers, so Article 16 rectification is a correcting entry.
4. Whether the son's Livret A can be presented as *held in trust* — excluded from household net worth,
   included in assets under administration — without breaking the projection maths.

**Ease of use.**
1. The narrative escalation prompt (§3.5) — one label, one decision, an obvious "categorise manually".
2. The quarantine queue, with the ten enumerated reasons rendered as things a person can act on.
3. The recovery-kit ritual at enrolment (§5.3), including the unrecoverability sentence.
4. The son-at-18 task, non-dismissible, with erasure as the 90-day default.
5. Sending a document to the other partner's private account produces "delivered", nothing more.

---

## 11. Build order and cost

Ordered so that each step either removes a risk or unblocks the next.

1. **Key hierarchy and unlock** (§5.1–5.3). Everything else stores data; this decides who can read it.
   Recovery kit in the first version, not the second — retrofitting recovery onto existing keys is
   painful.
2. **Client-side encrypted storage** (§4.1–4.2) with the server as a blob store. Do this before there
   is data to migrate.
3. **Template registry, local parser and arithmetic closure** (§3.2, §3.7). This alone makes the app
   usable with zero egress and satisfies INV5. Ship it and live on it for a month.
4. **Detector layers (a) and (b)** (§3.3) plus the labelled corpus and the canary harness (§3.9).
   Detectors and their measurement in the same step; a detector without a corpus is a guess.
5. **Egress gate and relay** (§3.6). Now — and only now — a payload can leave.
6. **Model extraction pass and four-eyes commit rule** (§3.7).
7. **Narrative escalation and merchant lexicon** (§3.5).
8. **NER layer (c)** (§3.3). Last, because §3.5 means it is a refinement rather than a dependency.
9. **Retention tiers, tombstones, subject-bundle export** (§6, §8.2).
10. **The son's key domain from the beginning**, but the majority-trigger task can come later.

**Cost against A1.** Model calls: ~10 documents a month, one Opus 5 pass each over a payload of a few
kilobytes, plus occasional narrative escalations — a few dollars a month, well inside "a streaming
subscription". Storage: a few hundred megabytes of encrypted blobs. Relay: the smallest instance
available. Market data is the only line item with a real price and it belongs to financial correctness.

**The expensive thing is engineering time on the redaction pipeline and the corpus**, and it is
front-loaded: roughly a day to label the corpus, one to two hours per template, and the detector and
gate work. Recurring cost is two or three new templates a year. Flagging this because it is the part
that gets quietly dropped when a schedule slips, and dropping it is what turns D1 from a control into
a claim.

**Two cost facts that should not be assumed away:** zero data retention requires a negotiated
arrangement and is not obtainable at household scale; and even with it, flagged content may be
retained for up to two years (§1).

---

## 12. Open questions

1. **Does Partner B accept D3?** Not a data-security question, but §5.1 hard-codes the answer into a
   key hierarchy. Confirm before enrolment, because changing the model later means re-wrapping every
   stanza and re-deriving every aggregate.
2. **Which relay host, and can it credibly promise no body logging?** A managed function platform
   that logs request bodies by default is disqualified regardless of its other merits.
3. **Does the household want narrative escalation at all,** or should the merchant lexicon be
   entirely manual? The zero-narrative-egress option is real (§3.5) and should be offered as the
   default at first launch, then relaxed only if the household asks.
4. **The market-data provider's query shape** (§7). Per-ISIN queries disclose the portfolio. This
   needs deciding with financial correctness before D5's mark-to-market is built.
5. **Does Crédit Agricole print the son's name and date of birth on the Livret A statement**, and in
   which zone? Determines whether his documents can be handled by a template parser with no detector
   layer at all.
6. **Recital 18's exact wording** should be checked against the Official Journal before anything in
   §8.1 is quoted to a third party; it is paraphrased here from secondary sources.
7. **Android in-browser NER performance** on the household's actual phone. If a 135 MB ONNX model is
   unusable there, layer (c) becomes desktop-only — which §3.5 makes survivable, but it should be
   measured rather than assumed.

---

*Third-party retention and eligibility statements were verified against published provider
documentation in August 2026 and change frequently; §7 is a living table and the app's transparency
notice is generated from it. Legal observations are structural, not advice.*
