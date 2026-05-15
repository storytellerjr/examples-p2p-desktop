# basic-photo-backup notes

## Q: What does the code do, in high-level non-technical terms?

It's a **shared photo & video album that two people can both add to — no cloud, no server**.

- **Drag photos or videos in.** Drop a file on the window (or click *Browse files* and pick one) and it appears in a gallery of thumbnails. Entry point: `ui/root.jsx:13` (`handleDrop`) and `ui/root.jsx:22` (`handleSelect`).
- **Click a thumbnail to open it.** Images show full size; videos play in a built-in player. See `ui/root.jsx:41` (`renderPlayer`).
- **Leave comments under each photo/video.** Like a tiny private Instagram thread, scoped to one item. Comments are stored as messages tagged with the item's `videoId` — `ui/root.jsx:36` (`onSend`) and `ui/root.jsx:44` (filter by `videoId`).
- **A second person joins by invite.** One user starts a room and prints an invite; the second user pastes that invite (`--invite <invite>` flag in `README.md:17`). Both galleries then stay in sync — anything one adds or comments on, the other sees.
- **Files flow directly between the two computers.** No central server holds the photos; the apps find each other peer-to-peer and copy files between themselves. Big videos get a small preview thumbnail generated locally so the gallery loads fast (`lib/create-preview-image.js`, `lib/create-preview-video.js`).

In spirit: a **two-person shared photo backup with comments**, where files only live on the participating devices.

Stack used (from `README.md:5`): holepunch, bare, pear, corestore, hyperswarm, blind-pairing, hyperdb, hrpc, autobase, hyperblobs, hypercore-blob-server, bare-media, bare-ffmpeg.

## Q: Why did the README's `--store` paths need to change?

The upstream README shipped with `--store /tmp/user1` and `--store /tmp/user2`. Those bare paths collide with every other example folder in this study repo, because each folder has its own schema (different namespace, collection IDs, hyperdispatch offsets — `schema.js`) and that metadata is baked into every autobase block. Reusing one folder's store directory while running a different folder makes the app try to reconstruct a record whose collection ID isn't registered locally and it crashes with `Uncaught Error: Unknown collection type: N`.

Fix applied (per the repo-wide convention in `../CLAUDE.md`):

- Replaced every `--store /tmp/user1` with `--store /tmp/photo-backup-user1`, and `/tmp/user2` with `/tmp/photo-backup-user2`, across the **Usage**, **Build Pear app**, and **Troubleshoot** sections of `README.md`.
- Added the explanatory blockquote above the first `pear run` block (`README.md:9`), matching the `basic-live-cam/README.md:10` reference template.

Outcome: this example's on-disk data now sits in its own folder, so Storyteller can hop between examples without `--reset` dances and without losing any folder's existing state.

## Q: What do `npm i` and `npm run build` actually do here?

**`npm i`** (short for `npm install`)

- Reads `package.json` and downloads every library this app depends on into a local `node_modules/` folder.
- In plain terms: "fetch all the puzzle pieces this project needs to actually run."
- For this app, that's ~30 libraries — React on the UI side (`package.json:64-65`), the whole Holepunch stack (`autobase`, `corestore`, `hyperswarm`, `hyperdb`, `hyperblobs`, `hypercore-blob-server`, `blind-pairing`, `hrpc`, …, `package.json:41-67`), ffmpeg + media bindings for thumbnail generation (`bare-ffmpeg`, `bare-media`, `package.json:43,45`), Pear runtime glue (`pear-bridge`, `pear-electron`, `pear-pipe`, `package.json:61-63`), plus the build-time devDependencies for Tailwind and SWC (`package.json:69-75`).

**`npm run build`**

- Runs the `"build"` script in `package.json:37`, which chains two steps:
  - **`build:tailwind`** (`package.json:36`) — takes `input.css` (which holds the Tailwind directives) and produces the final `output.css` the browser actually loads. This is what turns class names like `bg-blue-500`, `flex`, `cursor-pointer` into real CSS rules. Without this step, every class in `ui/root.jsx` is a no-op.
  - **`build:swc`** (`package.json:35`) — runs SWC (a fast JS compiler) over the `ui/` folder. `ui/root.jsx` is React-with-JSX, which the browser can't parse directly; SWC strips the JSX into plain `.js` and writes the result into `build/`. The `--copy-files --strip-leading-paths` flags copy non-JSX assets along and flatten the `ui/` prefix in the output.
- Net effect: converts the human-friendly source (`ui/root.jsx`, `input.css`) into the browser-friendly form that Pear's window loads at runtime via `index.html`.

**Why the order matters**

