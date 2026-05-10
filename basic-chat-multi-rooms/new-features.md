# basic-chat-multi-rooms — possible future improvements

This file collects forward-looking ideas surfaced while studying this folder. It is intentionally separate from `notes.md` (Q&A study notes) so the two don't get tangled. Each entry is anchored to code with `path:line` and explains *what*, *why*, and *rough how*.

## 1. Distinguish account-invite vs room-invite in the CLI

**What.** The single `--invite|-i <invite>` flag (`worker/index.js:12`) is consumed only by `ChatAccount` to pair you onto someone's *account base* (`worker/chat-account.js:42-52`). There is no CLI path to join a specific *room* by invite — that flow only exists in the UI (`ui/root.jsx:106-114` → `joinRoom`). Yet `README.md:18`'s comment says `# user2: join room`, which mirrors the `basic-chat` one-room mental model and is misleading here.

**Why.** Two genuinely different invites coexist in this app (see `notes.md` Q on architecture):

- **Account invite** — adds the bearer as a writer on your account base; effectively "another device of yours" (multi-device sync of your room list). Currently the only thing `--invite` does.
- **Room invite** — adds the bearer as a writer on a specific room base; "another participant in this conversation". Currently UI-only.

A first-time reader following the README will assume "user2 joins user1's room", but what actually happens is "user2 becomes a second device of user1" — a different security and sharing posture.

**Rough how.**

