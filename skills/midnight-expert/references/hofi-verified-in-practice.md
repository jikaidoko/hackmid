# Midnight — verified-in-practice findings (HoFi private voting)

> **What this file is.** The empirically-confirmed, hard-won Midnight knowledge from
> building a real ZK app (HoFi private voting) end-to-end: Compact circuits, the
> local simulator, the Wallet SDK, self-custodial submit via Lace, the local
> standalone network, and in-browser WASM proving. It is the **authoritative
> supplement** to the sibling Midnight skills (`midnight-compact`, `midnight-api`,
> `midnight-dapp-dev`, `midnight-wallet`, `midnight-network`): where a general skill
> describes the happy path, this file pins the **traps that actually cost days** and,
> at the bottom, an **[Audit: corrections to sibling skills](#audit-corrections-to-sibling-skills)**
> section listing where those skills are wrong or incomplete against what we verified.
> Version-specific — re-verify after `compact update` / SDK bumps.

Midnight is a separate paradigm from Cardano L1: its own language (**Compact**), its
own execution model (**ZK circuits + private witnesses**, not yes/no validators), and
its own off-chain stack (**TypeScript / Midnight.js**). Do not carry eUTXO instincts
in — re-read the mental-model section first.

> Status (mid-2026): mainnet launched **late March 2026**. Public **testnet**
> (preprod) + faucet are live for development. The **NIGHT** token shipped as a
> Cardano Native Asset (24 B fixed supply, distributed via the *Glacier Drop* /
> *Scavenger Mine*) before the native chain went live. Treat exact figures and
> compiler versions as fast-moving — verify against `docs.midnight.network`.
>
> The **"Verified in practice"** section below pins concrete, empirically-confirmed
> behavior against **compactc 0.31.0 · language 0.23.0 · runtime 0.16.0** (the
> toolchain used to build HoFi private voting). Re-verify after `compact update`.

## What Midnight is, in one screen

- A **partner chain** of Cardano (not a sidechain, not an L2). Independent network,
  own economics, secured with help from Cardano **SPOs** who can produce Midnight
  blocks and earn NIGHT without touching their ADA stake-pool operation.
- Goal: **"rational / programmable privacy"** — *validity without visibility*.
  Prove a computation followed the rules without revealing the inputs.
- **Dual-ledger architecture**:
  - **Public (unshielded) ledger** — visible state, ordering, coordination. This
    is the contract's on-chain `ledger` state.
  - **Shielded (private) ledger** — confidential state and value (Zswap-style
    shielded tokens), proven correct via zero-knowledge.
- **Selective disclosure**: reveal a *fact* (balance ≥ threshold, KYC passed,
  reputation ≥ N) without revealing the underlying data.
- **Two tokens**:
  - **NIGHT** — public, fixed-supply (24 B) governance/utility token. Not burned
    by usage. Held, transferred publicly (NIGHT txs are public).
  - **DUST** — shielded, **non-transferable, renewable** resource that *pays for
    transactions*. Holding NIGHT continuously generates DUST up to a cap tied to
    your NIGHT balance; it regenerates over time. This decouples gas cost from
    token price (predictable fees — an enterprise selling point). On testnet:
    `tNIGHT` → generates `tDUST`.

## The mental-model shift (this is the whole game)

On Cardano L1 a **validator** is a pure `Bool`: "given this tx context, is it
allowed?" The off-chain builder assembles inputs/outputs; the validator only
checks. **Everything is public.**

On Midnight a **circuit** is a function the compiler turns into a **zero-knowledge
proof**. When a user calls a circuit, *their machine* (via the local proof server)
generates a proof that "I ran this circuit correctly against the current public
state, using private inputs I will not reveal." The chain verifies the proof and
applies the declared public-state updates. So:

| Concept | Cardano L1 (Aiken) | Midnight (Compact) |
|---|---|---|
| Unit of logic | `validator` returning `Bool` | `circuit` compiled to a ZK proof |
| State | datums in UTXOs (all public) | public `ledger` ADTs **+** private off-chain state |
| Private data | impossible (chain is transparent) | **`witness` functions** feed private inputs; never on-chain |
| Auth | `extra_signatories` / authority NFT | prove knowledge of a secret via the circuit (e.g. commitment) |
| Off-chain | PyCardano / MeshJS build tx | **Midnight.js** (TS) calls circuits, runs proofs |
| "msg.sender" | none | none — identity is whatever you *prove*, via commitments |
| Cost | ADA fees, min-UTXO | **DUST** (regenerates from NIGHT) |

Default stance: **everything a circuit computes is presumed private.** To write a
value into the public `ledger` you must explicitly wrap it in **`disclose(...)`** —
the compiler *rejects* an implicit leak of witness-derived data into public state.
This "deny by default" disclosure is the single most important safety primitive;
treat an un-`disclose`d compile error as the compiler catching a privacy bug.

## Compact — the language

TypeScript-flavored syntax, but a **restricted, total** language built for ZK:
strongly typed, **all sizes fixed at compile time**, **no recursion**, **only
bounded loops** (compile-time-known bounds). Three building blocks:

- **`ledger`** declarations — the contract's **public on-chain state** (ADTs).
- **`circuit`s** — entry points / helpers, compiled to ZK circuits. `export
  circuit` = callable from TypeScript. A circuit is *impure* if it touches ledger
  state or calls a witness; mark side-effect-free ones `pure circuit`.
- **`witness`** functions — declared in Compact (signature only), **implemented in
  TypeScript**. They read/write **private, off-chain state** and feed secret values
  into circuits for proof generation. The value never appears on-chain or in the
  proof.

### Canonical example — the bulletin board (`bboard`)

```compact
pragma language_version >= 0.23;
import CompactStandardLibrary;

export enum State { VACANT, OCCUPIED }

// Public, on-chain ledger state:
export ledger state: State;
export ledger message: Maybe<Opaque<'string'>>;
export ledger sequence: Counter;          // bumped each cycle -> replay protection
export ledger owner: Bytes<32>;           // commitment to the poster, NOT their key

constructor() {
  state = State.VACANT;
  message = none<Opaque<'string'>>();
  sequence.increment(1);
}

// Private input, implemented in TypeScript, never revealed:
witness localSecretKey(): Bytes<32>;

export circuit post(newMessage: Opaque<'string'>): [] {
  assert(state == State.VACANT, "Board occupied");
  // publicKey() hashes the secret -> a commitment; disclose() = explicit reveal
  owner   = disclose(publicKey(localSecretKey(), sequence as Field as Bytes<32>));
  message = disclose(some<Opaque<'string'>>(newMessage));
  state   = State.OCCUPIED;
}

export circuit takeDown(): Opaque<'string'> {
  assert(state == State.OCCUPIED, "Board empty");
  // Proves "I am the owner" by re-deriving the commitment from the secret,
  // WITHOUT ever revealing the secret on-chain:
  assert(owner == publicKey(localSecretKey(), sequence as Field as Bytes<32>),
         "Not the owner");
  const formerMsg = message.value;
  state = State.VACANT;
  sequence.increment(1);
  message = none<Opaque<'string'>>();
  return formerMsg;
}

export circuit publicKey(sk: Bytes<32>, seq: Bytes<32>): Bytes<32> {
  // domain-separated one-way commitment
  return persistentHash<Vector<3, Bytes<32>>>([pad(32, "bboard:pk:"), seq, sk]);
}
```

Read this like a senior: the **secret never leaves the user's machine**. Ownership
is proven by re-deriving a hash commitment inside a circuit; the chain sees only
the commitment (`owner`) and a ZK proof. `sequence` is a nonce defeating replay.
Every write to public state goes through `disclose(...)` on purpose.

### Type system cheat-sheet

- **Value types**: `Boolean`, `Field` (unsigned int up to the ZK field order),
  `Uint<n>` / `Uint<0..n>` (sized/bounded), `Bytes<n>`, `Vector<n, T>` (homogeneous,
  fixed length), `[T1, T2, ...]` (heterogeneous tuple), `Opaque<'string'>` /
  `Opaque<'Uint8Array'>` (treat-as-blob), `Maybe<T>` (`none()` / `some(v)`).
- **Ledger-state ADTs** (public state only): `Cell<T>`, `Counter`, `Map<K,V>`,
  `Set<T>`, `List<T>`, `MerkleTree<n,T>` / `HistoricMerkleTree<n,T>` (n ≤ 32),
  `Kernel`. Nested ledger types are allowed **only as Map values**.
- **Ops**: `read()`/`write()`/`reset_to_default()` on `Cell` (or `F = v` / bare
  `F`); `increment`/`decrement`/`read` on `Counter` (or `+=` / `-=`);
  `insert`/`lookup`/`remove` on `Map`/`Set`/`List`; Merkle inclusion proofs.
  ⚠️ Two of these bite at *runtime* (not compile) — see **Verified in practice**:
  `Map<K, Counter>` does **not** auto-init on `increment`, and inclusion uses the
  free fn `merkleTreePathRoot`, not a `.checkRoot()` method.
- **Declarations**: `struct Name<T> { f: T; }` (nominal, non-recursive),
  `enum E { a, b }`, `ledger x: T;`, `sealed ledger x: T;` (write-once),
  `witness w(p: T): R;`, `[export] [pure] circuit c<T>(p: T): R { ... }`,
  one optional `constructor(...)`, `export module M<T> { ... }`.
- **Control flow**: `if/else`, `for (const x of vec)`, `for (const i of a..b)`,
  ternary `?:`, `assert(cond, "msg")`. **No recursion, no unbounded loops.**
- `const` bindings are immutable. Pattern-destructure tuples/structs in params.
- Generics are compile-time (`<T>`, numeric `<#N>`), specialized at compile time.

## DApp architecture & toolchain

A Midnight dApp is a layered TS project, not a single contract:

1. **Contract** — `*.compact` compiled with `compact compile` → generates a
   **TypeScript API** + JS impl + the ZK circuit artifacts (keys/IR) under
   `src/managed/<name>/`.
2. **Witnesses / private state** — TS implementing each `witness`, plus a
   private-state provider (stores secrets locally, e.g. in a level/indexeddb).
3. **Midnight.js** (`@midnight-ntwrk/*` packages) — the off-chain SDK. You wire up
   **providers**: ledger/indexer (public state), zk-config (proving keys), **proof
   provider** (talks to the proof server), wallet, and private-state provider. Then
   you call exported circuits; the SDK assembles the tx and obtains the proof.
4. **Proof server** — a **local Docker service** that generates the ZK proofs.
   *Must be running* to deploy or interact. It keeps private inputs on the user's
   machine — proofs are produced locally, only the proof is submitted.
5. **Wallet** — **Lace** (Midnight-enabled) for users; a **headless wallet** for
   CLI/automation/tests (seed-based).

### Setup commands (verify versions against the docs)

```bash
# 1. Install the `compact` toolchain manager (macOS/Linux; Windows = WSL)
curl --proto '=https' --tlsv1.2 -LsSf \
  https://github.com/midnightntwrk/compact/releases/latest/download/compact-installer.sh | sh
source ~/.bashrc                 # or ~/.zshrc; PATH = $HOME/.compact/bin

compact update                   # fetch + default the latest compiler
compact --version                # toolchain version
compact compile --version        # compiler version

# 2. Compile a contract -> TS API + ZK artifacts
compact compile src/bboard.compact src/managed/bboard

# 3. Run the local proof server (Docker Desktop required)
docker run -p 6300:6300 midnightntwrk/proof-server:latest midnight-proof-server -v

# 4. Node 22+, install the JS workspace (contract / api / cli / ui)
npm install

# 5. Testnet: get tNIGHT from the faucet -> it generates tDUST for fees
#    https://faucet.preprod.midnight.network/
```

The toolchain ships a compiler, **formatter**, a **"fixup"** migration tool, and a
**VS Code extension** (syntax highlighting / language support). Mental map to the
Aiken workflow you already know: `compact compile` ≈ `aiken build` (it regenerates
the typed interface the off-chain code binds to, like `plutus.json` does for
PyCardano/MeshJS); the proof server has **no L1 analogue** — budget for it.

## Senior gotchas & guidance

- **Privacy is a *property you design*, not a switch.** What you put in public
  `ledger` is public forever. Leak analysis = "what can an observer infer from the
  sequence of public-state diffs + tx graph?" Commitments, nullifiers, and nonces
  (`sequence` above) are your tools; copy the patterns from the standard library
  and audited examples rather than inventing crypto.
- **`disclose` is load-bearing.** A circuit that won't compile because of a missing
  `disclose` is usually telling you you're about to leak private data. Don't
  blanket-wrap to silence it — understand *why* that value is tainted.
- **Witnesses are trusted off-chain code.** The chain proves the *circuit* ran
  correctly, but a witness returns whatever your TS says. Validate witness outputs
  *inside the circuit* (range checks, commitment checks) — never assume honesty.
- **The proof server is mandatory infra.** "Nothing happens / hangs on deploy" is
  almost always: proof server not running, wrong port (6300), or Docker stopped.
- **No recursion / bounded loops only** — algorithms must be reframed with
  fixed-size data (`Vector<n>`, `MerkleTree<n>`). Same discipline as ZK everywhere.
- **`Field` ≠ a 256-bit int.** It's modular over the proving field; arithmetic
  wraps. Use `Uint<n>` with explicit bounds when you need integer semantics.
- **DUST, not ADA.** Test wallets need tNIGHT first (faucet) to regenerate tDUST;
  a "zero balance / can't pay fees" error often means DUST hasn't regenerated yet.
- **Two ledgers, two providers.** Public reads come from an indexer; private reads
  come from your local private-state provider. Forgetting to persist/restore
  private state across sessions silently breaks "ownership" proofs.
- **Fast-moving.** Compact `language_version`, compiler, and SDK package names
  change between releases. Pin versions; check `docs.midnight.network/relnotes`.

## Verified in practice (compactc 0.31.0 · language 0.23.0 · runtime 0.16.0)

Hard-won, empirically-confirmed facts from building a real circuit (HoFi private
voting). Version-specific — re-verify after `compact update`. The meta-lesson:
**a circuit compiling is NOT proof it's correct.** Several of these are
compile-true but runtime-false; the local simulator (below) is what catches them.

### Test circuits locally — no testnet, no proof server

`@midnight-ntwrk/compact-runtime` runs the generated circuit *logic* (asserts,
ledger state transitions) deterministically, **without** generating a real ZK
proof. This is the fast inner loop — use it before ever touching testnet/DUST:

```ts
import { createConstructorContext, createCircuitContext, dummyContractAddress }
  from '@midnight-ntwrk/compact-runtime';
import { Contract, ledger } from './managed/<name>/contract/index.js';
import { witnesses } from './witnesses.js';

const contract = new Contract(witnesses);
const ctor = createConstructorContext(initialPrivateState, '00'.repeat(32) /* coinPK */);
const init = contract.initialState(ctor, ...ctorArgs);   // -> { currentContractState, ... }

let state = init.currentContractState.data;              // ChargedState
const ctx = createCircuitContext(dummyContractAddress(), '00'.repeat(32), state, privateState);
const res = contract.circuits.myCircuit(ctx, ...args);   // THROWS on a failed assert
state = res.context.currentQueryContext.state;           // thread into the next call
const L = ledger(state);                                 // typed public-ledger view to assert on
```

Run it with `tsx --test` (Node 22; `node --test` can't resolve the `.js`→`.ts`
imports the generated code uses). Each circuit call returns `{ result, context,
proofData, gasCost }`; chain `res.context.currentQueryContext.state` into the next
`createCircuitContext`. `CoinPublicKey` is just a hex string here — a dummy works.

### Merkle inclusion — the actual API (and how to build the tree off-chain)

- `MerkleTreePath<n, T>` is a **struct** `{ leaf: T, path: Vector<n,
  MerkleTreePathEntry> }`; `MerkleTreePathEntry { sibling: MerkleTreeDigest,
  goes_left: Boolean }`; `MerkleTreeDigest { field: Field }`.
- There is **no `.checkRoot()` on the path** (that's a method of the ledger
  `MerkleTree` ADT). When you only store the *root snapshot*, verify with the
  **free function** `merkleTreePathRoot<n, T>(path): MerkleTreeDigest` (it hashes
  the leaf) and compare `==` to a `ledger root: MerkleTreeDigest`. Store the root
  as **`MerkleTreeDigest`, not `Bytes<32>`**.
- **Always bind `path.leaf == yourCommitment`** inside the circuit — else a witness
  can present a valid path for *someone else's* leaf and impersonate them.
- **Build the tree off-chain with the SAME hashing — don't hand-roll it.** Use
  `StateBoundedMerkleTree` (re-exported by compact-runtime; the same impl that backs
  the stdlib ADT). Insert *raw* leaves (`update(i, alignedLeaf)`), `rehash()` once,
  then `root()` / `pathForLeaf(i, leaf)` return `AlignedValue`s. Convert to the
  generated shapes with `CompactTypeMerkleTreeDigest.fromValue(av.value)` and
  `new CompactTypeMerkleTreePath(n, new CompactTypeBytes(32)).fromValue(av.value)`.
  The off-chain root then equals what `merkleTreePathRoot` reconstructs in-circuit —
  confirm it with the simulator (a mismatch silently rejects every eligible voter).

### `MerkleTree.checkRoot()` accepts ONLY the current root — never a historical one  ⚠️⚠️

There is **no `HistoricMerkleTree`** in compactc 0.31.0. `commitments.checkRoot(digest)`
compares against the tree's **current** root, so **any insert invalidates every other
member's path**.

This silently destroys any design where a *cohort* proves against a snapshot taken at the
start of a window (rotating accumulators, batched enrollment, epoch-based membership). The
first member to insert breaks everyone else; the rest must re-read the tree and **re-prove**
(minutes each, client-side). On a real chain that is a thundering-herd race, not a
correctness bug you'd catch by reading.

**The fix — freeze the root.** Store a snapshot digest as its own ledger field and verify by
*pure comparison*, exactly as `castVote` does with `eligibilityRoot`:

```compact
export ledger commitments: MerkleTree<20, Bytes<32>>;  // insert-only
export ledger epochRoot: MerkleTreeDigest;             // frozen at window open

// ✅ pure comparison → N members prove in parallel against the same root
assert(merkleTreePathRoot<20, Bytes<32>>(path) == epochRoot, "not registered");

// ❌ ledger op → only the current root; serializes the cohort
assert(commitments.checkRoot(disclose(merkleTreePathRoot<20, Bytes<32>>(path))), "...");
```

Two corollaries fall out:
- **`checkRoot` is a *ledger operation*** ⇒ the compiler's disclosure analysis demands
  `disclose(...)` around the reconstructed root. The `==` form is pure and needs none. If you
  find yourself wrapping a Merkle root in `disclose` just to satisfy the compiler, that's the
  smell.
- **`tree.root()` is runtime-only — it cannot be called in-circuit.** (`MerkleTree root is a
  runtime-only method, but was invoked in-circuit`.) So the frozen root must enter as a
  **circuit parameter**, read off-chain from `ledger(state).commitments.root()`. The circuit
  therefore **cannot verify it** ⇒ whatever circuit sets it **must be gated by an issuer
  signature**, or an admin can freeze an arbitrary root and admit leaves that were never
  inserted.

Discovered 9-jul-2026 building HoFi's `rotateReputation` (epoch-rotating private reputation
accumulator). The simulator test written to assert "the whole cohort can rotate against the
epoch-start root" is what caught it — the design read fine on paper.

