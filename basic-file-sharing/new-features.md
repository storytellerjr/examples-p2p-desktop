# basic-file-sharing — possible future improvements

This file collects forward-looking ideas surfaced while studying this folder. It is intentionally separate from `notes.md` (Q&A study notes) so the two don't get tangled. Each entry is anchored to code with `path:line` and explains *what*, *why*, and *rough how*.

## 1. Delete the source file after upload (move-instead-of-copy semantics)

**What.** When the user drops a file into the GUI, the worker handles `add-file` by `fs.copyFile`-ing the source path into `my-drive/` and never touches the original (`worker/worker-task.js:45-47`):

```js
this.rpc.onAddFile(async (data) => {
  await fs.promises.copyFile(data.uri, path.join(this.myDrivePath, data.name))
})
```

So a file dragged from `~/Downloads/foo.pdf` ends up living in **two** places: the original `~/Downloads/foo.pdf` and `Pear.app.storage/corestore/my-drive/foo.pdf`. There is no way to express "I'm sharing this *out of* my Downloads folder, please don't keep an extra copy."

**Why.**

- **Disk usage doubles per shared file.** Every drop silently duplicates bytes onto disk. For a peer sharing a few GB of media, this is the single biggest space cost in the app and there's no UI hint about it.
- **Stale originals drift from the share.** If the user later edits `~/Downloads/foo.pdf`, the version in `my-drive/` (and therefore the version every peer sees) is still the snapshot from drop time. The user has no signal that the two have diverged — they look at "their" file in Downloads and assume peers see the latest bytes. Removing the original closes that gap.
- **Move is the natural send semantic.** In every other "send a file to someone" UX (email attachments aside), users mentally model a transfer as moving a file out of their workspace, not duplicating it into a hidden corestore directory.

**Rough how.**

- Add an optional `delete` boolean to the HRPC `add-file` request (`schema.js:78-80`, then regenerate `spec/hrpc/`):
  ```js
  rpc.register({
    name: 'add-file',
    request: { name: '@basic-file-sharing/file', send: true } // already takes { name, uri, info }
  })
  ```
  and add a `delete` field on `@basic-file-sharing/file` in the schema (`schema.js:40-47`).
- In `worker-task.js:45-47`, branch on the flag — prefer `fs.rename` (atomic same-volume) and fall back to `copyFile` + `unlink` for cross-volume drops:
  ```js
  this.rpc.onAddFile(async (data) => {
    const dest = path.join(this.myDrivePath, data.name)
    if (data.delete) {
      try { await fs.promises.rename(data.uri, dest) }
      catch (err) {
        if (err.code !== 'EXDEV') throw err
        await fs.promises.copyFile(data.uri, dest)
        await fs.promises.unlink(data.uri)
      }
    } else {
      await fs.promises.copyFile(data.uri, dest)
    }
  })
  ```
- In `ui/root.jsx:18-30`, add a "Move (delete original)" checkbox near the drop zone and pass its state through `addFile({ name, uri: filePath, delete: moveChecked })`. Default off — destructive operations should be opt-in.
- Defer the delete until **after** the LocalDrive→Hyperdrive mirror has appended at least once if you want to be extra cautious (the upload `setInterval` is in `worker/drive-room.js:167-168`). For the simple case, deleting right after `copyFile` is fine because the LocalDrive at `my-drive/` is now the source of truth and the upload loop will pick it up within ~1 s.

**Edge cases worth flagging in the implementation.**

- **Name collision.** If `my-drive/` already has a file with the same name, `rename` overwrites silently on POSIX. Either rename-with-suffix or surface an error to the UI.
- **Permission errors on the source.** Files dragged from system-protected locations (e.g. `/Volumes/<read-only>/`, sandboxed directories on macOS) may not be deletable. Catch `EACCES`/`EPERM` and fall back to copy semantics with a UI toast like "Original kept (read-only source)".
- **Drag-from-browser / temp files.** Drops from a browser window resolve to OS temp paths (via `Runtime.media.getPathForFile`, `ui/root.jsx:20`). Deleting those is harmless and arguably correct — they were going to be GC'd anyway.

