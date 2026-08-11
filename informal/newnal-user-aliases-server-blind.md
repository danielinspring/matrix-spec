# Server-blind pseudonyms (`com.newnal.user_alias`, blind mode) — design spec

This informal document specifies the **server-blind variant** of the Newnal
user-alias / per-room-pseudonym extension. The base (server-mapped) variant
is specified in [newnal-user-aliases.md](./newnal-user-aliases.md) and is
already implemented; this variant is a **draft design, not yet implemented**.

Status: draft. Target platform: Newnal private phone (Android). iOS is
explicitly out of scope (APNs' one-token-per-install model reintroduces the
push join key this design removes; see §5).

The key words MUST, MUST NOT, SHOULD, MAY are to be interpreted as in
RFC 2119.

## 1. Goal and threat model

The base variant hides identity from **other users**: room members see only
per-room pseudonyms and per-pseudonym E2EE device keys, so they cannot
correlate one person across rooms. However the homeserver keeps a
deterministic record linking everything: the `room_pseudonyms` table, the
provisioning/login-token API trail, and the plaintext
`com.newnal.pseudonym` account data.

Blind mode removes the **deterministic** link from the server:

> After this change, no row in the homeserver's database and no single API
> exchange states that anchor account *U* owns pseudonym *P*. The operator
> (or anyone who compromises, subpoenas, or inherits the database) cannot
> answer "which rooms is *U* in?" by lookup — only by statistical traffic
> inference.

| Observer | Base variant | Blind mode |
|---|---|---|
| Room member / cross-room member | cannot link | cannot link |
| Homeserver DB / operator lookup | **full mapping** | **no mapping** — statistical inference only |
| Push (UP) server operator | subset of homeserver knowledge | new trust boundary — see §5.4 |
| Network observer (carrier) | TLS metadata only | TLS metadata only |

Explicit non-goals (kept honest): IP/timing correlation at the homeserver
(mitigated by mobile CGNAT, UP wake-based syncing, and jitter — never
eliminated), traffic-volume analysis, and collusion between the homeserver
operator and the UP server operator (§5.7).

## 2. Identity and data model

* **Anchor account** — the user's real Matrix account. Holds the alias
  (optional in blind mode, see below), requests blind tokens (§3), and
  stores the encrypted recovery blob (§6). It never joins rooms and
  registers **no pusher**.
* **Pseudonym account** — a fully independent account, one per
  (user, room), with `user_type: "pseudonym"`. In blind mode it is
  **self-registered by the client** (§3); the server performs no
  provisioning, mints no login tokens, and stores no ownership mapping.
  Credentials (password or refresh token) exist only client-side.
* **Client-held registry** — the only place the mapping
  `room_id ↔ pseudonym ↔ credentials` exists in plaintext is the device
  (hardware-keystore protected). Its ciphertext backup is specified in §6.

Removed relative to the base variant: the `room_pseudonyms` table, the
`POST /rooms/{roomId}/pseudonym[/login]` endpoints, the server-written
`com.newnal.pseudonym` account data, the `invite_by_alias` and
`create_room` wrapper endpoints, and the server-side deactivation cascade.

Kept unchanged: `user_type: "pseudonym"` semantics (excluded from the user
directory and MAU; cannot call anchor-only endpoints), and the
`require_pseudonyms_for_rooms` membership gate — both key off `user_type`
only and never needed the mapping.

**User alias in blind mode.** The alias registry (alias ↔ anchor) MAY be
kept as an anchor-level contact handle, but it plays **no role** in
pseudonym flows: there is nothing the server could resolve it *to* without
recreating the mapping. Deployments using room-alias rendezvous (Newnal
numbers) can omit it entirely.

## 3. Blind registration tokens

Pseudonym self-registration must be gated (otherwise the server-blind
property is bought with open registration and unlimited sybils). The gate
is a **blind signature token**: the server can verify "this bearer is
entitled to one account" without learning **which** anchor the token was
issued to.

Recommended scheme: RSA blind signatures (RFC 9474) or a Privacy-Pass-style
VOPRF issuance (RFC 9497). The choice is a deployment constant advertised
in the capability (§8).

Where these endpoints live depends on the deployment's auth architecture:
on a Synapse-native-auth deployment they are homeserver endpoints as shown;
on an MSC3861 deployment (Matrix Authentication Service) both issuance and
redemption belong to the **authentication node**, which owns registration
and token issuance — see §7.1. The wire contract is the same either way.

### 3.1 Issuance (anchor-authenticated)

