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

## 5. Leave-room / delete-room

**What.** No way to remove a room from your account once added. `chat-account.js:118-131`'s router has no `remove-room` op.

**Why.** Long-running accounts accumulate rooms forever. Storage grows monotonically (each room has its own corestore namespace and own autobase). Joining a test room becomes a permanent decision.

**Rough how.**

- Add `remove-room` to the dispatch namespace (`schema.js:90-95`).
- In `ChatAccount._setupRouter`, on `remove-room`, delete the row from `@basic-chat-multi-rooms/rooms` and tear down `this.rooms[id]` (close the room, drop the swarm topic, optionally delete the namespaced corestore).
- New HRPC method `removeRoom(id)`. UI gets a delete button per room.

Note: deleting the local namespace is destructive — for "leave but keep history", just stop replicating; for "leave and forget", remove the storage too.

## 6. Edit / delete messages (tombstones)

**What.** Messages are append-only, no edit or delete (`worker/chat-room.js:166-171`).

**Why.** Standard chat affordance; also useful when a sender realises they leaked an invite or sent the wrong room.

**Rough how.** Use the standard append-only pattern: an `edit-message {id, newText}` op that inserts a tombstone-style record HyperDB can join with the original on read (or stores the edit history inline). Same for delete. The view materialiser would then surface the latest text + an "edited"/"deleted" marker.

## 7. Tighten `joinRoom`'s metadata-mirroring loop

**What.** `chat-account.js:193-203` writes a fresh `add-room` to the account base **every time** the room base updates and the room name differs from the account's cached copy.

**Why.** If a room's metadata changes frequently (rename loop, repeated `addRoomInfo` from a paired device), the account base accumulates redundant `add-room` blocks. Each one is small but the writes are otherwise pointless.

**Rough how.** Compare against the *current account view*, not just `room.name`. Or debounce the mirror so a flurry of room updates collapses to one account write. Or only mirror when the room base reaches a stable steady state.

## 8. Persist UI room selection across reloads

**What.** `ui/root.jsx:9` initialises `selectedRoomId` to undefined, then falls back to `rooms[0]?.id` (`ui/root.jsx:14`). On every reload you land on whichever room sorts first.

**Why.** Annoying for users with many rooms; they have to re-select every time.

**Rough how.** Persist to `localStorage` (Pear UI is a regular browser context); restore on mount. Tiny change.

## 9. Document the `info` JSON shape in the schema

**What.** Three places use `info: { type: 'json' }` (`schema.js:34`, `:47`) — for rooms and messages. The actual shape is `{ name, at }` for messages and effectively unused for rooms, but nothing in the schema or code states it.

**Why.** A first-time reader of `schema.js` can't tell what's expected. Future contributors may stuff incompatible shapes in.

**Rough how.** Either replace `type: 'json'` with named subschemas (`message-info`, `room-info`) registered above, or document the shape inline with a comment. Subschemas are the more "Hyperschema-native" path.

## 10. Combine with the identity layer

**What.** This folder strips identity (see `notes.md`); `basic-chat-identity` adds it on top of `basic-chat`. A combined "multi-rooms + identity" example would be the natural next step.

**Why.** Real multi-room chat needs both: per-message authorship (identity) **and** per-room access control (room invites). Multi-device sync (account invites) on top of that gives you Keet's basic shape.

**Rough how.** Re-introduce `proof` on `add-message` and the `Identity.attestData` / `Identity.verify` calls (per `basic-chat-identity/worker/worker-task.js:43-44, 58-63`) inside `ChatRoom`, leaving `ChatAccount` untouched. Account writers don't need per-block identity proofs because the account base only carries metadata, not messages.
