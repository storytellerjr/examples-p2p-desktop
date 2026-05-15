## Q: What is this folder about, high level?

A P2P **video library + chat room** Pear desktop app. Despite the name, it is not live streaming (that was `basic-live-cam`) — it is a multi-writer room where peers upload **whole video files**, replicate them, and play them back in-app, with a side chat.

### What it does

- One peer creates a room and prints a blind-pairing invite; another peer joins with `--invite <invite>` (`worker/index.js:32-34`).
- Each peer can `addVideo(filePath)` — the file is streamed into a `Hyperblobs` core, and a record (`id, name, type, blob, info`) is appended to a shared **Autobase** (`worker/video-room.js:179-199`).
- Each peer runs a local `hypercore-blob-server` and the UI plays videos via a generated HTTP link (`worker/video-room.js:174`).
- A second collection, `@basic-video-stream/messages`, gives the room a chat alongside the videos (`worker/video-room.js:135-137`, `201-210`).

### Stack (per `package.json`)

- Pear desktop runtime: `pear-electron` + `pear-bridge` + a worker (`worker/index.js`) talking to the UI over `hrpc` framed over `pear-pipe`.
- Data: `autobase` (multi-writer log) → `hyperdb` view with two collections (`videos`, `messages`) plus an `invites` table for blind-pairing.
- Transport: `hyperswarm` + `blind-pairing`.
- Blobs: `hyperblobs` for the actual video bytes, served locally by `hypercore-blob-server`.
- UI: React 19 + Tailwind, built with swc into `build/` (`ui/root.jsx` is the only source file).

### Architecture shape (same pattern as earlier examples)

- `index.js` boots Pear + bridge + Electron runtime + spawns the worker.
- `worker/index.js` parses CLI flags (`--invite`, `--name`, `--reset`), opens the corestore, starts a `WorkerTask` exposed over HRPC.
- `worker/video-room.js` is the core: pairing, autobase apply, the four dispatch handlers (`add-writer`, `add-invite`, `add-video`, `add-message`), and the blob upload/serve glue.
- `spec/{db,dispatch,hrpc,schema}` are generated from `schema.js`.

### Comparison to siblings

Sits closest to `basic-file-sharing` (blob-backed shared library), but specialized to video MIME types (`worker/video-room.js:182-184`) and bundled with a chat collection — essentially "file-sharing for videos + chat".

## Q: Isn't this just the previous folder with a Play button? Looks like `basic-photo-backup` could already play videos.

Correct — and this folder is in some ways a **step backward** vs `basic-photo-backup`, not forward. **This folder is not necessary to study** if you already worked through `basic-photo-backup/`.

### `basic-photo-backup` already plays videos inline

`basic-photo-backup/ui/root.jsx:59` renders `<video src={video.info.link} controls autoPlay />` for any non-image MIME. Same blob-server link, same `<video>` element, same autoplay. Drop a video into photo-backup's drop zone, click the thumbnail tile, it plays.

### What `basic-video-stream` actually changes vs `basic-photo-backup`

- **UI shape, not mechanism.** video-stream uses a flat list with an explicit Play/Stop toggle button per row (`ui/root.jsx:67-74`). photo-backup uses a gallery of thumbnail tiles you click into (`basic-photo-backup/ui/root.jsx:86-118`). Both ultimately render the same `<video>` element; both only allow one video "open" at a time (`playerId` is single-valued in both).
- **Narrower MIME filter.** video-stream rejects non-video files at the worker (`worker/video-room.js:182-184`). photo-backup accepts both images and videos and branches on `type.startsWith('image/')`.
- **No thumbnail generation.** photo-backup pulls in `bare-media` + `bare-ffmpeg` to generate preview thumbnails for images. video-stream drops both deps and shows only filenames until you press Play.
- **Otherwise identical.** Same autobase with `videos` + `messages` collections, same blind-pairing flow, same `hyperblobs` + `hypercore-blob-server` plumbing, same per-item comments. Only the `schema.js` namespace meaningfully differs at the protocol layer.

### "Stream" is a misnomer here

Nothing about the data path is actually streaming. Videos are uploaded as whole blobs into a `Hyperblobs` core (`worker/video-room.js:186-193`) and played back via a local HTTP link from `hypercore-blob-server` — same as photo-backup. No new Holepunch primitive is introduced; this folder is a UI variant of photo-backup minus images and minus the ffmpeg preview path. If you want actual live streaming, `basic-live-cam/` is the relevant example.