```
POST /_matrix/client/unstable/com.newnal.user_alias/blind_tokens/issue
{"blinded_elements": ["<base64>", ...]}          // client-blinded nonces
→ 200 {"signatures": ["<base64>", ...], "quota_remaining": 17}
```

* The anchor blinds fresh random nonces locally and submits only the
  blinded form; the response signatures are unblinded locally. The server
  MUST NOT see the unblinded nonce at issuance.
* The server enforces a per-anchor issuance **quota** (rate + lifetime
  budget). It learns *how many* tokens an anchor holds — an upper bound on
  room count — but not where any token is spent. Deployments SHOULD set the
  quota loose enough that the count itself is low-signal (e.g. fixed-size
  monthly grants for every account, so all anchors look alike).

### 3.2 Redemption (unauthenticated registration)

```
POST /_matrix/client/v3/register
{"auth": {"type": "com.newnal.blind_token",
          "nonce": "<base64>", "signature": "<base64>"},
 "initial_device_display_name": ""}
→ 200 {user_id: "@p_…:server", access_token, device_id, refresh_token}
```

* The server verifies the signature, checks the nonce against a
  **spent-token set** (double-spend prevention; the set stores only nonce
  hashes, which are unlinkable to issuance), assigns a random
  `p_`-prefixed localpart, and sets `user_type: "pseudonym"`.
* The created account MUST behave like any account: own devices, own E2EE
  keys, own refresh-token rotation. The client stores the credentials in
  its registry (§2) and SHOULD set up recovery per §6 before first use.

### 3.3 Unlinkability requirements on the client