- Rename `--invite` → `--account-invite|-a` in `worker/index.js:12`.
- Add `--room-invite|-r <invite>` that, once `ChatAccount` is ready, calls `account.joinRoom(invite)` for the supplied invite (mirrors what `ui/root.jsx`'s join flow does — `worker/worker-task.js:35-37`).
- Update the README so each `pear run` example labels which invite kind it's passing, and split the Usage section into two scenarios: "second device" (account-invite) and "second participant" (room-invite).

## 2. Print every room's invite at startup, not just the account invite

**What.** `worker/index.js:34` logs only `Account Invite: …`. To get a room's invite you have to open the UI (`ui/root.jsx:45`).

**Why.** When working through CLI examples or scripting tests, having room invites in the worker stdout is much friendlier than spinning up the UI just to copy a string. Combined with feature 1, this would make the whole join flow scriptable.

**Rough how.** After `await workerTask.account.openRooms()` finishes, iterate `account.rooms` and `console.log` each `id`, `name`, and `invite`. Subscribe to a future `account.on('room-added', …)` event to log new ones too.

## 3. Replace `Math.random().toString(16).slice(2)` IDs with crypto-random IDs

**What.** Both new room IDs (`worker/chat-account.js:171`, `:187`) and new message IDs (`worker/chat-room.js:167`) are generated with `Math.random().toString(16).slice(2)`.

**Why.**

- Not collision-resistant — `Math.random()` is ~52 bits of entropy, and a single hex slice without padding can be as short as 12 hex chars (~48 bits). Across busy rooms with multiple writers, collisions are possible and silently break the `key: ['id']` collection invariant.
- Not seeded by a CSPRNG — fine here because IDs aren't security-sensitive, but inconsistent with the rest of the Holepunch stack which uses `hypercore-crypto` / `b4a` everywhere.

**Rough how.** Use `hypercore-crypto.randomBytes(16).toString('hex')` (already imported transitively) or `b4a.toString(crypto.randomBytes(16), 'hex')`. Single-line change in three call sites.

## 4. Order messages by an autobase-derived sequence, not `Date.now()`

**What.** Outgoing messages stamp `info.at = Date.now()` (`worker/worker-task.js:39`). Sorting in both `chat-account.js:218` and `worker-task.js`'s old equivalent uses `info.at`.

**Why.** Wall-clock ordering across peers is unreliable: clock skew between two writers reorders the conversation; a writer with a wrong timezone or NTP-broken clock pushes their messages to the wrong end of the list. Autobase already gives each appended block a deterministic position once linearised — that's the natural source of order.

**Rough how.** When materialising messages out of the view, expose the autobase sequence (or HyperDB's insertion order) and sort by that. Keep `info.at` as a display-only timestamp. Requires reading what HyperDB exposes about insertion order — possibly attach the seq during `_setupRouter`'s `add-message` handler before `view.insert`.

## 5. Leave room

**What.** No way to remove a room from your account once added. `chat-account.js:118-131`'s router has no `remove-room` op, and there's no UI affordance.

**Why.** Long-running accounts accumulate rooms forever. Storage grows monotonically — each room has its own corestore namespace and its own autobase (`chat-account.js:159, 173`). Joining a test room becomes a permanent decision.

**Rough how.**

- Add `remove-room` to the dispatch namespace (`schema.js:90-95`) — request type just `{ id: string }`.
- In `ChatAccount._setupRouter`, on `remove-room`: `view.delete('@basic-chat-multi-rooms/rooms', { id })`, then `await this.rooms[id].close()` and `delete this.rooms[id]`.
- New HRPC method `leaveRoom(id)`; UI gets a "leave" button per room.
- Two flavours worth distinguishing in the API:
  - **Leave (keep local history).** Close the room base and drop the swarm topic, but keep the namespaced corestore on disk. Reversible if you re-add the room via the same invite later.
  - **Leave and forget.** Above plus `await this.store.namespace(id).clear()` (or rm the underlying storage). Destructive, frees disk.
- True "remove me as a writer from the room itself" needs `Autobase.removeWriter` (if available in the version pinned) — out of scope for a first pass; the local leave above is enough for the common case.

## 6. Delete message

**What.** Messages are append-only — no delete op (`worker/chat-room.js:166-171`).

**Why.** Standard chat affordance. Also useful when a sender realises they leaked an invite or pasted into the wrong room and wants to retract.

**Rough how.**

- Add a `delete-message {id: string}` op to the dispatch namespace (`schema.js:92-95` is where the room-level ops are registered).
- In `ChatRoom._setupRouter`, on `delete-message`: either `view.delete('@basic-chat-multi-rooms/messages', { id })` for hard-delete, or `view.insert(..., { id, text: '', info: { ...existing.info, deleted: true } })` for a tombstone the UI can render as "[deleted]".
- New HRPC method `deleteMessage(roomId, id)`. UI gets a delete affordance per message.
- **Authorization caveat.** Without the identity layer, *any* writer in the room can delete *any* message — the autobase has no concept of "the original sender" beyond writer-key, and message records don't carry their writer-key. Flag this as a known limitation; the proper fix is the combined "multi-rooms + identity" example (entry 12) where `proof` lets the dispatcher verify deletion came from the original sender.

## 7. Edit message

**What.** Messages are append-only — no edit op either (`worker/chat-room.js:166-171`).

**Why.** Same as delete: standard chat affordance, fixes typos, retracts mistakes without losing thread context.

**Rough how.**

- Add an `edit-message {id: string, text: string, editedAt: int}` op to the dispatch namespace.
- In `ChatRoom._setupRouter`, on `edit-message`: re-insert the row at the same `key: ['id']` with the new text and an `info.editedAt` stamp (HyperDB upserts on the keyed field).
- New HRPC method `editMessage(roomId, id, text)`. UI shows "(edited)" suffix when `info.editedAt` is set.
- **Authorization caveat.** Same as delete — any writer can edit any message until identity is layered in. Worth a comment in the dispatcher pointing at entry 12.
- **Edit history (optional).** If preserving the original text matters, store edits in a sibling collection `message-edits` keyed by `[messageId, editedAt]` rather than overwriting in place; the read materialiser folds them in.

## 8. Rename room (existing room)

**What.** No way to rename a room after creation. `ChatRoom.addRoomInfo` (`chat-room.js:154-160`) writes the room's metadata once into its own base; `ChatAccount.joinRoom`'s mirror loop (`chat-account.js:193-203`) reflects that into the account base. Neither is exposed to the UI as an editable field.

**Why.** Test rooms get bad names. Group chats get repurposed. Today the only fix is `--reset` and re-pair, which loses the room's history and forces every participant to re-join.

**Rough how.**

The plumbing for this is half-built — the schema and routers already accept `add-room` (`schema.js:94`) keyed by `id`, so re-appending with the same id and a new name is a valid upsert. What's missing is a clean explicit op and a cross-base propagation path:

- **Explicit op.** Add `rename-room {id, name}` to the dispatch namespace, registered on **both** `ChatRoom._setupRouter` (so room writers see the new name) and `ChatAccount._setupRouter` (so paired devices on this account see it in their room list). Cleaner than overloading `add-room`.
- **Propagation.** New `ChatAccount.renameRoom(roomId, newName)`:
  1. Append `rename-room` to `this.rooms[roomId].base` (the room base) so all room participants converge on the new name.
  2. Append `rename-room` to `this.base` (the account base) so all of *your* devices see the change in their account view.
  3. Update `this.rooms[roomId].name` in memory.
- **Auto-mirror for joiners.** The existing `chat-account.js:193-203` mirror already pulls the room name from the room base into the account base on update — extend it so a `rename-room` on the room base triggers a mirror to the account base, even for non-originating devices. (Today it only fires on the joiner's first sync; subsequent renames wouldn't propagate to all account devices.)
- New HRPC method `renameRoom(id, name)`; UI inline-edit on the room title.
- **Authorization caveat.** Same shape as edit/delete — without identity, any room writer can rename. Acceptable for most rooms but worth noting.

## 9. Tighten `joinRoom`'s metadata-mirroring loop

**What.** `chat-account.js:193-203` writes a fresh `add-room` to the account base **every time** the room base updates and the room name differs from the account's cached copy.

**Why.** If a room's metadata changes frequently (rename loop, repeated `addRoomInfo` from a paired device), the account base accumulates redundant `add-room` blocks. Each one is small but the writes are otherwise pointless.

**Rough how.** Compare against the *current account view*, not just `room.name`. Or debounce the mirror so a flurry of room updates collapses to one account write. Or only mirror when the room base reaches a stable steady state.

## 10. Persist UI room selection across reloads

**What.** `ui/root.jsx:9` initialises `selectedRoomId` to undefined, then falls back to `rooms[0]?.id` (`ui/root.jsx:14`). On every reload you land on whichever room sorts first.

**Why.** Annoying for users with many rooms; they have to re-select every time.

**Rough how.** Persist to `localStorage` (Pear UI is a regular browser context); restore on mount. Tiny change.

## 11. Document the `info` JSON shape in the schema

**What.** Three places use `info: { type: 'json' }` (`schema.js:34`, `:47`) — for rooms and messages. The actual shape is `{ name, at }` for messages and effectively unused for rooms, but nothing in the schema or code states it.

**Why.** A first-time reader of `schema.js` can't tell what's expected. Future contributors may stuff incompatible shapes in.

**Rough how.** Either replace `type: 'json'` with named subschemas (`message-info`, `room-info`) registered above, or document the shape inline with a comment. Subschemas are the more "Hyperschema-native" path.

## 12. Combine with the identity layer

**What.** This folder strips identity (see `notes.md`); `basic-chat-identity` adds it on top of `basic-chat`. A combined "multi-rooms + identity" example would be the natural next step.

**Why.** Real multi-room chat needs both: per-message authorship (identity) **and** per-room access control (room invites). Multi-device sync (account invites) on top of that gives you Keet's basic shape.

**Rough how.** Re-introduce `proof` on `add-message` and the `Identity.attestData` / `Identity.verify` calls (per `basic-chat-identity/worker/worker-task.js:43-44, 58-63`) inside `ChatRoom`, leaving `ChatAccount` untouched. Account writers don't need per-block identity proofs because the account base only carries metadata, not messages.

## 13. Presence — see who is online (per room)

**What.** No notion of presence anywhere in the app. `worker/worker-task.js:21` wires every swarm connection straight into `store.replicate(conn)`; `worker/chat-room.js:69` joins each room's discovery key but nothing surfaces "who is connected right now". The UI's room view (`ui/root.jsx:40-67`) shows messages only — no member list, no online dots.

**Why.** Concretely framed by Pear School (see the Pear School Q in `basic-chat-identity/notes.md` for the full setup): an instructor needs to see which students are attending the live session before starting; students benefit from seeing their cohort online for study-group formation. Presence is also the prerequisite for typing indicators, "last seen" timestamps, and durable attendance records.

**Rough how.**

This depends on entry 12 (combine with identity layer) for the **"which students"** half of the question — without identity all you can show is anonymous peer counts. Build it in three layers:

- **Layer A — Anonymous peer count (works today, no identity).** Track `swarm.connections` per room's discovery topic. Easiest hook: count peers in `room.base.discoveryKey`'s `swarm.peers` list, debounced, push over HRPC as `presence-count`. Useful as a smoke test and as a fallback when identity is absent. Limitation: a peer with two devices counts as two; a peer that just dropped counts until the swarm notices.
- **Layer B — Identity-mapped soft presence (needs entry 12).** Open a **Protomux side-channel** on each peer connection (Protomux is already on the wire because corestore replication uses it). On connect, both sides exchange a signed hello:
  ```js
  // pseudo
  const hello = { name, identityPub, deviceKey, sentAt: Date.now() }
  const helloProof = Identity.attestData(encode(hello), deviceKeyPair, deviceProof)
  channel.message.send({ hello, helloProof })
  ```
  Receiver verifies `helloProof` against `hello.identityPub`, then pins `connection → identityPub`. Heartbeat every ~10s with `{ identityPub, ts }`; mark a peer offline when `lastSeen > 2 * heartbeatInterval`. Maintain `roomId → Map<identityPub, { name, since, lastSeen, devices: Set<deviceKey> }>` so multi-device users (entry that account-invites enable) collapse to one identity in the UI. Push the materialised map over HRPC as `presence-update`.
- **Layer C — Pear School: durable attendance log (optional, identity-required).** Soft presence is ephemeral — close your laptop, you disappear. For a real classroom you also want a permanent record. Add a `checkin {identityPub, sessionId, at}` op signed by the student's identity (`Identity.attestData`) and append-only on the room base. Instructor can then query "who checked in to session X" weeks later. Rate-limit per identity per session at the dispatcher to avoid log spam.

**UI.** New right-side panel in `ui/root.jsx` per selected room showing the member list with name, identity-shortcode, and an online dot. For Pear School: an instructor-only view (visibility gated by `identityPub === INSTRUCTOR_IDENTITY_PUBKEY`) listing every enrolled student with status `Online | Recently active | Offline | Not yet attended`.

**Privacy / authorization.**

- Anyone connected to a room base can already enumerate its writers from the autobase metadata, so "writer keys are visible" is not new. What presence adds is **identity-binding** ("this writer key right now is held by student S") and **liveness** ("S is connected at this moment").
- For Pear School, the default should be **mutual visibility within a room** (everyone enrolled in the course sees everyone else online). If the instructor wants invisible/anonymous student mode, gate the presence-hello broadcast behind a `roomPolicy.presenceVisible` flag on the room.
- The hello's `identityPub` is signed, so a peer cannot impersonate another student even on the protomux channel — this matters when "online" gates anything UX-significant (raise hand, submit attendance).

**Why protomux side-channel and not autobase ops.** Presence is transient and high-frequency (heartbeats every few seconds). Putting it through autobase would balloon the append-only log with noise that has no long-term value. Protomux gives you an out-of-band stream on the connection that's already there — same mechanism Keet uses for typing indicators and the "now playing" track in voice rooms.

## 14. Direct messages — teacher↔student and student↔student DM rooms

**What.** No way to chat 1-to-1. Every conversation today is a `ChatRoom` shown in the public Rooms sidebar (`ui/root.jsx:69-91`); there is no notion of a private peer-to-peer channel that bypasses the manual invite copy/paste step.

**Why.** Pear School use cases (see Pear School Q in `basic-chat-identity/notes.md`):

- **Teacher → student.** Private feedback on a submission, scheduling a 1:1, escalating a flagged answer — none of which belongs in the public course room.
- **Student → teacher.** Asking a question without broadcasting it to the cohort.
- **Student ↔ student.** Study partners, group-project coordination. Often gated per course (some instructors want it; others want all interaction on official channels).
- **Small group DMs.** "The four of us doing the capstone together" — same mechanism, N > 2.

### Core insight: a DM is just a small private room

Mechanically, a DM is a `ChatRoom` with exactly the same wire format and storage layout — autobase, encrypted, blind-paired writers, message dispatch. **The only thing that changes is how the pairing happens** and how the room is presented in the UI.

Concretely:

- One autobase per peer-pair (teacher↔alice, alice↔bob), namespaced in the corestore as `this.store.namespace('dm:' + sortedIdentityPubs.join(':'))`.
- Two writers in the typical case (one per identity); N writers for a group DM.
- Stored alongside the public rooms but tagged differently — see schema change below.

So the *room* primitive is unchanged. What needs to be designed is **invite-less pairing**.

### The pairing problem (the actually-new bit)

Today, joining a room means: (1) someone hands you a `z32`-encoded `BlindPairing` invite string, (2) you paste it into `--invite` or the UI's join box (`ui/root.jsx:106-114`), (3) `BlindPairing.addCandidate` does the handshake (`worker/chat-account.js:42-52` and `chat-room.js:39-49`).

For DMs that's user-hostile — clicking "DM Alice" should just work. The fix uses identity (entry 12) as the discovery anchor:

- **Identity-derived DM mailbox topic.** Each identity has an implicit Hyperswarm topic derived from its public key:
  ```js
  const mailboxTopic = crypto.hash([Buffer.from('dm-mailbox-v1\0'), identityPub])
  ```
  At worker startup, `swarm.join(mailboxTopic)` (alongside the existing account/room topic joins). Anyone who knows your `identityPub` can find you.
- **DM-request handshake over Protomux.** When the initiator connects to the recipient's mailbox topic, both sides open a `pear-school/dm` Protomux channel (mux it onto the same connection that's already replicating, like presence in entry 13). The initiator sends:
  ```js
  {
    fromIdentityPub,
    fromName,
    requestedAt: Date.now(),
    proof: Identity.attestData(encode({ fromIdentityPub, toIdentityPub, requestedAt }), deviceKeyPair, deviceProof)
  }
  ```
- **Recipient policy gate.** Recipient verifies `proof` against `fromIdentityPub`, then checks a per-deployment policy (see "Policy" below). If accepted: recipient creates a fresh DM `ChatRoom` (its own autobase + namespace), generates a `BlindPairing` invite, and ships `{ key, encryptionKey, invite }` back over the same Protomux channel — encrypted to `fromIdentityPub` so an eavesdropper on the swarm can't snoop.
- **Initiator pairs in.** Initiator decrypts, calls `BlindPairing.addCandidate` with the invite, becomes a writer. Both parties now share the DM room. The mailbox connection's job is done; from here on it's a normal autobase/replication flow.
- **Subsequent reconnects skip the handshake.** Once the DM room exists locally on both sides (registered in their account view as a `kind: 'dm'` row, see schema change below), the workers swarm its discovery key directly — same as for any room.

This is essentially `BlindPairing` with the invite delivered over an automatically-discovered side-channel instead of out-of-band paste.

### Policy: who is allowed to DM whom

Same mechanism, different policy per actor pair. Pear School defaults:

- **Teacher → any enrolled student.** Always accepted. Teacher's identity is the trust anchor; students implicitly authorise contact from it on enrollment.
- **Student → teacher.** Always accepted on the teacher's side (office-hours model). Optionally rate-limited to prevent abuse.
- **Student → student.** Gated by a course-level flag set by the teacher (`courseRoom.policy.studentDMs: 'open' | 'opt-in' | 'closed'`). On `'opt-in'`, the recipient sees a "Bob wants to DM you — accept?" prompt; on `'closed'`, the request is dropped.
- **Outsider → anyone.** Drop. The mailbox topic is technically discoverable by anyone with your `identityPub`, but in Pear School the identity-pub list is internal to the course autobase, so this is naturally bounded. Still worth a default-deny for unknown identities.

Implement the policy as a callback on the `dm-request` Protomux handler: `account.onDMRequest = (from, to) => 'accept' | 'reject' | 'prompt'`. Default to `prompt` for unknown identities.

### Schema change

Extend the existing `room` schema in `schema.js:28-36`:

- Add `kind: string` (values `'group' | 'dm' | 'group-dm'`).
- Add `participants: array of buffer` (identity pubs of the intended members; empty for `kind: 'group'` since group rooms admit by invite, not by identity allowlist).

The router already upserts on `id`, so existing rows stay compatible (default `kind: 'group'` if absent).

### UI

Split the sidebar (`ui/root.jsx:69-91`) into two sections:

- **Direct Messages** (top): rows tagged `kind: 'dm'`, labelled by the *other* participant's display name (resolved from `participants[0|1]` via the identity → name lookup the identity layer maintains). Online dot from entry 13.
- **Rooms** (below): rows tagged `kind: 'group'`, labelled by `room.name` as today.

Add a "DM" affordance everywhere a user is identified — for instance, click any name in the presence panel from entry 13 → `dmCreateOrOpen(thatIdentityPub)`. If a DM room already exists, surface it; otherwise initiate the handshake described above.

### Dependencies and cross-refs

- **Hard dependency on entry 12** (identity layer) — the mailbox topic is identity-derived, the handshake proves identity, the policy gate identifies parties. Without identity there is no "Alice" to DM.
- **Soft dependency on entry 13** (presence) — knowing whether the recipient is online lets the UI show "Alice is online" before you start typing, and lets the initiator queue the DM-request retry instead of failing immediately if the recipient is offline.
- **Builds on existing primitives** — no new Holepunch component is required. `BlindPairing`, autobase encryption, Protomux side-channels, identity attestation are all already in scope. The new code is the mailbox-topic-join, the Protomux channel handler, the policy callback, the schema `kind` field, and the UI split.

### What to skip in a v1

- **Forward-secrecy / Signal-style ratchets.** Autobase's per-block encryption is fine for Pear School's threat model; layering Signal Protocol on top is overkill until there's a concrete reason.
- **Self-DMs.** Mechanically work (single-writer room) but UX-noise; just hide them.
- **Read receipts and typing indicators.** Both belong on the same Protomux side-channel as presence (entry 13) — defer until presence lands.
