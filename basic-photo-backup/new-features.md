# basic-photo-backup — possible future improvements

This file collects forward-looking ideas surfaced while studying this folder. It is intentionally separate from `notes.md` (Q&A study notes) so the two don't get tangled. Each entry is anchored to code with `path:line` and explains *what*, *why*, and *rough how*.

## 1. Multiple named galleries with descriptions (categories) — multi-rooms-style architecture

**What.** Today the app has exactly one flat collection — every file ever dropped goes into the same `@basic-photo-backup/videos` view (`schema.js:72-76`), materialised by `VideoRoom.getVideos` (`worker/video-room.js:165`) and rendered as one big wrap-grid (`ui/root.jsx:86-118`). There is no notion of a *category* the user can create, name, or describe.

The desired model is multiple galleries side-by-side — e.g. **Videos**, **Templates**, **AI Prompt Templates** — each with its own name, description, contents, and ability to be paired into independently.

The integration model to follow is **`basic-chat-multi-rooms`**, which solves the structurally identical problem ("one chat" → "many named chat rooms"). The relevant primitives are visible in `../basic-chat-multi-rooms/worker/chat-account.js` and `../basic-chat-multi-rooms/worker/chat-room.js`. The mapping is:

- **`ChatAccount` → `PhotoAccount`.** An account-level autobase that holds *only metadata*: the list of galleries (with their per-gallery invite stored in plaintext) plus the account's own pairing invite for multi-device sync. No video rows live here.
- **`ChatRoom` → `PhotoGallery`.** A per-gallery autobase, opened in its own `store.namespace(galleryId)`, with its own BlindPairing instance, its own discoveryKey joined on the swarm, its own invite, and its own Hyperblobs store. Videos and comments for that gallery live only here.
- **`addRoom` → `addGallery`** and **`joinRoom` → `joinGallery`**: same shape, same flow.
- **`openRooms` → `openGalleries`**: walk the account view's `galleries` collection on startup, instantiate one `PhotoGallery` per row.

This is the move from today's single `VideoRoom` (`worker/video-room.js:19`) — which fuses "the pairing/swarming layer" and "the data layer" into one class — into the two-level shape that multi-rooms already validates.

**Why.**

- **Single bucket doesn't scale.** Past ~50 items the flat grid becomes a "find the right thumbnail" game. Named categories give an obvious first-level cut.
- **Mixed content types coexist awkwardly.** "Videos" and "AI Prompt Templates" want different list affordances (video thumbnails vs. text snippets), and the UI already branches on mime type (`ui/root.jsx:54-60`). A gallery gives the UI a stable hook to vary the rendering instead of pattern-matching every file.
- **Per-gallery sharing is the real payoff.** Storing each gallery in its own autobase + swarm topic (mirroring `ChatAccount.addRoom` at `../basic-chat-multi-rooms/worker/chat-account.js:170-184`) means a peer can be invited to **just** the "AI Prompt Templates" gallery without seeing your "Videos" gallery — same security boundary multi-rooms gives you between rooms. With a single shared base, the entire library is all-or-nothing.
- **Per-gallery sync isolation.** A peer who only cares about prompts doesn't have to download multi-GB video blobs from another gallery they're not in. Each gallery's `blobs` core (`worker/video-room.js:37`) and `view` core sync independently because they're in their own namespace.
- **Per-gallery deletion gets cheap.** Tearing down a gallery is `room.close()` + nuking the namespace (mirroring entry 5 in `../basic-chat-multi-rooms/new-features.md`). With a single base you can only soft-delete rows and the blob bytes stay forever.
- **Foundation for entry 3 (delete) and entry 4 (rename).** Both operations are far more useful when scoped per-gallery and have an obvious analogue in multi-rooms.

**Rough how (mirroring multi-rooms file-for-file).**

