# Message deniability for Newnal rooms — design space and key-disclosure spec

This informal document covers **message-authorship deniability** (repudiation)
for the Newnal fork: making it impossible for a third party to prove that a
given pseudonym authored a given message. It is a sibling to the identity /
correlation work in [newnal-user-aliases.md](./newnal-user-aliases.md) and
[newnal-user-aliases-server-blind.md](./newnal-user-aliases-server-blind.md);
those hide *who* a participant is, this hides *that they said it*.

Status: draft. Part 1 (design space) is analysis. Part 2 (session-key
disclosure, "idea ②") is a concrete draft spec — **the chosen approach**, in
two modes: immediate disclosure for 2-party rooms and deferred disclosure for
group rooms (§5.8). Not yet implemented; the repo-by-repo implementation
review lives in the messaging-node fork at
`custom_docs/todo/message-deniability-key-disclosure.md`.

A 2026-08 literature survey of what exists **outside** the seven approaches in
§2 is recorded in
[newnal-message-deniability-research-2026-08.md](./newnal-message-deniability-research-2026-08.md).
It changes no decision here, but three of its results bear on this document and
should be read before scoping work: attested-hardware peers defeat ideas ③ and
④ while leaving the publicly-forgeable ones (①, ②, ⑤) intact; deniability is
defeated at the **system** layer rather than the crypto layer, so §0's framing
is at most half the problem; and the only empirical study of the legal
counter-case found deniability never raised, with zero measured effect. The
survey's own verification infrastructure partly failed — read its §0 before
citing anything from it.

## 0. Why Matrix messages are non-repudiable today