## 2. Categorise drives (curriculum / recordings / module-videos / examples / templates / …)

**What.** The example today has exactly one notion of "drive": every peer publishes a single `myDrive` and the autobase view stores a flat `drives` collection (`schema.js:57-61`, `worker/drive-room.js:37-38, 164`). Every drive is a peer of every other drive — there is no way to say "this drive is the syllabus", "this is module-3 videos", "this is the templates folder I'm sharing with TAs". The UI consequently shows one giant flat list grouped only by `info.name` (`ui/root.jsx:51-64`, sort in `worker/worker-task.js:70-74`).

**Why.** The five Pear School use cases in `notes.md` (curriculum, recordings, per-student feedback, module videos, group projects) all want **structurally different drives**, not just different names. They differ on at least four axes:

- **Who can publish** — only the instructor publishes curriculum; any student publishes a submission; TAs publish to templates.
- **Who replicates** — every student pulls curriculum; only the instructor pulls submissions; TA cohort pulls templates.
- **Storage policy** — curriculum is fully mirrored (small, durable); module videos are sparse-on-demand (large, viewed once); recordings are mirrored only for the active week.
- **UI surface** — students want a tabbed sidebar ("Curriculum / Videos / Templates / My submissions"), not a single alphabetised list of 50 drive names.

Encoding category in the free-text `info.name` ("[CURRICULUM] week-1") is the obvious workaround and the wrong one — it's not queryable, not enforceable, and breaks the moment two peers spell it differently.

**Rough how.**

Three layered changes — schema, dispatcher, UI/storage — each independently shippable.

### a. Schema: add a `category` field to the `drive` record

In `schema.js:28-34`, extend the drive shape:

```js
schema.register({
  name: 'drive',
  fields: [
    { name: 'key', type: 'buffer', required: true },
    { name: 'category', type: 'string', required: true },
    { name: 'info', type: 'json' }
  ]
})
```

`category` is a free string at the schema level (no enums in hyperschema), but the **app pins it to a known set** at the dispatcher: `curriculum | recordings | module-videos | examples | templates | submissions | feedback | group-project`. Treat it like an MIME type — open set, but the runtime knows the well-known values.

Then promote it to a queryable index by registering the collection's secondary key:

```js
db.collections.register({
  name: 'drives',
  schema: '@basic-file-sharing/drive',
  key: ['key']
})
db.indexes.register({
  name: 'drives-by-category',
  collection: '@basic-file-sharing/drives',
  key: ['category', 'key']
})
```

Now `view.find('@basic-file-sharing/drives-by-category', { gte: { category: 'curriculum' }, lt: { category: 'curriculum\xff' } })` is an O(log N) prefix scan instead of full-table filter. Matters when a course has 50 students × 5 categories = 250+ drive rows.

### b. Dispatcher: enforce who-can-publish-what at the autobase op layer

Today `_setupRouter` (`worker/drive-room.js:131-141`) blindly inserts every `add-drive` op:

```js
this.router.add('@basic-file-sharing/add-drive', async (data, context) => {
  await context.view.insert('@basic-file-sharing/drives', data)
})
```

Replace with a category-aware policy check:

```js
this.router.add('@basic-file-sharing/add-drive', async (data, context) => {
  const writerKey = context.node.from.key   // exposed by autobase apply context
  if (!isAllowedToPublish(data.category, writerKey, this.policy)) return
  await context.view.insert('@basic-file-sharing/drives', data)
})
```

Where `this.policy` is built from the room's identity claims (when basic-chat-identity merges in: `instructorIdentityPub`, `taIdentityPubs`, `enrolledStudentPubs`). Default policy table:

