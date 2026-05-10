# Project context

This repo is a **study project** by **Storyteller** of the **pearopen** repositories — Holepunch / Pear / Bare example apps. The goal is to learn the stack by reading and annotating the examples.

Storyteller is a pseudonym (the user's nym in the Pear Baby Room on Keet, and the handle on the GitHub account `storytellerjr` where these notes will be pushed). **Never write the user's real name in any file in this repo or in any file Claude creates** — including notes, commits, READMEs, code comments, or memory. Use "Storyteller" everywhere.

## How to take notes

When Storyteller asks questions about an example folder (e.g. `basic-chat-blind-peering/`, `basic-chat/`, `basic-file-sharing/`, etc.), append the Q&A to a `notes.md` **inside that folder**. Create the file if it doesn't exist. Each note should be self-contained so the folder can be read on its own later.

- One `notes.md` per example folder being studied.
- Use `## Q:` headings for questions and put answers underneath.
- Reference code with `path:line` so notes stay anchored to the source.
- Keep cross-folder comparisons in the folder being studied (not in a sibling folder), unless Storyteller asks for a top-level summary.
- **Do not use Markdown tables.** Storyteller's viewer renders them unreadable for any non-trivial content. Always use bulleted lists or sub-headings instead, even when the content seems naturally tabular (multi-column comparisons, attack/effect pairs, actor/role rows, etc.). If you would have written a table, write one bullet per row with bolded key terms inline.

## README conventions for example folders

Each example folder is its own Pear app with its own schema namespace, hyperdb collection IDs, and hyperdispatch offsets baked into every autobase block it writes. Reusing a store path across folders therefore crashes the next app with `Uncaught Error: Unknown collection type: N` (HyperDB tries to reconstruct a record whose collection ID isn't registered in this folder's `spec/db`).

When writing or editing a folder's `README.md`:

- **Use folder-scoped `--store` paths** in every `pear run` example. Pattern: `/tmp/<folder-shortname>-user1`, `/tmp/<folder-shortname>-user2`, etc. (e.g. `/tmp/identity-user1`, `/tmp/multi-rooms-user1`). **Never** use the bare `/tmp/user1`, `/tmp/user2` the upstream pearopen READMEs default to — those collide across folders.
- **Apply the rule consistently across every code block** in the README: Usage, Build Pear app, Troubleshoot, and any other section that shows a `--store` flag.
- **Include a brief explanatory note** above the first `pear run` block explaining why the path is folder-scoped (link the failure mode: `Unknown collection type: N`). `basic-chat-identity/README.md:9` and `basic-chat-multi-rooms/README.md` are the reference templates.

This keeps every example's on-disk data isolated so Storyteller can hop between folders without `--reset` dances and without losing any folder's existing state.

## Folder-scoped improvement ideas (`new-features.md`)

When Storyteller (or you, while studying) spots a concrete way the code in an example folder could be improved or extended, capture it in a `new-features.md` **inside that folder**. This is the forward-looking counterpart to `notes.md` — keep them separate so study Q&A doesn't get tangled with speculative roadmap items.

- One `new-features.md` per example folder. Create on first idea; append thereafter.
- Each entry should state **what** is sub-optimal or missing, **why** it matters (concrete failure mode or UX gap, not vague "best practice"), and a **rough how** sketch. Anchor every claim to code with `path:line`.
- Keep ideas **folder-scoped** — they describe changes to *this* example, not cross-folder refactors. Cross-folder ideas belong in a top-level summary if Storyteller asks for one.
- Same formatting rules as `notes.md`: bulleted lists or sub-headings, no Markdown tables.
- `basic-chat-multi-rooms/new-features.md` is the reference template.
