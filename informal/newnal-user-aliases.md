# User aliases and per-room pseudonyms (`com.newnal.user_alias`)

This informal document specifies a **vendor extension** to the Matrix
Client-Server API implemented by the Newnal homeserver fork
(`newnal-web3-comm-messaging-node`). It is not part of the upstream Matrix
specification; all endpoints live under the `unstable` prefix in the
`com.newnal.user_alias` namespace.

Servers advertise support via `GET /_matrix/client/v3/capabilities`:

```json
{"capabilities": {"com.newnal.user_alias": {"enabled": true}}}
```

A **server-blind variant** — removing the server-held anchor↔pseudonym
mapping entirely (blind-token self-registration, per-pseudonym UnifiedPush,
client-held encrypted registry) — is specified separately in
[newnal-user-aliases-server-blind.md](./newnal-user-aliases-server-blind.md).

## Motivation

Matrix user IDs are permanent and are embedded in every event a user sends
(`sender`) and in their `m.room.member` state (`state_key`). Consequently:

1. every room member learns every other member's real MXID, and
2. two rooms containing the same MXID are trivially linkable, and
3. even with per-room display names, E2EE device keys (queried per user ID)
   cryptographically link a user's rooms.

This extension introduces:

* **User alias** — a unique, user-chosen, *changeable* handle used only for
  server-side invite resolution. Unlike room aliases (`#…`), user aliases
  are never written into room state or events.
* **Per-room pseudonym** — a dedicated shadow account, one per
  (user, room), which is the identity that actually joins the room. Each
  pseudonym has its own devices and E2EE keys, so protocol-level
  cross-room correlation is impossible for other members. The mapping is
  known only to the homeserver and the owning user.

## Identity model

| Concept | Grammar | Mutable | Visible to other users |
|---|---|---|---|
| Real MXID | `@localpart:domain` | no | never (with this extension used throughout) |
| Alias | `^[a-z0-9._=+-]{1,255}$` (case-folded) | yes | only as an out-of-band handle its owner shares |
| Pseudonym MXID | `@<prefix><32 hex>:domain` | per room, permanent | room members only |
| Displayname | free text | yes, per room | room members |

A released alias enters a server-configured *grace period* during which
only its previous owner may re-claim it (anti-impersonation).

## Client-Server API

All paths below are relative to
`/_matrix/client/unstable/com.newnal.user_alias`. All requests require
authentication. Endpoints marked *real accounts only* return
`403 M_FORBIDDEN` when called with a pseudonym's access token.

### Alias management (real accounts only)

| Method & path | Body | Response |
|---|---|---|
| `GET /account/alias` | — | `{"alias": "clara.k"}` or `{"alias": null}` |
| `PUT /account/alias` | `{"alias": "Clara.K"}` | `{"alias": "clara.k"}` (case-folded) |
| `DELETE /account/alias` | — | `{}` (starts the grace period) |
| `GET /alias_availability?alias=x` | — | `{"alias": "x", "available": true}` |

Errors: `400 M_INVALID_PARAM` (grammar), `400 M_USER_IN_USE` (taken, or
released by another user within the grace period).

### Pseudonym management (real accounts only)

| Method & path | Body | Response |
|---|---|---|
| `GET /pseudonyms` | — | `{"rooms": {"!r:s": {"pseudonym_user_id": "@p_…:s", "created_ts": 123}}}` |
| `POST /rooms/{roomId}/pseudonym` | `{}` | `{"pseudonym_user_id": "@p_…:s"}` — get-or-create, idempotent |
| `POST /rooms/{roomId}/pseudonym/login` | `{}` | `{"pseudonym_user_id", "login_token", "expires_in_ms"}` |

The `login_token` is consumed with the standard
`POST /_matrix/client/v3/login` `{"type": "m.login.token", "token": …}`,
which creates a **new device** (and therefore fresh E2EE identity) on the
pseudonym account. `404 M_NOT_FOUND` if no pseudonym exists for the room.

> **MAS/MSC3861 caveat.** This flow assumes Synapse-native authentication.
> On a deployment that delegates auth to the Matrix Authentication Service
> (MSC3861 — e.g. the Newnal telecom-authentication node), Synapse's
> legacy `/login` is disabled and Synapse-local pseudonym accounts are not
> reachable through MAS-issued tokens: pseudonym creation and token
> issuance must be mediated by MAS instead, keeping this endpoint's
> response shape. See §7.1 of the
> [server-blind spec](./newnal-user-aliases-server-blind.md) for the
> placement rules; the same relocation applies to this variant.

### Invites

`POST /rooms/{roomId}/invite_by_alias` — callable by any member session
with invite power in the room (typically a pseudonym session).

```json
{"alias": "clara.k"}
```

The server resolves the alias to the real user, provisions that user's
pseudonym for the room if needed, invites the pseudonym, and records the
mapping in the *target's* account data. Response:

```json
{"pseudonym_user_id": "@p_9f2c…:s"}
```

The inviter never learns the real MXID. `404 M_NOT_FOUND` for unknown,
deleted, or deactivated aliases.

### Room creation

`POST /create_room` — *real accounts only*. Accepts the standard
[`/createRoom`](https://spec.matrix.org/latest/client-server-api/#post_matrixclientv3createroom)
body with two extensions and one restriction:

* `invite_aliases` (`[string]`, optional) — invited via alias resolution
  after creation.
* `pseudonym_displayname` (`string`, optional) — displayname for the
  creator's pseudonym in this room.
* the standard `invite` field is **rejected** (`400 M_INVALID_PARAM`),
  since inviting raw MXIDs would defeat the extension.

The room is created *by a fresh pseudonym of the caller*, so the creator's
real MXID never appears in room state — not even in `m.room.create`.

```json
{
  "room_id": "!abc:s",
  "pseudonym_user_id": "@p_51d0…:s",
  "login_token": "syl_…",
  "expires_in_ms": 120000,
  "invited": {
    "clara.k": {"pseudonym_user_id": "@p_9f2c…:s"},
    "ghost":   {"errcode": "M_NOT_FOUND", "error": "Unknown alias"}
  }
}
```

## Account data

The server maintains global account data of type `com.newnal.pseudonym` on
each real account:

```json
{"rooms": {"!abc:s": {"pseudonym_user_id": "@p_…:s", "updated_ts": 1754870000000}}}
```

Clients watch this over `/sync` on the anchor (real) session to discover
new invites/pseudonyms, then obtain a login token per room.

## Client model

A client runs one *anchor* session (real account: alias management,
account data, this API — never joins rooms) plus one session per room
(pseudonym: sync, messaging, E2EE). Displayname is set per room on the
pseudonym; reusing one name across rooms re-links the user by their own
choice. Cross-signing between pseudonyms is impossible **by design** —
unlinkability and cross-room verifiable identity are mutually exclusive.

## Server behaviour

* Pseudonym accounts carry `user_type: "pseudonym"`, are excluded from the
  user directory and MAU counting, cannot log in by password, and cannot
  call the management endpoints (no nesting).
* Deactivating the real account deactivates all owned pseudonyms and
  releases the alias.
* With `require_pseudonyms_for_rooms` enabled the server refuses
  join/invite/knock of real local users (exempting existing members,
  appservice users, the server-notices user, and admins).
* The real↔pseudonym mapping is server-private. The threat model is
  "other room members", not "the homeserver operator": the server can
  always correlate, since it must route invites and mint pseudonym logins.

## Known limitations

* Alias resolution is local to one homeserver; cross-server invite-by-alias
  needs a discovery/federation extension (future work).
* Rooms joined with a real MXID before this extension keep that history.
* Timing/stylometric correlation is out of scope.
