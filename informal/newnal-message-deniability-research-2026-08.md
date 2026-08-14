# Deniability beyond the seven catalogued approaches — literature survey, 2026-08

Companion to `newnal-message-deniability.md`. That document catalogues seven
approaches (§2) and adopts idea ② (session-key disclosure). This one records an
automated literature survey run on **2026-08-14** asking what exists **outside**
those seven, and what 2024–2026 work says about the ones inside.

It is a **research record, not a design change.** Nothing here has been adopted.
Where a result bears on the adopted design, the last section says so explicitly
and leaves the decision open.

---

## 0. Read this first — the verification infrastructure failed

**Every scholarly host was blocked by this environment's egress proxy**:
`eprint.iacr.org`, `iacr.org`, `ia.cr`, `petsymposium.org`, `dl.acm.org`,
`arxiv.org`, `doi.org`, `dblp.org`, Crossref, OpenAlex, Semantic Scholar,
`infoscience.epfl.ch`. Reproduced first-hand (`CONNECT eprint.iacr.org:443 ->
HTTP/1.1 403 Forbidden`), with the proxy's own status log showing repeated
`connect_rejected` entries. The search budget was also exhausted (200/200).

Confirmed substance therefore came **only** from hosts that happened to be
reachable: GitHub (draft source repos, cryptobib, DBLP dumps, PoPETs Artifact
Evaluation Committee records, a conference-talk subtitle file) and authors'
homepages.

**Consequence: 20 of 25 verified claims were voted down, and the dominant
refutation reason was "quote not verifiable against primary text" — not "claim
is false."** Absence of confirmation in this document is **not** evidence of
absence. §6 lists what to re-fetch from a session with scholarly egress.

### Run metadata

| | |
|---|---|
| Date | 2026-08-14 |
| Method | 6 search angles → parallel search → source fetch → claim extraction → 3-vote adversarial verification (2/3 refutes kills) → synthesis |
| Agents | 106 (0 errors) |
| Sources fetched | 23 |
| Claims extracted / verified | 80 / 25 |
| Confirmed / killed | 5 / 20 |
| Findings after synthesis | 7 |
| Duration | ~2.4 h |

### Confidence labels used below