- **Schema.** Replace `worker/video-room.js`'s mixed responsibilities with two schema namespaces, mirroring `../basic-chat-multi-rooms/schema.js`:
  - **Account-level collection `gallery`** — fields `{ id: string, name: string, description: string, invite: string, info: json }`. Direct analogue of multi-rooms' `room` schema (`../basic-chat-multi-rooms/schema.js:28-36`); the `invite` field is the gallery's BlindPairing invite, stored plaintext on the account base so paired devices auto-join.
  - **Gallery-level collection `video`** — keep the existing shape (`schema.js:28-37`); it stays the same, just lives on the gallery base instead of the global one.
  - **`messages`** (comments) — same shape, also on the gallery base. Each gallery has its own comment thread, isomorphic to multi-rooms' per-room messages.
  - **Dispatch ops.** Account-level: `add-writer`, `add-invite`, `add-gallery`, `update-gallery`. Gallery-level: `add-writer`, `add-invite`, `add-video`, `delete-video`, `add-message`. The account router must explicitly reject `add-video` / `add-message` with `throw new Error('Invalid op')`, exactly as `../basic-chat-multi-rooms/worker/chat-account.js:128-130` does for `add-message`. This is the load-bearing guardrail that keeps content out of the account base.
- **Worker.** Split `worker/video-room.js` into two files mirroring multi-rooms layout:
  - `worker/photo-account.js` — clone of `../basic-chat-multi-rooms/worker/chat-account.js`. Owns `this.galleries: Record<string, PhotoGallery>`, has `openGalleries()` (mirror of `openRooms`, `:156-168`), `addGallery(name, description, info)` (mirror of `addRoom`, `:170-184`), `joinGallery(invite)` (mirror of `joinRoom`, `:186-206`). On `addGallery`, it: (1) generates `id = Math.random().toString(16).slice(2)`, (2) opens `this.store.namespace(id)` as a fresh gallery, (3) calls `gallery.addGalleryInfo()` which writes the gallery's own metadata into its own base, (4) appends `add-gallery {id, name, description, invite, info}` to the account base.
  - `worker/photo-gallery.js` — clone of `../basic-chat-multi-rooms/worker/chat-room.js`, but keeps the today-only-on-the-room pieces from `video-room.js`: `Hyperblobs` setup (`worker/video-room.js:37-39, 99-100`), `BlobServer` lifecycle, blob-cores swarm-join loop in `getVideos` (`worker/video-room.js:165-179`), preview generation in `addVideo` (`worker/video-room.js:181-203`). Add a `getGalleryInfo()` / `addGalleryInfo()` pair mirroring `getRoomInfo` / `addRoomInfo` (`../basic-chat-multi-rooms/worker/chat-room.js:150-160`) so a joiner's first sync mirrors the gallery name/description back into their account view.
- **Worker bootstrap.** Update `worker/worker-task.js` (and `worker/index.js`) to instantiate `PhotoAccount` instead of `VideoRoom`. This is the exact shape multi-rooms uses — see how `../basic-chat-multi-rooms/worker/worker-task.js` wires the account, and how messages/rooms HRPC streams are split per-room.
- **HRPC surface.** Mirror `../basic-chat-multi-rooms/schema.js:98-119`:
  - `galleries` (stream of gallery rows from the account view)
  - `add-gallery`, `join-gallery`, `update-gallery`
  - `videos` and `add-video` get a `galleryId: string` field added to their request types (same pattern as `add-message { text, roomId }` at `../basic-chat-multi-rooms/schema.js:62-68`)
  - `messages` likewise gets `galleryId`
- **UI** (`ui/root.jsx`). Left-side sidebar with the gallery list (rendered from the account view) and a "+ New Gallery" affordance. Click a gallery → it becomes the current scope; the existing drop zone and grid render only that gallery's items. Show `name` bold and the first ~60 chars of `description` muted. The view-mode toggle for a player (`ui/root.jsx:41-84`) stays unchanged — it just operates within the currently-selected gallery.
- **Migration from single-`VideoRoom` data.** First boot after upgrade detects the legacy single-base layout (an autobase already present at the root namespace with videos in it). On detection: open the legacy base as a read-only `PhotoGallery`, create a new account base, register the legacy gallery in the account view as `{id: 'legacy', name: 'Gallery', description: '', invite: <re-issued>}`. The legacy autobase is reused in place rather than re-imported — its `store.namespace` is just adopted under the new account's `this.galleries['legacy']`. No blob copy, no replay, fast.
- **Authorization note.** Same recurring caveat as multi-rooms: any writer in a gallery can add/edit/delete any video in that gallery; any writer on the account base can add/rename any gallery. Acceptable for a 2-peer photo backup; the proper fix is an identity layer (see entry 12 of `../basic-chat-multi-rooms/new-features.md`).

