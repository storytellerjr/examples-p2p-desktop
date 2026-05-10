# basic-chat-multi-rooms — study notes

## Q: What is the new functionality compared to `basic-chat-identity`?

Two changes at once: a **new feature added** (multiple rooms, organised under a per-user account) and a **feature removed** (the whole keet-identity-key signing/verification layer). Both directions matter — this is not a strict superset of `basic-chat-identity`.

### Added: a two-tier Autobase architecture

`basic-chat-identity` had **one** Autobase per process: the room itself. `basic-chat-multi-rooms` now has **N + 1** Autobases per process:

- One **account** Autobase (`worker/chat-account.js:13`). This is the base the worker pairs to first. Its only collections are `invites` (the account-level pairing invite) and `rooms` (the list of rooms this account belongs to). It explicitly refuses message ops — `chat-account.js:128-130` registers `add-message` as `throw new Error('Invalid op')`. Messages do not live here.
- One **room** Autobase per row in `rooms` (`worker/chat-room.js:11`). Each room has its own corestore namespace (`this.store.namespace(item.id)` at `chat-account.js:159` and `:173`), its own `invites` collection (room-level pairing) and its own `messages` collection. This is structurally what `chat-room-identity.js` was, minus the identity bits.

So the data model now has two kinds of invite and two kinds of writer:

- **Account invite**: printed at startup as `Account Invite:` (`worker/index.js:34`). Whoever you hand it to becomes a writer on your *account base* — they sync your room list, not your messages. Conceptually this is "another device of the same person" / multi-device sync.
- **Room invite**: returned by `ChatRoom.getInvite()` (`chat-room.js:132`) and surfaced to the UI per room. Whoever you hand it to becomes a writer on that specific *room base* — they get the messages of that room.

### Added: room lifecycle methods on the account

`ChatAccount` exposes three new operations the room-only example didn't need:

- `openRooms()` (`chat-account.js:156-168`) — on startup, walk the `rooms` collection and instantiate a `ChatRoom` for each one under its own namespace. This is what reconstitutes your room set after a restart.
- `addRoom(name, info)` (`chat-account.js:170-184`) — generate a fresh id, create a new `ChatRoom` under a new namespace, call `room.addRoomInfo()` to seed the room's *own* DB with `{name, invite, info}`, then append `add-room` to the **account base** so other devices on this account see the room appear.
- `joinRoom(invite)` (`chat-account.js:186-206`) — create a `ChatRoom` configured as a pairing candidate. When the room base updates and a name becomes available via `room.getRoomInfo()`, the account mirrors that name back into the account base. This is why the room ends up with a human-readable name on the joiner's side: it's pulled out of the room's own DB, not the invite.

### Added: per-room messages on the wire

The HRPC surface gains room-aware variants (`schema.js:98-120`):

- `rooms`: server pushes the current room list to UI.
- `add-room`, `join-room`: UI → worker.
- `messages`: now carries `{messages, roomId}` (`schema.js:55-61`) instead of a flat array.
- `add-message`: now carries `{text, roomId}` (`schema.js:62-68`).

The worker plumbs this in `worker-task.js:25` (`account.on('messages', (roomId, messages) => rpc.messages({messages, roomId}))`) and the UI splits messages by room id in `ui/root.jsx:15` (`messages[roomId] || []`).

### Added: room sidebar + create/join inputs in the UI

`ui/root.jsx:69-91` is brand new — a left column listing rooms with a select button. The right column is the message view, and now also shows the **room invite** at the top (`ui/root.jsx:45`) so you can hand it out. There are two top inputs (`ui/root.jsx:96-114`): one to create a room, one to join by invite. The verify-mark column from `basic-chat-identity` is gone (see "Removed" below).

### Removed: the entire identity layer

`basic-chat-identity` signed every message with a device key derived from a 24-word mnemonic and verified those signatures on read. **Multi-rooms drops all of it.** Concretely, the following are gone in `basic-chat-multi-rooms`:

