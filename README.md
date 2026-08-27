# JOC testnet (JOCT) metadata

Japan Open Chain testnet — chain ID `10081`, Clique proof-of-authority,
5-second blocks.

This directory contains the chain metadata, configuration parameters and
genesis information for JOCT. Everything in [`metadata/`](metadata/) is
verified against the live network by CI.

> **The p2p network id is `361257328`, not the chain id `10081`.** Geth
> defaults `--networkid` to the genesis `chainId`, so it must be passed
> explicitly or the node will never handshake with a testnet peer.
> `chain.json` carries `networkId: 10081` because that field follows the
> EIP-155 chain-list convention; the value geth needs is below and in
> [`metadata/genesis_details.yaml`](metadata/genesis_details.yaml).

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

| File | Contents |
|---|---|
| [`metadata/genesis.json`](metadata/genesis.json) | Execution-layer genesis. Feed to `geth init`. |
| [`metadata/genesis_details.yaml`](metadata/genesis_details.yaml) | Genesis hash, state root, clique params, fork blocks, allocation summary |
| [`metadata/enodes.yaml`](metadata/enodes.yaml) | Execution-layer bootnodes |
| [`metadata/chain.json`](metadata/chain.json) | EIP-155 chain metadata — id, RPC endpoints, native currency, explorer |

Proof-of-stake configuration for the planned PoA → PoS migration is a draft and
lives in [`pos-migration/`](pos-migration/), outside the CI-verified `metadata/`
directory.

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