**Cross-ref.** Once this is in place, entries 5–8 of `../basic-chat-multi-rooms/new-features.md` (leave-room, delete-message, edit-message, rename-room) become directly portable: their dispatch ops, router handlers, and HRPC channels translate one-to-one into leave-gallery, delete-video, rename-gallery on the gallery base. Don't re-design these — copy the multi-rooms patterns.

## 2. Add a folder of files (including subfolders)

**What.** The current intake paths are both file-only:

- Drag-and-drop handler builds a list from `e.dataTransfer.files` (`ui/root.jsx:13-16`, then `onAddFiles` at `:29-34`).
- The "Browse files" input is `<input type='file' multiple>` (`ui/root.jsx:128-135`) — no `webkitdirectory` attribute, no folder semantics.

Both routes resolve each `File` to a path via `Runtime.media.getPathForFile(file)` (`ui/root.jsx:24, 31`) and call `addVideo(filePath)`. There is no recursion; a dropped folder is silently ignored on most platforms.

**Why.**

- **Real photo collections live in folders.** Importing a year-by-month tree by clicking each file is unusable for any non-trivial backup. The whole point of "photo backup" is bulk ingest.
- **Already accepted file types degrade gracefully.** `addVideo` already filters by mime type (`worker/video-room.js:184-186`); a recursive walk that skips non-media files is a small extension, not a rethink.
- **Folder structure is information.** If `~/Photos/2024/Italy/` is being ingested, the folder names are useful metadata — they map naturally onto entry 1's gallery concept (one top-level subfolder per gallery) or onto a future tag/album field.

**Rough how.**

- **Drop side.** Switch the drop handler from `e.dataTransfer.files` to `e.dataTransfer.items`. For each item, call `item.webkitGetAsEntry()` and walk recursively:
  ```js
  // pseudo
  async function walk (entry, prefix = '') {
    if (entry.isFile) {
      const file = await new Promise(r => entry.file(r))
      const path = Runtime.media.getPathForFile(file)
      await addVideo(path, { folder: prefix })
    } else if (entry.isDirectory) {
      const reader = entry.createReader()
      const children = await readAll(reader)
      for (const child of children) await walk(child, prefix ? `${prefix}/${entry.name}` : entry.name)
    }
  }
  ```
- **Browse side.** Two input fields side-by-side: the existing `<input type='file' multiple>` and a new `<input type='file' webkitdirectory>` for folder mode. Chrome/Electron both honour `webkitdirectory`; Pear inherits.
- **Mime filtering stays in the worker.** Let the walk hand every leaf to `addVideo`; the existing `Only image/video files are allowed` check at `worker/video-room.js:184-186` becomes a skip rather than an `Error`. Logging the skipped count back to the UI is friendlier than silent drops.
- **Native folder picker (alternative).** If `webkitdirectory` proves unreliable inside Pear's Electron build, fall back to Electron's `dialog.showOpenDialog({ properties: ['openDirectory'] })` exposed over IPC and walk the absolute path with `bare-fs` — already a dependency (`package.json:44`).
- **Folder → gallery mapping (depends on entry 1).** When the user drops a folder onto the gallery list (not into an existing gallery), auto-create a gallery named after the folder and import its contents into it. Subfolders become a `info.folder` breadcrumb on each video for later display; no new collection needed in v1.
- **Performance.** Large imports should not block the UI thread. Stream additions through the worker (which already runs off-thread per `worker/index.js`), and emit progress events over HRPC so the UI can render a `42 / 1200 imported …` indicator.

## 3. Delete selected files and folders via a delete button

**What.** No delete path exists end-to-end:

- The schema/dispatch only defines `add-*` ops (`schema.js:86-89`); no `delete-video`, no `delete-gallery`.
- `VideoRoom`'s router only handles inserts (`worker/video-room.js:127-140`).
- The UI has no selection state and no delete affordance (`ui/root.jsx:86-118` renders thumbnails clickable for *view*, not *select*).

The desired flow: click a thumbnail (or a whole gallery) to select it, click a **Delete** button, the item disappears for both peers.

**Why.**

