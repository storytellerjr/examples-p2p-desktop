# Basic Video Stream

- P2P basic video stream app running in desktop environment

- Stack: holepunch, bare, pear, corestore, hyperswarm, blind-pairing, hyperdb, hrpc, autobase, hyperblobs, hypercore-blob-server

## Usage

> **Why `/tmp/video-stream-user*` instead of `/tmp/user*`?** Each example in this study repo has its own schema (different namespace, collection IDs, and hyperdispatch offsets — see `schema.js`), and that metadata is baked into every block the autobase writes. Reusing a store path across folders makes this app try to read blocks written under a different schema and crash with `Uncaught Error: Unknown collection type: N`. Folder-scoped paths keep each example's on-disk data separate so you can hop between exercises without `--reset` dances.

```shell
npm i
npm run build

# user1: create room + print invite
pear run --store /tmp/video-stream-user1 . --name user1

# user2: join room
pear run --store /tmp/video-stream-user2 . --name user2 --invite <invite>
```

## Build Pear app
```shell
npm i
npm run build

pear stage <channel>
pear seed <channel>

# user1: create room + print invite
pear run --store /tmp/video-stream-user1 <pear-link> --name user1

# user2: join room
pear run --store /tmp/video-stream-user2 <pear-link> --name user2 --invite <invite>
```

## Troubleshoot
Use `--reset` to reset everything, e.g.
```shell
pear run --store /tmp/video-stream-user1 . --name user1 --reset
```
