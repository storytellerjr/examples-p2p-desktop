# basic-chat-identity — Study Notes

Notes for Storyteller while studying this example.

---

## Q: High-level overview — what does this example add, and how does it fit together?

### One-line summary

A P2P chat where every message is signed by a **stable user identity** (24-word mnemonic) via a **per-device key**, so receivers can cryptographically tell who actually sent each message — even though the autobase writer keys are device-scoped and rotateable.

### What's new vs the `basic-chat` / `basic-chat-blind-peering` skeleton

The Pear/Bare/Autobase/HyperDB/HRPC plumbing is the same shape as the earlier examples. The deltas are concentrated in three places:

1. **New dep** — `keet-identity-key` (`package.json:53`).
2. **`message` schema gains a `proof: buffer` field** — `schema.js:33`. Persisted alongside `id`, `text`, `info`.
3. **Worker bootstraps an identity + device key + device proof** — `worker/worker-task.js:34-37`:
   ```js
   this.identity      = await Identity.from({ mnemonic: this.mnemonic })
   this.deviceKeyPair = crypto.keyPair()
   this.deviceProof   = await this.identity.bootstrap(this.deviceKeyPair.publicKey)
   ```

Also:
- New CLI flag `--mnemonic|-m` (`worker/index.js:14`).
- Mnemonic is generated on first run and persisted to `<storage>/identity-mnemonic.txt` so the identity is stable across launches (`worker/index.js:30-37`).
- Blind-**peering** (mirror) is **gone** vs the previous folder — no `blind-peering` dep, no `--blind-peer-key`. This folder is about identity, not availability, so it goes back to direct-only replication.

### The three keys (this is the conceptual core)

There are three distinct keys at play, each with a different lifetime:

- **Identity key** — derived deterministically from the 24-word mnemonic. This is the recognizable "you." It **never signs messages directly**; it only signs *device certificates*. Stable across devices and reinstalls — same mnemonic, same key, forever.
- **Device key pair** — `crypto.keyPair()` generated fresh per process / install. This is what actually signs each message. Not stable: every device gets its own, and a reinstall on the same device gets a new one.
- **Device proof** — produced by `identity.bootstrap(devicePub)`. An identity-signed certificate that says "this device public key speaks for me." Stable until you choose to revoke it.

Why three layers instead of just signing with the identity key directly? So you can sign from multiple devices, revoke a lost laptop, and recover from a device compromise — all without rotating the public-facing identity that other people recognize you by. Same model Keet uses.

### Per-message flow

Send (`worker-task.js:42-45`):
```js
this.rpc.onAddMessage(async (data) => {
  const proof = Identity.attestData(Buffer.from(data), this.deviceKeyPair, this.deviceProof)
  await this.room.addMessage(data, proof, { name: this.name, at: Date.now() })
})
```
`text` + `proof` are appended to the autobase. The proof bundles: device sig over text + identity-signed device cert.

Receive / re-render (`worker-task.js:55-65`):
```js
const res = Identity.verify(msg.proof, Buffer.from(msg.text), {
  expectedIdentity: this.identity.identityPublicKey
})
msg.info.verified = !!res
```

### Architecture map (everything else)

```
index.js (renderer)
  └─ pear-bridge + pear-electron Runtime
        └─ spawns Bare worker (worker/index.js)
              ├─ paparam: parses --invite / --name / --mnemonic / --reset
              ├─ persists mnemonic to <storage>/identity-mnemonic.txt
              └─ WorkerTask (worker/worker-task.js)
                    ├─ Corestore @ <storage>/corestore
                    ├─ Hyperswarm  (replicates every conn into the store)
                    ├─ Identity (mnemonic) + device keypair + device proof
                    ├─ ChatRoomIdentity (worker/chat-room-identity.js)
                    │     ├─ BlindPairing  ← *invite pairing*, NOT blind-peering
                    │     ├─ Autobase  (encrypted, writer-set managed via dispatch)
                    │     │     └─ HyperDB view: invites, messages
                    │     └─ Router: add-writer / add-invite / add-message
                    └─ HRPC over FramedStream to UI
                          ├─ rpc.onAddMessage(text)        ← UI → worker
                          └─ rpc.messages(messages[])      ← worker → UI (debounced on autobase 'update')
```