### Widening: `Uint<32> + 1` is not a `Uint<32>`

```
expected right-hand side of = to have type Uint<32> but received Uint<0..4294967297>
```

Arithmetic widens the type. Narrow it back explicitly when assigning to a ledger field:
`epoch = disclose((epoch + 1) as Uint<32>);`. Related to the exclusive-bound gotcha below.

### Measuring a circuit's real proving cost: read `k`, not the constraint count

Proving time is set by **`k`** (log2 of the domain size), which quantizes to a power of two —
not by how many constraints sit inside it. So *adding work is free until it isn't*, and then
it doubles.

```js
import { Zkir } from "@midnight-ntwrk/zkir-v2";   // only getK() is exposed
const k = Zkir.deserialize(new Uint8Array(readFileSync("managed/X/zkir/X.bzkir"))).getK();
```

Measured for HoFi (compactc 0.31.0): `castVote` **k=15** (≈94 s in-browser, main thread),
`rotateReputation` **k=15** too — *despite* carrying two depth-20 Merkle paths, because
`castVote` already carries one of depth **32**. `register` k=14, `closeEpoch` k=7.

⚠️ **k=15 is the local ceiling**: the SRS params in `.zk-params/` stop at
`bls_midnight_2p15`. One more assert can push a circuit to k=16 → **2× proving time** plus a
params file that isn't on disk. **Re-measure as you add asserts, not at the end.**

