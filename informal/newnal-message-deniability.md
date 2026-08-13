# Message deniability for Newnal rooms — design space and key-disclosure spec

This informal document covers **message-authorship deniability** (repudiation)
for the Newnal fork: making it impossible for a third party to prove that a
given pseudonym authored a given message. It is a sibling to the identity /
correlation work in [newnal-user-aliases.md](./newnal-user-aliases.md) and
[newnal-user-aliases-server-blind.md](./newnal-user-aliases-server-blind.md);
those hide *who* a participant is, this hides *that they said it*.

Status: draft. Part 1 (design space) is analysis. Part 2 (session-key
disclosure, "idea ②") is a concrete draft spec — the recommended first
increment. Not yet implemented.

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
| ② | **Session-key disclosure** | On retiring a session, publish its Ed25519 **private** key into the room; from then on anyone who held the session can forge its past messages | **None to the wire format** — one new event type | Live: kept. Retroactive: dissolved | Yes |
| ③ | **Pairwise MACs** (OTR/Signal model) | Replace the signature with per-recipient MACs keyed from the Olm pairwise secret; each recipient can verify *and* forge toward themselves, nobody else's MAC | Medium — n MACs/message (feasible: Newnal caps rooms at ≤20) | **Kept** | Yes |
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
③/④  pairwise-MAC Megolm variant   ← endgame: only un-provable material ever exists
 ②   session-key disclosure         ← deployable now, no wire change, covers existing history retroactively
 ⑥   plaintext-only storage + real server delete  ← device-seizure / screenshot layer
 ⑦   pseudonyms + server-blind        ← the "who sent it" record layer (already designed)
```

② is the natural first increment: it changes **no Megolm wire format**, only
adds one event type and a rotation policy.

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

## 2.1 Mechanism

Each Megolm session already owns an ephemeral Ed25519 keypair whose **public**
half is the `session_id`. This spec adds: when a session is retired (rotated
out and no longer used for new messages), its owner publishes the **private**
half into the room. Anyone who holds the (already-shared) session ratchet can
then produce messages at any ratchet index they possess, sign them with the
disclosed key, and the result is byte-indistinguishable from a genuine message
of that session. The session's entire past history becomes forgeable-by-any-
member, i.e. non-attributable to a third party.

Nothing about the on-wire Megolm message format changes. Live messages of the
*current* session remain signed and in-group-authentic; only *retired*
sessions become deniable. Retirement cadence (§2.4) sets the deniability lag.

## 2.2 Event type

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
  **no longer authenticatable** (see §2.3) and SHOULD store the disclosed key
  so its own "these are deniable" UI and any forge tooling can use it.

## 2.3 Client verification-state semantics

The key change on the receiving side is that a disclosed session's messages
must stop being presented as authenticated:

* Before disclosure: message shows normal E2EE verification state.
* After disclosure: message MUST be reclassified to a "deniable / not
  attributable" state — never shown with a verified/authenticated badge, and
  `EncryptionInfo`-derived trust must be treated as void for attribution.
* This reclassification MUST apply retroactively to already-received messages
  of that session and MUST survive restart (persisted).

## 2.4 Rotation / retirement policy

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

## 2.5 What ② does and does not give

| Property | ② |
|---|---|
| Third party cannot prove a *retired* session's authorship | yes — anyone who had the session can forge it |
| Live (current-session) messages stay in-group authentic | yes |
| Scope of a disclosure | exactly one session; device identity & other sessions untouched |
| Works even if peer keeps everything | yes (peer's copy becomes forgeable-by-peer) |
| Defeats contemporaneous notarization | no (§4.1) |
| Provides in-group deniability *during* the live window | no — use ③ for that |

## 2.6 Interaction with key backup / history sharing

Disclosure and long-term history are in tension: a device that keeps
per-session **megolm key backups** (for message-history recovery) is also
keeping the ratchets that make disclosed sessions forgeable — which is fine
for deniability (forgeable = good) but means "deniable history" still decrypts
for the owner. Deployments choosing maximal deniability SHOULD prefer the
plaintext-only storage of ⑥ over retaining per-session backups, so that what
survives on the device is plaintext (owner-forgeable) rather than
ratchet+ciphertext.
