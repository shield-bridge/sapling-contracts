# sapling-contracts

> The Tezos smart contracts behind [Shield Bridge](https://github.com/shield-bridge/shield-bridge):
> privacy-preserving pools that shield, privately transfer, and unshield assets using the **Sapling**
> protocol, plus the on-chain **relay registry** that lets clients discover privacy relays.

Written in **JsLIGO**, built with **Taqueria** + **LIGO**. Two generations live here: **V2** (current,
a factory model) and **V1** (the original standalone wrappers, kept for reference and deprecated).

## How the privacy model works

Each pool holds a Sapling state. Funds enter by **shielding** (a public deposit of the real asset),
move between shielded accounts by **private transfer** (no public leg), and leave by **unshielding**
(a public payout). On-chain, every operation is an opaque `sapling_transfer`: the sender, recipient,
amount, and asset are encrypted, and only a viewing key can decrypt them. The contracts here are the
trust anchor for that, they verify each zero-knowledge `sapling_transaction` and move the matching
public asset leg. Proofs are generated client-side (see the [SDK](https://github.com/shield-bridge/shield-bridge-sdk)).

All pools use a Sapling state of **memo size 8** (`sapling_state<8>`) and the modern `Tezos.Next.Sapling.*`
LIGO API.

---

## V2 — current contracts

V2 splits the old monolith into three focused contracts: a **Factory** that deploys and indexes
per-asset pools, the **Sets** that are the pools themselves, and a **Relay Registry** for discovery.
The Factory is *not* in the transfer path: a client looks up a Set's address, then calls that Set
directly. This gives clean per-asset isolation and trustless on-chain discovery.

### Sapling Factory — `contracts/v2/sapling_factory.jsligo`

A pure **deployer + registry** for per-asset Sets. It originates a new Set for a token, records the
`asset -> set address` mapping, and exposes views so anyone can resolve a token's Set or verify that
an address is a legitimate Factory-deployed Set. It never holds or moves funds.

| Entrypoint / view | Kind | What it does |
|---|---|---|
| `init_token_sapling_set` | entry | Originates a new per-asset Set and registers it. `token_id = None` deploys an **FA1.2** Set (requires the token to expose `%transfer` + `%approve`); `token_id = Some(id)` deploys an **FA2** Set (requires `%transfer` + `%update_operators`). Rejects attached XTZ and duplicate assets (`SAPLING_SET_ALREADY_EXISTS`). |
| `get_tez_set` | view | The single pre-deployed **Tez** Set address. |
| `get_fa1_2_set` | view | `token_contract -> option<address>` of its Set. |
| `get_fa2_set` | view | `{ contract, token_id } -> option<address>` of its Set. |
| `is_registered_set` | view | `true` if the address was originated by this Factory (used to confirm a counterparty is a real Shield Bridge Set, not an arbitrary contract). |

**Storage:** `{ tez: address; token_fa_1_2: big_map<address,address>; token_fa_2: big_map<{contract,token_id},address>; registered_sets: big_set<address> }`.
FA1.2 keys by token contract; FA2 keys by `contract + token_id` (one FA2 contract hosts many token_ids,
each gets its own Set). The **Tez** Set is special: it is deployed out of band and hard-wired into the
Factory's storage at origination (so `init_token_sapling_set` only ever creates FA1.2 and FA2 Sets).

### Sapling Sets — `contracts/v2/sapling_set_{tez,fa1_2,fa2}.jsligo`

The shielded pools. Each instance holds one asset's Sapling state and exposes a **single `@default`
entrypoint** taking `list<sapling_transaction<8>>`. It folds the batch through
`Tezos.Next.Sapling.verify_update`, and the **sign of the verified balance** dispatches the public leg:

| Net balance | Meaning | Public leg |
|---|---|---|
| `< 0` | **Shield** | pull the asset in from the caller |
| `> 0` | **Unshield** | pay the asset out to the `key_hash` decoded from `bound_data` |
| `= 0` | **Private transfer** | none (state change only) |

| Variant | Storage | Public asset move |
|---|---|---|
| `sapling_set_tez` | the bare `sapling_state<8>` | XTZ, via balance arithmetic against `Tezos.get_amount()` |
| `sapling_set_fa1_2` | `{ state, contract }` | FA1.2 `%transfer` (allowance model) |
| `sapling_set_fa2` | `{ state, contract, token_id }` | FA2 `%transfer` (operator model) |

The shield payer is always `Tezos.get_sender()` (the direct caller), never a configurable address, so
a Set cannot be tricked into pulling funds from a third party.

### Relay Registry — `contracts/v2/relay_registry.jsligo`

A **permissionless, ownerless** on-chain directory of Shield Bridge privacy relays. Clients read it
trustlessly over RPC to discover who runs a relay, where its descriptor lives, and which worker `tz1`s
its fees are paid to, so users never hand-paste relay URLs and no maintainer can censor the list. It
stores discovery facts only, **never reputation** (clients judge reliability from their own experience).

| Entrypoint | What it does |
|---|---|
| `register` | `{ worker_keys, descriptor_url }` + a **refundable deposit** (`min_deposit`). Sender becomes the operator. Fails if the deposit is short, the operator already registered, or any worker key is already claimed by another relay. |
| `update` | Change only `descriptor_url` (worker keys are immutable, re-register to rotate). |
| `deregister` | Stop being discovered now; starts the `unbond_period` timer. |
| `withdraw` | After the unbond period, refund the deposit and free the entry + its worker keys. |

**Storage:** `{ relays: big_map<address,Entry>; index: set<address>; claimed_workers: big_set<key_hash>; min_deposit: tez; unbond_period: int }`.
`index` is a *plain* set on purpose, so a client enumerates every live operator in one `GET storage`
call (big maps are membership-only and need an indexer). `claimed_workers` enforces global worker-key
uniqueness. There is **no admin key, no pause, no force-remove, and no parameter mutation** after
origination, that uncensorable property is what makes permissionless discovery safe; to evolve, you
originate a new version and ship its address to clients. Full spec and the `register` ABI:
[`RELAY_REGISTRY.md`](./RELAY_REGISTRY.md).

---

## Deployments

| Contract | mainnet | shadownet |
|---|---|---|
| **Sapling Factory** | `KT1WqGXxe5Anam6Hm6zQqGmaXdtZrzZRynnw` | `KT1Q81ZGgciw6tLfbwuPuYiJ8WyxkwzJeESQ` |
| **Relay Registry** | `KT1AsHxHLTBLofmBJAVHVRVRxYdghaRCJcRk` | `KT1PvhGAz7zfxiNC5KMYRXnDjbb9paEyJYNx` |

Per-asset **Sets** are not listed here, you resolve them at runtime through the Factory views
(`get_tez_set` / `get_fa1_2_set` / `get_fa2_set`). The Relay Registry uses a **5 XTZ** refundable
deposit and a **3-day** unbond period.

---

## V1 — legacy (deprecated)

The original standalone wrappers under `contracts/v1/`, superseded by the V2 factory model. Kept for
reference and for any V1 pools still on-chain; new work targets V2.

- `sapling_tez.jsligo`, `sapling_fa1_2.jsligo`, `sapling_fa2.jsligo` — one self-contained shielded pool
  per asset (same shield / transfer / unshield dispatch as the V2 Sets, but each is its own contract).
- `sapling_state_map.jsligo` — an aggregator that holds many pools (tez + FA1.2 + FA2 maps) inside a
  single contract, with `init_token_sapling_pool` to add a pool at runtime.

The V2 split (a Factory that originates isolated per-asset Sets, plus on-chain discovery) replaced this
"one contract per asset, or one mega state-map" approach.

---

## Repository layout

```
contracts/
  v2/                      # current
    sapling_factory.jsligo       # deploys + indexes per-asset Sets (views only; not in the tx path)
    sapling_set_tez.jsligo       # XTZ shielded pool
    sapling_set_fa1_2.jsligo     # FA1.2 shielded pool
    sapling_set_fa2.jsligo       # FA2 shielded pool
    relay_registry.jsligo        # permissionless relay discovery
  v1/                      # legacy (deprecated)
    sapling_tez / sapling_fa1_2 / sapling_fa2 / sapling_state_map
artifacts/{v1,v2}/         # compiled Michelson (.tz) + default storage
RELAY_REGISTRY.md          # relay registry spec + register ABI
quickstart.md              # generic Taqueria onboarding
```

Each contract `foo.jsligo` has sibling `foo.storageList.jsligo` and `foo.parameterList.jsligo` files
(the Taqueria convention): they `#import` the contract and define the default storage / parameters used
for origination and invocation.

---

## Build, test, deploy

Toolchain: **[Taqueria](https://taqueria.io)** (`taq`) driving the **LIGO** compiler (in Docker),
**Flextesa** (local sandbox), and **Taquito** (origination). Install plugins once with `taq install`.

```bash
# compile (auto-discovers the sibling storageList/parameterList and emits artifacts/)
taq compile sapling_factory.jsligo      # one contract
taq compile-all                         # everything

# local sandbox round-trip
taq start sandbox local
taq deploy sapling_set_tez.jsligo -e development

# deploy to a public network
taq deploy relay_registry.jsligo -e testing      # ghostnet
taq deploy sapling_factory.jsligo -e production   # mainnet

# generate TypeScript types for clients
taq generate types
```

Networks (`.taq/config.json`): `development` (local Flextesa), `testing` (ghostnet), `production`
(mainnet). Shield Bridge also deploys to **shadownet** (see [`RELAY_REGISTRY.md`](./RELAY_REGISTRY.md)).
There are no LIGO unit tests in the repo today; contracts are exercised end-to-end against the sandbox
and the [SDK](https://github.com/shield-bridge/shield-bridge-sdk).

---

## Related

- [shield-bridge](https://github.com/shield-bridge/shield-bridge) — the web app
- [shield-bridge-sdk](https://github.com/shield-bridge/shield-bridge-sdk) — the TypeScript SDK (generates the proofs, calls these contracts)
- [shield-relay](https://github.com/shield-bridge/shield-relay) — a self-hostable privacy relay that registers in the Relay Registry

## License

MIT
