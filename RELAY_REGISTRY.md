# Relay Registry (`shield-relay/1` discovery)

`contracts/v2/relay_registry.jsligo` is a permissionless, ownerless on-chain directory of
Shield Bridge privacy relays. It lets a client **discover** relays without anyone pasting a
URL by hand, and it does so without a maintainer who could censor the list.

This note explains what it stores, why it stores *only* that, and how a client turns it into a
trust ranking — because the most important design decision here is what the contract
**deliberately does not do.**

## Deployed addresses

| Network | Registry KT1 | `min_deposit` | `unbond_period` |
|---|---|---|---|
| mainnet | `KT1AsHxHLTBLofmBJAVHVRVRxYdghaRCJcRk` | 5 XTZ | 3 days |
| shadownet | `KT1PvhGAz7zfxiNC5KMYRXnDjbb9paEyJYNx` | 5 XTZ | 3 days |

Baked into the clients: `RELAY_REGISTRY_CONFIG` (shield-bridge `src/context/constants.ts`) and
`DEFAULT_REGISTRY` (shield-relay `src/config/schema.ts`), each overridable by env. An operator
registering locks the 5 XTZ deposit from its operator key (pool worker 0 by default), so that
key needs the deposit **plus** gas — see *Registering a relay*.

## Registry ≠ reputation

These are two different questions, and conflating them is the trap:

| | Registry (this contract) | Reputation (the client) |
|---|---|---|
| Answers | *Who* runs a relay, *where* it is, which worker tz1s are theirs | *How good* is it — does it complete jobs it's paid for? |
| Lives | On-chain | Off-chain (client) |

**Reputation cannot be computed on-chain for a privacy relay**, by construction:

- The user's Phase-2 operation is an **opaque Sapling proof** — no on-chain fact says "relay X
  completed job Y."
- The two-phase **distinct-worker** design deliberately unlinks the fee payment from the
  broadcast, so even a contract can't pair "fee received" with "op broadcast."
- A contract has no oracle into "the job completed."

So any on-chain reputation needs *attestations*, which are either a deanonymization leak
(clients attest → a public "who used which relay" record), forgeable (free → review-bombing),
or self-serving (relays attest their own success, unfalsifiable because the op is opaque). And
**wash-trading** makes a self-dealt score indistinguishable from a real one — there is no
on-chain distinguisher, *because* of unlinkability. The contract therefore stores **zero**
reputation and stays trivial. The client computes trust off-chain (see *Reputation*, below).

This is safe because the protocol bounds the worst case to **one refused fee** — never theft,
never deanonymization. Discovery and reputation are a reliability/UX problem, not a security
one.

## What's on-chain

```
storage {
  relays          : big_map<address, Entry>   // operator tz1 -> entry (lazy, per-key reads)
  index           : set<address>               // live operators — a PLAIN set, read in ONE storage fetch
  claimed_workers : big_set<key_hash>          // every bound worker tz1 — global uniqueness
  min_deposit     : tez                         // immutable; 0 relies on storage burn as the spam cost
  unbond_period   : int                         // immutable; seconds (deployed: 3 days)
}
Entry {
  operator        : address       // controls the entry (== sender at register)
  worker_keys     : set<key_hash> // tz1s Phase-1 fees are paid to — makes PoH attributable
  descriptor_url  : string        // https origin serving /info + /.well-known/shield-relay.json
  registered_at   : timestamp
  deposit         : tez           // refundable; NOT slashable (refusal is unprovable)
  unbond_at       : option<timestamp> // set on deregister
}
```

`index` is a **plain `set`, not a `big_set`**, on purpose: Tezos `big_map`/`big_set` are lazy
(membership-only, no key enumeration without an indexer), so a client reads the entire
live-operator list in a single `GET …/storage` RPC call, then does per-operator `big_map`
lookups only for the entries it cares about — **no indexer, no trusted intermediary.**

### Entrypoints (no admin — ownerless by design)

- `register({ worker_keys, descriptor_url })` — lock the deposit (≥ `min_deposit`), bind
  worker keys (must be globally unclaimed), appear in `index`. `Tezos.get_sender()` is the
  operator; control is proven by signing future calls from it.
- `update({ descriptor_url })` — change the URL. Worker keys are immutable (rotate via
  deregister + re-register) so the claim set stays simple.
- `deregister()` — leave `index` immediately; start the unbond timer.
- `withdraw()` — after the timer, refund the deposit and remove the entry.