UI side: `ui/root.jsx` (React 19) + `lib/use-worker.js` hook around HRPC, built via `swc` + `tailwind` into `build/` (`package.json:29-35`).

### Pairing vs peering (easy to confuse)

- **`blind-pairing`** (used here, `chat-room-identity.js:19`) — the **invite handshake** that adds a new writer to the autobase. Owner: `addMember`/`onadd`. Joiner: `addCandidate`/`onadd`. Same as in `basic-chat`.
- **`blind-peering`** (NOT used here) — the **third-party mirror** for offline availability from the previous folder.

Same "blind" prefix, totally different libraries. This folder uses the first, drops the second.

### Notable quirk in the verification

`_messages()` verifies with `expectedIdentity: this.identity.identityPublicKey` (`worker-task.js:60-61`) — i.e. **the local user's own identity**. So `info.verified` only flips true for messages *you* wrote. It's a smoke-test of the signing pipeline, not a realistic sender check. A real app would verify against the *claimed sender's* identity public key (e.g. derived from the autobase writer-key → identity mapping the message implicitly carries via `proof`).

Worth keeping in mind when extending this example.

---

## Q: Why would you use identity + proof? Why does it matter?

### What plain `basic-chat` actually gives you (and doesn't)

In `basic-chat` / `basic-chat-blind-peering`, "who sent this" is two unrelated things glued together:

- **Autobase writer key** — a random per-device hypercore key. The autobase knows "writer X appended this block," but X is just bytes. Not a person.
- **`info.name`** — a free-form string the writer types. Unsigned. Anyone added as a writer can claim any name on any message.

Concretely: in `basic-chat-blind-peering`, once user2 is paired in via invite, nothing stops them appending `{ name: 'user1', text: 'send me your wallet seed' }`. The autobase happily replicates it. user1's UI happily renders it. There is **no cryptographic link between a message and a person**.

There's also no notion of "same person across devices." Alice on her laptop and Alice on her phone are two different writers — the system can't tell they're one human, and if Alice reinstalls she becomes a third stranger to her own room.

### What identity + device proof actually adds

Every message now carries a `proof` field that binds **the message text** to **a stable identity public key**, via a device the identity has authorized:

```
proof = identity_sig(device_pub)   ← the device cert (= deviceProof, issued once)
      + device_sig(text)           ← per-message signature
```

Anyone holding the message + the sender's identity public key can verify both halves. That gives you four things plain chat can't:

1. **Unforgeable authorship.** A malicious co-writer can sign with *their* identity, but they can't produce a proof that verifies against *Alice's* identity public key. Display-name impersonation becomes detectable, not just trust-me-bro.

2. **Stable persona across devices and reinstalls.** Identity is mnemonic-derived (`Identity.from({ mnemonic })`, `worker-task.js:35`). Same 24 words → same identity public key, forever. Laptop, phone, fresh install — all the same "Alice" to everyone else, even though each has its own writer key.

3. **Multi-device without sharing the master key.** The mnemonic stays on whichever device(s) you choose. Each device generates its own ephemeral keypair (`crypto.keyPair()`, `worker-task.js:36`) and gets a device proof from the identity. A compromised device leaks its own signing key, not the identity itself. Revoke the device, identity intact.

4. **No central authority.** Verification needs only the identity public key and the message bytes — no server, no PKI, no Holepunch. Matches the "no servers" property of the rest of the stack.

### Why this matters *especially* in P2P

In a centralized app the server vouches: it checks your password at login and stamps your user-id on every message. P2P has no such vouching layer, so without explicit cryptographic identity:

- **The "writer" set is the trust set.** Invite someone in → they can speak as anyone. There's no narrower permission than "writer."
- **Untrusted relays poison the well.** The previous folder's blind-peer mirrors encrypted bytes — fine for confidentiality, but a malicious mirror (or a buggy one, or one that gets compromised) could in principle replay/inject blocks. With identity proofs, "who delivered this" is independent of "who said this." The proof field either verifies or it doesn't, regardless of how the bytes got to you.
- **History sticks around.** Autobase blocks are permanent. If authorship is just a display-name string, you can never go back and prove which messages were really yours.

### Concrete attacks this blocks

Four scenarios, each shown as: *the attack* → *what plain `basic-chat` does* → *what identity does*.

- **Co-writer impersonates Alice via `info.name`.** A user who got paired into the room appends `{ name: 'alice', text: '…' }`.
  - Plain `basic-chat`: works — `info.name` is just a string, no one can tell.
  - With identity: the proof would have to verify against Alice's identity public key, and only Alice's authorized devices can produce one. Forgery rejected at verify time.

- **Compromised or malicious mirror injects a fake message.** Especially relevant once you add blind-peering — the mirror has the bytes.
  - Plain `basic-chat`: works once the mirror is in the swarm; receivers have no way to distinguish injected from authentic.
  - With identity: the injected block has no valid proof for any expected sender → `verified` flag stays false, UI hides or flags it.

- **Alice's old device leaks and attacker keeps posting as her.**
  - Plain `basic-chat`: indistinguishable from real Alice, because authorship is just "this writer key appended a block."
  - With identity: revoke that specific device's proof. Messages from her other (still-authorized) devices keep verifying; messages from the leaked device stop. The identity itself never has to rotate.

- **Alice reinstalls and wants her old persona back.**
  - Plain `basic-chat`: new install = new writer key = a stranger to the room. No way to reclaim "I am Alice."
  - With identity: same 24-word mnemonic regenerates the same identity public key. Other peers see continuity automatically; her new device just needs a fresh device proof.

### When you'd reach for this

- **Chat / docs / anything where impersonation has real consequences** (financial, social, trust-graph).
- **Apps using untrusted storage/relay** (blind-peering, future mobile sync, IPFS-like patterns).
- **Multi-device personas** — the entire reason Keet uses this exact `keet-identity-key` lib.
- **Apps where users may reinstall or migrate** and need a persistent public face.

### When you might skip it

- Tiny rooms with manually-vouched writers, single-device, ephemeral content.
- Apps where you already have a stronger out-of-band identity (a wallet signature, an org SSO token, a Keet identity from a parent app you can pass through).
- Throwaway pads / pastebin-style sharing where authorship genuinely doesn't matter.

### TL;DR

`basic-chat` answers "who has write access?" Identity answers "who *wrote this specific message*?" Those are different questions, and in P2P only the second one survives without a server.

---

## Q: Applied — how would identity work in a "PearSchool" app (instructor + paying students + course material)?

PearSchool is the perfect stress test for this feature, because it has **three distinct trust questions** that plain writer-key access can't answer separately, but identity can:

1. *Did this material really come from the instructor?* (authenticity)
2. *Is this person actually an enrolled, paid student?* (authorization)
3. *Can a third party (employer, another school) trust the certificate this student is showing?* (portable credentials)

All three collapse into "are you on the writer list?" without identity. Identity splits them apart cleanly.

### Actors and their identity setup

Each actor has their own mnemonic-derived identity, with one or more device key pairs bootstrapped from it.