- **`curriculum`** — instructor only.
- **`recordings`** — instructor + TAs.
- **`module-videos`** — instructor only.
- **`examples`** — instructor + TAs.
- **`templates`** — instructor + TAs.
- **`submissions`** — any enrolled student, but only their own (`writerKey === student's key`).
- **`feedback`** — instructor only, audience-keyed (see entry 1's per-student-feedback note in `notes.md`).
- **`group-project`** — members of the group only (group membership stored in a sibling collection).

Rejected ops are silently dropped, not erroneously inserted — autobase still records the op in the writer's hypercore, but the view never materialises it. This is the same enforcement pattern multi-rooms uses for cross-writer ops.

### c. Filesystem: namespace `shared-drives/` by category

Today every drive lands in `shared-drives/<key>/` (`worker/worker-task.js:25, 53`). For 250 drives this is unbrowsable in Finder. Group by category:

```
shared-drives/
  curriculum/<key>/
  recordings/<key>/
  module-videos/<key>/
  examples/<key>/
  templates/<key>/
  submissions/<key>/        # only on instructor's machine
  feedback/<key>/           # only on the recipient student's machine
  group-project/<key>/
```

Change `_downloadSharedDrives` (`worker/drive-room.js:143-160`) to read each drive's `category` from the view row and use `path.join(this.sharedDrivesPath, item.category, key)` instead of the bare `path.join(this.sharedDrivesPath, key)` on line 149. Mirror call to the snapshot loop in `worker-task.js:53` (which currently rebuilds the same path).

There's one own-drive too: `myDrivePath` becomes `myDrivesPath` (plural) and gains a category dimension — a peer might publish *both* a `submissions` drive (their homework) *and* a `templates` drive (a starter pack they're sharing with classmates). Replace the single `myLocalDrive` / `myDrive` pair (`worker/drive-room.js:37-38`) with a `Map<category, { local, drive }>`. `_uploadMyDrive` (`worker/drive-room.js:162-169`) iterates the map.

### d. UI: tabbed groups instead of one flat list

Replace the single `<ul>` in `ui/root.jsx:51-64` with category sections. Pseudo-shape:

```jsx
const grouped = useMemo(() => groupBy(drives, d => d.category), [drives])
const order = ['curriculum', 'module-videos', 'recordings', 'examples', 'templates', 'submissions', 'feedback', 'group-project']

return order.filter(c => grouped[c]?.length).map(category => (
  <section key={category}>
    <h3 className='font-bold mt-4'>{labelFor(category)}</h3>
    <ul>{grouped[category].map(renderDrive)}</ul>
  </section>
))
```

The drag-and-drop zone needs a "publish into category…" picker — a `<select>` next to the file input that determines which of the user's category-drives receives the dropped file. Categories the user isn't authorised to publish to are disabled (with a tooltip "instructor only").

### e. Discovery scope: optional, but worth flagging

Today every drive joins the swarm by its own `discoveryKey` (`worker/drive-room.js:158, 165`). For Pear School at scale, you may want to **separate swarm topics by category** so a student running on hotel wifi doesn't discover and start replicating connections for 49 submission drives they have no business pulling. Two options:

- **Per-category swarm topic** — `swarm.join(hash('pearschool/' + courseId + '/' + category))`, drives advertise themselves on their category's topic only. Cheaper discovery, but requires a second coordination layer to map `category → drive keys`.
- **Per-drive (status quo) + role-gated mirror** — keep current discovery, but the *mirror* logic in `_downloadSharedDrives` skips drives whose category the local peer isn't supposed to pull. Wastes a few connections but keeps the swarm topology simple.

Default to the second; reach for the first only if connection count becomes a problem.

**Migration note.** Existing rooms have flat `drives` rows with no `category`. Treat `category === undefined` as `'legacy'` in the dispatcher and surface it as an "Uncategorised" section in the UI until the room is rebuilt. No need for a migration op — Pear School rooms will be created fresh against the new schema.

**Why this isn't just "use folders inside one drive".** A subdirectory layout inside a single Hyperdrive (`my-drive/curriculum/`, `my-drive/templates/`) seems simpler and is the wrong call for Pear School:

- **Authorisation is at the drive level, not the path level.** Hyperdrive has no per-path ACLs — anyone with the drive's key reads the whole tree. Curriculum and submissions need different audiences, which means different drives, which means different keys.
- **Replication scope is at the drive level.** You can sparse-fetch within a drive, but you still need the drive's full discovery to mirror anything. Separate drives let you skip whole categories on bandwidth-constrained peers.
- **Ownership is at the drive level.** Curriculum has one writer (instructor); submissions has one writer per student. A single shared drive would either need a shared writer key (insecure) or autobase-managed multiwriter semantics inside Hyperdrive (which doesn't exist — Hyperdrive is single-writer; multiwriter shared folders are what `autodrive` and the upcoming Hyperdrive-on-Autobase work address, but that's a bigger lift than just splitting drives).

## 3. Folder upload (drag a directory tree into the drop zone)

**What.** Today the drop zone in `ui/root.jsx:36-50` only accepts individual files — dragging a folder either silently does nothing or lands a phantom zero-byte entry. The HRPC `add-file` shape (`schema.js:40-47, 78-81`) and the worker handler (`worker/worker-task.js:45-47`) both assume a single leaf file at a time, so even if the UI sent a path-like `name`, the worker would flatten it.

**Why.** Folder upload is a prerequisite for the categorised-drives work in entry 2 to be usable in practice. Every Pear School category is naturally a tree:

- **Curriculum** — `week-1/lecture.pdf`, `week-1/exercises/`, `week-2/…`.
- **Module videos** — `module-3/lesson-2.mp4` plus chapter markers, transcripts.
- **Examples & templates** — project scaffolds with `src/`, `tests/`, `README.md`.

Without folder upload, an instructor publishes each file individually and then manually arranges them into subdirectories on disk *after* the fact, which round-trips them through the LocalDrive→Hyperdrive mirror twice and is hostile to anyone publishing more than a handful of files. Once Pear School courses ship real curricula, this stops being convenience and becomes the bottleneck.

The fortunate part: **only the input side is broken.** The downstream pipeline (LocalDrive ↔ Hyperdrive mirror, the snapshot loop's recursive `fs.readdir`, the UI's `file://` link list) already handles nested paths correctly. So the fix is bounded to the UI input, the HRPC contract, and the worker's `add-file` handler.

**Rough how.** Four coordinated changes that have to land together:

1. **UI: walk the dropped tree.** Replace `e.dataTransfer.files` (`ui/root.jsx:11`) with `e.dataTransfer.items[]` + `webkitGetAsEntry()` recursion. Add `webkitdirectory` to the file input (`ui/root.jsx:42-48`) so the OS picker exposes "select folder" mode. For each leaf, compute the relative path from the drop root.
2. **HRPC: carry the relative path.** Reuse `name` and document that it may contain `/` — Hyperdrive paths are slash-delimited strings already. (Alternative: add a `path` field to `@basic-file-sharing/file` in `schema.js:40-47`. Less ambiguous but requires a schema regen.)
3. **Worker: `mkdir -p` before copy.** In `worker/worker-task.js:45-47`:
   ```js
   const dest = path.join(this.myDrivePath, data.name)
   await fs.promises.mkdir(path.dirname(dest), { recursive: true })
   await fs.promises.copyFile(data.uri, dest)
   ```
4. **Or batch at the worker.** Alternative to leaf-by-leaf: add an `add-folder` HRPC method that takes a single root URI and have the worker walk it with `fs.cp(src, dest, { recursive: true })`. Less network chatter on the IPC channel, but loses per-file progress reporting.

**See also.** `notes.md` — the Q "I dragged a folder with sub-folders into the drop zone and nothing happened. Are folders just not supported?" walks through the layer-by-layer analysis (HTML5 drag event → file→URI step → HRPC contract → worker handler) that this entry's *why* and *rough how* are condensed from. Read that first if the *why each layer breaks* matters before touching code.

**Interaction with entry 1 (delete after upload).** If both ship, the "delete original" semantics need a pass on directories: `fs.rename` on a dir works on POSIX same-volume; cross-volume needs `fs.cp` + recursive delete. Worth a single test case per OS.
