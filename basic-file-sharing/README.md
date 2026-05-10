# Basic File Sharing

- P2P basic file sharing app running in desktop environment

- Stack: holepunch, bare, pear, corestore, hyperswarm, blind-pairing, hyperdb, hrpc, autobase, hyperdrive, localdrive

## Usage

> **Note:** `--store` paths are folder-scoped (`/tmp/file-sharing-user1`, …) so this app's corestore doesn't collide with other example folders. Sharing a bare `/tmp/user1` path across folders crashes the next app with `Uncaught Error: Unknown collection type: N`, because each example registers its own HyperDB collection IDs.

```shell
npm i
npm run build

# user1: create room + print invite
pear run --store /tmp/file-sharing-user1 . --name user1

# user2: join room
pear run --store /tmp/file-sharing-user2 . --name user2 --invite <invite>
```

## Build Pear app
```shell
npm i
npm run build

pear stage <channel>
pear seed <channel>

# user1: create room + print invite
pear run --store /tmp/file-sharing-user1 <pear-link> --name user1

# user2: join room
pear run --store /tmp/file-sharing-user2 <pear-link> --name user2 --invite <invite>
```

## Troubleshoot
Use `--reset` to reset everything, e.g.
```shell
pear run --store /tmp/file-sharing-user1 . --name user1 --reset
```