- **Instructor.** One identity, published once and treated like a public business card (printed on the school site, in the app's "About" pane, on a poster). Used from laptop, phone, conference machine — each device gets its own device proof from the same identity, so the instructor signs from anywhere without sharing the mnemonic.
- **TA / co-instructor.** Their own separate identity, *delegated* by the instructor via a signed delegation proof (see below). Same multi-device story.
- **Student.** Own identity, generated on signup (mnemonic shown once, stored in app storage). Used from laptop, phone, tablet — same identity across all of them.
- **Registrar / payments backend** (optional, only if payment is automated). Own identity, treated as the "enrollment authority." Runs wherever you host it — could even be the instructor's own laptop in a small school.

The instructor's identity public key is the **trust anchor** for the whole school — much like a CA root, but mnemonic-derived and self-managed. Pin it in the app once; verify everything against it.

### Where identity is essential (feature by feature)

**1. Course material authenticity.** Every video drop, PDF, slide-deck reference posted to the course autobase is signed by the instructor's device, attested by the instructor's identity:
```js
const proof = Identity.attestData(materialBytes, instructorDeviceKeyPair, instructorDeviceProof)
await room.append(encode('add-material', { id, blobRef, proof, info }))
```
Students verify with `expectedIdentity: INSTRUCTOR_IDENTITY_PUBKEY` (pinned). If a co-writer or compromised TA tries to slip in a fake "Final Exam Answers.pdf," the proof won't verify against the instructor's identity — UI marks it untrusted.

This is exactly the scenario the example example mishandles: PearSchool must verify against the *known sender's* identity (instructor for material, student for submissions), **not** the local user's identity. See "Notable quirk" above — that's the bit you replace.

**2. Pay-gated enrollment as a signed credential.** Today the example pairs anyone holding the invite. PearSchool can't do that — the invite is worth real money. The fix:

- After payment, the registrar (or instructor) issues an **enrollment credential**:
  ```
  cred = Identity.attestData(
    encode({ studentIdentityPub, courseId, paidAt, expiresAt }),
    registrarDeviceKeyPair,
    registrarDeviceProof
  )
  ```
- The student presents that cred *plus* a fresh signature with their own identity (proving "I'm the same identity the cred names") during the `blind-pairing` handshake. Today's flow is `chat-room-identity.js:75-85` — the `onadd` callback just calls `addWriter` blindly. PearSchool's version checks the cred and the student's matching signature first; rejects otherwise.

This turns the room from "anyone with an invite is a writer" into "only paid, identity-proven enrollees are writers" — without a server.

**3. Submission integrity.** Student assignment posts mirror the same pattern as course material, but signed by the student's identity. The instructor (when grading) verifies with `expectedIdentity: thatStudentsIdentityPub` (looked up from the enrollment table the autobase already holds). Now you can prove which student submitted what, even if their writer key changed (new device) between submissions.

**4. Portable, verifiable certificates of completion.** This is the killer feature for an education app:
```js
const cert = Identity.attestData(
  encode({ studentIdentityPub, courseId, grade, completedAt }),
  instructorDeviceKeyPair,
  instructorDeviceProof
)
```
The student exports `{cert, courseInfo}` as a file or QR code. **Any third party** — an employer, another school — verifies it offline with only the instructor's identity public key (which they look up on the school's site). No PearSchool server needs to be online. No revocation lookup needed for the basic case. This is a real, self-sovereign credential, structurally similar to a verifiable credential / signed JWT — but P2P-native.

**5. TA delegation without giving away the keys.** The instructor signs a delegation proof:
```
delegation = Identity.attestData(
  encode({ taIdentityPub, scope: ['grade', 'reply'], expires }),
  instructorDeviceKeyPair,
  instructorDeviceProof
)
```
TA grades a submission → grade post is signed by TA's identity *and* carries the delegation proof. Students verify two-deep: TA signed this grade, instructor authorized this TA. Compromise the TA's laptop and the instructor revokes the delegation — instructor's own signatures stay valid.

**6. Multi-device students and instructors.** Out of the box. A student starts a quiz on the laptop, finishes on the phone — both devices sign with the same identity, so to the autobase it's continuous authorship. No "you're on a new device, please re-enroll." The mnemonic *is* their student account.

**7. Reinstall / new semester continuity.** Same mnemonic → same identity → alumni records / completed-course history all still attribute correctly. The school's "graduates registry" is a list of identity public keys, not ephemeral writer keys.