### The cost is `persistentHash` **blocks**, not asserts and not Merkle depth  ⚠️⚠️

The `k` above tells you *how much* a circuit costs. This tells you *where the cost is* — and it
is almost never where people look.

`persistentHash` is **SHA-256 evaluated inside the circuit**. It is not charged per call: it is
charged per **64-byte compression block**, at roughly **2 048 rows per block**. SHA-256 appends
9 bytes of padding, so a preimage of **64 B costs two blocks** while one of **55 B costs one**.
`transientHash` (Poseidon) is comparatively free.

Measured (a probe contract with ten circuits, one `compact compile`):

| SHA-256 blocks | 0 | 1–2 | 3 | 4–6 | 8–13 | **15** | **16** |
|---|---|---|---|---|---|---|---|
| `k` | 9 | 13 | 13 | 14 | 15 | **15** | **16** |

⇒ **k=15 holds exactly 15 blocks. The 16th falls off the cliff.**

The consequences invert the usual intuition:

- **A `assert` with no hash in it is free.** Adding an anti-replay `assert(expected == epoch)`
  to a circuit cost **zero** blocks and did not move `k`.
- **One hash is not.** Gating a circuit with a single `persistentHash` preimage check took it
  from k=7 to **k=13** (128 → 8 192 rows).