- `npm i` first: nothing else can run without `node_modules/` (SWC and Tailwind are dev-deps, so they're not on disk yet).
- `npm run build` second: produces `build/` and `output.css` so the UI has something to display.
- `pear run …` last (per `README.md` Usage): launches the Pear runtime, which loads `index.html` → bundled UI → spawns the worker, joins the swarm, etc.

**Sibling helpers in `scripts` (`package.json:33-38`)**

- `clean`: `rm -rf build` — nukes the SWC output, useful when a stale build is misbehaving.
- `build:db`: `node schema.js` — regenerates the hyperdb/hyperdispatch spec into `spec/`. Only needs running after editing `schema.js`; not part of the default build pipeline.
- `start`: `pear run .` — convenience shortcut, but the README prefers the explicit `--store …` form so paths stay folder-scoped.

## Q: `pear stage basic-photo-backup` errored with "A valid pear link must be specified" — what went wrong?

The `<channel>` placeholder in `README.md:25, 28-32` is **not the folder name** — it's an arbitrary channel label you pick (like `dev`, `latest`, `release`, `production`). Pear combines the channel name with the project (from `package.json:2`) to mint the actual `pear://…` link.

Calling `pear stage basic-photo-backup` made Pear interpret `basic-photo-backup` as the `<link>` positional in `pear stage [flags] <link> [dir=.]`, and since that string isn't a `pear://…` URL it rejected it. The folder name happens to match the project name but that's coincidence — Pear doesn't infer "channel = folder name".

**Fix.** Pick a channel label and pass that:

```shell
pear stage dev
pear seed dev
# pear prints a pear://… link — use it below
pear run --store /tmp/photo-backup-user1 <pear-link> --name user1
pear run --store /tmp/photo-backup-user2 <pear-link> --name user2 --invite <invite>
```

**When to use stage vs. local dev mode.** Two different flows exist in `README.md`:

- **Usage section (`README.md:7-19`)** — local dev. `pear run … . --name user1` points Pear at the current folder. No staging, no seeding, no pear-link. Fastest study loop because there's no link bookkeeping; rebuild → re-run. This is the right path while reading and annotating the code.
- **Build Pear app section (`README.md:23-33`)** — distribution. `pear stage <channel>` snapshots the current build into the Pear network and `pear seed <channel>` keeps that snapshot reachable to peers. `pear run` then takes a `pear://…` link so the app can be launched on machines that don't have the source checked out. Use this when shipping a build to another device.

Rule of thumb for this study repo: stay in the local-dev path (`pear run … .`) unless explicitly testing the "ship to another machine" workflow. Storyteller's typical loop is `npm run build && pear run --store /tmp/photo-backup-user1 . --name user1` in one terminal and the `--invite` variant in another.

## Q: How do I actually exercise `pear stage` and `pear seed` to learn what they do?

Goal of this session was specifically to **learn the stage/seed flow** (not just sidestep it). The plan below was the hands-on walkthrough run from this folder.

**Concept first**

- **`pear stage <channel>`** snapshots the current built project and publishes it under a **channel name** (a label of your choice) attached to your project key. First run mints the `pear://<key>/<channel>` link. Subsequent runs print a diff vs. the previous snapshot.
- **`pear seed <channel>`** keeps that snapshot **reachable** on the network. Without seeding, the link exists but no peer can pull from it. It's a long-running command — leave it open in its own terminal.
- **`pear run pear://<key>/<channel>`** fetches the staged snapshot and launches it — this is what another machine (or even another `--store` on yours) sees.

**Step-by-step exercise**

```shell
# 1. Make sure the build is current (stage snapshots build output, not source)
npm run build

# 2. First stage — mints the pear:// link
pear stage dev
#   → prints "pear://<key>/dev" plus a list of staged files

# 3. Stage again with no changes — should report "no changes"
pear stage dev

# 4. Change something tiny (e.g., a string in ui/root.jsx),
#    rebuild, stage again, and watch the diff
npm run build
pear stage dev
#   → now prints only the changed files

# 5. Preview without writing
pear stage --dry-run dev

# 6. Try tree-shaking
pear stage --compact dev

# 7. Try a second channel — same project, different link suffix
pear stage prod
#   → pear://<same-key>/prod   (different snapshot history)

# 8. In a SEPARATE terminal, start seeding
pear seed dev
#   → stays running; this is what makes the link fetchable

# 9. In a THIRD terminal, run from the link (not from `.`)
pear run --store /tmp/photo-backup-user1 pear://<key>/dev --name user1
#   → copy the printed invite

# 10. FOURTH terminal — second user, same link, different store
pear run --store /tmp/photo-backup-user2 pear://<key>/dev --name user2 --invite <invite>
```

**Observations to expect while running it**

- The `pear://<key>` part is **stable** across all your stages — it's derived from the project key (which in turn lives in your local Pear keystore). Only the `/<channel>` suffix and its snapshot history vary.
- Different channels (`dev`, `prod`) give you **parallel release tracks** from the same project. Handy for "stable for peers, breaking for me". Each channel has its own independent version history.
- `pear stage` is **idempotent** on identical content; running it twice in a row is cheap and the second invocation tells you nothing changed.
- `pear seed` is **what makes a stage useful**. Stop the seed terminal and then `pear run pear://…/dev` from a fresh store — it'll hang waiting for a peer. Resume seeding and the run unblocks.
- Even on a single machine, the `pear run pear://…` flow **exercises the real fetch path** — Pear treats your other `--store` as a remote peer. This is what makes single-machine testing of staged builds meaningful, not just theatre.

**Useful flags to explore (from `pear stage --help`)**

- `--dry-run | -d` — preview a stage without writing.
- `--compact | -c` — minimise via static analysis (drops modules not reachable from `main`).
- `--ignore <paths>` / `--only <paths>` — comma-separated path filters for what gets included.
- `--purge` — remove files that *were* in a previous stage but are no longer present locally. Without this, deletions on disk don't propagate to the stage.
- `--truncate <n>` — rewind the channel to version `n`. Advanced; use to undo a bad stage.
- `--json` — newline-delimited JSON output for scripting around the command.

**stage vs. seed vs. run — the mental model that stuck**

- **Stage** = "freeze a build into an immutable snapshot and attach it to a channel name". Local act; doesn't put anything on the wire.
- **Seed** = "advertise that I have these snapshots and serve them to anyone who asks". This is the act that puts content onto the Holepunch network.
- **Run** = "given a `pear://…` link, fetch the latest snapshot for that channel from whichever seeder I can find, cache it locally, and launch it". The launched app then runs exactly as if it had been run from a local checkout — same code, same Bare/Electron runtime, same swarm joins.

The pieces are intentionally split so the same app can be staged from a dev machine, seeded from a server, and run from a phone — none of which has to be the same device.

## Q: Do I need to run `npm run build` again before each `pear stage dev`?

**Short answer.** Only if anything inside `ui/` or `input.css` (or `schema.js` for `build:db`) changed since the last build. But it's cheap to re-run as a habit and avoids stale-artifact bugs.

**Why it matters specifically here.** The `pear.stage.ignore` list in `package.json:11-22` excludes the *sources* that go through a compile step:

- `ui/` — React-with-JSX source. The compiled output in `build/` is what actually gets staged.
- `input.css` — Tailwind source. The compiled `output.css` is staged.
- `schema.js` — schema source. The generated `spec/` directory is what gets staged.

If those staged artifacts are stale (or missing), peers running your `pear://…/dev` link see stale UI, missing styles, or — worst case — a dispatch router that doesn't recognise the ops the worker is appending.

**Rule of thumb**

- Changed `ui/*.jsx` or `input.css` → `npm run build` before `pear stage dev`.
- Changed `schema.js` → `npm run build:db` (regenerates `spec/`). The default `npm run build` (`package.json:37`) does **not** run `build:db` — `clean` and `build:db` are separate scripts at `:33-34`.
- Changed `index.js`, `worker/**`, `lib/**`, or anything else outside the ignore list → no rebuild needed; those files are staged from disk as-is.

**Practical habit for this study session**

```shell
npm run build && pear stage dev
```

One line, always fresh, build is fast on incremental runs. If `schema.js` was touched too:

```shell
npm run build:db && npm run build && pear stage dev
```

**Symptom-to-cause mapping** (so a future Storyteller knows what's happening if a stage looks wrong)

- Peer sees old UI text after you changed `ui/root.jsx` → forgot `npm run build`. The JSX wasn't recompiled into `build/`, so the old `.js` was staged.
- Peer sees unstyled HTML → forgot `npm run build` (or the Tailwind half of it). `output.css` is stale.
- Worker crashes with `Unknown route` or `Unknown collection type: N` after a schema edit → forgot `npm run build:db`. The router on disk doesn't know about the new op.

## Q: Status of the stage/seed exercise — what got done, what's deferred

**Outcome on 2026-05-15.** The hands-on `pear stage`/`pear seed` walkthrough did **not** complete in this folder. Both `pear stage basic-photo-backup` and `pear stage dev` returned `✖ A valid pear link must be specified.`, which means this version of Pear requires a project to have a minted `pear://…` link before staging — the README's `pear stage <channel>` shorthand doesn't bootstrap that link from nothing.

**Working hypothesis (not verified in this folder).** The `pear` block in `package.json:6-31` is missing a `name` or project-key field that the staging command expects. The likely fix is one of:

- `pear init` to mint a project key and patch `package.json` automatically, or
- adding `"name": "basic-photo-backup"` (or the project's chosen identifier) inside the `pear` block and retrying.

Neither path was tested this session.

**Deferral.** Continue the stage/seed exercise in a later session — likely in whichever folder is the active study target at the time, since the question is Pear-version-wide and not folder-specific. The conceptual walkthrough and command-by-command observations above stand; only the "actually run it end-to-end" half is outstanding.

**Local-dev path still works.** All study and feature work in this folder used the `pear run … . --name user1` form from `README.md:7-19`, which doesn't require staging at all. Storyteller can continue to study and iterate without ever resolving the stage path.

