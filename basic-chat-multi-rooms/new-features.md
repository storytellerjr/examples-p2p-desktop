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