- **A depth-20 Merkle path is nearly free**: its 20 levels are `transientHash`. Two depth-20
  paths (40 Poseidons) do not move `k`; the *three* 38-byte leaf hashes around them cost 1 block
  each.
- ⇒ **The optimization lever is shortening preimages, not deleting asserts.** A `struct` of
  `tag:Bytes<32> + sid + salt + holon + consent + 3 small ints` is 168 B = **3 blocks**. Folding
  the attributes into one 32-byte digest gets it under 119 B = 2 blocks — and if it is hashed
  twice per circuit, that is 2 blocks recovered at once.

Count the blocks straight out of the **textual** `.zkir` (the compiler emits `X.zkir` next to
`X.bzkir`); `persistent_hash` ops carry their preimage `alignment`:

```js
const ir = JSON.parse(readFileSync("managed/X/zkir/X.zkir", "utf8"));
const bytes = (a) => a.reduce((n, x) => n + (x?.value?.length ?? 0), 0);
const blocks = (n) => Math.ceil((n + 9) / 64);          // +1 byte 0x80, +8 byte length
const total = ir.instructions.filter((i) => i.op === "persistent_hash")
                             .reduce((n, i) => n + blocks(bytes(i.alignment)), 0);
```

The op histogram of that same file (`persistent_hash` vs `transient_hash`) is the fastest way to
see why a circuit costs what it costs — far faster than bisecting compiles.

### Preimage arithmetic is not enough: the hash cost depends on the *shape*  ⚠️

Measured while executing the fold above (three variants compiled and measured — byte
arithmetic alone predicted none of this):

- **A `Field` or `Uint` member inside a `persistentHash<Struct>` splits the hash into an
  extra compression.** `{tag, sid, seal: Field, rep: Uint<16>, epoch: Uint<32>}` costs
  3 blocks even though its byte count says 2. Moving the ints into the Poseidon fold and
  leaving `{tag, sid, seal: Bytes<32>}` (96 B) *still* measured 3 as a struct.
- **The shape that actually yields 2 blocks in one call is
  `persistentHash<Vector<3, Bytes<32>>>([tag, sid, seal])`** with the fold cast:
  `transientHash<SealInput>(...) as Bytes<32>`. (`Field as Bytes<32>` is a legal cast.)
- **The 38-byte companion compressions that accompany every Merkle path/insert are the
  ADT's own leaf hash** — a 6-byte domain prefix + the 32-byte leaf, visible as
  `alignment: [{length: 6}, {length: 32}]` in the `.zkir`. One per `merkleTreePathRoot`
  call and one per `insert`. They are part of the tree; shortening *your* preimages
  cannot remove them. (Don't misattribute them to your own hashes when reading the
  histogram — attribution is positional.)
- **Don't fold what is already minimal**: a 70 B struct is 2 blocks; folding it to 96 B
  is still 2 blocks and only churns domain tags.
- `sealed` is a **reserved keyword** in Compact — as a struct member name it's a parse
  error (`found keyword "sealed"`). Use `seal`.

Net effect on HoFi (etapa 4c): commit and enrollment hashes went 3 blocks → 2 each;
`rotateReputation` 13 → 11 blocks (still k=15, margin 2 → 4 blocks); the four credential
circuits (`prove*`: commit + nullifier + one path leaf = 5 blocks) land at **k=14** —
half the rows of the rotation.

### The disclosure analysis also constrains *return values*

An exported circuit that **returns** a value derived from witnesses is a leak, even if it
never writes the ledger:

```
the value returned from exported circuit register might disclose a hash of the witness value
```

`register(): Bytes<32> { ... return commit(subjectId(), ...); }` fails. Return `[]` and let the
caller recompute the commitment locally from its own private state.

### `Map<K, Counter>` does NOT auto-initialize  ⚠️

`m.lookup(k).increment(1)` on an **absent** key *compiles* but **throws at runtime**:
`"expected a cell, received null"`. Create the counter first:

```compact
if (!m.member(k)) { m.insert(k, default<Counter>); }
m.lookup(k).increment(1);
```

`insertDefault` does **not** exist in this stdlib. Prefer this over a `Map<K, Uint>`
read-modify-write: `Counter` increments are **conflict-free** under concurrent
transactions — the right choice for a tally / vote count, where many writers race.

When writers *don't* race (e.g. M-of-N approvals where each signer approves once,
enforced by a spent-marker), the manual pattern sidesteps the gotcha entirely and
compiles fine in-circuit (verified: HoFi `approveClose`):

```compact
if (approvals.member(k)) {
  approvals.insert(k, (approvals.lookup(k) + 1) as Uint<16>);
} else {
  approvals.insert(k, 1);
}
```

Also verified in the same contract: a **`for (const s of signers)` loop over a
`Vector<8, Bytes<32>>` constructor arg** compiles and inserts into a ledger `Set`
(zero-padding entries are harmless — a zero commitment would need a SHA-256 preimage
of zero); the TS side passes a plain `Uint8Array[]` of length 8.

### `Uint<0..N>` — the upper bound is **exclusive** at runtime  ⚠️

`Uint<0..N>` *compiles* for any value but the **runtime rejects `N` itself**: passing
`7n` to an arg typed `Uint<0..7>` throws `"type error: ... expected value of type
Uint<0..7> but received 7n"`. So `Uint<0..7>` admits **0..6** (7 values). To allow a
vote gradient of levels 0..7 inclusive (8 values), declare **`Uint<0..8>`**. The
generated `.d.ts` erases the bound to plain `bigint`, so TS won't catch this — only the
runtime/simulator does. Same applies to a bounded `Uint` used as a **struct field**.

### Hashing a composite key — use a `struct`, not just a `Vector`

The usual hash takes a homogeneous `Vector<N, Bytes<32>>`
(`persistentHash<Vector<4, Bytes<32>>>([pad(32,"dom:"), a, b, c])`). To key a `Map` by
a **mix of types** (e.g. a tally cell per `(decisionId, choice)` where `choice` is a
small `Uint`), `persistentHash` is generic over **any** Compact type — declare a struct
and hash it:

```compact
struct TallyKeyInput { tag: Bytes<32>, decision: Bytes<32>, level: Uint<0..8> }
export pure circuit tallyKey(decisionId: Bytes<32>, choice: Uint<0..8>): Bytes<32> {
  return persistentHash<TallyKeyInput>(
    TallyKeyInput { tag: pad(32, "dom:tk:"), decision: decisionId, level: choice });
}
```

Don't try to cast the `Uint` into `Bytes<32>` (there's no clean coercion); hash the
struct. Verified to compile (compiler 0.31.0) and to round-trip via
`pureCircuits.tallyKey(...)` in the simulator. Factor shared logic (e.g. Merkle
eligibility used by several circuits) into a non-exported `circuit` that **returns the
witness-derived secret** — internal circuit-to-circuit values stay private without
`disclose`; only writes to public ledger / public outputs need it.

### Everything is private until disclosed — including circuit *parameters*

The compiler taints not just witness outputs but **circuit parameters** too. Writing
a parameter into public ledger state — even as a **Map key** via `member`/`insert` —
trips "potential witness-value disclosure" and needs `disclose(...)`. Disclose the
genuinely-public ones once: `const k = disclose(param);`. The compiler **never** asked
to disclose the actual secret (the witness `voterSecret`) — trust that signal: if it
demands a `disclose`, understand the leak before silencing it.

