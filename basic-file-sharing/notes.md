# basic-file-sharing — study notes

## Q: What is this app, in one sentence?

A Pear desktop app where each peer publishes one local folder as a shared **Hyperdrive**, and joining peers automatically mirror everyone else's drive into a local folder on disk — so dragging a file into one window makes it appear, as a real file, in every other peer's `shared-drives/<drive-key>/` directory.

## Q: What's the high-level process layout?

Two-process model with an HRPC channel between them, brokered by `pear-bridge`:

- **GUI process (Electron renderer)** — `index.html` loads `ui/root.jsx` via `build/root.js`. It uses `lib/use-worker.js` to open an HRPC connection over the Pear pipe.
- **Worker process (Bare)** — `index.js` boots `pear-bridge`, starts the Electron runtime, then `runWorker(pipe)` in `worker/index.js` owns the corestore, swarm, autobase, and drives.
- They talk over a `FramedStream` wrapping the bridge pipe (`worker/index.js:17`, `lib/use-worker.js:7-10`), with HRPC built from `spec/hrpc`.

The HRPC contract (from `schema.js:71-81`) is asymmetric:
- Worker → UI: `drives` (stream of the latest drive+files snapshot).
- UI → Worker: `add-file` (path + name of a file the user dropped).

## Q: How is on-disk storage organized?

Set up in `worker/worker-task.js:20-25`:

- `Pear.app.storage/corestore/` — the corestore where the autobase, view (HyperDB), and every Hyperdrive's hypercores live.
- `Pear.app.storage/corestore/my-drive/` — `LocalDrive` source. **Files the user drops live here as plain files.** Mirrored *into* the Hyperdrive periodically.
- `Pear.app.storage/corestore/shared-drives/<z32-drive-key>/` — one folder per remote peer's drive. **Hyperdrive content is mirrored *out* to here as plain files,** so the OS file manager and `file://` links work natively.

The `--store` flag in `pear run` sets `Pear.app.storage`, which is why the README scopes the path per example folder.

## Q: How does a file get from "I dropped it" to "it appears in every peer's folder"?

1. UI: `ui/root.jsx:18-23` resolves the dropped `File` to an absolute path via `Runtime.media.getPathForFile(file)` and calls `rpc.addFile({ name, uri })`.
2. Worker: `worker/worker-task.js:45-47` handles `onAddFile` by `fs.copyFile`-ing the source into `myDrivePath` (`my-drive/`). It's now a plain file on disk in the publisher's `my-drive/`.
3. Worker loop (`worker/drive-room.js:167-168`): every 1 s, a debounced `myLocalDrive.mirror(myDrive).done()` syncs `my-drive/` (LocalDrive) → `myDrive` (Hyperdrive in the corestore). New blocks get appended to the Hyperdrive's hypercore.
4. The Hyperdrive's `discoveryKey` is announced on the swarm (`drive-room.js:165`), so connected peers find it.
5. On every joiner: `_downloadSharedDrives` (`drive-room.js:143-160`) sees a new entry in the autobase view, creates `Hyperdrive(store, item.key)` + a `LocalDrive(shared-drives/<key>/)`, joins the swarm by that discovery key, and on every `append` runs `drive.mirror(local).done()` — Hyperdrive → LocalDrive — landing the bytes as real files in `shared-drives/<key>/`.
6. Worker polls (`worker-task.js:49-76`) every 1 s, reads `fs.readdir` of each `shared-drives/<key>/`, builds a snapshot with `file://` URIs, and pushes it to the UI via the `drives` HRPC stream.

