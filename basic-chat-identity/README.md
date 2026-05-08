# Basic Chat Identity

- P2P basic chat app with identity running in desktop environment

- Stack: holepunch, bare, pear, corestore, hyperswarm, blind-pairing, hyperdb, hrpc, autobase

## Usage

> **Why `/tmp/identity-user*` instead of `/tmp/user*`?** The previous examples in this study repo (`basic-chat`, `basic-chat-blind-peering`) also tell you to use `--store /tmp/user1`, `/tmp/user2`, etc. Each example has its own schema (different namespace, collection IDs, and hyperdispatch offsets — see `schema.js:11-12, 58-63`), and that schema metadata is baked into every block the autobase writes. So if you reuse a store path across folders, this app tries to read blocks written under a different schema and crashes with `Uncaught Error: Unknown collection type: N`. Folder-scoped paths keep each example's on-disk data separate so you can hop between exercises without `--reset` dances.

```shell
npm i
npm run build

# user1: create room + print invite
pear run --store /tmp/identity-user1 . --name user1

# user2: join room
pear run --store /tmp/identity-user2 . --name user2 --invite <invite>
```

## Build Pear app
```shell
npm i
npm run build

pear stage <channel>
pear seed <channel>

# user1: create room + print invite
pear run --store /tmp/identity-user1 <pear-link> --name user1

# user2: join room
pear run --store /tmp/identity-user2 <pear-link> --name user2 --invite <invite>
```

## Troubleshoot
Use `--reset` to reset everything, e.g.
```shell
pear run --store /tmp/identity-user1 . --name user1 --reset
```