### The generated API is the source of truth

`compact compile src/x.compact src/managed/x` writes `src/managed/x/`:

- `contract/index.d.ts` — `Witnesses<PS>`, `Ledger`, `Contract<PS>` (`.initialState`,
  `.circuits`), `Circuits`/`PureCircuits`/`ImpureCircuits<PS>`, `pureCircuits`,
  `ledger()`. **Type your TS `witnesses` as `Witnesses<PS>`** so the signatures (and
  the Merkle-path shape) can't drift from the circuit.
- `compiler/contract-info.json` — authoritative ledger layout (storage: Cell/Map/
  Set/Counter), circuit signatures, and the exact **compiler / language / runtime
  versions**. **Pin `@midnight-ntwrk/compact-runtime` to the `runtime-version` it
  reports** — the generated code imports it and Midnight moves fast between releases.
- `keys/*.prover|*.verifier`, `zkir/` — the ZK artifacts.

The stdlib has **no shipped source** — it's embedded in `compactc.bin`. Introspect
the API via `contract-info.json` + the generated `.d.ts`, or `grep -aoE` symbol
names out of the binary (that's how `merkleTreePathRoot` and the absence of
`insertDefault` were confirmed).

### Windows reality: the compiler runs in WSL

`compactc` is Linux/macOS-only. On Windows it lives in your **WSL Ubuntu** distro —
and the *default* WSL distro is often `docker-desktop` (BusyBox, no `bash`), so
target Ubuntu explicitly. The native-Windows `compact` on `PATH`
(`C:\WINDOWS\system32\compact`) is the unrelated **NTFS-compression** tool, not this
compiler. Compile via a login shell:

```bash
wsl -d Ubuntu -- bash -lic 'cd /mnt/c/.../pkg && compact compile src/x.compact src/managed/x'
```

`npm run typecheck` and the `tsx` simulator test run **natively** on Windows Node —
only `compact compile` and testnet deploy need WSL (+ the proof server for deploy).

⚠️ **Shell variables do not survive into `wsl -- bash -lc '...'` from Git-Bash.** MSYS mangles
the argument, so `D=$HOME/x; ls "$D"` sees `D` empty and `ls` silently lists the CWD instead —
a maddening symptom that reads as "the path doesn't exist". Use **absolute literal paths with
no shell variables**, or drop a `.sh` in `scripts/` and pipe it CRLF-safe:
`tr -d '\r' < scripts/foo.sh | wsl -d Ubuntu -- sh`. Same root cause: a bare `/home/...` used as
the *command* gets rewritten to `C:/Program Files/...`.

Running a circuit test from a **git worktree**: junction its `node_modules` to the main
checkout's (`New-Item -ItemType Junction -Path <wt>/node_modules -Target <main>/node_modules`).
`src/managed/` and `node_modules/` are gitignored, so nothing leaks into the commit.

### Computing commitments/nullifiers in the **browser** (the WASM bundling trap)

You don't need the proof server (or the generated contract) to compute a **commitment**
or a **nullifier** client-side — they're just `persistentHash`. Replicate the pure
circuit with **only `@midnight-ntwrk/compact-runtime`** (`persistentHash` +
`CompactTypeVector`/`CompactTypeBytes`), and **pin a cross-language vector test** that
asserts it equals `pureCircuits.commitIdentity` / `voteNullifier` from the generated
contract. This lets the device derive its identity leaf / anti-double-vote nullifier so
the secret **never leaves the browser** — only the opaque output is sent.

But `compact-runtime` pulls `@midnight-ntwrk/onchain-runtime-v3`, whose WASM is
**wasm-bindgen "bundler" target**. **Next 14 / webpack 5 can't parse it** (`"Module
parse failed: parseVec could not cast the value"`); there's no `web`/fetch glue to fall
back to. The working escape (verified):

1. Pre-build that engine with **Vite** (`vite-plugin-wasm` + `vite-plugin-top-level-await`,
   `build.lib` ESM, `target: "esnext"`, `publicDir: false`) into a **self-contained**
   `public/vote-engine/index.js` — the WASM ends up **inline as a `data:` base64 URL**.
2. Load it at runtime **outside** webpack: `import(/* webpackIgnore: true */ url)` with
   the URL in a **variable** (not a literal, else webpack still resolves it).
3. `vite-plugin-top-level-await` defers the real exports behind a `__tla` promise —
   `await mod.__tla` **before** calling `mod.computeCommitment`.
4. Add a `prebuild` script so `next build` regenerates the bundle; gitignore the output.

Proof generation/submission still needs the proof server + a deployed contract — defer
that. But UX + device-side commitment/nullifier + an off-chain store that dedupes by
nullifier is fully buildable and testable without it.

### Deploy/interaction: use the **Wallet SDK 1.x**, NOT `@midnight-ntwrk/wallet@5.0.0`  ⚠️

To actually deploy + call circuits (proof gen + balance + submit) you wire a wallet into
the midnight-js providers. **The biggest version trap in the whole stack:**
`@midnight-ntwrk/wallet@5.0.0` is npm's `latest` but is **stale** — it speaks the old
indexer GraphQL subscription `wallet`/`ViewingUpdate` (the 4.x indexer dropped it; it's now
`shieldedTransactions`) and derives HRP `_test`. Against a current indexer you get a silent
sync hang; against preprod the indexer rejects the viewing key:
`"invalid viewing key: expected HRP mn_shield-esk_preprod, but was ..._test"`.

The wallet that pairs with `compact-runtime` 0.16.0 / `midnight-js` 4.1.1 is the **Wallet
SDK 1.x**: `@midnight-ntwrk/wallet-sdk` + **`@midnight-ntwrk/ledger-v8`** (a *different*
package from `@midnight-ntwrk/ledger`). **The compatibility matrix is the source of truth —
check it before pinning anything:** <https://docs.midnight.network/relnotes/support-matrix>
(runtime 0.16.0 ↔ Wallet SDK 1.x · indexer 4.0.1 preprod / 4.3.3 preview · node 0.22.x ·
proof-server 8.0.3). When sync silently fails, **diagnose with a WS proxy** that logs frames
between the wallet and the indexer (`ws` lib forwarding to the real `wss://…/graphql/ws`) —
it surfaces the exact `connection_ack` / schema / HRP error the SDK swallows.

Wiring (verified): `HDWallet.fromSeed(Buffer.from(seedHex,'hex'))` →
`.selectAccount(0).selectRoles([Zswap, NightExternal, Dust]).deriveKeysAt(0)` → build
`ledger.ZswapSecretKeys.fromSeed` / `DustSecretKey.fromSeed` / `createKeystore`, then
`WalletFacade.init({ configuration, shielded, unshielded, dust })` (the constructor is
**private** — use `init`, not `new`). Gotchas that only show at runtime:
- **Each sub-wallet needs a `txHistoryStorage`** or it throws `Cannot read properties of
  undefined (reading 'upsert')`. `new NoOpTransactionHistoryStorage()` (from
  `@midnight-ntwrk/wallet-sdk-abstractions`) works; the `InMemory…` variant needs a schema.
- `walletProvider.balanceTx` = `wallet.balanceUnboundTransaction(tx, {shieldedSecretKeys,
  dustSecretKey}, {ttl})` → sign the unshielded intents → `wallet.finalizeRecipe(recipe)`;
  `midnightProvider.submitTx` = `wallet.submitTransaction(tx)` (same object).