- **Storyteller needs to undo mistakes.** Dragging the wrong folder in is otherwise permanent without a `--reset`.
- **Photo libraries are pruned, not just grown.** Without delete, "backup" turns into "hoard".
- **Companion to entry 2 (folder import).** Mass-import without mass-delete is asymmetric — once you can pull in a thousand items in one drop, you need to be able to evict them just as easily.

**Rough how.**

- **Selection model in the UI** (`ui/root.jsx`):
  - Add `selectedIds: Set<string>` and `selectionMode: 'item' | 'gallery'` state.
  - Single-click on a thumbnail in selection mode toggles inclusion; Shift+click extends a range. Esc clears. Distinguish *view* (current behaviour at `:93` — opens the player) from *select* (new) with a checkbox overlay on hover, or by entering selection mode explicitly with a toolbar toggle.
  - Show a sticky toolbar when `selectedIds.size > 0`: `Delete (3)` button plus a `Cancel` link.
- **Schema / dispatch ops** (`schema.js`):
  - `delete-video { id: string }` and (with entry 1) `delete-gallery { id: string }`. Both register under `hyperdispatch.register` next to `add-video` (`schema.js:86-89`).
- **Router** (`worker/video-room.js:127-140`):
  - On `delete-video`: `await context.view.delete('@basic-photo-backup/videos', { id })`. Optionally also enqueue cleanup of the blob (see "Blob lifecycle" below).
  - On `delete-gallery`: delete the gallery row, then iterate `videos` filtered by `galleryId` and delete each, then delete any `messages` whose `info.videoId` references a deleted video (cascade).
- **Worker API.** `deleteVideo(id)`, `deleteVideos(ids[])` (single batched autobase append where possible), `deleteGallery(id)`.
- **HRPC.** Add corresponding channels alongside `add-video` at `schema.js:99-101`.
- **Blob lifecycle (the genuinely-hard bit).** Today every dropped file pushes bytes into the local Hyperblobs store via `this.blobs.createWriteStream()` (`worker/video-room.js:189-194`). HyperDB-row deletion doesn't reclaim those bytes; the blob core grows monotonically. Options for v1:
  - **Tombstone only.** Mark the row deleted but leave blob bytes on disk. Cheapest; pruning is a future "compact" command.
  - **Best-effort blob clear.** Call `this.blobs.clear(id)` (if supported in the pinned `hyperblobs` version) inside the router op. Frees local disk but does not propagate to peers automatically — they still hold their copy until they run the same op.
  - Either way, document that "delete" is *autobase-level*: removes the row from the materialised view, but the historical autobase block recording the original `add-video` is immutable. Restoring is non-trivial; this is true today for any HyperDB-backed app.
- **Authorization caveat.** Same shape as the chat examples: any room writer can delete any file/gallery until an identity layer (`basic-chat-identity`-style `proof`) is layered in. Acceptable for a 2-person setup; flag as future work.
- **Soft-delete alternative (worth considering).** Instead of a hard delete, set `info.deleted = true` and filter it out at read time. Reversible from the same UI as a "Trash" view, and the blob-lifecycle problem becomes irrelevant. Trade-off: storage grows unbounded.

## 4. Update name and description of a category (gallery)

**What.** Depends on entry 1 — once galleries exist as schema rows with `name` and `description`, those fields need to be editable. There is no rename/edit op today because there is no gallery primitive today.

**Why.**

- **First-name choice is rarely the final one.** Storyteller drops a folder named `tmp-import-2026-05` and later wants it named `Spring trip`. Without edit, the only fix is delete-and-recreate, which loses the gallery's contents (or forces a re-import).
- **Descriptions evolve.** What started as "scratch" becomes "Onboarding videos for new team members" — a one-line description is cheap to add and pays back every time a peer opens the app.
- **Pairs with entry 3 (delete).** Together they give the gallery list its full CRUD shape (create from entry 1, read from the gallery list, **update** here, **delete** from entry 3).

**Rough how.**

- **Op.** `update-gallery { id: string, name?: string, description?: string }` in the dispatch namespace (`schema.js:86-89`). Optional fields so partial updates don't clobber the other.
- **Router** (`worker/video-room.js:127-140`):
  - On `update-gallery`: read the existing row (`await context.view.findOne('@basic-photo-backup/galleries', { id })`); merge non-undefined fields from the request; re-insert (HyperDB upserts on the `['id']` key, so this is a true update, not a duplicate). Skip the write if both fields are undefined.
