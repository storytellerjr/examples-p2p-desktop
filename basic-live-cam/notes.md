# basic-live-cam notes

## Q: What does this example actually do, in plain English?

`basic-live-cam` is a small peer-to-peer **live broadcast** app — think of it as a stripped-down Twitch or Instagram Live.

- The first person to start the app becomes a **broadcaster**. The app turns on their webcam and microphone, and it prints out an **invite code**.
- They share that invite code with anyone they want to let in — over chat, email, whatever. There is no server, no account, no website hosting the stream.
- The second person starts the same app with that invite code. As soon as they connect, they see the broadcaster's camera feed playing live in their window.
- Underneath the video there is a **comments box**. Either side can type a message, hit Send, and it shows up for everyone in the room, with the sender's name and a timestamp.
- More people can join with the same invite and they'll all see the same video and the same shared comment thread.

So in spirit it's: **"share my camera with my friends, and let us all chat under it, with no middleman."** The "basic" in the name means it's the smallest possible working version — one broadcaster, one ongoing video stream, one comment feed, peer-to-peer over the Pear/Holepunch stack.

## Q: What Pear School use cases does this folder enable?

`basic-live-cam` is the foundation for any Pear School experience that needs **one person broadcasting live to many, with a synchronous text backchannel, and no server in the middle**. Several concrete classroom shapes fall out of it (most are stronger once features 1–3 in `new-features.md` — audio, device pickers, screen-share — are in place):