There is no `pause`, no `remove(other)`, no `setDeposit`. The contract cannot be edited or
censored by anyone, including its deployer. To evolve it, originate a v2 and ship the new
address to clients (they may list several registry contracts).

### The deposit is anti-spam, not anti-theft

Earlier design rejected bonds/stake/slashing as a *reputation* mechanism: they defend a threat
the protocol already kills, slashing is unenforceable under our own unlinkability, and a big
bond is plutocratic. The `deposit` here is a *different, narrow* thing — **anti-Sybil-spam for
the list itself**, and it's **refundable** (there's nothing to slash for). On Tezos the storage
burn of a `big_map` entry is *already* a spam cost, so `min_deposit` can be `0` and rely on
that; raise it for stronger resistance. It is at most a weak ranking input, never the trust
root.

## How a client reads it (cheap, no indexer)

```
1. GET …/contracts/<registry_KT1>/storage          → { index: [op1, op2, …], min_deposit, … }
2. for each operator you care about:
   GET …/big_maps/<relays_id>/<script_expr(op)>     → Entry { worker_keys, descriptor_url, … }
3. fetch <descriptor_url>/info                       → live fee / network / liveness (re-derived,
                                                        never trusted from chain or any feed)
```

Tier 0 (the built-in default relay) is always present, so a registry failure never leaves the
client with zero relays. An optional TzKT-style indexer may speed up enumeration, but every
entry it returns must be re-validated against a direct RPC read — an indexer can hide a relay,
never forge one.

## Reputation: the two layers the client actually ranks by

The contract enables a **public** signal and the client owns a **private** one. The private one
always wins.

1. **Proof-of-Operating-History (PoH) — public, trustless, no attestation.** Because Phase-1
   fees are public unshields to the `worker_keys` bound here, anyone can compute from chain
   data alone: **age** (block level of the worker's first op) and **volume** (count of
   fee-receipts over a window). This is the cold-start prior for relays you've never used. Its
   property: faking it costs exactly as much as having it (real funds, real time). It is *not*
   "behaved well" — only "has genuinely been operating."

2. **First-party local history — private, on-device, dominant.** The client's own observed
   outcomes per relay (`sb-relay-rep` in shield-bridge: jobs / completed / `feeBurned` /
   latency). It never leaves the device (a shared reputation stream would be a deanonymization
   fingerprint). It *overrides* PoH: a relay with stellar PoH that burned **your** fee twice
   ranks below one that completed 40/40 for you.

Picker ranking (`sort by trust, never by fee`):

```
rank = first-party local score   (dominant)
     → PoH                        (public cold-start prior: age + volume)
     → liveness                   (the live /info probe)
     → fee                        (LAST, display-only)
```

PoH is the asymptote (P4c); first-party local ships first (P4a) because it's the layer that
actually protects the user and needs no contract at all.

## Deploying

```bash
# compile (LIGO 1.13.x; same toolchain as the factory)
ligo compile contract contracts/v2/relay_registry.jsligo > artifacts/v2/Relay_Registry.tz
ligo compile expression jsligo 'default_storage' \
  --init-file contracts/v2/relay_registry.storageList.jsligo > artifacts/v2/Relay_Registry.default_storage.tz

# originate per network via Taqueria (one registry per network; bake its KT1 into the client
# alongside the factory address, e.g. RELAY_REGISTRY_CONTRACT in shield-relay).
```

Edit `relay_registry.storageList.jsligo` first to set `min_deposit` / `unbond_period` for the
target network (they're immutable after origination).

## Registering a relay

A relay operator never hand-crafts an operation — `shield-relay` ships a one-liner:

```bash
relay registry register --url https://relay.example.com
```

It reads the relay's pool, binds its active worker tz1s, locks the registry's `min_deposit`,
and signs from worker 0 (override with `--operator-key` / `--from`). Lifecycle:
`relay registry update --url …`, `deregister`, `withdraw`, `show`.

## Known residual (bounded, documented)

An operator can bind worker tz1s it doesn't control **iff** they aren't already claimed by
another *registered* relay — borrowing an *unregistered* relay's fee history to inflate its
PoH. `claimed_workers` blocks the registered-vs-registered case; the rest is bounded-loss-
tolerable (worst case: a flatteringly-ranked relay refuses = one fee) and corrected fast by the
dominant first-party local history. A future hardening: require the relay to prove worker-key
custody via a `/challenge` the client cross-checks before counting PoH.