* Tokens MUST be pre-fetched in pools at times unrelated to room joins;
  a client MUST NOT issue-then-redeem in one breath (the issuance/spend
  timestamps are the operator's best correlation signal at this layer).
* The blinded element MUST be a fresh random nonce carrying no anchor- or
  device-derived structure.
* Registration SHOULD be jittered relative to the user action that
  triggered it (a join can proceed optimistically once the account
  exists).

## 4. Room flows without server resolution

There is no server-side invite-by-alias in blind mode — the server cannot
resolve a handle to a pseudonym it does not know. Rendezvous moves to
rooms and to existing E2EE channels:

* **Contact room (Newnal-number model).** The number is a *room alias*.
  Anyone who knows it resolves the alias, self-registers a pseudonym
  (§3.2), and joins. The owner sits in their own number-room through a
  pseudonym as well.
* **Group rooms.** The invitee mints a fresh pseudonym for the new room
  and sends its MXID to the inviter **over an existing E2EE room**; the
  inviter invites that MXID normally. Alternatively the room address is
  shared over the E2EE channel and the room uses `knock` or a shared join
  secret. Either way the server only ever sees pseudonym-to-pseudonym
  membership operations.
* **Room creation.** The client registers a pseudonym first, then calls
  the standard `POST /createRoom` *as that pseudonym session*. (The base
  variant's `create_room` wrapper existed only because the server had to
  write the mapping; with no mapping there is nothing to wrap.)

The `require_pseudonyms_for_rooms` gate continues to refuse join/invite/
knock of local non-pseudonym users, so an anchor can never end up in room
state by accident.

## 5. Push — normative

Push is the layer most likely to silently rebuild the link this design
removes, so all of the following are requirements, not suggestions.

### 5.1 Summary

| # | Requirement | Level |
|---|---|---|
| P1 | Push transport is UnifiedPush | MUST |
| P2 | One UP registration (endpoint) **per pseudonym**, never shared | MUST |
| P3 | Pusher registered by the pseudonym's own session; pushkey = its UP endpoint | MUST |
| P4 | The anchor account registers no pusher | MUST |
| P5 | Pusher `data.format` = `event_id_only` | MUST |
| P6 | UP server not operated by (or resolvable to) the homeserver operator | MUST |
| P7 | Endpoints contain no user/device-derived identifiers | MUST |
| P8 | Pusher deleted and endpoint rotated on room leave / pseudonym deactivation | MUST |
| P9 | Wake-driven syncs jittered; no permanent per-pseudonym sync loops | SHOULD |
| P10 | Shared `app_id` across all pseudonyms of the deployment | SHOULD (k-anonymity) |

### 5.2 Why per-pseudonym endpoints (P1–P3)

A single FCM token per install is a **join key**: every pseudonym's pusher
row carries the same pushkey and the homeserver DB links them in one
query. UnifiedPush issues a distinct endpoint per registration, and an app
may hold arbitrarily many registrations — so each pseudonym session
registers its own endpoint via `POST /_matrix/client/v3/pushers/set`:

```json
{"app_id": "ai.newnal.comm", "kind": "http",
 "pushkey": "https://up.example/xxxxxxxxxxxx",
 "data": {"url": "https://up.example/_matrix/push/v1/notify",
          "format": "event_id_only"}}
```

The homeserver still knows which pusher belongs to which pseudonym (its
own table — that is unavoidable and harmless); what it must never get is a
**common value across pseudonyms**. `app_id` is technically common, but it
is common to *every* user of the app, so it identifies nothing (P10 —
deployments MUST NOT vary app_id per user or per anchor).

### 5.3 Payload minimization (P5)

What transits the UP server per notification, by pusher format
(field list verified against the Synapse fork, `synapse/push/httppusher.py`):

| Field | default format | `event_id_only` |
|---|---|---|
| `event_id`, `room_id`, `prio`, `counts.unread` | yes | yes |
| event `type` | yes | no |
| `sender` (counterparty's pseudonym MXID) | yes | no |
| `sender_display_name`, `room_name` | yes | no |
| `content` (ciphertext + session/device IDs in E2EE rooms; plaintext otherwise) | yes | no |

`event_id_only` costs nothing in an all-E2EE deployment — the client must
fetch-and-decrypt to render a preview in either format — and removes
sender pseudonyms, names and megolm metadata from the UP server's logs.
Badge-only pushes (unread count updates) carry no event data and are fine.

### 5.4 UP server trust and operator separation (P6)

UnifiedPush delivery is one persistent connection per device, subscribing
to **all** of that device's endpoints. The UP server therefore learns
"endpoints E1…En belong to one device" — exactly the grouping this design
erases from the homeserver. Consequences:

* If the homeserver operator also runs the UP server, blind mode is
  defeated at the push layer. P6 forbids this. "Operated by" includes
  hosting the UP server on operator infrastructure or behind operator TLS
  termination.
* Acceptable deployments: a user-selected public distributor/server; a
  third-party-operated server; or — for the private phone — a
  **device-embedded distributor** whose server URL the user can point
  anywhere (with a non-operator default). The distributor app itself runs
  on the device and may see everything; that is in scope of the device,
  not a network adversary.
* The UP server implements the Matrix push gateway
  (`/_matrix/push/v1/notify`) natively (as ntfy does), so no Sygnal sits
  in the path. Deployments migrating from FCM MUST scope any legacy
  gateway rewriting by `app_id` + exact URL so UP pushers are untouched.
* What the (compliant, separated) UP server ends up knowing: device IP and
  online pattern, endpoint↔device grouping, per-endpoint notification
  timing/volume, and the `event_id_only` payload. It does not know
  pseudonym MXIDs, aliases, anchors, or message content — and it cannot
  map endpoints to accounts without the homeserver's cooperation (§5.7).

### 5.5 Endpoint hygiene (P7, P8)

* Endpoints are server-generated random capabilities; clients MUST NOT
  derive topics from MXIDs, room IDs, or install IDs.
* On leaving a room or deactivating a pseudonym, the client MUST delete
  the pusher (`pushers/set` with `kind: null`) and unregister the UP
  endpoint. Stale endpoints keep receiving room-activity timing.
* On UP re-registration (distributor change, server migration), all
  endpoints rotate; clients MUST re-register pushers per pseudonym and
  MUST NOT reuse a single new endpoint across pseudonyms "temporarily".

### 5.6 Wake behaviour (P9)

On a push for endpoint E, the client opens **only** the pseudonym session
mapped to E (client-local table), runs a bounded sync/notification fetch,
then closes it. Clients SHOULD add random jitter before the fetch and
SHOULD batch catch-up syncs across pseudonyms at coarse intervals, so the
homeserver's access log does not show a lockstep
"push→sync-within-200ms" fingerprint per device. No pseudonym runs a
permanent sync loop.

### 5.7 Residual push-layer risks (accepted, documented)

* **Collusion / compulsion** of homeserver + UP operators re-links
  endpoints to pseudonyms to devices. Operator separation raises the cost
  from "one SQL query" to "two organizations cooperating"; it does not
  make the link impossible.
* **Timing**: the homeserver knows when it pushed each pseudonym's
  endpoint; anyone observing the device (or the UP server) can correlate
  wakes. Jitter (P9) and CGNAT reduce, not eliminate, this.
* Traffic volume per endpoint approximates per-room activity; this is
  inherent to push and is not mitigated.

## 6. Recovery and lifecycle

* **Registry backup.** The client maintains a versioned registry
  `{room_id, pseudonym_mxid, refresh_token/password, store_key}[]`,
  encrypted client-side (key in hardware keystore, recoverable via a
  user-held recovery code, SSSS-style). The ciphertext MAY be stored as
  anchor account data or media — the server hosts an opaque blob either
  way. Losing device **and** recovery code loses the pseudonym rooms;
  this is the accepted price of the server holding no mapping.
* **Message-history recovery** additionally requires per-pseudonym megolm
  key backups (standard mechanism, one per pseudonym account), OPTIONAL.
* **Deactivation.** The server cannot cascade (it does not know the
  pseudonyms). The client MUST iterate its registry and deactivate each
  pseudonym, then the anchor. Deployments SHOULD expire long-idle
  pseudonym accounts (no device activity for N months) as garbage
  collection for lost registries.
* **Alias** (if kept): released as in the base variant, including the
  reuse grace period.

## 7. Server-side delta vs the base variant

| Area | Change |
|---|---|
| Config | `user_aliases.mode: "server_mapped" \| "server_blind"` (capability-advertised) |
| Removed in blind mode | pseudonym provisioning + login-token endpoints, `invite_by_alias`, `create_room` wrapper, plaintext pseudonym account data, deactivation cascade, `room_pseudonyms` table |
| Added | blind-token issuance endpoint (+ per-anchor quota), `com.newnal.blind_token` registration auth type, spent-token set, idle-pseudonym GC (optional) |
| Unchanged | `user_type: "pseudonym"` exclusions (directory, MAU), membership gate, alias registry (optional), all standard Matrix APIs |

### 7.1 Deployments with MAS (MSC3861)

When the homeserver delegates authentication to a Matrix Authentication
Service (as the Newnal stack does — the telecom-authentication node is a
MAS fork with DID/wallet login), the auth-touching pieces of this design
move from Synapse into MAS. Synapse cannot host them: under MSC3861 the
legacy `/login` and `/register` endpoints are disabled and accounts that
exist only in Synapse's local tables are unreachable through MAS-issued
tokens.

Placement under MAS:

| Concern | Owner under MSC3861 |
|---|---|
| Blind-token issuance (§3.1) | MAS endpoint, authenticated by the anchor's MAS session |
| Blind-token redemption (§3.2) | MAS: verifies token + spent-set, creates the pseudonym **as a MAS user**, issues its device/refresh tokens directly |
| Pseudonym → Synapse propagation | MAS→Synapse provisioning as for any user, plus marking `user_type: "pseudonym"` on the Synapse side so directory/MAU exclusion and the membership gate keep working |
| Session/refresh lifecycle for pseudonyms | MAS (standard token machinery) |
| Deactivation | client iterates its registry against MAS; MAS propagates to Synapse |

Additional MUSTs specific to MAS deployments:

* Pseudonym registration MUST NOT pass through the deployment's identity
  flows — no telecom verification, no DID/VC presentation, no upstream
  IdP link. Any such step would attach the wallet/phone identity to the
  pseudonym and defeat the design. The blind token is the *only*
  credential consumed at redemption. In the Newnal MAS fork this means a
  registration path that bypasses the `newnal_did` login flow entirely.
* MAS MUST NOT persist any issuance→redemption correlation: no anchor ID
  on the created user row, no shared trace/request ID spanning the two
  operations, and audit logs for redemption keyed only by the (unlinkable)
  token nonce hash.
* MAS user records for pseudonyms carry no email, phone, or upstream
  subject; profile fields stay empty.

**Note on the base (server-mapped) variant under MAS.** The implemented
base variant assumes Synapse-native auth: it registers pseudonyms via
Synapse's internal registration and mints Synapse `m.login.token` values.
Under MSC3861 both assumptions fail. A MAS deployment of the base variant
needs the same relocation in miniature: pseudonym creation via MAS (with
the mapping retained, since that variant is deliberately server-mapped)
and a MAS-issued compat/device token in place of the Synapse login token,
returned through the existing `POST /rooms/{roomId}/pseudonym/login`
response shape.

## 8. Capability advertisement

```json
{"capabilities": {"com.newnal.user_alias": {
    "enabled": true,
    "mode": "server_blind",
    "blind_token_scheme": "rfc9474-rsa-2048"}}}
```

## 9. Summary of guarantees

| Claim | Holds? |
|---|---|
| Room members cannot link a person across rooms (IDs or device keys) | yes |
| Homeserver holds no record mapping anchor ↔ pseudonym ↔ rooms | yes |
| Push layer introduces no shared join key across pseudonyms | yes (P1–P8) |
| Homeserver can still infer links statistically (IP, timing, token counts) | yes — bounded, not eliminated (§3.3, §5.6, §5.7) |
| Survives homeserver + UP operator collusion | no (§5.7) |
| Device loss without recovery code preserves room access | no (§6) |