- The `keet-identity-key` import, the `--mnemonic` flag, the `identity-mnemonic.txt` read/write dance and the `mkdir`-before-write fix in `worker/index.js`.
- The `identity`, `deviceKeyPair`, `deviceProof` fields on `WorkerTask` and the `Identity.from` / `bootstrap` calls in `_open` (`basic-chat-identity/worker/worker-task.js:34-37`).
- `Identity.attestData(...)` around outbound messages (`basic-chat-identity/worker/worker-task.js:43-44`).
- `Identity.verify(...)` and the `msg.info.verified` flag in the per-message loop (`basic-chat-identity/worker/worker-task.js:58-63`).
- The `proof` field in the `message` schema and the `addMessage(text, proof, info)` signature on the room.
- The `✅` / `🛑` per-message badge in the UI.

So messages in `multi-rooms` are *not* cryptographically tied to a specific human identity — the only auth is "you are a writer on this room base because someone admitted you via the room invite". There is no cross-room "this is the same person" guarantee.

### Net take

The two examples are siblings, not parent/child. `identity` proves who sent a message inside one room; `multi-rooms` proves who has access to which set of rooms (via account writers) and to each room individually (via room writers), but says nothing about message authorship. A combined "multi-rooms + identity" example would presumably re-introduce `proof` on `add-message` and verify per message in `ChatRoom._messages`, while keeping the account/room split intact.

## Q: So `basic-chat-multi-rooms` is really an extension of `basic-chat`, not `basic-chat-identity`?

Yes — the file genealogy confirms it. `basic-chat-identity` was a *fork* off the same base (`basic-chat`); `basic-chat-multi-rooms` is a different fork off the same base. Neither one descends from the other.

Evidence:

- `basic-chat/worker/` already contains the file `chat-room.js` (plain name, no `-identity` suffix). `basic-chat-multi-rooms/worker/chat-room.js` is that same file, **purely additive**: same constructor shape gains `{ name, info }`, same router gains an `add-room` op so the room can store its own metadata, plus two new helpers `getRoomInfo` / `addRoomInfo` (`worker/chat-room.js:150-160`). No identity-related code was removed because it was never there.
- `basic-chat-identity` instead renamed the file to `chat-room-identity.js` and threaded `proof` through `addMessage(text, proof, info)` — a divergent change `multi-rooms` never inherited.
- The new file in `multi-rooms` is `worker/chat-account.js`, which sits *above* `chat-room.js` and owns the room list + account-level pairing. That layer doesn't exist in either of the other two examples.

So the right mental model is:

- `basic-chat` = single room, no identity. The trunk.
- `basic-chat-identity` = `basic-chat` + per-message identity attestation/verification.
- `basic-chat-multi-rooms` = `basic-chat` + an account layer that owns N rooms.

The two extensions are orthogonal. Combining them would mean re-introducing `proof` on `add-message` inside `ChatRoom` while keeping the `ChatAccount` wrapper untouched.

## Q: Can I run `basic-chat-multi-rooms` without overwriting the `basic-chat` stores? Can both keep working side-by-side?

Yes — but only if you give them different `--store` paths. The two examples are fully isolated *at the code level*, but the upstream READMEs collide *at the path level*.

### Code-level isolation: complete

Every block written to disk is stamped with this folder's schema metadata: namespace `@basic-chat-multi-rooms/...` (`schema.js:11-12`), its own hyperdb collection IDs (`schema.js:71-72`), its own hyperdispatch offsets (`schema.js:90-91`). `basic-chat` bakes `@basic-chat/...` instead. Different ID space → multi-rooms cannot read or modify `basic-chat`'s data, and vice versa. There is no shared on-disk format and no shared codepath.

### Path-level collision: was present in the README

