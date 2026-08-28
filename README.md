# JOC testnet (JOCT) metadata

Japan Open Chain testnet — chain ID `10081`, Clique proof-of-authority,
5-second blocks.

This directory contains the chain metadata, configuration parameters and
genesis information for JOCT. Everything in [`metadata/`](metadata/) is
verified against the live network by CI.

> **The p2p network id is `361257328`, not the chain id `10081`.** Geth
> defaults `--networkid` to the genesis `chainId`, so it must be passed
> explicitly or the node will never handshake with a testnet peer. Every file
> here carries `361257328` — [`chain.json`](metadata/chain.json),
> [`genesis_details.yaml`](metadata/genesis_details.yaml) and
> `DEPOSIT_NETWORK_ID` in [`config.yaml`](metadata/config.yaml).

### A note on `chain.json`'s `networkId`

The upstream [ethereum-lists/chains](https://github.com/ethereum-lists/chains)
entry for JOCT (`eip155-10081.json`) publishes `networkId: 10081`. **That is
wrong**, and this repo deliberately disagrees with it: the field is the p2p
network id, and the network answers `net_version` with `361257328`.

```bash
curl -sS -X POST -H 'Content-Type: application/json' \
  --data '{"jsonrpc":"2.0","id":1,"method":"net_version","params":[]}' \
  https://rpc-3.testnet.japanopenchain.org | jq -r .result
# 361257328
```

Mirroring `chainId` into `networkId` is not a chain-list convention — it is
just what most entries look like, because for most chains the two values are
equal. Of the 2726 entries in the registry, **48 record a `networkId` that
differs from their `chainId`**: Ethereum Classic is `chainId 61 / networkId 1`,
Camino C-Chain is `500 / 1000`, Mordor is `63 / 7`. The schema expects the real
value. `gu-corp/sandbox1` sets it the same way, `networkId: 1456260212` against
chain ID `1337`.

The upstream entry still needs a PR to match.

## Genesis information

```yaml
chain_id: 10081
network_id: 361257328         # geth --networkid — differs from chain_id
genesis_time: 1543235253      # 2018-11-26T12:27:33Z
genesis_hash: 0x0fb7b4779aae36dc557227283f182bc9a3b232c07fae4a2553734c5817df1d06
genesis_state_root: 0x803cd322251f777a2e5ad83e1177195650fc0d358b526fedcd74abdc22fb13f0
gas_limit: 470000000
clique:
  period: 5
  epoch: 30000
  genesis_signers:
    - 0x34facaa6747bec72f8119f79cc78a7b6f0539b6b
berlin_block: 1330295
london_block: 1330295
```

The full machine-readable form, including every fork block and the allocation
summary, is [`metadata/genesis_details.yaml`](metadata/genesis_details.yaml).
Reproduce it from `genesis.json` with `scripts/verify_genesis.sh`.

## Genesis allocation

256 placeholder accounts at `0x00…00` – `0x00…ff` holding 1 wei each, plus a
single funded account:

```
0xe546882a744f43d20b84e8a0e5a6ac9f83f9d6e0   1,000,000,000 JOCT
```

## Files

The layout follows [eth-clients](https://github.com/eth-clients/mainnet), the
same one [`gu-corp/sandbox1`](https://github.com/gu-corp/sandbox1) uses.

Everything the live PoA chain runs on is filled in and checked by CI:

| File | Contents |
|---|---|
| [`metadata/genesis.json`](metadata/genesis.json) | Execution-layer genesis. Feed to `geth init`. |
| [`metadata/genesis_details.yaml`](metadata/genesis_details.yaml) | Genesis hash, state root, clique params, fork blocks, allocation summary |
| [`metadata/enodes.yaml`](metadata/enodes.yaml) | Execution-layer bootnode enode URLs |
| [`metadata/chain.json`](metadata/chain.json) | EIP-155 chain metadata — id, RPC endpoints, native currency, explorer |

The consensus-layer files exist so the layout is complete, but JOCT has no
beacon chain yet, so their contents are placeholders. **Nothing below is
usable** — see [`pos-migration/`](pos-migration/) for what is still open:

| File | State |
|---|---|
| [`metadata/config.yaml`](metadata/config.yaml) | Beacon chain config. Carries a `DRAFT — NOT ACTIVE` banner; 15 values are `TBD`. |
| [`metadata/bootstrap_nodes.yaml`](metadata/bootstrap_nodes.yaml) | Consensus-layer bootnode ENRs — empty, none published |
| [`metadata/deposit_contract.txt`](metadata/deposit_contract.txt) | Deposit contract address — `TBD`, not deployed |
| [`metadata/deposit_contract_block.txt`](metadata/deposit_contract_block.txt) | Eth1 block it was deployed in — `TBD` |
| [`metadata/deposit_contract_block_hash.txt`](metadata/deposit_contract_block_hash.txt) | Hash of that block — `TBD` |
| `metadata/genesis.ssz` | Beacon genesis state — **absent**, and deliberately not stubbed |

`genesis.ssz` is the one gap with no placeholder: its `genesis_validators_root`
domain-separates every signature on the network, so a fabricated one would make
clients compute fork digests that match no peer. See
[`pos-migration/README.md`](pos-migration/README.md#why-there-is-no-genesisssz).

The execution-side draft of the merge — `genesis.json` with the fork-activation
fields added — stays in [`pos-migration/`](pos-migration/), where
`scripts/check_pos_migration.sh` holds it to the live genesis.

## Endpoints

| | |
|---|---|
| RPC | `https://rpc-1.testnet.japanopenchain.org:8545` (all endpoints in [`metadata/chain.json`](metadata/chain.json)) |
| Explorer | https://explorer.testnet.japanopenchain.org |

## Run a node

```bash
geth init --datadir ~/.joct metadata/genesis.json
geth --datadir ~/.joct --networkid 361257328 --syncmode full \
     --bootnodes "$(sed -n 's/^-[[:space:]]*\(enode:\/\/[^[:space:]#]*\).*$/\1/p' \
                    metadata/enodes.yaml | paste -sd, -)"
```

## License

CC0 1.0 Universal. See [`LICENSE`](LICENSE).
