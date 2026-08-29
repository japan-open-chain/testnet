# JOC testnet (JOCT) metadata

Japan Open Chain testnet — chain ID `10081`, Clique proof-of-authority,
5-second blocks.

This directory contains the chain metadata, configuration parameters and
genesis information for JOCT, execution and consensus layer both. The beacon
chain is live — genesis fired 2026-08-28T15:16:25Z.

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

## Deposit contract

```yaml
address:    0x2d871682c97d93401F0348835af88A1D98ed6564
block:      19262622          # 2026-08-28T07:09:25Z
block_hash: 0xb948dc02090568aef6337f5761eefb043cf9888d23a70ff845bcffa560a2b980
```

It holds the 2 deposits that seeded the beacon chain (`get_deposit_count()`,
blocks `19267254` and `19267265`).

That it is a genuine deposit contract was checked at deployment, not assumed:
while still empty it answered `get_deposit_root()` with `0xd70a2347…7e5e`, the
canonical empty-tree root, reproducible offline from the contract's own
zero-hash ladder and matching nothing else:

```bash
python3 -c "
import hashlib; sha=lambda b: hashlib.sha256(b).digest()
z=[b'\x00'*32]
for i in range(32): z.append(sha(z[i]+z[i]))
n=b'\x00'*32
for h in range(32): n=sha(n+z[h])
print('0x'+sha(n+(0).to_bytes(8,'little')+b'\x00'*24).hex())"
# 0xd70a234731285c6804c2a4f56711ddb8c82c99740f207854891028af34e27e5e
```

### Finding the deployment block took some care

The obvious route does not work. This node prunes state — `eth_getCode` at the
deployment block fails with `missing trie node`, so a binary search over
history is out (it succeeds only at genesis and within roughly the last 128
blocks).

The explorer names creation transaction `0x25887f79…32e7`, and the node's own
receipt confirms that transaction and its block. But that receipt's
`contractAddress` is `0x4c8cbfac…b61d`, a **factory**, not the deposit
contract, and the deposit contract is not among the addresses that emitted logs
in the transaction. It was created by an internal `CREATE`, which no public RPC
on this node will show.

So it was confirmed arithmetically instead. A `CREATE` address is
`keccak256(rlp([creator, nonce]))[12:]`, and from that factory:

```
nonce 1 -> 0xb2bedb50…8aa9   [emitted a log in the tx]
nonce 2 -> 0x09297165…da16   [emitted a log in the tx]
nonce 3 -> 0x2d871682…6564   <- the deposit contract
nonce 4 -> 0xb85a6df7…8b1e   [emitted a log in the tx]
```

Nonces 1, 2 and 4 are three of the addresses that emitted logs in that
transaction, per the node's own receipt. Nonces are sequential, so the nonce-3
creation cannot have happened later than the nonce-4 one — the deposit contract
was created in that transaction, in block `19262622`. Only the list of internal
creations came from the explorer; everything load-bearing came from the node.

Reproduce the derivation with:

```bash
cast compute-address --nonce 3 0x4c8cbfaca8675deb6457a022cc7b3f4f2b41b61d
```

## Files