- **Live lectures.** Instructor broadcasts face + voice + slides (or whatever's on screen) to a cohort. Each student joins with the same invite; the comments box becomes the live Q&A backchannel without disrupting the lecturer's audio. No streaming-server bill — peers carry the load.
- **Office hours / 1:1 tutoring.** Tutor and student in a two-peer room. The persistent comment thread *is* the session notes — the student keeps the corestore around and the conversation stays browseable as plain text, anchored to the same time as the recording.
- **Code-along / pair-programming sessions.** Once screen-share (feature 3) lands, an instructor shares an IDE while talking; students follow along and ask questions in the comments without interrupting. Same room model as lectures, just a different capture source.
- **Student presentations + structured feedback.** A student becomes broadcaster, classmates join, and feedback flows into the comment thread tagged with each peer's `--name`. The instructor can later read the thread as a record of who-said-what and grade or summarise it.
- **Cohort standups / study groups.** Students self-host a daily room — no admin spinning up a Zoom link, no account creation barrier. Anyone with the invite joins.
- **Guest-speaker events.** Instructor generates an invite and forwards it to an external speaker; the speaker becomes broadcaster for the day. No platform login, no calendar dance.
- **"Show your work" sessions.** Student points a phone or webcam at their paper notes, a physical experiment, or a hardware build. Same pipeline, no special tooling.

Two architectural facts make Pear School use of this app structurally interesting compared to off-the-shelf Zoom / Twitch:

- **The session is persistent, not ephemeral.** Every video fragment and every comment is appended to an autobase view (`worker/live-cam-room.js:138-143, 268-272`). A student who missed the class can join the room later and the playback loop (`ui/root.jsx:30-49`) walks the fragments from `fragIdx=0` — so they get the lecture *and* the chat in correct order, without anyone having to "publish a VOD" or "share the recording link". The recording, the chat, and the live experience are the same object.
- **Joiners don't need accounts, only an invite.** The invite is a `blind-pairing` token (`worker/live-cam-room.js:151-161`); pairing happens on the swarm. For a school product, that means a student onboards with a single link and zero identity setup, and an instructor controls access by who they shared the invite with.

Limitations to flag honestly before pitching this for Pear School lessons:

- No audio yet — see feature 1 in `new-features.md`. A lecture-style use case is not viable until audio ships.
- The broadcaster is fixed at room creation (`if (!this.invite) this._startLiveCam()` at `worker/live-cam-room.js:103`). Roles can't rotate — useful for "instructor broadcasts" but not for "student takes over for their presentation" without restarting the app.
- No moderation surface on comments (no delete, no rate-limit, no ban) — fine for trusted cohorts, not yet defensible against bad actors.
- The 2-second-GOP latency floor (`worker/live-cam-room.js:217`) makes anything that needs back-and-forth interactivity (e.g. a Socratic-style seminar where the instructor reads the room) feel sluggish. Feature 4 (quality presets) helps but doesn't eliminate this.

## Q: Where are the video fragments stored, what happens when the broadcaster drops, and is there blind mirroring of the whole stream?

There are **two storage layers** in this app — once you separate them, the broadcaster-drop and mirroring questions answer themselves.

### Layer 1 — autobase view (HyperDB): metadata only

- One per peer, in `Pear.app.storage/corestore/` (set by `--store`).
- Holds rows in four collections (`@basic-live-cam/invites`, `/videos`, `/messages`, `/add-writer` — see `spec/db`).
- A `/videos` row is just a **pointer**: `{ id, blob: { key, blockOffset, blockLength, byteOffset, byteLength }, info: { fragIdx } }`. The actual video bytes are **not** in this row. See `worker/live-cam-room.js:266-272`.
- This view core is **eagerly fully downloaded** on every peer: `this.view.core.download({ start: 0, end: -1 })` at `worker/live-cam-room.js:83`. So every joiner pulls every metadata row, indefinitely.

### Layer 2 — Hyperblobs core: the actual video bytes

- A **single, separate hypercore** named `'blobs'`, owned by the broadcaster's corestore: `this.blobs = new Hyperblobs(this.store.get({ name: 'blobs' }))` at `worker/live-cam-room.js:35`.
- Every MP4 fragment FFmpeg produces is appended as a new blob inside this **one** core (`_onNewFragment` at `worker/live-cam-room.js:260-273`). All fragments share the same `blob.key` — only `byteOffset`/`byteLength` differ between rows.
- A `hypercore-blob-server` runs in-process on each peer (`worker/live-cam-room.js:36, 101`). It serves byte-ranges out of the local corestore over HTTP so the React UI can `fetch(link)` (`ui/root.jsx:42-46`).
- Joiners learn the blob core's key from the metadata row, open the remote core via `this.store.get({ key })`, and **join the swarm on the blob core's discovery key** the first time they see any video row (`worker/live-cam-room.js:182-188`). Subsequent rows hit the cache at `this.blobsCores[item.blob.key]`, so this happens once per joiner, not per fragment.

### What this means for "blind mirroring of the whole video"

Almost, but not eagerly:

- **Metadata** (the `/videos` rows) is force-downloaded on every peer (`worker/live-cam-room.js:83`). So every peer knows about every fragment that ever existed.
- **Bytes** are pulled **on demand** by the playback loop at `ui/root.jsx:36-49`. There is **no** explicit `blobsCore.download({ start: 0, end: -1 })` in this codebase, so hypercore's default sparse replication applies — a peer only fetches the blocks the blob server asks for.
- The playback loop walks fragments in order from `fragIdx=0`, so **in practice anyone who watches the broadcast end-to-end ends up with the full byte set in their local corestore**, and once it's there it stays — hypercore is append-only and the corestore doesn't garbage-collect.
- A peer who joins but never opens the room (or scrubs away before fragment N) will not have the bytes for fragment N+1, even though they have the metadata for it.

So "blind mirroring" is **effectively yes if you watch, no if you don't**. There's no eager pre-fetch of the whole stream.

### What happens after the broadcaster drops

The broadcaster going offline does **not** kill the room. Concretely:

- **Comments and metadata up to the drop persist on every joiner.** They were already replicated into each joiner's local view core (eager download, see above), so they remain visible and the comment thread still renders.
- **Video bytes up to the drop persist on whichever peers downloaded them.** A joiner who watched fragments 0–N has those bytes in their local corestore. They will continue to **serve** those bytes to any other peer connected to them on the swarm — the autobase view core and the blob core are both being announced from every peer that has them, not only from the broadcaster.
- **A late joiner can still watch a recording of the dropped broadcast**, *provided at least one peer who has the bytes is online*. The late joiner gets the metadata from the swarm (any peer can serve it), opens the blob core, requests the bytes, and the playback loop walks `fragIdx=0` upward. From their perspective the broadcaster's offline status is invisible — bytes flow from whoever has them.
- **New comments still work.** Any peer is a writer on the autobase once `addWriter` has run for them (`worker/live-cam-room.js:91-97` — `pairMember.onadd` calls `addWriter` for every joiner). The HRPC `addMessage` flow at `worker/worker-task.js:35-37` appends locally and replicates to anyone connected, regardless of whether the broadcaster is online.
- **What stops** is new video. The broadcaster is the only source of `_onNewFragment` events because of the `if (!this.invite) this._startLiveCam()` guard at `worker/live-cam-room.js:103`. No broadcaster → no new fragments → no growth in the blob core. Viewers reach the last `fragIdx` and the playback loop sits at `await new Promise(resolve => setTimeout(resolve, 100))` (`ui/root.jsx:38-40`) waiting for fragments that never arrive.
- **What stops absolutely** is when the *last* peer holding the bytes goes offline. There is no persistent seeder, no DHT-pinned mirror — durability is "whoever happens to have a corestore open". For Pear School use cases that need replay-after-the-fact, this is a real gap (worth a future-feature entry: a "always-on seeder" peer, or pinning to a Hyperbee/Hyperdrive durability service).

### One-line summary

- Metadata: one autobase view core per peer, force-downloaded, holds pointers to bytes.
- Bytes: one Hyperblobs hypercore on the broadcaster, replicated sparsely on-demand to viewers as they play.
- Broadcaster drops → existing content remains playable as long as at least one peer with the bytes is online; only *new* fragments stop.

## Q: How can I test this on one computer?

Yes — it's designed to run both peers on one machine. Only **user1** (the one started *without* `--invite`) actually opens the webcam — see the `if (!this.invite) this._startLiveCam()` guard at `worker/live-cam-room.js:103`. user2 (and any further joiners) just receive the encoded video fragments via blob replication and replay them, so there's no camera-contention between the two windows.

Steps:

1. **Install ffmpeg**, and on macOS grant Camera + Microphone permission to your terminal / Pear runtime the first time it asks. The React side requests the permissions at `ui/root.jsx:14-21`.
2. Open **two terminal windows/tabs**, both `cd`'d into `basic-live-cam`.
3. In the first terminal, run the broadcaster:
   - `npm i`
   - `npm run build`
   - `pear run --store /tmp/live-cam-user1 . --name user1`
   - The app window opens and prints an **invite** at the top of its UI.
4. **Copy that invite string** out of user1's window.
5. In the second terminal, paste it into the join command:
   - `pear run --store /tmp/live-cam-user2 . --name user2 --invite <paste-invite-here>`
6. Arrange the two app windows side-by-side. You should see:
   - user1's window playing its own camera feed.
   - user2's window playing the same feed, with a few seconds of catch-up delay (it has to receive enough MP4 fragments before the browser's `MediaSource` will start playback — `ui/root.jsx:30-49`).
   - A **comments box** under each video. Type into either window → it appears in both, tagged with the sender's `--name` and a timestamp.

