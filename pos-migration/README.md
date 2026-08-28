# PoA → PoS migration (draft)

**JOC testnet (JOCT) runs Clique proof-of-authority today.** The execution-layer
configuration that actually drives the live network is in
[`../metadata/`](../metadata/) and is verified against the chain by CI. Nothing
about the merge is active.

This directory holds the execution-side draft of the migration: the genesis file
with the merge fields added. The consensus-layer (beacon chain) files live in
[`../metadata/`](../metadata/) alongside the execution genesis, following the
[eth-clients](https://github.com/eth-clients/mainnet) layout that
[`gu-corp/sandbox1`](https://github.com/gu-corp/sandbox1) uses — but every value
in them that defines the chain is still `TBD`, and they carry a
`DRAFT — NOT ACTIVE` banner saying so.

## Contents

| File | Contents |
|---|---|
| [`genesis.json`](genesis.json) | Execution genesis with the merge fields added |
| [`presets.md`](presets.md) | Which spec preset the config is based on, and why |

## `genesis.json`

The merge does not change the genesis block — it only adds fork-activation
fields to `config`. This file is therefore the live
[`../metadata/genesis.json`](../metadata/genesis.json) plus exactly five keys:

```
terminalTotalDifficulty   shanghaiTime   cancunTime   pragueTime   depositContractAddress
```

[`../scripts/check_pos_migration.sh`](../scripts/check_pos_migration.sh)
enforces that invariant in CI: if the draft ever diverges from the genesis the
chain is actually running — a changed `alloc`, a moved fork block, an unexpected
config key — the build fails.

`depositContractAddress` is now real. The other four are still the string
`"TBD"` — deliberately, rather than `null`. It keeps the file valid JSON, and
geth hard-fails parsing a string into `*big.Int` / `*uint64` instead of quietly
reading `null` as "terminal total difficulty zero", which would mean *merge
immediately*. So the file still cannot be used by accident.

## What is still open

The deposit contract is **deployed** — `0x2d871682c97d93401F0348835af88A1D98ed6564`,
block `19262622`. That is the first migration parameter to become real. It
changes nothing yet: `get_deposit_count()` is `0`, so no validator has
deposited, and every parameter that depends on the deposits is still open. See
[`../README.md#deposit-contract`](../README.md#deposit-contract) for how the
address and block were verified.

Every value marked `TBD` in [`../metadata/config.yaml`](../metadata/config.yaml)
is undecided — 14 of them, alongside 4 forks parked at the max-uint64 stub.

The undecided values include everything that defines the chain's identity —
`TERMINAL_TOTAL_DIFFICULTY`, `MIN_GENESIS_TIME`,
`MIN_GENESIS_ACTIVE_VALIDATOR_COUNT`, the fork epochs — plus the whole set of
fork versions. Both other networks encode their chain id in the low bytes:
mainnet uses `0x0X000051` (`81 = 0x51`) and sandbox1 uses `0x0X000539`
(`1337 = 0x539`). The testnet equivalent would be `0x0X002761`
(`10081 = 0x2761`), but that has not been decided.

Alongside those, in `../metadata/`:

| File | State |
|---|---|
| [`../metadata/bootstrap_nodes.yaml`](../metadata/bootstrap_nodes.yaml) | empty list — no consensus-layer bootnodes exist yet |
| `../metadata/genesis.ssz` | absent — see below |

Also still to be produced: the beacon genesis details — fork digest, validators
root, genesis time.

### Why there is no `genesis.ssz`

`genesis.ssz` is the beacon chain genesis state, derived from the deposit
contract's contents at a chosen block. The contract now exists, but it is
empty — `get_deposit_count()` is `0`, so there is nothing to derive a state
from — and its remaining inputs are still unset: `MIN_GENESIS_TIME`,
`MIN_GENESIS_ACTIVE_VALIDATOR_COUNT` and `GENESIS_FORK_VERSION` are all `TBD`.

It is the one missing file that gets no `TBD` placeholder, because there is no
such thing as a placeholder for it. A generated stand-in would carry a
`genesis_validators_root` that is pure fiction — and that value is what every
signature on the network is domain-separated by. Shipping one publicly is worse
than shipping nothing: clients would compute fork digests that match no peer,
and anything signed against it would be signed under the wrong domain. It gets
committed when the migration parameters are real, not before.

## Do not use

None of this can be verified — there is no beacon chain to check it against.
Do not point a production beacon node at
[`../metadata/config.yaml`](../metadata/config.yaml), and do not `geth init` a
production node with the `genesis.json` in this directory.