Megolm authenticates every message with a **per-session Ed25519 signature**
([megolm.md](../content/olm-megolm/megolm.md): "the authenticated message is
signed using the Ed25519 keypair; the 64 byte signature is appended"). At
session sharing only the **public** part of that keypair is distributed (the
sharing format carries `Kpub`, not the private key), and the public key
doubles as the `session_id`.

Consequence: a recipient can *decrypt* (they hold the ratchet) but cannot
*forge* a validly-signed message (they lack the session's Ed25519 private
key). So a recipient's stored copy — ciphertext + signature — is transferable
proof to a third party that the session's owner authored it. That is exactly
non-repudiation, and it is what deniability must dismantle.

Two facts make this tractable:

* The signing key is **per-session and ephemeral**, distinct from the device
  identity key (`ed25519:DEVICEID`) and from cross-signing keys. Disclosing it
  compromises forgeability of **only that session's** messages and touches
  nothing else.
* The session key is shared over **Olm** to-device messages, and Olm's 3DH
  handshake is itself deniable. So there is no transferable signature binding
  "session → device → user"; a recipient's belief about the sender is local
  (`EncryptionInfo`/sender-data), not a third-party-verifiable proof.

## 1. The governing principle

> **Make the ability to verify entail the ability to forge.**

If verifying a message's authenticity requires a capability that also lets the
verifier fabricate an indistinguishable message, then the verifier's copy
proves nothing to anyone else ("you could have made it yourself"). Signatures
are the opposite of this (anyone verifies, only the holder signs), which is why
Megolm's Ed25519 kills deniability. Every technique below removes that
asymmetry in some form.

## 2. Design space

Ordered by how fundamental the guarantee is, not by ease. "In-group
authenticity" = can honest members still tell each other apart / detect
impersonation *within* the room while the conversation is live.

| # | Idea | Mechanism | Protocol change | In-group authenticity | Independent of peer client? |
|---|---|---|---|---|---|
| ① | **Drop the signature** | Megolm ratchet is already shared with all members; without the per-message signature every member can forge every member | Minimal (omit signature in a room-version/algorithm variant) | **Lost** (members can impersonate each other) | Yes |
| ② | **Session-key disclosure** | Publish a session's Ed25519 **private** key to the room, either on retirement (deferred) or at session-share time (immediate); anyone holding the session can then forge its messages | **None to the wire format** — one new event type / one `m.room_key` field | Deferred: live kept, past dissolved. Immediate: dissolved from the start | Yes |
| ③ | **Pairwise MACs** (OTR/Signal model) | Replace the signature with per-recipient MACs keyed from the Olm pairwise secret; each recipient can verify *and* forge toward themselves, nobody else's MAC | **Large** — new algorithm ID + new message format; n MACs/message (feasible: Newnal caps rooms at ≤20) | **Kept** | Yes |
| ④ | **Ring / designated-verifier signatures** | "Signed by sender OR recipient" OR-proof; the theoretical completion of ③ | Large (new crypto in vodozemac) | Kept | Yes |
| ⑤ | **Time-locked forgery opening** | Escrow session keys to a delay function (e.g. drand); "anything older than T is forgeable by all" | Infra-dependent, exotic | Time-bounded | Yes |
| ⑥ | **Plaintext-only storage + first-class edit** | On decrypt, discard ciphertext + signature + `EncryptionInfo`; keep only plaintext locally, in a format an in-app editor produces bit-identically. The app itself is a forgery tool, publicly | Client storage path only | n/a (storage layer) | **No** — relies on peer app also discarding |
| ⑦ | **Erase the authorship record** | Orthogonal to payload: "pseudonym P sent *something* at T" persists server-side; real-delete + short retention + pseudonyms/server-blind sever "P = person" | Server policy | n/a | n/a |

Notes:

* ⑥ is the earlier "local edit + server delete" idea; its missing piece was
  *never storing the ciphertext/signature in the first place*. Without that,
  a forensic extract of the device still yields signed originals, and locally
  edited messages are trivially distinguished from genuine ones (no valid
  signature) — which is worse than nothing.
* ① sacrifices in-group authenticity; ③ is the version that keeps it, and is
  the OTR/Signal-proven "right answer" at the cost of n MACs per message.
* ①'s effect is reachable **without** a new algorithm by disclosing the
  session signing key at share time (② immediate, §5.2b): signatures keep
  verifying — they simply stop proving anything. That makes ① as a distinct
  wire-level change unnecessary in practice.

### 2.1 What "in-group authenticity" actually costs

Precisely stated, because it drives the tiering in §5.8: a Matrix event's
`sender` is asserted by the **homeserver**, not by Megolm. The per-session
signature is what stops a *lying homeserver* from attributing an event to a
user who did not send it — the server never holds the ratchet or the signing
key, so it cannot forge alone.

Therefore disclosing a signing key does **not** by itself let a member inject
messages attributed to another member: the member's own `sender` is stamped by
the server. The residual risk is **member + homeserver collusion**, which
together can produce an event with someone else's `sender` *and* a
valid-looking signature.

* In a **2-party** room this is vacuous: the only party who could be deceived
  is the forger themselves.
* In a **group** room it is real, though bounded — a conforming client marks
  every disclosed session as *not attributable* (§5.4), so the fabricated
  message is displayed without any authenticity claim rather than as a
  verified message from the impersonated member.

## 3. Fundamental split, and the recommended stack

**Cryptographic deniability (①–⑤) holds regardless of what the peer does.**
Even a peer running a modified client that hoards everything only ends up with
material *it could have produced itself* (unsigned ciphertext / a MAC it can
mint / a message forgeable with a disclosed key). **Deletion-based deniability
(⑥) depends on peer-client cooperation** — a peer that keeps the signed
original defeats it. So the ultimate targets are ①/②/③, and ⑥ is a
secondary layer (against seizure of *your own* device), not the foundation.

Recommended layering, near-term to long-term:

```
 ②-immediate  disclose at session share   ← 2-party rooms: full live-window deniability, no wire change
 ②-deferred   disclose on retirement      ← group rooms: retroactive deniability, no wire change
 ⑥   plaintext-only storage + real server delete  ← device-seizure / screenshot layer
 ⑦   pseudonyms + server-blind        ← the "who sent it" record layer (already designed)
 ③/④  pairwise-MAC Megolm variant   ← only if live-window deniability is required in *group* rooms
```

② is the chosen first increment: it changes **no Megolm wire format** and no
algorithm identifier, so unmodified Matrix clients keep interoperating. ③ is
deferred — it needs a new algorithm ID (which breaks interop outright) and a
new message format, a substantially larger and higher-risk change than ②;
see §5.9.

## 4. Limits no scheme defeats (state honestly in product copy)

1. **Instant notarization.** At the moment of receipt the recipient *does*
   know the message is genuine (that is what makes communication work). If they
   anchor a hash to a chain / third-party notary *then*, later forgeability
   doesn't erase "it existed and verified at time T." Deniability blocks
   *after-the-fact* proof, not *contemporaneous* commitment.
2. **Cryptographic ≠ legal standard.** "Indistinguishable" drives the
   *cryptographic* evidentiary weight to zero; courts weigh circumstantial
   evidence, testimony, and device forensics too. The achievable goal is to
   demote "cryptographically certain" to "one party's word against another,"
   which is valuable but not absolute immunity.
3. **In-group impersonation cost (①).** Dropping signatures opens
   member-to-member spoofing; preserving in-group authenticity requires ③.
4. Do **not** add duress-PIN / decoy stores: their presence is itself
   forensically detectable and, in many jurisdictions, aggravating.

---

# Part 2 — Session-key disclosure (idea ②): draft spec

## 5.1 Mechanism

Each Megolm session already owns an ephemeral Ed25519 keypair whose **public**
half is the `session_id`. This spec adds: when a session is retired (rotated
out and no longer used for new messages), its owner publishes the **private**
half into the room. Anyone who holds the (already-shared) session ratchet can
then produce messages at any ratchet index they possess, sign them with the
disclosed key, and the result is byte-indistinguishable from a genuine message
of that session. The session's entire past history becomes forgeable-by-any-
member, i.e. non-attributable to a third party.

Nothing about the on-wire Megolm message format changes, and no algorithm
identifier changes: signatures are still produced and still verify. What
changes is that the key which produces them is no longer exclusive to the
sender, so a valid signature stops being evidence of authorship.

## 5.2 Two disclosure modes

### 5.2a Deferred disclosure (retire, then disclose)

The session is used normally; once rotated out and no longer used for new
messages, its private signing key is published via the event in §5.3. Past
messages of that session become forgeable-by-any-member; the **current**
session stays exclusively signable by its owner. Retirement cadence (§5.5)
sets the deniability lag: messages are attributable during the live window
and deniable afterwards.

### 5.2b Immediate disclosure (disclose at session share)

The private signing key is distributed **together with the session key**, in
the same per-device `m.room_key` to-device message that already shares the
session. Every recipient can forge from the very first message, so the live
window is deniable too — the property ① provides, but with **no new
algorithm identifier and no wire-format change**, because signatures continue
to be generated and to verify normally.

Suggested `m.room_key` content addition (the to-device message is already
Olm-encrypted per device, so the key never transits in clear):

```jsonc
{
  "algorithm": "m.megolm.v1.aes-sha2",
  "room_id": "!abc:server",
  "session_id": "<base64 Kpub>",
  "session_key": "<standard session sharing blob>",
  "com.newnal.disclosed_signing_key": "<base64 Ed25519 private scalar>"
}
```

Because the field rides along with the session key, **any device that later
receives the session automatically receives the forge key** — no supplementary
distribution step and no extra rotation is required when a device is added.

**Constraint — 2-party rooms only.** Immediate disclosure removes in-group
authenticity for the whole lifetime of the session, which per §2.1 (Part 1) is
harmless only when the room has exactly two participants. Deployments:

* MUST restrict immediate disclosure to room types with a hard cap of two
  members (the fork's `max_member_count == 2`);
* MUST rotate to a **non-disclosed** session before a third member is
  admitted, and MUST NOT disclose that new session immediately;
* SHOULD fall back to deferred disclosure (§5.2a) for every other room.

## 5.3 Event type

A state-free room message event, end-to-end encrypted like any other so the
disclosure itself isn't visible to the server:

```jsonc
// decrypted content of a com.newnal.key_disclosure event
{
  "type": "com.newnal.key_disclosure",
  "content": {
    "disclosed_sessions": [
      {
        "room_id": "!abc:server",
        "session_id": "<base64 Kpub — the public signing key / session id>",
        "signing_key_private": "<base64 Ed25519 private scalar>",
        "first_known_index": 0,          // ratchet index from which forgery is enabled
        "retired_ts": 1754870000000
      }
    ]
  }
}
```

Requirements:

* MUST be sent only for sessions the sender has **stopped using** for new
  encryption (retired). Disclosing a live session's key would let members
  forge messages the sender is still actively authoring under it — a
  correctness/impersonation break, not deniability.
* MUST be sent into the same room, encrypted under the room's *current* session
  (not the one being disclosed), so all members and joined-history readers
  receive it.
* SHOULD batch multiple retired sessions.
* A client receiving it MUST mark all cached messages of the named sessions as
  **no longer authenticatable** (see §5.4) and SHOULD store the disclosed key
  so its own "these are deniable" UI and any forge tooling can use it.

## 5.4 Client verification-state semantics

The key change on the receiving side is that a disclosed session's messages
must stop being presented as authenticated:

* Before disclosure: message shows normal E2EE verification state.
* After disclosure: message MUST be reclassified to a "deniable / not
  attributable" state — never shown with a verified/authenticated badge, and
  `EncryptionInfo`-derived trust must be treated as void for attribution.
* This reclassification MUST apply retroactively to already-received messages
  of that session and MUST survive restart (persisted).

## 5.5 Rotation / retirement policy

Deniability lag = how long a session stays live before it can be retired and
disclosed. Deployments SHOULD drive Megolm rotation aggressively so keys become
disclosable promptly:

* short `rotation_period_ms` / `rotation_period_msgs` (e.g. rotate hourly or
  every N messages), balanced against the extra key-sharing traffic;
* a background task that, on each rotation, schedules disclosure of the
  now-retired session after a short safety delay (ensuring no in-flight use);
* disclosure of all retired sessions on graceful client actions (room leave,
  account/pseudonym deactivation).

Trade-off to document: a shorter live window = stronger deniability but more
key rotation/sharing overhead; a longer window = the reverse.

## 5.6 What ② does and does not give

| Property | ②-deferred | ②-immediate |
|---|---|---|
| Third party cannot prove authorship of a *retired* session | yes | yes |
| Third party cannot prove authorship during the **live window** | no | **yes** |
| In-group authenticity during the live window | kept | given up (2-party only, §2.1) |
| Scope of a disclosure | one session; device identity & other sessions untouched | same |
| Works even if the peer keeps everything | yes (peer's copy becomes forgeable-by-peer) | yes |
| Wire format / algorithm ID change | none | none |
| Interoperates with unmodified Matrix clients | yes | yes |
| Defeats contemporaneous notarization | no (§4.1) | no (§4.1) |

## 5.7 Implementation notes (verified against vodozemac / matrix-rust-sdk)

* The signing keypair is a field of the outbound Megolm session
  (`GroupSession.signing_key: Ed25519Keypair`), and `session_id()` is its
  public half. Disclosure therefore only needs a **read accessor** for the
  private half — the material already exists and is already persisted in the
  session pickle.
* `MegolmMessage::add_signature()` already accepts an externally-produced
  signature and validates it against a key, so the signing half of forging is
  in place. What is missing is the ability to **encrypt at a chosen ratchet
  index from an inbound session** (`InboundGroupSession` holds the ratchet but
  exposes no encrypt path) — that is the one genuinely new primitive.
* `InboundGroupSession::decrypt` verifies the signature *before* the MAC, so
  the "disclosed ⇒ not authenticatable" state (§5.4) must be applied at the
  layer above decryption, not by skipping verification.
* Megolm rotation is triggered by device **removal**, visibility change and
  algorithm change — **not** by device addition; a newly added device is given
  the existing session. For ②-immediate this is convenient (the disclosed key
  travels inside the same `m.room_key`, so new devices get it automatically);
  for ②-deferred it means the retirement bookkeeping must not assume the
  recipient set is frozen.
* Exported / backed-up sessions carry only the **public** signing key, and
  imported sessions are already flagged as not-verified — so disclosure adds
  no new trust state for history-shared sessions.

## 5.8 Tiered deployment

| Room shape | Mode | Rationale |
|---|---|---|
| Exactly 2 members (DMs, contact/number rooms) | **②-immediate** | Live-window deniability at near-zero cost; in-group authenticity is vacuous (§2.1) |
| 3+ members | **②-deferred** | Keeps in-group authenticity live; history becomes deniable on rotation |
| 3+ members needing live-window deniability | ③ (out of scope here) | Requires pairwise MACs; see §5.9 |

The room type must **enforce** the 2-member cap for ②-immediate; a room that
can grow past two MUST use deferred mode, and any transition from a 2-member
room to a larger one MUST rotate to a fresh, non-disclosed session first.

## 5.9 Why ③ is not the first increment

③ (pairwise MACs) is the only option that provides live-window deniability in
**group** rooms, but it is a substantially larger and riskier change than ②:

* it requires a **new algorithm identifier**, which unmodified Matrix clients
  cannot decrypt at all (hard interop break — relevant because Newnal rooms
  federate), whereas ② keeps `m.megolm.v1.aes-sha2` intact;
* the Megolm message structure carries exactly one MAC and one signature with
  fixed-length suffixes; per-recipient MACs change the message *shape*, plus
  the outbound/inbound session state, both pickle formats, and the
  single-blob `m.room_key` sharing model (each recipient needs different key
  material);
* it introduces new cryptographic design (per-recipient key derivation)
  warranting external review, where ② only exposes an existing primitive;
* message overhead grows with the recipient device count (~0.2–1.7 KB per
  message at the fork's ≤20-member cap).

Order of magnitude: ② is a contained change to two forks; ③ is a
protocol-level fork of Megolm. Revisit ③ only if live-window deniability in
group rooms becomes a firm product requirement.

## 5.10 Interaction with key backup / history sharing

Disclosure and long-term history are in tension: a device that keeps
per-session **megolm key backups** (for message-history recovery) is also
keeping the ratchets that make disclosed sessions forgeable — which is fine
for deniability (forgeable = good) but means "deniable history" still decrypts
for the owner. Deployments choosing maximal deniability SHOULD prefer the
plaintext-only storage of ⑥ over retaining per-session backups, so that what
survives on the device is plaintext (owner-forgeable) rather than
ratchet+ciphertext.