The layout follows [eth-clients](https://github.com/eth-clients/mainnet), the
same one [`gu-corp/sandbox1`](https://github.com/gu-corp/sandbox1) uses.

| File | Contents |
|---|---|
| [`metadata/genesis.json`](metadata/genesis.json) | Execution-layer genesis. Feed to `geth init`. |
| [`metadata/genesis_details.yaml`](metadata/genesis_details.yaml) | Genesis hash, state root, clique params, fork blocks, allocation summary |
| [`metadata/config.yaml`](metadata/config.yaml) | Beacon chain config. Feed to a consensus client. |
| [`metadata/genesis.ssz`](metadata/genesis.ssz) | Beacon genesis state. Feed to a consensus client. |
| [`metadata/chain.json`](metadata/chain.json) | EIP-155 chain metadata — id, RPC endpoints, native currency, explorer |
| [`metadata/enodes.yaml`](metadata/enodes.yaml) | Execution-layer bootnode enode URLs |
| [`metadata/bootstrap_nodes.yaml`](metadata/bootstrap_nodes.yaml) | Consensus-layer bootnode ENRs |
| [`metadata/deposit_contract.txt`](metadata/deposit_contract.txt) | Deposit contract address |
| [`metadata/deposit_contract_block.txt`](metadata/deposit_contract_block.txt) | Eth1 block it was deployed in |
| [`metadata/deposit_contract_block_hash.txt`](metadata/deposit_contract_block_hash.txt) | Hash of that block |
| [`scripts/discv5_probe.py`](scripts/discv5_probe.py) | Proves a discv5 node is alive by making it answer `WHOAREYOU` |

### How the bootnodes are checked

The two layers are held to **different** standards, and it is worth knowing
which is which.

The consensus bootnodes in
[`metadata/bootstrap_nodes.yaml`](metadata/bootstrap_nodes.yaml) are **proven**
alive. A TCP connect only shows a port is open, and discovery runs over UDP
where a connect shows nothing at all — there is no handshake, so `nc -zu`
reports success against a black hole. Instead
[`scripts/discv5_probe.py`](scripts/discv5_probe.py) sends a real discv5
packet. Its masking key is the *recipient's* node id, keccak256 of the public
key in the ENR, so only a node that agrees its id is that can unmask it — and
it must answer `WHOAREYOU`. Getting that reply binds the key in the record to
whatever is actually listening.

The execution bootnodes in [`metadata/enodes.yaml`](metadata/enodes.yaml) get a
**weaker** check. There is no cheap equivalent of the `WHOAREYOU` trick for
devp2p: proving a node id belongs to an address needs a full RLPx handshake,
ECIES over secp256k1 ECDH. So they are only checked for a well-formed URL and a
port that accepts TCP — enough to catch rot, not enough to prove identity.

`scripts/check_bootnodes.sh` runs both, and CI runs it weekly, so a bootnode
going away surfaces on its own.

## Beacon chain

**Live.** `MIN_GENESIS_TIME` elapsed before any deposit, so genesis fired at
the eth1 block carrying the 2nd deposit (block `19267265`) plus
`GENESIS_DELAY` — confirmed both by computing that sum and by the node.

| | |
|---|---|
| `genesis_time` | `1787930185` — 2026-08-28T15:16:25Z |
| `genesis_validators_root` | `0x93531a50099acd3f91c38790b944b5e034f0f35c48ce3e06d6dacc782c78b19d` |
| Validators at genesis | `2` (= `MIN_GENESIS_ACTIVE_VALIDATOR_COUNT`) |
| `GENESIS_FORK_VERSION` | `0x00002761` — low bytes are chain id `10081` |
| Forks scheduled | Altair epoch 5, Bellatrix epoch 10 |
| Forks disabled | Capella onwards, at `2**64-1` |
| `TERMINAL_TOTAL_DIFFICULTY` | `2**64-1` — merge not scheduled |

With 2 validators the chain finalizes, but finality stops if either drops —
blocks would still be proposed, and never finalized. Bellatrix at
epoch 10 does not merge the chain — that needs `TERMINAL_TOTAL_DIFFICULTY`,
which is parked at the max-uint64 stub. The execution layer stays Clique PoA
until a real value is set.

### `PRESET_BASE` is `gnosis`

Consensus clients compile the preset in; it cannot be overridden from a config
file. JOC produces a block every 5 seconds, matching Gnosis Chain rather than
mainnet's 12s. The preset sets `SLOTS_PER_EPOCH` to 16, so an epoch is 80
seconds, not 384 — worth remembering when reading the fork epochs above.

- https://github.com/gnosischain/specs/tree/master/consensus/preset/gnosis
- https://github.com/sigp/lighthouse/tree/stable/consensus/types/presets/gnosis

### `genesis.ssz` comes from the node, not from a script

[`metadata/genesis.ssz`](metadata/genesis.ssz) is the beacon node's own genesis
state, taken from `/eth/v2/debug/beacon/states/genesis` (Lighthouse v7.0.1) —
not rebuilt from the eth1 deposits, which is the kind of reconstruction the
sandbox1 `berlinBlock` lesson warns against. Its decoded header (`genesis_time`,
`genesis_validators_root`, fork version `0x00002761`, slot 0) matches the
node's `/eth/v1/beacon/genesis`, and every key
[`config.yaml`](metadata/config.yaml) sets that the node reports in
`/eth/v1/config/spec` comes back with the same value.

`genesis_validators_root` domain-separates every signature on the network, so a
wrong file here would make clients compute fork digests matching no peer.

## Endpoints

| | |
|---|---|
| RPC | `https://rpc-1.testnet.japanopenchain.org:8545` (all endpoints in [`metadata/chain.json`](metadata/chain.json)) |
| Explorer | https://explorer.testnet.japanopenchain.org |

## Run a node

The execution layer is Clique PoA and follows the head on its own; the beacon
chain runs alongside it until the merge is scheduled.

### Execution layer

```bash
geth init --datadir ~/.joct metadata/genesis.json
geth --datadir ~/.joct --networkid 361257328 --syncmode full \
     --bootnodes "$(sed -n 's/^-[[:space:]]*\(enode:\/\/[^[:space:]#]*\).*$/\1/p' \
                    metadata/enodes.yaml | paste -sd, -)"
```

`--networkid` must be passed explicitly: geth would otherwise default it to the
genesis `chainId`, `10081`, not the `361257328` this network uses.

### Consensus layer

```bash
lighthouse beacon_node \
  --testnet-dir metadata \
  --boot-nodes "$(sed -n 's/^-[[:space:]]*\(enr:[^[:space:]#]*\).*$/\1/p' \
                  metadata/bootstrap_nodes.yaml | paste -sd, -)" \
  --execution-endpoint http://localhost:8551 \
  --execution-jwt ~/.joct/jwt.hex
```

`--testnet-dir metadata` picks up `config.yaml` and `genesis.ssz` from this
repo. Other clients want the same two files under different flag names.

### Verify

```bash
scripts/verify_genesis.sh    # geth init reproduces the genesis hash
scripts/check_bootnodes.sh   # published bootnodes are well-formed and answer
```

## License

CC0 1.0 Universal. See [`LICENSE`](LICENSE).
