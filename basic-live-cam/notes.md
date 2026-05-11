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