So there are **two mirror directions per peer**:
- Publisher side: LocalDrive → Hyperdrive (push my files into the shared replica).
- Subscriber side: Hyperdrive → LocalDrive (pull others' files into a browseable folder).

## Q: What's the autobase actually storing? Isn't the file content in the Hyperdrives?

Yes — file content lives in each Hyperdrive's hypercore. The autobase is only the **room metadata index**: who is a writer, what invites exist, and **which drive keys belong to this room**. See `_setupRouter` (`drive-room.js:131-141`) and the dispatch ops registered in `schema.js:66-69`:

- `add-writer` — promotes a peer's local core into the autobase's writer set.
- `add-invite` — records a pairing invite (id, invite blob, publicKey, expiry).
- `add-drive` — appends a `{ key, info: { name } }` row so every peer learns about a new drive.

The view is a `HyperDB` bee (`drive-room.js:117`) with two collections from `schema.js:52-61`: `invites` (keyed by `id`) and `drives` (keyed by `key`). `getDrives()` (`drive-room.js:194-196`) just reads the `drives` collection back out.

## Q: How does pairing work between user1 and user2?

It uses `blind-pairing` over the same Hyperswarm. Two roles run on the same `BlindPairing` instance (`drive-room.js:27`):

- **Candidate (joiner)**: only runs when `localBase.length === 0 && this.invite` is set — i.e. fresh store with an invite (`drive-room.js:52-62`). `pairing.addCandidate({ invite, userData: localKey, onadd })` waits until the host confirms, then returns the `{ key, encryptionKey }` needed to open the autobase.
- **Member (host)**: always runs once the base is ready (`drive-room.js:87-100`). For each incoming pairing request, it looks up the matching `invites` row by `inviteId`, opens the request with the stored `publicKey`, calls `addWriter(request.userData)` to grant write access, and confirms with the autobase key/encryption key.

`getInvite()` (`drive-room.js:176-186`) is idempotent: returns the existing invite if there is one, otherwise generates one via `BlindPairing.createInvite(base.key)` and persists it through `add-invite`. That's why every peer (after pairing) can call it and get the *same* invite back — useful for re-sharing.

## Q: Why is there both `localBase` and `base`?

`Autobase.getLocalCore(store)` is called early (`drive-room.js:33`) to peek at whether this store has *ever* written to the room. If `localBase.length === 0`, this is a fresh peer and an `--invite` is mandatory; otherwise the room key is already encoded in the store namespace, so the second `new Autobase(store, key?, encryptionKey?)` can re-open without them (`drive-room.js:67-73`, note the comment block at 64-65). The `localBase.close()` at 66 releases the temporary handle before reopening it as part of the full base.

## Q: How is "my drive" distinguished from others in the UI?

`worker-task.js:59` normalizes both keys with `idEnc.normalize` and compares: `key === idEnc.normalize(this.room.myDrive.key)`. The snapshot pushed to the UI flags it with `info.isMyDrive`, which `ui/root.jsx:54` uses to append `"(My drive)"`. The polling also pins the user's own drive to the top via the sort in `worker-task.js:70-74`.

## Q: Why does the worker poll every second? Couldn't it be event-driven?

It is event-driven for the *autobase* side — the `update` listener triggers `_downloadSharedDrives` on every base mutation (`drive-room.js:103`). The 1 s `setInterval` in `worker-task.js:49` is only there to detect **filesystem-level changes** in each `shared-drives/<key>/` directory (i.e. fresh files that landed via the Hyperdrive→LocalDrive mirror). There's no fs-watcher in this example; the loop just re-reads `fs.readdir` and pushes the snapshot. See `new-features.md` for the obvious improvement.

## Q: What does the `--reset` flag actually wipe?

`worker/index.js:23-25` deletes `Pear.app.storage/corestore` before opening anything. That nukes the autobase writer key, the HyperDB view, all the Hyperdrive cores, *and* the `my-drive/` + `shared-drives/` directories (since they live under the same `corestore/` root, see `worker-task.js:24-25`). The peer effectively becomes a fresh joiner — it'll need a new `--invite` even to rejoin a room it created previously.

## Q: What's in `spec/`?

Generated artifacts from `node schema.js` (`npm run build:db`):

- `spec/schema/` — Hyperschema definitions (writer, invite, drive, drives, file).
- `spec/db/` — HyperDB collection wiring (`invites`, `drives`). Note: the collection IDs baked here are what causes `Unknown collection type: N` when a sibling folder reuses the same `--store` path — IDs are folder-local.
- `spec/dispatch/` — Hyperdispatch op codec for `add-writer` / `add-invite` / `add-drive` autobase ops (offset 0).
- `spec/hrpc/` — HRPC codec for the worker↔UI channel (`drives`, `add-file`).

`schema.js` is the *source of truth*; `spec/` is regenerated. `package.json:18` excludes `schema.js` from `pear stage` for the same reason — it's a build tool, not runtime code.

## Q: What gets shipped vs. ignored at `pear stage` time?

From `package.json:9-19`, staging ignores: `.git`, `.github`, `.gitignore`, `.swcrc`, `.swcrcdev`, `ui` (because `swc` builds it into `build/`), `input.css`, `README.md`, `schema.js`. So the stage contains `index.js`, `index.html`, `output.css`, `build/` (compiled JSX), `lib/`, `worker/`, `spec/`, and `package.json` — enough for `pear run <link>` to boot the worker and the GUI.