- **Worker API.** `updateGallery(id, { name, description })`.
- **HRPC.** Add an `update-gallery` channel.
- **UI** (`ui/root.jsx`):
  - Inline-edit on the gallery's name in the sidebar (double-click → input → blur or Enter to commit).
  - Separate "Details" panel for the longer description — multi-line textarea, "Save" button that calls `updateGallery`.
- **Validation.** Trim whitespace; reject empty names (UI-side, since the schema has no `required: true` on `name` after partial updates). Description has no length cap in v1; cap to ~500 chars if it later starts bloating the autobase.
- **Concurrent edits.** Two peers renaming simultaneously will both append `update-gallery` blocks; autobase linearises them and the last-applied one wins. Acceptable — same shape as message edits in chat. If "last-write-wins" surprises Storyteller, add an `info.updatedAt` and surface "renamed by other peer just now" hint in the UI.
- **Authorization caveat.** Same recurring note: any room writer can rename any gallery. Out of scope for v1.

## 5. Rename the folder — the name no longer fits what the app does

**What.** The folder is called `basic-photo-backup`, but the code has already grown past photos:

- `addVideo` accepts both `image/*` and `video/*` mime types (`worker/video-room.js:184-186`), and the UI's gallery renders both (`ui/root.jsx:54-60, 96-112`).
- The schema's primary entity is literally named `video` (`schema.js:28-37`) even though it holds images too — the name is already a known liar.
- With entry 1 landed, galleries like **Templates** and **AI Prompt Templates** become first-class — non-media payloads (Markdown snippets, text templates) are explicitly in scope.
- With entry 2 landed, a user drops *any* folder tree in and expects it to be ingested; the file-type filter is only one layer deep.

**Why.**

- **The folder name is a discoverability signal in this study repo.** Storyteller (and anyone reading the repo on GitHub later) scans the `examples-p2p-desktop/` listing looking for the right example. "Photo backup" understates what's actually here and undersells the broader use cases — it's the only multi-blob shared store in the repo and deserves a name that flags it.
- **"Backup" is misleading too.** This isn't a backup tool in the data-recovery sense (no versioning, no scheduled snapshots, no integrity verification). It's a *shared mutable library* — the second peer can add and comment too, not just receive a copy. The word "backup" sets a wrong expectation.
- **Naming cost is paid once, ambiguity cost is paid every read.** The longer the folder lives with a misleading name, the more cross-references rot (`CLAUDE.md`, `README.md`, the folder-scoped `/tmp/photo-backup-user*` paths, any future top-level `examples-p2p-desktop/README.md`).

**Candidate names** (least-radical to most):

- **`basic-media-backup`** — minimal change; just generalises "photo" to "media". Still keeps the misleading "backup" word, and still doesn't cover non-media payloads (prompts/templates from entry 1).
- **`basic-media-library`** — drops "backup", uses "library" which matches the *shared mutable collection* nature. Still narrow to media.
- **`basic-shared-library`** — neutral, accurate, doesn't presuppose content type. Pairs naturally with multi-gallery in entry 1.
- **`basic-shared-vault`** — emphasises the P2P shared-private aspect; "vault" hints at the encrypted autobase + blob storage that's actually under the hood.
- **`basic-shared-gallery`** — leans into the user-visible primitive (the gallery, plural after entry 1). The closest analogue to the existing `basic-chat-*` naming where the folder names what the user *sees*, not how it's stored.
- **`basic-shared-blobs`** — most technically descriptive (`Hyperblobs` + autobase view is literally the architecture, `worker/video-room.js:37-39`). Good for the study-repo audience who's here to learn the stack; bad for anyone scanning by domain.

**Recommendation.** **`basic-shared-gallery`** — once entry 1 lands, "gallery" is the user-facing noun, the schema entity name (`gallery`), and the folder reads naturally next to `basic-chat`, `basic-chat-multi-rooms`, `basic-file-sharing`. It captures both the shared-with-a-peer nature *and* the multi-bucket structure the app is heading toward. Fallback: **`basic-shared-library`** if Storyteller prefers a more generic noun for galleries that hold non-media content.

**Rough how.**

