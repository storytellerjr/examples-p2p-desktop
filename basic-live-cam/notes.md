# basic-live-cam notes

## Q: What does this example actually do, in plain English?

`basic-live-cam` is a small peer-to-peer **live broadcast** app — think of it as a stripped-down Twitch or Instagram Live.

- The first person to start the app becomes a **broadcaster**. The app turns on their webcam and microphone, and it prints out an **invite code**.
- They share that invite code with anyone they want to let in — over chat, email, whatever. There is no server, no account, no website hosting the stream.
- The second person starts the same app with that invite code. As soon as they connect, they see the broadcaster's camera feed playing live in their window.
- Underneath the video there is a **comments box**. Either side can type a message, hit Send, and it shows up for everyone in the room, with the sender's name and a timestamp.
- More people can join with the same invite and they'll all see the same video and the same shared comment thread.

So in spirit it's: **"share my camera with my friends, and let us all chat under it, with no middleman."** The "basic" in the name means it's the smallest possible working version — one broadcaster, one ongoing video stream, one comment feed, peer-to-peer over the Pear/Holepunch stack.

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