Tips for one-machine testing:

- The folder-scoped store paths (`/tmp/live-cam-user1`, `/tmp/live-cam-user2`) are what make running both peers on the same disk safe — each one has its own isolated corestore. See the store-path Q below for the full rationale.
- If something gets wedged, kill both processes, add `--reset` to one or both commands, and re-run. **Caveat:** `--reset` on user1 wipes the autobase, which invalidates the old invite — you'll need to reissue the new invite to user2.
- You can add a third peer the same way (`/tmp/live-cam-user3`, same invite) to see fan-out: one broadcaster, multiple viewers, one shared comment thread.
- The broadcaster is fixed for the lifetime of the room — there's no "promote viewer to broadcaster" flow. If user1 quits, the video stream stops; user2's video element will simply stop receiving new fragments.

## Q: Why did we change the `--store` paths in the README from `/tmp/user1`/`/tmp/user2` to `/tmp/live-cam-user1`/`/tmp/live-cam-user2`?

The upstream pearopen README defaults to bare `/tmp/user1`, `/tmp/user2` for both `pear run` invocations (`README.md:14-19` after edit, originally lines 15/18 in the Usage block and 30/33 in the Build Pear app block). Every other example folder in this study repo did the same. That collides.

- Each example folder is its own Pear app with its **own schema namespace, hyperdb collection IDs, and hyperdispatch offsets** — see `schema.js` and `spec/` in this folder. Those numbers are different per app.
- That schema metadata is **baked into every autobase block** the app writes. The block on disk only makes sense to the app that wrote it.
- If you ran `basic-chat` (or any other example) earlier with `--store /tmp/user1`, then run `basic-live-cam` with the same `--store /tmp/user1`, this app opens the corestore, finds blocks written under a different schema, and crashes with:
  - `Uncaught Error: Unknown collection type: N` — HyperDB cannot find collection ID `N` in *this* folder's `spec/db`, because it was registered by the previous folder's app.
- Folder-scoped paths (`/tmp/live-cam-user1`, `/tmp/live-cam-user2`) keep each example's on-disk data isolated, so hopping between folders doesn't require `pear run ... --reset` every time.

The four call sites updated in `README.md`:

- **Usage / dev run**: `pear run --store /tmp/live-cam-user1 . --name user1`
- **Usage / join**: `pear run --store /tmp/live-cam-user2 . --name user2 --invite <invite>`
- **Build Pear app / staged run**: `pear run --store /tmp/live-cam-user1 <pear-link> --name user1`
- **Build Pear app / staged join**: `pear run --store /tmp/live-cam-user2 <pear-link> --name user2 --invite <invite>`
- **Troubleshoot / `--reset` example**: `pear run --store /tmp/live-cam-user1 . --name user1 --reset`

Also added an explanatory note above the first `pear run` block (`README.md:9`) modelled on `basic-chat-identity/README.md:9`, so a future reader hitting `Unknown collection type: N` understands the failure mode without having to dig.