- **Folder rename.** `git mv basic-photo-backup basic-shared-gallery`.
- **Schema namespace.** Every `@basic-photo-backup/…` string in `schema.js` (12 references) becomes `@basic-shared-gallery/…`. The namespace shows up across `schema.register({…})` calls, `db.collections.register({…})`, dispatch/HRPC registrations, **and inside the worker** in router-key strings (`worker/video-room.js:128, 131, 134, 137`, `:88, :135, :136`, etc.) and the `getInvite` lookup (`worker/video-room.js:148`). Easy `grep -r '@basic-photo-backup' .` to find them all.
- **Regenerate `spec/`.** Run `npm run build:db` (`package.json:34`) after the schema-string change so `spec/db`, `spec/dispatch`, `spec/hrpc`, and `spec/schema` are written with the new namespace baked in. Without this step the dispatch router will throw `Unknown route` on every op — same failure mode as the cross-folder `Unknown collection type: N` documented in `../CLAUDE.md`.
- **`package.json:2`.** `"name": "basic-shared-gallery"`.
- **`README.md`.** Title, intro line, and **all `--store` paths**. Per the `../CLAUDE.md` convention, switch `/tmp/photo-backup-user1` → `/tmp/shared-gallery-user1` (and `…-user2`) across Usage, Build Pear app, and Troubleshoot sections. Update the explanatory blockquote above the first `pear run` block to reference the new shortname.
- **Anything in `CLAUDE.md`** at the parent level that says "`basic-photo-backup/` is the active study folder" (`../CLAUDE.md:9`) — update to the new name when this rename lands. Same for any future top-level README that lists examples.
- **`notes.md` and `new-features.md`.** Their content stays valid; cross-references that say "this folder" stay valid; explicit references to `basic-photo-backup` (e.g. notes.md mentions `basic-photo-backup` in `path:line` anchors and stack lists) get a search-and-replace pass.
- **Data layout note.** Existing on-disk data under `/tmp/photo-backup-user1` is *not* automatically picked up under `/tmp/shared-gallery-user1`. For Storyteller's study workflow that's fine — restart fresh. If preserving existing data ever matters: `mv /tmp/photo-backup-user1 /tmp/shared-gallery-user1` before the next `pear run`.
- **Sequencing.** Land this rename **before** entry 1's multi-base refactor lands. Once entry 1 ships, the worker file split (`photo-account.js`, `photo-gallery.js`) bakes "photo" into many more filenames; renaming after means a bigger diff. Renaming first means the multi-base refactor lands with the right names from day one (e.g. `gallery-account.js`, `photo-gallery.js` → `gallery.js`).

## 6. Pear School use cases — how this folder fits an educational deployment

Not an implementation entry — a use-case scan so the features above can be prioritised against what Pear School actually needs. Non-technical; the value is in matching the app's primitives (shared galleries, drag-and-drop ingest, per-item comments, peer-to-peer sync) to classroom workflows.

### Primary use cases

- **Course materials library.** Instructor drops recorded lectures, slide images, and demo videos into a course gallery. Every paired student sees the materials appear automatically and can rewatch on demand. Comments under each lecture become a per-lecture Q&A thread ("can you re-explain the part at 14:32?"). Closest analogue today: a course's Google Drive folder — but private to the cohort and free of any school-server cost.
- **Assignment submissions.** Each student has a private gallery shared with just the instructor. They drop in a screenshot of code, a photo of worked-out math, or a recorded explainer video. Instructor's feedback lives as comments attached to the specific submission — feedback never gets separated from the work it refers to.
- **Reference libraries per course.** Multiple named galleries side-by-side — "AI Prompt Templates", "Code Snippets", "Reading Materials", "Past Exams", "Sample Solutions". Students find what they need without scrolling past unrelated content. Directly motivates entry 1 of this file; without multi-gallery support this use case collapses into "one big folder".
- **Peer-review portfolios.** Students share their gallery with assigned peer reviewers. Comments turn into structured, persistent feedback the student keeps long after the course ends. Pairs naturally with the comments primitive that already exists in this folder (`worker/video-room.js:209-214`, `ui/root.jsx:62-81`).
- **Project documentation for group work.** Capstone or group-project teams drop photos and videos of work-in-progress into a shared team gallery. Comments capture context ("prototype v2 — fixed the wobble"). Doubles as the team's project record at submission time.
- **Office-hours archive.** Instructor records office-hours sessions and adds them to an "Office Hours" gallery the whole cohort can rewatch — covers students who couldn't attend live without the instructor having to manage a separate video-hosting platform.
- **Onboarding for late-joiners.** A student joining mid-term gets one invite and instantly has every prior lecture, template, and resource the cohort accumulated — no email chains, no "can you send me the slides from week 3". This is a single-action workflow that's expensive on traditional LMSes.

