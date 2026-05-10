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
