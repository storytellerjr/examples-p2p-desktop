# basic-chat-blind-peering — Study Notes

Notes for Storyteller while studying this example.

---

## Q: What is "blind peering"? How does this project differ from `basic-chat`?

### Code delta vs `basic-chat`

The two folders are the same app except for the blind-peering layer. Aside from a global rename of the namespace `basic-chat` → `basic-chat-blind-peering` (in `schema.js`, `spec/**`, dispatch/HRPC names, etc.), the *real* additions are:

1. **New dependency** — `blind-peering` (^1.15.0) in `package.json`.
2. **New CLI flag** — `worker/index.js:11` adds `--blind-peer-key|-b` (multi-value).
3. **A `BlindPeering` instance in the worker** — `worker/worker-task.js:24-26, 37, 46`:
   ```js
   this.blindPeering = new BlindPeering(this.swarm, this.store.namespace('blind-peering'), {
     mirrors: this.blindPeerKeys
   })
   …
   await this.blindPeering.addAutobase(this.room.base)
   …
   await this.blindPeering.close()
   ```

The README also documents running an external mirror first:
```
npm i -g blind-peer-cli@latest
npx blind-peer -s /tmp/blind1            # prints a listening-key
pear run … --blind-peer-key <listening-key>
```

The chat protocol, autobase, hyperdb collections, RPC — all unchanged.

### What "blind peering" actually means

A **blind peer** is a third-party peer that **stores and replicates Hypercores/Autobases on your behalf, without being able to read the contents** and without being a writer.

- In plain `basic-chat`, peers only have data while they're online. If user1 posts and goes offline, user2 must have been connected then (or connect to user1 later) to pull it.
- In `basic-chat-blind-peering`, both clients pass the same blind-peer's listening key as a `mirror`. When `addAutobase(this.room.base)` runs, the blind peer is asked to replicate and persist the autobase blocks. It joins the swarm topic and pulls encrypted hypercore blocks like any other peer — but never gets the keys to decrypt records.
- Result: an **always-on, untrusted, encrypted mirror**. user1 can post, go offline, user2 syncs from the blind peer later. Neither has to be online simultaneously, and the blind-peer operator can't read the chat.

The "blind" part is the key property: the mirror has the **bytes** but not the **meaning**. Holepunch's equivalent of a trustless backup/relay node — availability and offline-tolerance without a server that can snoop.

---

## Q: What happens when many users (user1, user2, …) join the swarm? Is the blind peer storing all blocks of all these new users?

Yes — and the code makes it concrete.

### What the blind peer ends up storing

Every user does:
```js
await this.blindPeering.addAutobase(this.room.base)   // worker-task.js:37
```
…against the **same** blind-peer listening key passed via `--blind-peer-key`.

`addAutobase` tells the blind peer: "join the swarm for this autobase and replicate all of its cores." An Autobase is not a single hypercore — it's a bundle:

- **One writer hypercore per writer** (user1, user2, user3, …). Each is created locally via `Autobase.getLocalCore(this.store)` (`chat-room.js:25`) and registered into the autobase via the `add-writer` dispatch (`chat-room.js:111`).
- **The view core** — the linearized HyperDB view (`chat-room.js:96`) holding merged messages/invites.

So when N users join the room:
- The autobase has **N writer cores + 1 view core**.
- Each client independently asks the same blind peer to mirror that same autobase. Requests **dedupe** on the blind-peer side (same discovery key → one mirror), so the blind peer ends up with one copy of all N+1 cores for the room.
- As writers append messages, the blind peer pulls and persists those new blocks.

### How storage scales

- **Per room:** grows with `(writers) × (their append history) + view`. All encrypted — the blind peer can't decrypt; it holds bytes.
- **Across rooms:** in this app every client points at the *same* blind peer for *every* autobase they open, so that one blind peer accumulates every room's cores from every user that names it as a mirror. No per-room separation on the blind peer — it just mirrors whatever autobases peers point at it.
- **`store.namespace('blind-peering')`** (`worker-task.js:24`) is the *client-side* corestore namespace where the blind-peering library keeps its own bookkeeping; it does not partition the blind peer's storage.

### Practical implications