### Why this beats cloud-based equivalents for a school

- **Lives on the participating devices.** No school-server cost, no admin overhead, private by default — the institution never has to host or back up the content itself.
- **No accounts, no logins.** Pairing is by invite. Lower friction for students; no IT-managed identities to provision or deprovision.
- **Works offline for already-synced content.** Students on flaky home connections, on transit, or in a class with no wifi still have full access to materials they've already received.
- **Comments and content stay together.** Annotations don't get orphaned when files move, get renamed, or get reorganised — they live in the same autobase as the items they refer to.

### Honest limits to flag to Pear School

- **No identity layer in v1.** A comment can't yet be reliably attributed to "student S" — any room writer is currently indistinguishable from any other. This blocks assignment grading and any feature that depends on "who said what". The fix is to layer in the identity primitive from `../basic-chat-identity/` (referenced as entry 12 in `../basic-chat-multi-rooms/new-features.md`).
- **No delete or rename in v1.** Needed before students/teachers will trust this for graded materials — a wrong-folder drop or a typo'd gallery name shouldn't be a "burn it down with `--reset`" event. Entries 3 and 4 of this file address these.
- **Two-peer mental model.** Multi-student galleries technically work (autobase admits N writers), but the invite flow is one-at-a-time today and there's no presence indicator. For a 30-student cohort, the multi-rooms-style sharing in entry 1 plus a presence layer (entry 13 of `../basic-chat-multi-rooms/new-features.md`) is the real path.
- **No teacher↔student direct channel.** Private feedback today means a separate gallery per student-pair — workable but clunky. The DM design in entry 14 of `../basic-chat-multi-rooms/new-features.md` ports over cleanly once identity is in.

### Feature prioritisation through a Pear School lens

If Pear School is the load-bearing customer for this folder, the implementation order from entry 7 below shifts slightly:

- **Rename (entry 5)** — still first; cheap and clears ambiguity for anyone evaluating the example for a school deployment.
- **Multi-gallery (entry 1)** — second; unlocks the **Course materials library**, **Reference libraries**, and **Assignment submissions** use cases. Without it none of those work at scale.
- **Identity layer** (not in this file — borrowed from `../basic-chat-identity/`) — third, before delete/rename. Pear School can't grade on comment-attribution without it; flagging here even though the work itself lives in a different folder.
- **Update gallery (entry 4)** — fourth; minimum viable correction for typos and reassignments.
- **Delete (entry 3)** — fifth; trust threshold for graded materials.
- **Folder import (entry 2)** — sixth; quality-of-life for instructors with existing course archives on disk.

## 7. Cross-cutting: dependency graph for the features above

Not a feature on its own — a reading guide so the implementation order is obvious:

- **Entry 5 (folder rename)** should land **first**. Cheap when the folder is small, expensive after entry 1 splits the worker into multiple files. Pure rename — no behavioural change.
- **Entry 1 (galleries)** is the architectural foundation. Build second; it shapes the schema, the two-level autobase split, and the sidebar layout. Mirrors `../basic-chat-multi-rooms/` file-for-file.
- **Entry 2 (folder import)** can land independently of 1 but is most useful *after* it — dropping a folder onto the gallery list to auto-create a gallery is the natural UX.
- **Entry 4 (update gallery)** strictly requires entry 1; it has no meaning without a gallery primitive to rename.
- **Entry 3 (delete)** needs the schema additions from entry 1 to delete *galleries*; deleting individual *items* can ship sooner with just the file-level `delete-video` op.

Suggested order: **5 → 1 → 2 → 4 → 3**. Rename first (paid once, removes a long-running ambiguity). Delete last because it forces decisions about blob lifecycle that are easier once the other surfaces are stable.