### Why you'd combine this with blind-peering (the previous folder)

PearSchool absolutely needs the blind-peering layer too — students must be able to download lectures when the instructor is offline. But that introduces an untrusted mirror:

- **Blind-peering alone:** mirror has the encrypted bytes, can't read them, but a malicious or buggy mirror could in principle inject blocks the room would render.
- **Identity alone:** content authenticity is solid, but availability is hostage to the instructor being online.
- **Both:** mirror provides 24/7 availability of encrypted content; identity proofs let students reject anything the mirror tries to inject. They're orthogonal layers and PearSchool wants both.

This is why these two examples sit next to each other in the repo — each is half of what a real app needs.

### Concrete code changes vs the example

If you forked `basic-chat-identity` to start PearSchool, the must-change list is short and pointed:

1. **`schema.js`** — add `material`, `submission`, `enrollment`, `certificate`, `delegation` collections, all with a `proof` field.
2. **`chat-room-identity.js:75-85`** — replace the unconditional `addWriter` in `onadd` with a check: parse `request.userData` as `{studentIdentityPub, enrollmentCred}`, verify the cred against the registrar's identity, verify the student's signature on the pairing nonce, only then `addWriter`.
3. **`worker-task.js:58-63`** — replace `expectedIdentity: this.identity.identityPublicKey` (own identity) with a per-record lookup: instructor's pinned key for material, the student's enrolled key for submissions, etc.
4. **Pin the instructor's identity public key** at app build time (e.g., a constant in `worker-task.js` or fetched from a known pear:// link on first run). Students need an out-of-band way to know it's the real one — exactly the same trust-on-first-use problem SSH solves with host keys.
5. **Add `blind-peering`** from the previous folder for the course content autobase (and probably *not* for the private student↔instructor channel).

### TL;DR

For PearSchool, identity isn't a nice-to-have — it's load-bearing for **three** different jobs that plain `basic-chat` collapses into one:

- **Material** → instructor signs, students verify against pinned instructor key. *Stops impersonation of the instructor.*
- **Enrollment** → registrar signs a cred for the student's identity, used during pairing. *Turns paying-customer into a cryptographic, P2P-verifiable status.*
- **Certificates** → instructor signs `{student, course, grade}`. *Anyone, anywhere, anytime can verify it with just the instructor's public key — that's the whole "Keet identity" pitch made concrete.*

Without identity, PearSchool would have to put a server in the middle to vouch for any of these. With it, the school is just "an instructor's identity public key + an autobase + some students."

---

## Q: I ran the README's `pear run --store /tmp/user1 . --name user1` and got `Uncaught Error: Unknown collection type: 18`. What's going on?

### What the error means

```
Uncaught Error: Unknown collection type: 18
    at Object.collection1_reconstruct [as reconstruct] (pear://dev/spec/db/index.js:95:25)
    at BeeEngine.finalize (pear://dev/node_modules/hyperdb/lib/engine/bee.js:171:38)
```

HyperDB opened the on-disk view, started reading records, hit a record stamped with collection-type ID `18`, and that ID isn't registered in this folder's generated `spec/db`. So it aborts.

### Why it happened

The `/tmp/user1`, `/tmp/user2`, `/tmp/user3` paths the README suggests are the *exact same paths* the README of the previous folders (`basic-chat`, `basic-chat-blind-peering`) suggests. So if you've been working through the examples in order — which is the whole point of this study repo — those store directories already exist on your machine, populated by an earlier example's app.

Each example has a different schema:

- Different namespace: `basic-chat-identity` here (`schema.js:11-12`) vs `basic-chat-blind-peering` next door.
- Different set of dispatch entries with different offsets (`schema.js:58-63`).
- Different collection IDs assigned by `hyperdispatch` at build time.

That metadata is **baked into every block** the autobase writes. When this folder's HyperDB opens a corestore that was written by a different folder's schema, the bytes are still there but the collection-ID table doesn't match. It reads a record tagged `18`, looks it up, and crashes.