- `levelPrivateStateProvider` requires `privateStoragePasswordProvider` (password with **≥3
  character classes**) **and** `accountId` (use the wallet's coin public key).
- `setNetworkId('undeployed' | 'preprod')` is a **lowercase string**; the zswap `NetworkId`
  enum (used by `@midnight-ntwrk/wallet@5.x` legacy `buildFromSeed`) is numeric/capitalized —
  don't confuse them.

### Local standalone network — full E2E ZK with no testnet, no funds

For the real proof-gen E2E (deploy→call→read) without tNIGHT, run Midnight's **local
standalone** from `midnightntwrk/midnight-local-dev` (`standalone.yml`): `midnight-node`
(`CFG_PRESET=dev`) + `indexer-standalone` (`APP__APPLICATION__NETWORK_ID=undeployed`,
`APP__INFRA__NODE__URL=ws://node:9944`) + `proof-server`. NetworkId `undeployed` (HRP
`_undeployed`); a **genesis wallet (seed `0x00…01`)** holds all the initial tNIGHT. The
proof server downloads **~33 MB of public params** (BLS + zswap/dust provers) from S3 on the
first proof — **slow, and host download is throttled the same**, so persist them with a bind
mount (`./.zk-params:/.cache/midnight/zk-params`) so it happens once. Flip the same code to
preprod by only changing `.env` (network + endpoints + a funded seed).

### Self-custodial submit via the **Lace dApp connector** (`@midnight-ntwrk/dapp-connector-api` v4.0.1)

Two ways to get a proven tx on-chain from a browser dApp:

- **Relayer (variant B):** the device proves locally → an **opaque serialized tx** →
  a backend/sidecar relayer balances + submits + pays fees. Tenzo-in-the-loop.
- **Self-custodial (variant A):** the **user's own Lace wallet** balances + submits and
  pays the tDUST. **Stronger blindness** — no third party in the write path, and Lace's
  prover runs locally on the user's machine, so the `voterSecret` never leaves the device.

The connector is the extension-injected runtime; install the package as a **dev-dep for
types only**. API surface (verified, v4.0.1):

- `window.midnight` is a map keyed by wallet id; pick the provider whose `name === "lace"`.
- `provider.connect(networkId)` → the wallet API (prompts the user). Validate the returned
  network matches what you expect.
- `api.getConfiguration()` → `{ networkId, indexerUri, indexerWsUri, proverServerUri,
  substrateNodeUri }` — use **these** endpoints for the dApp's public reads + proof server,
  so the proof goes to the user's own configured prover.
- `api.getShieldedAddresses()` → `{ shieldedAddress, shieldedCoinPublicKey,
  shieldedEncryptionPublicKey }` (Bech32m). Decode to the hex pubkeys the proof bundle
  needs: `setNetworkId(networkId); MidnightBech32m.parse(addr).decode(ShieldedAddress,
  networkId)` (from `@midnight-ntwrk/wallet-sdk-address-format`) → `.coinPublicKey
  .toHexString()` / `.encryptionPublicKey.toHexString()`.
- `api.getDustBalance()` → `{ balance, cap }`.
- `api.balanceUnsealedTransaction(txHex, { payFees? })` → `{ tx }` — balances + adds fee
  inputs to an already-**proven** tx.
- `api.submitTransaction(tx)` → **`void`** (NOT a txId — don't log `txId=…`, it'll be
  `undefined`; confirm success by reading the ledger / nullifier count instead).

#### The "unsealed" tx format — the trap that cost a day  ⚠️

`balanceUnsealedTransaction` rejects almost everything with a cryptic
`expected header tag 'midnight:transaction[v9](signature[v1],proof,embedded-fr[v1]):',
got '�'`. Two independent requirements hide behind that one error:

1. **It must be HEX, not base64.** Base64 → the `'�'` (replacement char) message.
   Serialize then hex-encode the bytes.
2. **It must be PROVEN, not unproven.** The header wants the `proof` marker — pass a tx
   that already went through proof generation (against `getConfiguration().proverServerUri`),
   not the unproven call tx.

The header's **`embedded-fr[v1]`** is just the **on-wire form of ledger-v8's `pre-binding`**
binding marker (the ledger-v8 wasm literally contains `embedded-fr[v1]`; the `Bindingish`
type only names binding / pre-binding / no-binding). So a proven tx serialized by ledger-v8
is **already in the right shape — no binding conversion needed.** The fix is purely: prove
first, then `toHex(provenTx.serialize())`. Build a dedicated `buildUnsealedVoteTx` that
proves with the **self (Lace) pubkeys** and returns hex.

#### Lace ↔ local "undeployed" network

Lace supports an **"Undeployed"** network (Settings → Midnight → Undeployed) that points at
`localhost:9944` (node) / `8088` (indexer, path `/api/v4/graphql`) / `6300` (proof) — i.e.
the exact local standalone stack above (networkId `"undeployed"`). This gives a **real E2E
with Lace against local**, no testnet/faucet dependency.

- **Funding a fresh Lace wallet on undeployed:** the genesis wallet (seed `0x00…01`) holds
  all the tNIGHT. A `fund-wallet` script builds the genesis wallet and `transferTransaction`s
  NIGHT to the Bech32m address Lace shows (decode with `UnshieldedAddress`). `nativeToken().raw`
  is NIGHT.
- **DUST gotcha:** a fresh Lace wallet shows `getDustBalance: 0n`. **NIGHT must be registered
  for DUST generation** (signed by the NIGHT owner — so `fund-wallet` can't do it for them).
  DUST then regenerates over time; on mainnet this is automatic, on undeployed the "public key
  method" needs a manual registration step. A "could not balance dust / can't pay fees" error
  is almost always unregistered or not-yet-regenerated DUST, not a code bug.
- **De-risk variant A without Lace** by replaying the same tx through a mock: build the
  unproven tx with the **genesis** pubkeys, prove via the node proof provider, then reuse the
  relayer balance+submit path. Confirms the tx is valid independent of the wallet extension.

### In-browser (WASM) proving — closing the custodial proof-server leak  ⚠️

The **custodial** blindness gap is one line: a relayer/backend that proves for the user calls
`httpClientProofProvider(proofServerUrl).proveTx(unprovenTx)` — so **the proof server sees the
`voterSecret` (the witness)**. To close it, prove **in the user's browser** (WASM), same as Lace
does: the witness never leaves the device, no proof server in the path. **Validated E2E on-chain**
(relay `SucceedEntirely`, on-chain nullifier count +1) against the local standalone stack.

**The engine is `@midnight-ntwrk/zkir-v2`** (the same WASM prover Lace uses), driven through a
`KeyMaterialProvider`:

```ts
// KeyMaterialProvider = {
//   lookupKey(keyLocation): Promise<{ proverKey, verifierKey, ir } | undefined>;
//   getParams(k: number): Promise<Uint8Array>;     // the SRS, sized by circuit
// }
import { Zkir, provingProvider, type KeyMaterialProvider } from "@midnight-ntwrk/zkir-v2";
import { CostModel } from "@midnight-ntwrk/ledger-v8";

// An UnprovenTransaction from createUnprovenCallTx (midnight-js-contracts) IS a
// ledger-v8 Transaction — prove it directly on the MAIN THREAD:
const proven = await unprovenTx.prove(provingProvider(kmp), CostModel.initialCostModel());
```

`Zkir.deserialize(ir).getK()` gives the **exact params size** the circuit needs (HoFi `castVote`
→ **k = 15**), so `getParams` can fetch the right SRS. `midnight-js-protocol/ledger` is literally
`export * from '@midnight-ntwrk/ledger-v8'` (deduped single copy) — the `CostModel` type matches.

**The architecture finding (the one that cost the most):** the **`WasmProver`**
(`@midnight-ntwrk/wallet-sdk-prover-client/effect`, the higher-level Lace path) runs the prover in
a **Web Worker** and **hangs** — the worker is created, receives the `prove` op, then **never
requests keys/params and never errors** (silent, indefinite). The **main-thread** `tx.prove(
provingProvider(kmp), costModel)` **works**. Tradeoff: main-thread proving **freezes the tab
~94 s** (vs 6.9 s on the proof server). For an MVP that's acceptable; the "don't block the UI"
worker path is what to escalate upstream.

**Four blockers, all of which fail *silently* (no error) if unmet:**

1. **`CostModel` is obligatory.** `tx.prove(provider)` / `proveTransaction(tx)` **without** the
   2nd `cost_model` arg throws a cryptic `ClientError: Failed to prove transaction` / cause
   `expected instance of e` (the ledger-v8 wasm needs `cost_model`). Pass
   `CostModel.initialCostModel()`.
2. **Cross-origin isolation (COOP/COEP) is mandatory — and its absence is a SILENT HANG.** zkir-v2
   uses **WASM threads (rayon)** → needs `SharedArrayBuffer` → the page must be
   **`crossOriginIsolated`**. Serve it with **`Cross-Origin-Opener-Policy: same-origin`** +
   **`Cross-Origin-Embedder-Policy: credentialless`** (and `Cross-Origin-Resource-Policy:
   cross-origin` on cross-origin subresources). Without isolation, `self.crossOriginIsolated` is
   `false`, `SharedArrayBuffer` is `undefined`, and proving **hangs with no error, no param fetch,
   forever**. Guard the UI: `if (!self.crossOriginIsolated) warn(...)`. This is the nastiest gap —
   nothing tells you it's the cause.
3. **The SRS params need a CORS-enabled mirror.** The SDK's default params live in an **S3 *dev*
   bucket with no CORS** — `curl` returns 200 without an `Origin`, but with an `Origin` there's no
   `Access-Control-Allow-Origin`, so the **browser can't fetch them** (`getParams` → "Failed to
   fetch"). Fix: **self-host `bls_midnight_2p<k>`** (e.g. `bls_midnight_2p15`) **same-origin** and
   serve it from `getParams`, with a fallback to the SDK default.
4. **The worker must be bundled explicitly.** The prover worker is instantiated as
   `new Worker(new URL('…/proof-worker.js', import.meta.url), { type: 'module' })` with the base
   URL **aliased into a variable** — which **defeats Vite's static worker detection** (404 at
   runtime). Fix: a **separate Vite build** that bundles
   `node_modules/@midnight-ntwrk/wallet-sdk-prover-client/dist/proof-worker.js` to the
   runtime-resolved path (`public/dist/proof-worker.js`).

**Reproducible smoke rig (HoFi `contracts-midnight/vote-proof/`):** `bootLocalProver({zkBaseURL})`
validates blockers 3/4 with no chain (keys fetch + params size + worker boot);
`proveUnprovenTxMainThread(unprovenTx, kmp)` is the **validated** path;
`proveUnprovenTxLocally` (the `WasmProver`-worker variant) is kept as the **hang repro**. Serve the
smoke page COOP/COEP with a tiny static server (`scripts/coi-server.py`, `no-store` to defeat the
browser's module cache) and **cache-bust the dynamic import**
(`import('/vote-proof/index.js?v='+Date.now())`, then `await mod.__tla` before reading exports —
top-level-await WASM init). Metrics from the validated run: prove ≈ 93.8 s, relay `SucceedEntirely`
block 13832, on-chain nullifiers 5 → 6. Full runbook: `docs/07e-in-browser-proving-spike.md`.

### Selective-disclosure over an attested leaf — the soundness pattern (hofi-passport F3, 22-jul-2026)

Verified building `identity_disclosure.compact` (passport F3): the reusable pattern for
"prove a fact about a private value without revealing it" — attribute equality, an age/threshold,
or a **private counter ≥ N** (the shape a judicial "≥N filings" or a reputation "≥N" needs).

- **Commit the value INTO the leaf, then re-derive it in-circuit.** Leaf =
  `persistentHash([tag, sk, attrDigest(...values...)])`. A `boundIdentity()` helper recomputes the
  leaf from the `witness` values and asserts `path.leaf == leaf` **and**
  `merkleTreePathRoot(path) == attestedRoot`. **This is the whole soundness argument:** a prover
  who lies about any committed value (claims `count=5` when the attester committed `3`) recomputes a
  *different* leaf that isn't in the tree → rejected. The value stays private (only the `≥N` boolean
  / the nullifier is disclosed) yet is sound (bound to the public root).
- **Prove it with an ADVERSARIAL simulator test, not a happy-path one.** The property "can't inflate
  the count / lie about the attribute" is invisible unless you *try to break it*: feed a lying
  private state (`{...real, count: 5n, merkleProof: realProofForCount3}`) and assert it throws
  `"leaf != identity+attrs"`. Happy-path tests using the true attributes never exercise this.
- **`sealed` is a RESERVED keyword** in Compact — a circuit/param named `sealed` is a parse error.
- **Threshold without subtraction** (re-confirms the widening rule): `birthYear + 18 <= currentYear`,
  never `currentYear - birthYear >= 18` (Uint underflow).

**⚠️ Dual-ledger aggregate forgeability.** A public `Counter` (e.g. `eventTotal`) fed by a
`register(commitment)` circuit whose `commitment`/seal comes from an **unconstrained `witness`** is
**spammable**: one identity calls it repeatedly with fresh seals and inflates the public number; the
per-(id,seal) nullifier only blocks the *same* seal twice. To make the public aggregate authoritative
you must **anchor the seal to unique external evidence** (an authoritative `case_id`, inclusion in a
case padrón, …) — that binding is outside the circuit. The **private** threshold proof
(`count ≥ N` over the *attested* count) is unaffected; only the public tally is soft. Decide the
anchoring before relying on the "N this period" number (it's a common demo/rubric trap).

### Wallet sync OOMs against a public network — and never against your local one (26-jul-2026)

Verified twice: first in `hofi-consensus` F1 (blocked the Preprod cut of blind voting), then
**re-discovered from scratch** in `hofi-passport` before anyone remembered it had been solved.
If your `mn:up` harness works locally and dies the moment you point it at **Preprod**, this is why.

- **Symptom.** `buildWallet` / the wallet facade rebuilds full Zswap state on every run. Syncing
  from genesis on Preprod (~1.77M blocks at the time of measuring) the in-flight buffer has **no
  upper bound**: the heap grows monotonically and the process OOMs at ~5.2 GB even with
  `--max-old-space-size=6144`. On a standalone local network the history is trivial, so the bug is
  **invisible until you touch a public network**. Do not read "it works locally" as evidence.
- **Fix 1 — backpressure on the sync.** In the wallet config set `bufferSize` (~2000),
  `resumeThreshold` (~100) and `batchUpdates.spacing` (~8). This keeps memory **flat** through the
  historical sweep instead of letting it climb. ⚠️ **These options require
  `wallet-sdk-shielded ≥ 3.0.2` + `facade ≥ 4.1.0`** — on shielded 3.0.1 / facade 4.0.1 they are
  silently ignored and you will conclude the fix "doesn't work". Check the versions first.
- **Fix 2 — serialize/restore.** Persist the sub-wallets' state periodically (~30s) and restore on
  start. Turns a ~3 h cold sync into a ~2 min warm start; without it every run pays full history.
- **Both are needed.** Backpressure makes the sweep survivable; serialize/restore makes it something
  you can iterate on.

## Why this matters for HoFi (care-economy privacy)

HoFi handles intrinsically sensitive data — who cares for whom, health/dependency
context, voice. On Cardano L1 the design is deliberately **minimalist on-chain**
(only invariants on-chain; rich history in Neon + CIP-20/68 metadata) precisely
because L1 is fully public. Midnight is the natural home for the parts of that
history you'd otherwise keep entirely off-chain but still want **verifiable**:

- **Reputation without exposure** — prove `reputation ≥ N` (membership SBT
  threshold) to unlock a benefit, without revealing the score, the tasks, or the
  caregiver's identity. Selective disclosure replaces "trust the DB".
- **Private care receipts** — a task happened and was approved by quorum (provable)
  while the task content / parties stay shielded.
- **Compliance / payouts** — prove a holón distribution respected its cap or a
  member met eligibility, without publishing the underlying ledger.

Architecture fit: keep **value, caps and public coordination on Cardano L1
(Aiken/eUTXO)** where you already have working validators; consider **Midnight as a
privacy module** for the sensitive, disclosure-controlled claims — bridged via
NIGHT/DUST and the partner-chain relationship. This is forward-looking (post-MVP):
the L1 line is the committed path; treat Midnight as the privacy R&D track, not a
rewrite. Do **not** port the eUTXO validators to Compact — the models don't map;
you'd re-design around circuits, witnesses, and disclosure.

## Identity credentials: commitments, nullifiers & the off-chain bridge

The recurring identity pattern on Midnight (Semaphore / World ID style):

- **Register** a `commit(subject_id, attrs, salt)` as a **leaf** of a `MerkleTree`
  in the public `ledger`. The leaf is opaque; it binds an identity + attributes
  without revealing them.
- **Prove membership** = prove your leaf is in the tree (the `merklePath` is a
  witness) **without revealing which leaf**. That is what makes a credential
  *unlinkable*: two presentations don't correlate.
- **Nullifier per context** = a deterministic `hash(secret, context)` the ledger
  records as spent, so the same person can't double-use a credential in one context
  (vote twice, claim twice) while staying unlinkable *across* contexts. This is how
  `proveReputationAtLeast` / `proveHolonMembership` / `proveAdult` / `proveConsent`
  enforce one-shot use without identifying anyone. Validate the witness-supplied
  attributes *inside the circuit* (range/commitment checks) — never trust the witness.

**The bridge to the off-chain blindness layer** (`references/privacy-identity.md`):
the **same opaque `subject_id`** that the off-chain vault uses as a pseudonym is what
goes into the Midnight commitment (e.g. `toBytes32(subject_id)` in `witnesses.ts`).
So the person is consistent across both layers and **neither layer ever sees the
PII**. This also fixes a subtlety the off-chain layer alone can't: a biometric
**fuzzy extractor's helper data `P` is not unlinkable by itself** — but if `commit(R)`
is a tree leaf and credentials are presented via per-context nullifiers, on-chain
*use* stays uncorrelated even if `P` leaks something off-chain. Midnight's
verifiability and the off-chain blindness are complementary halves of one design;
build both, don't expect either alone to deliver privacy.

## Audit: corrections to sibling skills

Audited the `midnight-*` skills against the verified findings above (toolchain
compactc 0.31.0 · language 0.23.0 · runtime 0.16.0, HoFi build). Trust **this file**
where they conflict; each item says which skill to fix.

**Contradictions (a sibling skill is wrong):**

- **`midnight-compact/references/community-gotchas.md`** ships the proof-server image
  as **`midnightnetwork/proof-server`** — the correct org is **`midnightntwrk`**
  (`docker run -p 6300:6300 midnightntwrk/proof-server:latest …`). The misspelled pull
  fails.
- **`midnight-api` / `midnight-dapp-dev`** show `const result = await
  connectedAPI.submitTransaction(tx)` as if it returns a usable value. **Verified:
  `submitTransaction` returns `void`** — there is no txId; confirm success by reading
  the ledger / nullifier count, not the return value.
- **`midnight-compact/references/community-gotchas.md`** calls **`Cell<T>` "deprecated"**.
  `Cell<T>` is a live ledger ADT at runtime 0.16.0 (direct-type `ledger x: Field` is
  sugar over it; `.read()`/`.write()` work). Treat "deprecated" as imprecise.
- **`midnight-compact`** documents ledger ops via `StateValue<T>` and a **free-function
  `read(totalSupply)`**; the API we verified is method-style (`counter.read()`,
  `cell.write()`). Version divergence — verify against the generated `contract-info.json`
  for the compiler in use.

**Gaps (verified knowledge no sibling skill has — see the sections above):**

- The **`@midnight-ntwrk/wallet@5.0.0` trap**: it is npm's `latest` but **stale** (old
  indexer subscription + `_test` HRP) → silent sync hang / HRP rejection. Use **Wallet
  SDK 1.x + `@midnight-ntwrk/ledger-v8`** and the **support-matrix**. No sibling skill
  warns about this; `midnight-wallet` documents only the happy `wallet-sdk-facade` path.
- **`Map<K, Counter>` does NOT auto-initialize** (`insertDefault` doesn't exist) →
  runtime `"expected a cell, received null"`. Absent everywhere.
- **`Uint<0..N>` upper bound is exclusive at runtime.** The sibling skills only ever use
  single-bound `Uint<n>`, so the ranged-form trap is undocumented.
- **The "unsealed" tx format**: `balanceUnsealedTransaction` needs **hex (not base64)
  AND a PROVEN tx** (`embedded-fr[v1]` = ledger-v8 pre-binding). The connector docs show
  the call but not this requirement.
- **In-browser WASM proving**: COOP/COEP + `SharedArrayBuffer` (silent hang if absent),
  CORS-mirrored SRS params, explicit worker bundling, `WasmProver` worker hangs vs
  main-thread `tx.prove(provingProvider(kmp), CostModel.initialCostModel())` works.
- **Next/webpack can't parse bundler-target Midnight WASM** → the Vite self-contained +
  `webpackIgnore` escape.
- **Windows: `compactc` runs in WSL Ubuntu** (native `compact` on PATH is the NTFS tool).
- **Local simulator** via `@midnight-ntwrk/compact-runtime` (`createCircuitContext`) as
  the fast inner loop, and **Merkle root-snapshot verification** via the free function
  `merkleTreePathRoot` (no `.checkRoot()` on the path). `midnight-compact` only covers
  `findPathForLeaf` off the ledger ADT — a different scenario.

## Authoritative sources

- Docs hub: <https://docs.midnight.network>
  (Compact lang ref `/develop/reference/compact/lang-ref`, the `bboard`
  tutorial `/tutorials/bboard/smart-contract`, install
  `/getting-started/installation`, release notes `/relnotes`).
- Compact compiler/language repo (LFDT): <https://github.com/LFDT-Minokawa/compact>.
- OpenZeppelin Compact tools: <https://github.com/OpenZeppelin/compact-tools>.
- Token/network overview: <https://midnight.network> · NIGHT: <https://midnight.network/night>.