* **high** — primary text read directly and quoted verbatim.
* **medium** — load-bearing substance confirmed from a reachable secondary
  artifact (e.g. the authors' own recorded talk), primary PDF unreachable.
* **low** — abstract text relayed by a search index only; treat as a lead.

Anything marked "my inference" is the survey's reasoning from mechanism
descriptions, **not** a claim made by the cited source. No source located in
this pass reasons about our specific constraint set (≤20-member rooms, mobile,
unmodified-client interop), so **all per-approach interop and in-group
authenticity columns below are inference.**

---

## 1. Bottom line

Little confirmed cryptographic machinery exists beyond the seven catalogued
approaches, and what was confirmed mostly **undermines the framing of the
question** rather than adding options.

Three results are solid enough to act on:

1. **No IETF-standardised deniability anywhere in MLS/MIMI.** There is no design
   to port. (high)
2. **Attested-hardware peers defeat the recipient-forgery family** — catalogue
   items ③ and ④ and the DAKE/designated-verifier family generally — while
   leaving **publicly-forgeable** designs (①, ②, ⑤) largely intact. This is an
   argument *for* the adopted choice. (medium)
3. **Deniability fails at the system layer, not the crypto layer.** Signal —
   the canonical deployed "deniable" messenger — is not deniable in practice
   because the server authenticates the sender before relaying. Matrix's system
   layer is heavier than Signal's. (medium)

The most internally consistent reading of the whole survey is uncomfortable:
attestation kills the recipient-forgery family; the whole-system result kills
protocol-only fixes; an MLS author's objection kills plaintext-only storage
against a determined peer; and the one legal survey says none of it is exercised
in court anyway. **What is left standing is the publicly-forgeable subset already
in our catalogue (② and ⑤) plus the unlinkability axis (⑦) that the primary
authors themselves recommend instead.**

---

## 2. Findings

### F1 — No IETF specification of deniable authentication for MLS or MIMI

**Confidence: high** (3-0)

In `draft-mahy-mimi-identity`, "Deniable credentials" appears only as a bare
bullet under *"Other possible mechanisms … not investigated due to a lack of
time"* — no prose, no citation, no construction. The gap persists unchanged from
**draft-04 (2025-10-20) into draft-05 (2026-07-06)**.

Two corrections to how this is usually cited:

* The document is the **individual, non-WG-adopted** "MIMI Identity Concepts"
  I-D ("not endorsed by the IETF and has no formal standing") — **not** the MIMI
  identity architecture draft, which is `draft-barnes-mimi-identity-arch-03` and
  also contains no deniability mechanism.
* The **adopted** MIMI WG documents (`mimi-content-09`, `mimi-protocol-05`,
  `mimi-room-policy-04`) contain nothing either.

**Consequence: MLS/MIMI offers no design to port into Matrix, and any "MLS is
deniable" assertion is not standards-backed.**

Evidence: the verifier read the primary text in the editor's own designated
source repo (datatracker and ietf.org were egress-blocked) and confirmed the
quote verbatim in both -04 and -05: *"Below are other mechanisms which were not
investigated due to a lack of time. […] - Deniable credentials"*. §6.4 of that
draft lists anonymous credentials (RFC 9576, `draft-schlesinger-cfrg-act`) and
ZKP/JWP (`draft-ietf-jose-json-web-proof`) in the same not-investigated list —
the nearest adjacent primitives anyone might build a deniable credential from.
The nearest counterexample was searched for and found wanting: RFC 9750 has one
conditional sentence on deniability, a precondition rather than a mechanism.

Sources: `datatracker.ietf.org/doc/draft-mahy-mimi-identity/` ·
`github.com/rohanmahy/mimi-identity` (editor's canonical source repo;
`versioned/draft-mahy-mimi-identity-04.txt` and `-05`) ·
`datatracker.ietf.org/doc/draft-barnes-mimi-identity-arch/` · RFC 9750.

**Time-sensitivity:** the draft is at -05 as of 2026-07-06, so any citation to
-04 is nine months stale. Check -06+ before publishing anything from this.

---

### F2 — A candidate eighth approach: deniable *provenance of the signing key*

**Confidence: low** (0-3 as originally submitted against a GitHub copy of
`draft-ietf-mls-architecture`; the underlying RFC 9750 sentence was
independently quoted by the verifier of F1)

The only deniability lever the MLS standards text itself names is **deniable
provenance of the signing key**, not weakening or removing the per-message
signature. RFC 9750: *"Message content can be deniable if the signature keys are
exchanged over a deniable channel prior to signing messages"* — characterised by
the verifier as *"a conditional statement that deniability depends on
out-of-band deniable key exchange, not an MLS mechanism."*

Mapped onto Matrix this would be a distinct eighth approach: keep Megolm's
per-session Ed25519 signature exactly as it is, and attack attributability at
the **key-publication / identity layer** so nobody can prove the signing keypair
was ever yours.

| Dimension | Assessment (inference) |
|---|---|
| In-group authenticity | Survives fully |
| Wire format / algorithm ID | **No change** — the most interop-friendly option in the catalogue |
| Holds against hoarding peer | **No** |
| Maturity | Standards prose only; no construction, no prototype |

**Why it is dead on arrival in today's Matrix:** device Ed25519 keys are
published to the homeserver and bound by **cross-signing** to a user identity.
The peer hoards that binding alongside the messages, so "this key was never
mine" fails unless the entire key-distribution path changes. It also collapses
substantially into catalogue item ⑦ and inherits its objection — some issuer or
server retains the mapping.

The Matrix mapping and the cross-signing objection are **inference**, not from
any verified source.

Sources: RFC 9750 (MLS Architecture), Intended Security Guarantees ·
`github.com/mlswg/mls-architecture/blob/main/draft-ietf-mls-architecture.md`.

---

### F3 — Remote attestation is the sharpest objection, and it cuts unevenly

**Confidence: medium** (2-1; a companion claim naming Signal and OTR as the
specific circumvented protocols was 0-3, and a companion claim that the paper
also proposes attestation as a *defence* was 1-2 — see F7d)

Verbatim from the abstract (quoted, not paraphrased): *"We show how an adversary
can use remote attestation to undetectably generate a non-repudiable transcript
from any deniable protocol (including messaging protocols) providing sender
authentication, proving to skeptical verifiers what was said."*

**Mechanism.** The adversary confines the recipient's long-term secret inside an
attested enclave and attests that the enclave never released it. The recipient
can then prove to a third party that it *could not* have produced the
transcript — undetectably from the sender's side — converting a deniable
protocol into a non-repudiable one.

**It defeats every design whose deniability rests on "the designated verifier
could have forged it":**

| Catalogue item | Under an attested peer |
|---|---|
| ③ Pairwise MACs from the Olm secret | **Defeated** |
| ④ Ring / designated-verifier signatures | **Defeated** |
| X3DH/PQXDH offline deniability, DAKE generally | **Defeated** |
| ① Drop the signature | Largely intact |
| ② Publish the session Ed25519 key | Largely intact |
| ⑤ Time-locked forgery opening | Largely intact |

For the publicly-forgeable designs the enclave can attest only *"these bytes
verified under a key many parties hold"*, which proves nothing about authorship.
The residual attack degrades to **timestamped receipt** — i.e. the ordinary
contemporaneous-notarisation objection (§4.1 of the design spec) rather than this
paper's key-monopoly result.

**Two qualifications.**

1. It relocates trust to a TEE vendor root that has been broken repeatedly —
   Foreshadow/L1TF (2018), **SGAxe (2020, attestation-key extraction enabling
   forged quotes)**, AEPIC Leak (2022), SGX.fail (2023) — so an informed sceptic
   can discount a quote. *(This list is the verifier's contextual knowledge, not
   the paper's.)*
2. The 2018 SGX instantiation is **stale** (Intel removed SGX from client CPUs
   from 11th-gen Core), but the substitutes are **stronger for mobile Matrix
   clients**: Android hardware-backed key attestation / Play Integrity, Apple
   Secure Enclave / App Attest, ARM CCA, AMD SEV-SNP, Intel TDX.

**Net: attested-peer capability is more available in 2026 than in 2018 — a live
argument for publicly-forgeable designs over designated-verifier ones.**

**Not established:** the literal universality of "*any* deniable protocol". Full
text unreachable, so the formal model, prototype performance, the paper's stated
limits, and **whether it covers the timestamped-receipt variant** are unverified.

Source: `eprint.iacr.org/2018/424.pdf` — Gunn, Vieitez Parra, Asokan,
"Circumventing Cryptographic Deniability with Remote Attestation", PoPETs
2019(3).

---

### F4 — Deniability must be evaluated at the whole-system level; Signal fails there

**Confidence: medium** (2-1; the paper's bibliographic identity separately 3-0;
a companion claim spelling out the server-relay mechanism in the receiver's
local commit path was 0-3 on quote-fidelity grounds, though the RWC transcript
corroborates its substance)

Collins–Colombo–Huguenin-Dumittan, "Real-World Deniability in Messaging",
PoPETs 2025(1):320–340, DOI 10.56553/POPETS-2025-0018. **Peer-reviewed and
artifact-evaluated** (PoPETs AEC badges available / functional / reproduced) —
not a stale preprint.

X3DH/PQXDH offline deniability is **not disputed**. The break is architectural.
Above the handshake sit multi-device, multiple groups, and above all
**authentication to the server**, which in the authors' words *"harms
deniability in practice … independent of the crypto somehow, so if you plug in
these other schemes, deniability is also lost."*

Sealed sender makes it worse: the delivery certificate Alice obtains *"binds
Alice to this communication … this certainly limits Alice's deniability."*

**Consequences.**

* A Matrix/Megolm design **cannot justify itself by pointing at Signal's
  protocol-level proof.**
* *(Inference, not the paper's claim)* Removing or weakening Megolm's
  per-session Ed25519 signature leaves Matrix's system-layer attribution
  untouched — token-authenticated event submission, the PDU `sender` field,
  origin-server signatures and the event DAG are exactly the corroborating
  server-side authentication events the paper identifies as unkillable by any
  pairwise-MAC or deniable-AKE argument.
* **The framing "the Ed25519 per-session signature is what destroys deniability
  today" is at most half true and should be re-examined before scoping protocol
  work.**

See §4 for a Matrix-specific check that materially narrows this inference.

Evidence path: all PDF hosts were egress-blocked, but the verifier recovered the
authors' **RWC 2023 talk transcript** and confirmed every load-bearing element
in Collins's own words, including his explicit acknowledgement of — and refusal
to rely on — the composition theorem showing X3DH + Double Ratchet deniable.
Bibliography corroborated by four independent sources (cryptobib
`crypto_PoPETS25.bib`, a DBLP dump, PoPETs AEC records, and all three authors'
homepages).

**Two details weaker than usually cited:** the abstract's "formal model" is
stronger than Collins's own talk description (*"not quite a cryptographic model
but something a little less"*), and **DKIM is never mentioned in the talk**, so
the DKIM half of the result rests on search-index abstract text alone.

Sources: `eprint.iacr.org/2023/403` (full version) ·
`petsymposium.org/popets/2025/popets-2025-0018.pdf` · RWC 2023 talk transcript
(LemonSec/tipofmytongue, "Real World Crypto 2023/Messaging and
Encryption.eng.srt", talk begins ~01:31:41) · `github.com/si-co/rwdm` (artifact).

---

### F5 — The same paper backs item ⑥ and argues against the whole project

**Confidence: medium** (the item-⑥-remedy claim was voted 0-3 twice on quote
fidelity, but its substance plus both Q&A objections were confirmed in the
verified RWC transcript underlying F4)

**Support for ⑥.** The paper's concrete remedy for Signal is **client-side
editable local storage** — let the user modify or add locally stored messages
through the shipped UI, making the app itself the forgery tool. This is
peer-reviewed backing for catalogue item ⑥ and implies **the missing ingredient
is UI-level forgeability, not a new signature scheme.**

**Argument against the project.** The same work finds deniability *essentially
unexercised in real disputes*, and **Collins closes by questioning whether
deniability is even desirable, suggesting anonymity / metadata-minimisation is
the better axis** — i.e. an argument for catalogue item ⑦ over items ①–⑥.

**Two published objections to ⑥ specifically**, both from the recorded Q&A:

1. An **MLS author** argues clients normally discard ciphertext and signature
   anyway, so *"only the plaintext is kept and that is completely deniable
   again"* — making ⑥ close to a **no-op relative to current client behaviour.**
2. A determined framer defeats even a perfectly deniable protocol by running the
   client inside an SGX enclave — F3 applied directly to ⑥, and the reason ⑥
   does not hold against a sufficiently determined hoarding peer.

| Dimension for ⑥ as the paper frames it | |
|---|---|
| In-group authenticity | Unaffected (nothing changes on the wire) |
| Wire format / algorithm ID | No change; unmodified clients interoperate |
| Holds against hoarding peer | **No** — attested client, or simply timestamp + notarise receipt |
| Maturity | Proposed remedy in a peer-reviewed paper with a public artifact repo; no shipped implementation |

**Cherry-picking hazard, flagged by the verifier:** this source is as much
ammunition *against* building cryptographic deniability into Matrix as it is
against citing Signal.

---

### F6 — The counter-case has been measured once, weakly, and the effect is zero

**Confidence: medium** (existence and direction corroborated inside the 2-1
verification of F4; the n=140 formulations voted 0-3 **twice**, from two
different source URLs)

The same authors surveyed **openly accessible court records in French-speaking
Swiss cantons** where messaging conversations were entered as evidence.
Verbatim from the verifier reading the RWC transcript: *"empirical leg: survey
of openly accessible court records in French-speaking Swiss cantons; transcripts
(often screenshots) routinely admitted, only one failed validity challenge
found."* **Deniability was never raised.**

**This is the only empirical measurement located, and its measured effect on
outcomes is zero.**

**Do not repeat these versions** — both failed verification because the number
came only from search-index abstract text:

* the specific "**140 cases**" count (0-3);
* the phrasing "reviewed 140 Swiss court cases … deniability was never raised or
  considered" (0-3).

Treat **direction and existence as established, the exact n as unverified**, and
the scope as a serious external-validity limit: **one country, one language
region, openly accessible records only.** Nothing measures common-law evidence
practice, adversarial cross-examination, or forensic device imaging.

A separate claim citing `eprint.iacr.org/2025/1949` for the stronger thesis that
**courtroom process already assumes all evidence is forgeable, so deniability
buys no evidentiary exclusion**, was voted 0-3 and must be re-fetched. If real it
is the most direct statement of the counter-case and **the single highest-value
item on the re-fetch list** for this question.

---

### F7 — Four candidate mechanisms outside the seven; all unverified

**Confidence: low** (0-3, 0-3, 0-3, 1-2 respectively — **all on provenance, none
on substance**)

Every one of these rests on abstract text relayed by a search index, never on
fetched primary text. They are **leads, not findings.**

#### (a) Epochal signatures — `eprint.iacr.org/2020/1138`

**The single most promising unverified lead for the Matrix case.**

A near drop-in replacement for ordinary digital signatures that binds each
signature to a **discrete epoch** and evolves the secret key at epoch
boundaries: **unforgeable within the live epoch, forgeable by anyone once the
epoch expires.** In effect a peer-reviewed, formally modelled version of
catalogue item ⑤ — **without needing an external delay service such as drand.**

Two claimed properties matter directly for us, and reportedly avoid what killed
mpOTR deployment:

* **No pairwise key establishment** among participants — Megolm exists precisely
  to avoid O(n) pairwise work.
* **Members can be added or removed without re-initialising the session** —
  normal room churn.

Its stated precondition is that the protocol *would already be deniable if the
signature were simply deleted* (= catalogue item ①), so it is best read as
**the fix that recovers the in-group authenticity item ① throws away.**

| Dimension, if it verifies (inference) | |
|---|---|
| In-group authenticity | Survives **within the epoch** — that is the point |
| Wire format / algorithm ID | **New signature algorithm ID → unmodified Matrix clients would not verify.** An interop break; this is the main cost against item ② |
| Holds against hoarding peer | Only for **expired** epochs; the peer can notarise within the live epoch |
| Maturity | Paper only; no known prototype |

#### (b) Split-KEM deniable handshake — `eprint.iacr.org/2025/853`

Claims MLWE split-KEMs make **post-quantum deniable X3DH** as compact as the
ring-signature route at lower computational cost — i.e. for the *pairwise
handshake*, item ④ is dominated. **But the property at stake is offline
deniability of the key exchange, not per-message group authorship, so it does
not touch Megolm's signature.** With the host unreachable, even the paper's
existence and topic rest on search-index snippets.

#### (c) KEM-only impossibility + DVS template — `eprint.iacr.org/2021/769` (Brendel et al.)

Claims **no construction from KEMs alone** gives asynchrony, deniability and
compromise resilience simultaneously, because encapsulate/decapsulate is
asymmetric — only the encapsulator's contribution is "free", so the responder
cannot symmetrically simulate a transcript the way DH allows. The positive
result is a generic template: **static KEM + ephemeral KEM + designated-verifier
signature**, with PQ DVS built directly or from ring signatures.

If true this answers the post-quantum sub-question — **PQ KEMs do break the
classical deniability argument, and DVS is the accepted route back** — which
**collides head-on with F3**, since DVS deniability is exactly what attestation
removes. See the open question in §6.

#### (d) Attestation as a *defence* — `eprint.iacr.org/2018/424`

The same paper as F3 reportedly also argues that protocols can be designed to
stay deniable against an attestation-capable adversary, and that **attestation
can itself restore deniability** by ruling out realistic adversary classes. A
mechanism not among the seven. Voted **1-2** — the closest any lead came to
surviving.

---

## 3. Impact on the catalogue in `newnal-message-deniability.md`

Summarising the above against §2 of the design spec. **"Weakened" / "supported"
here means by this survey's evidence, at the confidence stated — not a design
decision.**

| Item | Effect of this survey |
|---|---|
| ① Drop the signature | **Supported** as a family (publicly forgeable ⇒ survives attestation, F3). Still loses in-group authenticity; F7a claims to be the fix for exactly that |
| ② **Session-key disclosure (adopted)** | **Supported.** Survives the attestation attack that kills ③/④ (F3). Residual exposure is timestamped receipt — the objection §4.1 already states. Open question: does an enclave-attested timestamp defeat it after all? |
| ③ Pairwise MACs | **Weakened, seriously.** Defeated by an attested peer (F3). Deferring it now has a security argument, not only an interop one |
| ④ Ring / designated-verifier | **Weakened, seriously.** Same as ③ (F3) — while F7c claims DVS is the *only* PQ route back, so PQ and attestation resistance may be mutually exclusive |
| ⑤ Time-locked forgery opening | **Strengthened as a direction.** F7a claims the same effect with no external beacon, no pairwise setup, churn tolerance — at the cost of a new algorithm ID |
| ⑥ Plaintext-only storage | **Both.** Peer-reviewed backing as the practical remedy (F5), and two published objections: near no-op vs. current client behaviour, and defeated by an attested peer (F5) |
| ⑦ Pseudonyms + server-blind | **Strengthened.** The primary authors recommend anonymity / metadata-minimisation *over* cryptographic deniability (F5). This is the axis already implemented |
| **New: deniable key provenance** | Candidate eighth approach (F2), but dead on arrival against cross-signing, and collapses into ⑦ |

**The order of operations implied by F4 is the uncomfortable part:** if
system-layer attribution dominates, Megolm-layer work is not the first thing to
do. §4 narrows how much that applies to us.

---

## 4. Matrix-specific check — done locally, not part of the survey

F4's inference names "origin-server signatures" as part of Matrix's system-layer
attribution. That is checkable in our own fork, so it was checked rather than
assumed.

`synapse/events/utils.py`, `format_event_for_client_v2`:

```python
drop_keys = (
    "auth_events", "prev_events", "hashes", "signatures",
    "depth", "origin", "prev_state",
)
for key in drop_keys:
    d.pop(key, None)
```

**Clients never receive the origin-server signature.** A hoarding peer's copy of
`sender` is unsigned JSON it could have written itself. Matrix has no
client-visible artifact equivalent to Signal's sealed-sender delivery
certificate.

The threat therefore splits, and F4 applies unevenly:

| Adversary | Severity |
|---|---|
| **Peer client hoarding everything** | **Weaker than Signal.** No server-signed artifact reaches the client; the peer's transcript is its word |
| **Federated remote server** | **Stronger.** Remote servers receive and retain **server-signed PDUs carrying `sender`**, outside our control and our retention policy |
| **Our own database, seized or subpoenaed** | Strong — but this is the operator-facing risk that ⑦ (pseudonyms + server-blind) and retention policy address |

**Implication not previously recorded anywhere: enabling federation materially
weakens what idea ② buys.** A federated deployment hands every remote server a
signed, sender-attributed record of every event, permanently. This is a
deployment decision that belongs next to the ones in
`newnal-user-aliases-server-blind.md`.

Note also that the design spec's §4.1 treats contemporaneous notarisation as
something a recipient must do *deliberately*. Under federation, **remote servers
do a weaker form of it automatically for every event.** Content stays encrypted,
so what is notarised is "this sender emitted this ciphertext at T" — which is
why ⑦ (the sender is a pseudonym nobody can link) is what keeps the composition
working.

---

## 5. Unreliable details — do not repeat verbatim

Each of these appears in secondary write-ups of the same sources and **failed
verification here**:

* "**140 Swiss court cases**" (0-3, twice, from two different URLs).
* "reviewed 140 Swiss court cases … deniability was never raised or considered".
* Calling the Collins et al. model a "**formal model**" — the author's own talk
  said *"not quite a cryptographic model but something a little less."*
* The **DKIM** half of the Collins et al. result — never mentioned in the talk;
  rests on search-index abstract text alone.
* Collins's affiliation as **Purdue** — paper-time only; postdoc at NYU /
  Hebrew University since Sep 2025.
* **Colombo's EPFL affiliation** — unconfirmed.
* "**MLS deliberately designs for non-repudiation to support abuse reporting**"
  (0-3, unsupported).
* "The MLS architecture document **explicitly disclaims** any deniability
  guarantee / calls it an open question" (0-3).
* "MIMI's answer to authorship privacy is per-room pseudonyms, draft reached
  only -00 and is no longer active" (0-3).
* "MIMI pseudonymous credentials require an issuer trusted by every participant
  and providers cap pseudonyms per account" (0-3).

The last four are all plausible and may well be true — they died on **quote
fidelity against primary text**, which was unreachable.

---

## 6. Re-fetch list, in priority order

Requires a session with scholarly egress. All of §0's blocked hosts are needed.

1. **Epochal signatures (`2020/1138`)** — do they deliver per-epoch
   unforgeability, post-expiry universal forgeability, no pairwise setup, and
   membership-churn tolerance? Concrete epoch length, key size, and per-message
   verification cost on a mobile client? *If it holds, the design question
   becomes whether an interop-breaking new signature algorithm ID is an
   acceptable price versus ②'s no-wire-change disclosure.*
2. **`2025/1949`** — the "courts already assume all evidence is forgeable"
   thesis. The most direct statement of the counter-case; drives the decision on
   whether to build ② at all.
3. **Does the attestation attack extend to publicly-forgeable designs via
   timestamped receipt?** An enclave attesting *"I recorded these bytes at T,
   before the session key was published at T+δ"*. **If yes, items ② and ⑤ lose
   their advantage over ③ and ④ and the entire cryptographic route collapses
   against an attested peer.** The 2018 paper's full text should settle whether
   the authors considered this.
4. **Is the post-quantum tension real?** KEM-only cannot be deniable
   (`2021/769`) ⇒ DVS is the only PQ route back; attestation (`2018/424`)
   destroys DVS deniability. If both hold, **deniable authentication may be
   unachievable against an attested peer in a post-quantum Matrix** — a
   decision-grade impossibility argument that should be resolved before any
   implementation effort.
5. **Does the Collins et al. whole-system model place Matrix's homeserver-side
   attribution inside the adversary's evidence set?** Their artifact is
   reproducible (`github.com/si-co/rwdm`), so this is answerable by instantiating
   it on Matrix rather than by argument. §4 above is a partial answer for the
   client-peer case only.
6. **What replaced mpOTR, and why did multi-party OTR fail to deploy?** Asked and
   returned **nothing at all**; it is the closest historical analogue and the
   most likely source of deployment-failure lessons for a ~20-member mobile room.

---

## 7. Scope gaps — asked for and not found

* **mpOTR successors / why multi-party OTR never deployed** — nothing located.
* **Chameleon hashes, trapdoor commitments, malleable / homomorphic signatures
  as deniability mechanisms** — nothing located.
* **Transparency-log-based approaches** — nothing located.
* **No source reasons about our constraint set** (~20-member rooms, mobile,
  unmodified-client interop). Every interop and in-group-authenticity column in
  this document is inference from mechanism descriptions.

---

## 8. Sources

Rated by the survey. "unreliable" means the fetch failed or yielded no usable
claims — not that the work is poor.

| Source | Quality | Claims |
|---|---|---|
| `github.com/mlswg/mls-architecture/.../draft-ietf-mls-architecture.md` | primary | 5 |
| `datatracker.ietf.org/doc/draft-mahy-mimi-identity/` | primary | 5 |
| `petsymposium.org/popets/2025/popets-2025-0018.pdf` | primary | 5 |
| `eprint.iacr.org/2020/1138` (epochal signatures) | primary | 5 |
| `eprint.iacr.org/2025/853.pdf` (split-KEM) | primary | 5 |
| `eprint.iacr.org/2021/769.pdf` (Brendel et al.) | primary | 5 |
| `eprint.iacr.org/2018/424.pdf` (attestation) | primary | 5 |
| `eprint.iacr.org/2023/403` (real-world deniability) | primary | 5 |
| `eprint.iacr.org/2025/1949` | primary | 5 |
| `eprint.iacr.org/2024/741` | primary | 4 |
| `eprint.iacr.org/2025/1090` | primary | 5 |
| `usenix.org/system/files/sec21-specter-keyforge.pdf` (KeyForge) | primary | 5 |
| `obj.umiacs.umd.edu/ieeesp23/Cryptographic_Deniability.pdf` | primary | 5 |
| `ieee-security.org/TC/SP2015/papers-archived/6949a232.pdf` | primary | 5 |
| `dl.acm.org/doi/10.1145/3688459.3688479` | primary | 5 |
| `usenix.org/system/files/usenixsecurity23-yadav.pdf` | unreliable | 0 |
| `eprint.iacr.org/2018/1097.pdf` | unreliable | 0 |
| `eprint.iacr.org/2025/204` | unreliable | 0 |
| `arxiv.org/abs/2305.09799` | unreliable | 1 |
| `link.springer.com/chapter/10.1007/978-3-031-70903-6_21` | unreliable | 0 |
| `dl.acm.org/doi/10.1145/2517840.2517867` | unreliable | 0 |
| `cypherpunks.ca/~iang/pubs/mpotr.pdf` | unreliable | 0 |
| `datatracker.ietf.org/doc/draft-barnes-mimi-identity-arch/` | primary | — |

Note that three sources appearing in the fetch list with usable claim counts —
KeyForge (`sec21-specter-keyforge.pdf`), `Cryptographic_Deniability.pdf`, and
`6949a232.pdf` — produced claims that did not survive to synthesis. KeyForge is
the closest published relative of the "make forgery cheap and public" family
(forward-forgeable signatures for email non-attribution) and was **not** examined
on its merits in this pass. Add it to the re-fetch list if that family is
revisited.