### Two ways to fix

**Option A — reset the existing store.** The flag is handled at `worker/index.js:25-27`:
```sh
pear run --store /tmp/user1 . --name user1 --reset
```
Caveat: `--reset` wipes `<storage>/corestore`, which also takes the persisted mnemonic file with it. You'll come up with a fresh identity. Fine for a clean slate, not what you want if you're testing identity continuity.

**Option B — folder-scoped store paths (recommended for the study repo).** Sidestep the collision entirely:
```sh
pear run --store /tmp/identity-user1 . --name user1
pear run --store /tmp/identity-user2 . --name user2 --invite <invite>
```
Each example gets its own namespace on disk. You can hop between folders without `--reset` dances and without losing identities. The README has been updated to use this pattern.

### Takeaway

The `pear run --store ...` path is **not** just a working directory — it's the corestore root, and the schema that wrote those blocks travels with the bytes. Treat one store as bound to one app/schema.

---

## Q: After switching to a fresh `--store /tmp/identity-user1`, I now get `ENOENT: no such file or directory, open "/tmp/identity-user1/identity-mnemonic.txt"`. Why?

### What the error means

```
Uncaught FileError: ENOENT: no such file or directory,
  open "/tmp/identity-user1/identity-mnemonic.txt"
    at async runWorker (pear://dev/worker/index.js:37:3)
```

The crash is on the `fs.promises.writeFile(mnemonicPath, mnemonic)` call in `worker/index.js`. ENOENT on a `writeFile` doesn't mean the *file* is missing (writeFile creates files) — it means the *parent directory* is missing. So `/tmp/identity-user1/` itself doesn't exist yet.

### Why the upstream example works on `/tmp/user1` but not on a fresh path

Look at the order of operations in `worker/index.js`:

```js
const storage = path.join(Pear.app.storage, 'corestore')   // 1. compute paths
…
await fs.promises.readFile(mnemonicPath, …)                // 2. try to read mnemonic
await fs.promises.writeFile(mnemonicPath, mnemonic)        // 3. write mnemonic ← crashes here
…
await workerTask.ready()                                   // 4. Corestore opens; *now* it
                                                           //    creates Pear.app.storage
```

The mnemonic file is written **before** anything has touched `Pear.app.storage` on disk. Corestore is what would normally create that directory, but it doesn't run until `workerTask.ready()` later. So on a brand-new store path, the parent dir doesn't exist when `writeFile` runs.

We didn't hit this on `/tmp/user1` earlier because that directory was already on disk — left over from previous runs of `basic-chat` / `basic-chat-blind-peering`. Folder-scoped paths like `/tmp/identity-user1` start clean, so the latent bug surfaces.

This is a real bug in the upstream pearopen example, not something we introduced by changing the path.

### The fix (applied in `worker/index.js:37-40`)

One line of `mkdir -p` semantics before the write, plus a comment explaining why it's needed:

```js
// Pear.app.storage is not guaranteed to exist on disk yet: Corestore creates
// it as a side effect, but that runs later in WorkerTask._open(). On a fresh
// --store path the writeFile below would otherwise ENOENT.
await fs.promises.mkdir(Pear.app.storage, { recursive: true })
await fs.promises.writeFile(mnemonicPath, mnemonic)
```

### Lesson worth keeping

- `Pear.app.storage` is a **logical path** — Pear sets the value, but it doesn't materialise the directory on disk until something (typically Corestore) writes into it.
- Any code that touches `Pear.app.storage` directly *before* Corestore opens (mnemonic files, config, logs, any persistent metadata) needs to either `mkdir` first or be moved to run after `WorkerTask.ready()`.
- The general pattern: side-effects in initialization order matter. "It worked on the first try" can mean "the side-effect from a previous run papered over a missing call." Folder-scoped, clean store paths are also useful for *finding* these latent bugs.

---