`basic-chat-multi-rooms/README.md` originally suggested `--store /tmp/user1` and `--store /tmp/user2` — the **exact same paths** `basic-chat/README.md:14-17` suggests. If you copy-paste both READMEs back-to-back, the second app to start hits `Uncaught Error: Unknown collection type: N` from HyperDB the moment it tries to reopen the existing on-disk view.

The crash is *protective*, not *destructive*: the new app dies before writing anything, so the existing data on that path is never corrupted. But neither app is usable on the shared path until you wipe it (`--reset`) or move one of them.

Same failure mode is documented in `basic-chat-identity/notes.md` for the same reason.

### Fix applied

This folder's README now uses folder-scoped paths everywhere — `/tmp/multi-rooms-user1`, `/tmp/multi-rooms-user2` — across the Usage, Build Pear app, and Troubleshoot blocks. With those paths in place, `basic-chat` keeps using its own `/tmp/user1` and `/tmp/user2` undisturbed, and both apps coexist on the same machine.

The general rule is also now written into `CLAUDE.md` ("README conventions for example folders") so every future folder's README starts out with folder-scoped paths and the same explanatory note.

## Q: A Pear School student paid, joined a room and received an invite. They want a refund. How do I rotate the room's invite after their writer key has been removed?

This question sounds like it has a small mechanical answer, and at one level it does. But the underlying problem (cutting an ex-member off from the room) is fundamentally bigger than invite rotation in this stack — and it's worth understanding why before writing any code.

### Short answer (the mechanical part)

An "invite" in this app is a row in the `@basic-chat-multi-rooms/invites` collection keyed by `id` (`schema.js:73-77`). The pairing handler looks the invite up by id every time someone presents it: `chat-room.js:78-79` does `findOne(..., { id: request.inviteId })` and silently drops the request if the row is missing. So to deactivate an invite, you only need to delete (or mark) that row.

Today's code, however, assumes a single perpetual invite — `getInvite()` returns the existing one if present (`chat-room.js:132-136`). To support rotation you need two small additions to the dispatch namespace (`schema.js:92-95`):

- **`revoke-invite { id }`** — `_setupRouter` handler calls `view.delete('@basic-chat-multi-rooms/invites', { id })`.
- **A `rotateInvite()` helper on `ChatRoom`** that, in one logical operation: appends `revoke-invite` for the current invite's id, then `add-invite` with a fresh `BlindPairing.createInvite(this.base.key)`, and returns the new `z32`-encoded string.

Once that lands, any old invite string the refunded student still holds (or shared with friends) is dead at the pairing handler — `findOne` returns null, request drops. New cohort members get the new string.

That's the literal answer to "change the invite code".

### Why rotating the invite is only half the answer

The invite controls **who can become a *new* writer**. It does *not* control:

- What the ex-student already has on disk.
- What they can decrypt going forward.
- Whether they can keep replicating the room.

The student paired in once and that handshake gave their device three things:

1. **The autobase key** (`chat-room.js:46-48`'s `res.key` from `BlindPairing.addCandidate`).
2. **The autobase encryption key** (`res.encryptionKey`, used at `chat-room.js:55`'s `encrypt: true, encryptionKey`).
3. **Their own writer key on the base** (`addWriter(request.userData)` at `chat-room.js:81`).

Removing their **writer key** (item 3) only stops them appending *new* blocks. Items 1 and 2 they keep forever, on disk, beyond your reach. With both, they can:

- Compute the room's discovery topic (`base.discoveryKey` is a deterministic function of the autobase key) and join the swarm.
- Decrypt every block they replicate, past and future, including everything other writers append after the refund.
- Be served by any honest writer who isn't tracking your revocation list — Hyperswarm doesn't ask "is this peer authorised" before gossipping.

So **invite rotation closes the door for new joiners; it does not evict the existing ex-member**. For the Pear School refund scenario, that's not enough — the whole point of the refund is "you no longer get the course content".

### What real revocation actually requires in this stack

The honest answer for Pear School: revocation = **re-key the room**.

The mechanic is essentially "create a new room and bring everyone except the ex-student along":

1. **Instructor (or whoever holds the trust anchor) creates a new `ChatRoom`** with its own autobase, its own encryption key, its own discovery topic.
2. **Issue fresh invites to every remaining cohort member** over a side-channel that the ex-student isn't on (over their existing identity-mailbox connections — see DM mechanic in `new-features.md` entry 14, same shape).
3. **Each remaining member pairs into the new room** the normal way (`BlindPairing.addCandidate`).
4. **Old room is abandoned.** The ex-student still has the old encryption key but nobody else is appending there anymore, so there's nothing new for them to decrypt.
5. **Optionally migrate history forward.** A trusted writer (instructor) reads the old base's `messages` collection, re-appends the entries worth keeping under their own writer key in the new base. The ex-student is *not* a writer on the new base, so they can never produce new entries here. Their historical messages, if migrated, are now attributed to the migrating writer — for Pear School this is fine because the instructor's identity becomes the canonical record of "what happened in the course".

Two important caveats:

- **The ex-student keeps everything they already saw.** Anything appended before the rekey is theirs to decrypt forever; that's a property of any append-only encrypted log without per-message ratcheting (Signal Protocol-style PFS). For a course this is usually acceptable — they paid, then they got refunded, so they keep the lectures up to that moment but lose access to anything new.
- **`Autobase.removeWriter` (if your pinned version supports it) is a *complement* to re-keying, not a substitute.** It cleanly invalidates any future blocks the ex-student tries to publish on the *old* base — useful in the window between "decision to refund" and "new room is up". But it doesn't touch decryption.

### Pear School: what to actually build

A clean refund flow on top of the multi-rooms structure:

1. **Refund event arrives** (from the registrar/payments backend, or instructor's manual action). Carries `{ studentIdentityPub, courseId, refundedAt }`, signed by the registrar's identity.
2. **Course room is re-keyed.** Instructor's worker creates a new `ChatRoom`, pairs every remaining enrolled student in via their identity-mailbox (DM handshake from `new-features.md` entry 14), optionally migrates historical material the instructor wants to preserve (`add-material` ops, pinned announcements, etc).
3. **Old room is marked superseded.** A `superseded-by { newRoomId, supersededAt, reason: 'refund' }` op on the *old* room base tells any client still watching it where the conversation moved. UI stops showing the old room as active.
4. **Refund record is durable.** Append `refund { studentIdentityPub, courseId, refundedAt, refundedBy, oldRoomId, newRoomId, proof }` to the *registrar's* base (or the course's enrollment table, whichever holds the truth). This is the audit trail.
5. **Repeat per affected room.** If the student was in a study-group sub-room (group DM, see entry 14), each of those is also a candidate for rekey. The participants in those rooms decide independently whether they want to bring the ex-student along (i.e. is it a paid-cohort artefact or a personal friendship channel?).

Step 1's signed refund event is what `_setupRouter` would gate revocation on — the instructor's worker doesn't act on a refund unless the registrar's signature verifies, otherwise anyone with write access could weaponise this and rekey everyone out.

### Cross-refs

- The mechanical invite-rotation op is worth capturing as a focused future feature; covered alongside the deeper rekey design in `new-features.md` (would slot in next to entries 5/6/7 — "leave room" / "delete message" / "edit message" — as the membership-churn family).
- The DM mailbox mechanism from `new-features.md` entry 14 is the natural carrier for "deliver a fresh invite to every remaining member without putting it in a public channel" — same Protomux side-channel, same identity-pinning.
- The identity layer entry (`new-features.md` entry 12) is a hard prerequisite — without identity you can't tell which writer key on the old base belongs to the refunded student, and you can't address the rekey invites to the right people.
- The Pear School Q in `basic-chat-identity/notes.md` sets out the actor model (instructor as trust anchor, registrar as enrollment authority) this answer assumes.
