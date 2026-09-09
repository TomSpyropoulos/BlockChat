# BlockChat

A peer-to-peer blockchain for coins and chat messages, with Proof-of-Stake consensus. Every node is a Dockerized FastAPI service that holds a wallet, gossips over HTTP, and independently arrives at the same chain.

Built for the Distributed Systems course at NTUA.

## 🏗️ How it works

Each node runs one FastAPI app (`api.py`) on port 8000 inside its own container. There is no coordinator and no shared database. Nodes learn about each other once, at bootstrap, and from then on every node holds the full chain and the full wallet state.

**Bootstrap.** Node 0 is the bootstrap node. Every other node starts, generates an ECDSA keypair, and POSTs its public key and address to `/get\_id`. When the last expected node has joined, the bootstrap node mints a genesis block crediting itself 5000 BCC, then a second block transferring 1000 BCC to every other node, and broadcasts the chain and the peer mapping to everyone.

**Identity.** A wallet is an ECDSA (SECP256k1) keypair generated at startup. The hex-encoded public key *is* the address, so there is no registry to look anything up in.

**Gossip.** `broadcaster.py` fans out over a full mesh. Transactions, blocks and the initial chain each go to every known peer as an HTTP POST.

**Minting.** Every node holds the same transaction pool. When the pool reaches capacity, each node independently computes who the validator should be. Exactly one of them concludes it is itself, builds the block, and broadcasts it. The others validate and append.

## 🎲 Consensus

Validator selection is stake-weighted and fully deterministic. Given the previous block's hash, every node derives the same validator without exchanging a single message:

```python
random.seed(last\_hash)
validator\_bag = \[addr for addr, w in validators.items() for \_ in range(int(w.stake))]
validator\_bag.sort()          # identical ordering on every node
return random.choice(validator\_bag)
```

Seeding on the previous hash makes the draw reproducible, sorting the bag makes it order-independent across nodes, and repeating each address `stake` times makes it proportional to stake. Staking is itself a transaction: send to the reserved address `"0"` and the amount becomes your stake.

A received block is accepted only if the validator is the one the receiver independently derived, every transaction verifies, the block hash recomputes correctly, and the previous hash matches the local chain tip.

## 💸 Transactions

Two kinds, both signed and both fee-bearing:

||Amount|Fee|
|-|-|-|
|**Coins**|what you send|3% to the validator|
|**Message**|1 BCC per character|3% to the validator|
|**Stake**|amount staked|none|

Each transaction is a SHA-256 digest of sender, receiver, type, amount, message and nonce, signed with the sender's private key. Validation checks the signature, then the sender's *pending* balance rather than their settled balance, so several in-flight transactions cannot collectively overdraw an account. A monotonic per-wallet nonce rejects replays.

## 🚀 Running it

Requires Docker. From `src/`:

```bash
./dorun.sh 4 5
```

The first argument is the highest node index, so `4` starts 5 nodes. The second is block capacity, the number of transactions that triggers a mint. Node *i* is reachable on `localhost:800i`.

Then attach the CLI to any node:

```bash
python cli.py -c 2
```

It shows your balance and stake, and lets you send coins, send a message, set your stake, or inspect the last block.

## 📊 Benchmarks

```bash
python benchmark.py
```

Runs 5-node and 10-node networks at block capacities of 5, 10 and 20, replaying the canned transaction sets in `input\_5/` and `input\_10/`, and reports total time, transactions per second and mean block time for each.

It then runs a fairness check: every node stakes 10 except one, which stakes 100, and the final balances show whether validator selection actually favours stake in the proportion it claims to. That run is repeated with each node taking the large stake in turn, so the result is not an artifact of one node's position in the network.

## ⚙️ Configuration

Set per container by `dorun.sh`:

|Variable|Meaning|
|-|-|
|`IP\_ADDRESS`|this container's hostname on the Docker network|
|`BOOTSTRAP`|hostname of the bootstrap node|
|`CAPACITY`|transactions per block|
|`NUM\_NODES`|expected node count, triggers bootstrap once reached|

## 📁 Layout

```
src/
├── api.py               FastAPI routes and node startup
├── node.py              orchestration: wallets, minting, bootstrap
├── block.py             block structure, hashing, validator selection
├── blockchain.py        the chain and its validation
├── transaction.py       signing, verification, fees
├── transaction\_pool.py  pending transactions, capacity trigger
├── wallet.py            keys, balances, stake, nonce
├── broadcaster.py       HTTP gossip to peers
├── cli.py               interactive client
├── benchmark.py         throughput and fairness experiments
└── dorun.sh             builds the image and starts N containers
```

## 🔌 API

|Endpoint|Purpose|
|-|-|
|`POST /create\_transaction`|send coins or a message|
|`POST /set\_stake`|change your stake|
|`GET /balance`, `GET /stake`|wallet state|
|`GET /view\_last\_block`|chain tip|
|`GET /get\_mapping`|known peers|
|`GET /get\_avg\_block\_time`|mean block interval|
|`POST /receive\_transaction`|peer gossip|
|`POST /receive\_block`|peer gossip|
|`POST /receive\_blockchain`|bootstrap sync|
|`POST /receive\_mapping`|bootstrap sync|
|`POST /get\_id`|join the network|



