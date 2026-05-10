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

## Q: How would we use this folder in Pear School?

Background on the actor model (instructor as trust anchor, registrar as enrollment authority, refund → re-key) lives in `basic-chat-identity/notes.md` and the Pear School Q in `basic-chat-multi-rooms/notes.md`. This section is specifically about **what role file-sharing plays** in Pear School and what would need to change vs. the example as it stands today.

### The five concrete use cases

1. **Curriculum drive (instructor → cohort).** The instructor publishes a single drive containing the syllabus, slide decks, PDFs, problem sets, and lecture recordings. Every enrolled student mirrors it locally, so course materials are available offline (train rides, planes, flaky wifi). Updates propagate as soon as the student is online — `_downloadSharedDrives` (`worker/drive-room.js:143-160`) already handles "new drive appears, start mirroring" and Hyperdrive's append events handle "new file landed, mirror again."

2. **Submission drive (student → instructor).** Each student publishes their own drive into the room; the instructor mirrors all of them. This is **exactly what the example already does** for every peer — except in a course you don't want every *student* mirroring every *other student's* submissions. See "What needs to change" below.

3. **Per-student feedback drive (instructor → one student).** After grading, the instructor drops the marked-up PDF or feedback notes into a private drive that only that student can decrypt. Different encryption posture from the cohort-wide curriculum drive.

4. **Recordings drop (instructor / TA → cohort).** Live-session recordings auto-published into the curriculum drive (or a sibling `recordings/` drive) at the end of each session. Same dynamics as case 1, but bursty and large.

5. **Group project drives (subset of students ↔ each other).** Teams of 3–5 share a working folder for a group project. Same many-to-many model as the example, but scoped to the team subset, not the whole cohort.

### What needs to change vs. the example as it stands

The example assumes **a small symmetric cohort where everyone publishes and everyone mirrors everyone**. Pear School breaks that on three axes — scale, role asymmetry, and revocation.

**Scale: don't mirror full content by default.** With 50 students × ~5 GB each, a naïve port mirrors 250 GB onto every laptop. The current `_downloadSharedDrives` (`worker/drive-room.js:143-160`) calls `drive.mirror(local)` immediately on discovering a drive, which downloads everything. For Pear School, **separate metadata from content**: list each drive's file tree (already streamed via the `drives` HRPC method, `worker/worker-task.js:75`) but only fetch a file's bytes when the user clicks it. Hyperdrive supports sparse downloads natively — `drive.download(path)` per file instead of `drive.mirror(local)` for the whole drive. The `shared-drives/<key>/` LocalDrive folder stops being a 1:1 mirror and becomes a cache.

**Role asymmetry: not every peer should be a writer or a publisher.**

- The example calls `addWriter(request.userData)` for *every* successful pairing (`drive-room.js:94`) and `addDrive(this.myDrive.key, ...)` for *every* peer's local drive (`drive-room.js:164`). For Pear School, role gates the writes:
  - **Curriculum drive:** instructor is the only `add-drive` author; students are read-only (they pair, but they're not promoted to writers on the curriculum autobase, or there's a separate single-writer hypercore for it).
  - **Submission drive:** each student publishes, but the dispatcher rejects `add-drive` ops where the writer isn't the student themselves submitting their own drive — prevents impersonation in the metadata layer.
- This needs an identity check at the `_setupRouter` dispatch level (`drive-room.js:131-141`). Today the router blindly inserts whatever the autobase appended; tomorrow it inspects the autobase block's writer key and the op payload, and rejects mismatches. Pear School identity (signed `identityPub`) becomes the gate.

**Revocation: refunded student keeps a copy of everything.** Same fundamental problem the multi-rooms refund Q calls out — Hyperdrive content the student already replicated is on their disk forever, and the encryption key is the room key. To stop *future* leaks after a refund:

- Re-key the **curriculum drive** (publish a new drive, abandon the old one's discoveryKey) — same mechanic as re-keying a room, propagated to every student except the refunded one.
- The submission drives are per-student so revocation is trivial there: just stop mirroring the refunded student's drive (or evict its key from the `drives` collection and skip it in `_downloadSharedDrives`).

### Per-use-case mapping to the existing code

- **Use case 1 (curriculum):** one drive, instructor as sole writer on its hypercore, encryption key shared with the cohort autobase. Replace `addDrive(this.myDrive.key, ...)` (`drive-room.js:164`) with role-gated logic: only the instructor's worker calls it; students just `new Hyperdrive(store, instructorKey)` after pairing and join the swarm by its discoveryKey.
- **Use case 2 (submissions):** keep the existing per-peer publish flow but flip the mirror direction — only the instructor's worker runs `_downloadSharedDrives` for student drives; students skip it. UI for instructor lists "Alice's submission, Bob's submission, …"; UI for student shows only their own drive plus the curriculum drive.
- **Use case 3 (per-student feedback):** new collection in the autobase view, `feedback-drives` keyed by `studentIdentityPub`. The drive is encrypted with a per-student-derived key (e.g. ECDH between instructor identity and student identity). Mirror logic only runs for the entry whose key matches your own identity. Today the schema's `drive` shape (`schema.js:28-34`) is just `{ key, info }` — extend `info` to include the audience identity-pub.
- **Use case 4 (recordings):** mechanically identical to case 1 — either append into the curriculum drive (one growing tree) or publish a sibling `recordings` drive (separate hypercore, separate discoveryKey, easier to gate behind a "recordings included" enrollment tier).
- **Use case 5 (group projects):** spawn a sub-room with its own autobase + its own DriveRoom — same pattern multi-rooms uses for chat rooms inside an account. Each team is essentially a small instance of the current example, scoped under the parent course.

### What this example gives Pear School for free

Even before any of those changes, the example already nails the pieces that are tedious to get right:

- **Hyperdrive ↔ LocalDrive bidirectional sync** — `drive.mirror(local)` handles both publish and subscribe directions (`drive-room.js:154`, `:167`). The decision "files on disk in `shared-drives/<key>/`" rather than blob storage in a database is exactly right for course materials — students can open PDFs in their normal viewers, drag recordings into VLC, etc.
- **Pairing via blind-pairing** (`drive-room.js:53-100`) — the enrollment flow ("instructor sends invite, student joins") is already there; Pear School just adds an identity-binding step on top (the registrar countersigns the join).
- **Append-driven mirror** — `drive.core.on('append', () => mirror())` (`drive-room.js:155`) means new course materials show up in students' folders within seconds of upload, no polling needed at the byte level. (The 1 s `setInterval` in `worker-task.js:49` is only for the *file-listing snapshot* sent to the UI; the actual byte sync is event-driven.)
- **`file://` URIs in the UI snapshot** (`worker-task.js:65-66`) — clickable links open files in the OS default app. Free desktop integration for course materials.

### What to look at next

Before building Pear School's file layer, the prerequisites are:

- Identity binding for drives (extend `schema.js`'s `drive` to carry a signed `identityPub` + role claim).
- Sparse / on-demand download (replace blanket `drive.mirror(local)` with per-file `drive.download(path)`).
- A role-aware dispatcher (`_setupRouter` checks the writer's identity claim before applying ops).

Each of those is a self-contained change to *this* example and worth prototyping here before porting into the actual Pear School app.

## Q: How would we add multiple drives or categories of drives (video recordings, curriculum, module videos, examples, templates) to this folder, all clearly separated?

Short answer: each category becomes its own **drive**, not a sub-folder, and the room's autobase view learns to tag each drive with a `category` field so peers can route, authorise, and group them independently. Folders inside a single drive don't work because Hyperdrive's three boundaries — **authorisation, replication scope, and ownership** — are all at the drive level, not the path level.

The thinking-level pieces:

- **Why categories aren't directories.** Hyperdrive has no per-path ACLs and is single-writer. "Curriculum readable by all enrolled students" and "submissions readable only by the instructor" need different keys, which means different drives. Same for ownership (instructor publishes curriculum; each student publishes their own submission) and for replication scope (you want to skip whole categories on bandwidth-constrained peers, which the swarm only lets you do per-discoveryKey).
- **Where the type lives.** A `category` field on the existing `drive` record (`schema.js:28-34`), plus a secondary `drives-by-category` index so the view can prefix-scan rather than full-table-filter once a course has 50 students × 5 categories of drives.
- **Where authorisation lives.** The dispatcher (`worker/drive-room.js:131-141`) — currently a blind `view.insert`. It becomes a policy gate keyed on `(category, writerKey, identityClaim)`: only instructor publishes `curriculum`; only the student themselves publishes their own `submissions`; TAs publish `templates`; etc. Rejected ops are silently dropped by the view.
- **Where the storage layout lives.** `shared-drives/<category>/<key>/` instead of flat `shared-drives/<key>/`. The single `myDrive` (`drive-room.js:37-38`) generalises into a `Map<category, drive>` because one peer may publish into several categories (a student's `submissions` and the `templates` they're sharing back with classmates).
- **Where the UI lives.** `ui/root.jsx`'s flat `<ul>` becomes category sections in a fixed display order, and the drop zone gets a "publish into…" picker that disables categories the user isn't authorised for.
- **Where discovery scope lives.** Optional: per-category swarm topics if connection counts blow up. Default: keep current per-drive discovery and gate the *mirror* on category, not the connection.

Concrete code sketch — schema diff, dispatcher rewrite, filesystem path changes, UI restructure, discovery trade-offs, and a migration note for existing rooms — is in `new-features.md` entry **2. Categorise drives**. That entry also includes a "why not folders inside one drive" section that walks through the three Hyperdrive boundaries above with code references.

## Q: I dragged a folder with sub-folders into the drop zone and nothing happened. Are folders just not supported?

Correct — only individual files. This isn't a bug in any one layer; it's a flat-file assumption that runs through the whole pipeline. The good news: **the read/sync side is already directory-aware**, so only the *write* side needs fixing.

### What goes wrong, layer by layer

The path of a dragged item is: HTML5 drag event → React handler → HRPC call → worker `copyFile` → LocalDrive → Hyperdrive → peers. Folders break at the very first step and the assumption gets re-baked at every step after.

- **HTML5 drag event (UI input).** `e.dataTransfer.files` (`ui/root.jsx:11`) is a flat `FileList` — it does *not* recurse into directories. When you drag a folder, browsers populate `dataTransfer.files` with either nothing or a zero-byte phantom entry for the folder itself; the actual contents are only reachable via `dataTransfer.items[].webkitGetAsEntry()`, which exposes a `FileSystemDirectoryEntry` you have to walk yourself. The `<input type="file" multiple>` on `ui/root.jsx:42-48` has the same restriction — to allow selecting a folder, the input would need the `webkitdirectory` attribute, which switches the OS picker into "select directory" mode.
- **The file → URI step.** `Runtime.media.getPathForFile(file)` (`ui/root.jsx:20, 27`) resolves a `File` to an absolute path on disk. For the phantom folder entry, depending on browser/OS, it returns either an empty string, the folder path, or undefined behaviour. There's no contract for "here's a tree."
- **HRPC contract.** `add-file` takes a single `{ name, uri, info }` shape (`schema.js:40-47, 78-81`). `name` is a leaf filename, `uri` is a single-file path. There's no field for "this is a tree, here's the relative path within it" — the schema literally cannot express subdirectories.
- **Worker `add-file` handler.** `worker/worker-task.js:45-47` calls `fs.copyFile(data.uri, path.join(this.myDrivePath, data.name))`. `fs.copyFile` errors with `EISDIR` if the source is a directory, and even if it didn't, `path.join(myDrivePath, data.name)` would flatten any path separators inside `data.name` because `data.name` is treated as a leaf. So even if the UI *did* send `"reports/2024/q1.pdf"`, the worker would `mkdir`-fail or write `reports%2F2024%2Fq1.pdf` to the drive root — neither is what the user wants.

### What's already fine (the read side)

Once a file is in `my-drive/<some>/<nested>/<path>.pdf` by any means, the rest of the pipeline handles it correctly:

- **LocalDrive → Hyperdrive mirror.** `myLocalDrive.mirror(myDrive)` (`worker/drive-room.js:167`) walks the local directory recursively and pushes every leaf into the Hyperdrive at its full relative path. Hyperdrive paths are slash-delimited strings — no flat-file constraint at the storage layer.
- **Hyperdrive → LocalDrive mirror on receivers.** Symmetric — `drive.mirror(local)` (`worker/drive-room.js:154`) recreates the tree under `shared-drives/<key>/`.
- **Snapshot loop for the UI.** `fs.readdir(dir, { recursive: true })` (`worker/worker-task.js:55`) already returns nested paths like `'subdir/file.pdf'`, and `worker-task.js:66` builds a `file://` URI by joining `dir` with that name. Nested files render correctly in the UI's `<a href={file.uri}>` link list (`ui/root.jsx:56-60`) — they'd just appear as `subdir/file.pdf` in the displayed name.

So the moment you can land bytes on disk under `my-drive/` with a relative path, **the entire downstream pipeline already works**. This is a UI/HRPC/worker-input problem, not a Hyperdrive problem.

### What it would take to fix

Four coordinated changes. None individually large, but they have to land together because the contract changes shape.

1. **UI: walk the dropped tree.** Replace the `e.dataTransfer.files` loop with one that iterates `e.dataTransfer.items[]`, calls `item.webkitGetAsEntry()`, and recurses into directory entries. For each leaf `File`, compute its relative path from the drop root (e.g. `"week-1/exercises/q1.pdf"`). For the file picker, add `webkitdirectory` (or a separate "Select folder" button — having both modes is friendlier).
2. **HRPC: carry the relative path.** Either add a `path` field to the `@basic-file-sharing/file` record (`schema.js:40-47`) or just reuse `name` and document that it may contain `/`. The latter is simpler — Hyperdrive already treats paths as strings — but make sure the worker side doesn't `path.basename` it away.
3. **Worker: `mkdir -p` before copy.** In `worker/worker-task.js:45-47`, replace the single `copyFile` with:
   ```js
   const dest = path.join(this.myDrivePath, data.name)  // data.name may include '/'
   await fs.promises.mkdir(path.dirname(dest), { recursive: true })
   await fs.promises.copyFile(data.uri, dest)
   ```
4. **Or, batch at the worker.** Alternative to fixing it leaf-by-leaf in the UI: add an `add-folder` HRPC method that takes a single root URI, and have the worker do the walk locally with `fs.cp(src, dest, { recursive: true })`. Less network chatter, but loses the per-file progress hook the UI might want.

### Why this matters for Pear School

Every category in entry 2 of `new-features.md` (curriculum, module-videos, examples, templates) is naturally a tree, not a flat file list:

- **Curriculum** — `week-1/lecture.pdf`, `week-1/exercises/`, `week-2/...`.
- **Module videos** — `module-3/lesson-2.mp4` plus chapter markers and transcripts.
- **Examples** — project scaffolds with `src/`, `tests/`, `README.md`.
- **Templates** — same shape as examples.

Without folder uploads, an instructor would have to upload each file one at a time and then manually drag them into subdirectories on disk *after* the upload, which round-trips them through the LocalDrive→Hyperdrive mirror twice and is hostile to anyone publishing more than ten files. Folder upload is effectively a prerequisite for the categorisation work in `new-features.md` entry 2 to feel usable. Worth flagging there too.