- A blind peer is an **always-on storage node** offering bytes-but-not-meaning. More users/rooms pointing at it = more disk used.
- `npx blind-peer -s /tmp/blind1` is a wide-open mirror: anything that asks gets mirrored. In production you'd run your own, or use one with quotas/auth (the `blind-peering` library and `blind-peer-cli` support gossip/limits — but this example doesn't configure any).
- Failure mode: if the blind peer is overloaded or evicts an autobase, clients fall back to direct peer-to-peer replication when both happen to be online — same as `basic-chat`. The blind peer is a *convenience for availability*, not a source of truth.

---

## Q: How do I test that the blind peer is actually working? (two users running, both with `--blind-peer-key`)

The decisive test is **offline message delivery** — that's the whole point of the blind peer.

### Test 1 — Offline delivery (the main feature) ✅ verified working

With blind peer + user1 + user2 running and chatting:

1. Kill user2 (Ctrl-C). user2 is offline.
2. From user1, send `"sent while user2 offline"`.
3. Kill user1 too. Both clients offline; only the blind peer holds that message.
4. Restart user2 with the same command (same `--store /tmp/user2`, same `--blind-peer-key`, same `--invite`).
5. The new message should appear in user2's UI within seconds.

**Result (Storyteller, 2026-05-07):** ran successfully. user2 received the message that was sent while it was offline, with user1 also offline at restart time — so the message can only have come from the blind peer. Confirms the encrypted offline-mirroring path works end-to-end.

Pass → blind peer pulled user1's append while user1 was online and served it to user2 on reconnect.
Fail → check both clients used the same `--blind-peer-key` and that the blind peer is still running.

> Don't use `--reset` for this test — it wipes the corestore, including user2's own writer core, so you'd lose your seat in the room. Local storage must stay intact; only the network availability is what's being tested.

### Test 2 — Storage growth on the blind peer

```shell
du -sh /tmp/blind1     # before
# … send some messages …
du -sh /tmp/blind1     # after
ls /tmp/blind1         # see corestore-style cores/ layout
```
Bytes grow as encrypted hypercore blocks are persisted. Contents are encrypted; chat text is not visible.

### Test 3 — Blind peer logs

`npx blind-peer -s /tmp/blind1` logs to stdout. Connections + replication activity should appear when clients pair and chat. Silence after pairing = the `--blind-peer-key` flag did not take effect on the clients.

### Test 4 — Late joiner (stronger evidence)

Proves the **split** between two phases:

| Phase | Needs writer online? | Blind peer enough? |
|---|---|---|
| Pairing (joining a new member) | ✅ yes | ❌ no |
| Replication (syncing messages) | ❌ no | ✅ yes |

Why pairing needs a live writer: in `chat-room.js:37-47` the joining side does `this.pairing.addCandidate({ invite, userData, onadd: resolve })` and **awaits** the resolve. That resolve only fires when an existing writer (running `pairing.addMember` at `chat-room.js:72-85`) receives the request and calls `request.confirm({ key, encryptionKey })`. The blind peer mirrors encrypted blocks but does not know your invite or hold autobase keys — so it cannot complete the pairing handshake.

**Correct sequence:**

1. Blind peer + user1 + user2 running; chat for a bit.
2. Kill user2.
3. Start user3 with `--invite <invite>` and `--blind-peer-key <key>`.
   user3 pairs with **user1** (the live writer). Success signal: `Invite: …` printed in user3's terminal (line at `worker/index.js:35`, only runs after `workerTask.ready()` resolves).
4. Once user3 has the chat history, kill user1.
5. Both original writers are now offline. To prove blind-peer-only replication, restart user1, send a message, kill user1 again — user3 should still receive it via the blind peer.

**Failure mode observed (Storyteller, 2026-05-07):** ran user3 with the invite while *both* user1 and user2 were offline. Result: user3's UI showed an empty screen with no messages and no error. Diagnosis: `addCandidate` hangs forever waiting for a writer to confirm; `room.ready()` never resolves; the worker never sends any messages over RPC, so the UI stays in its default empty state. **No `Invite:` line in user3's terminal** is the giveaway.

**Recovery observed (Storyteller, 2026-05-07) ✅:** brought user1 back online while user3 was still hanging in step 3. user1's `pairing.addMember` handler (`chat-room.js:72-85`) confirmed user3, appended an `add-writer` op for user3's local key (`chat-room.js:111-113`), and user3's pending `addCandidate` Promise resolved. user3 then constructed its `Autobase` with the received key/encryptionKey, joined the swarm topic, and **all the chat history appeared**. Confirms the two-phase split: pairing requires a live writer, replication does not.
