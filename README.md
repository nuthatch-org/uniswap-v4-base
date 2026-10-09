# Uniswap V4 Base nest

A [Nuthatch](https://github.com/nuthatch-org/nuthatch) nest for the Uniswap V4 Base deployment
used by Graph deployment `Qmbsc6XQWbiv4DfLVfaNciScqYLyDWUYjWzrFBbzzmRsMB`.

It is a standalone Base nest, not a replacement for
[uniswap-v4-ethereum](https://github.com/nuthatch-org/uniswap-v4-ethereum). Nuthatch nests have
one chain each, so the two can be mounted in the same multichain runtime without blending their
cursors, data, or RPC credentials.

## Run it

```sh
cargo install --git https://github.com/nuthatch-org/nuthatch nuthatch
git clone https://github.com/nuthatch-org/uniswap-v4-base
cd uniswap-v4-base

# Keep your authenticated Base archive RPC outside the repository.
export RPC=https://your-base-archive-node/

# A short, useful index. Restart after seal-direct so the SQL views load.
nuthatch dev --seal-direct --backfill 10000 --rpc "$RPC"
nuthatch dev --rpc "$RPC"
```

For a full history, omit `--backfill`. This deployment begins at block `25,350,988`, so use an
archive-capable endpoint. The repository deliberately contains no provider URLs or credentials.

## What it indexes

| Contract | Address | Start block | Events |
| --- | --- | ---: | --- |
| PoolManager | `0x498581ff718922c3f8e6a244956af099b2652b2b` | 25,350,988 | Initialize, ModifyLiquidity, Swap |
| PositionManager | `0x7c5f5a4bbd8fd63184577525326123b519429bdc` | 25,350,993 | Subscription, Unsubscription, Transfer |
| ArrakisHookFactory | `0xef129a430032c8183aba158c1a70799e3b840df9` | 28,450,225 | LogCreatePrivateHook |

The seven raw tables are complete for those event selections. `block_timestamp`, transaction hash,
block hash, log index, and emitting address are present on every row.

The views provide pool keys, hook adoption, pool activity, and liquidity summaries. V4 swap amounts
are signed from the pool's perspective, so the activity view uses absolute values rather than a
misleading signed sum. `currency0 = 0x0000000000000000000000000000000000000000` means native ETH,
not a missing token.

## Graph compatibility, stated plainly

This nest reproduces the Graph deployment's on-chain event inputs. It does **not** claim entity-for-
entity Graph parity: that mapping also performs `eth_call` token metadata reads and hook calls to
derive token names, decimals, supply, valuation, and time-series entity fields. Nuthatch's authored
SQL is event-derived and makes no hidden RPC calls. Those fields require a separately versioned
enrichment layer and should not be inferred from this dataset.

## Verification

```sh
# Stop the dev server first. Checks read the local store directly.
nuthatch check --dir .
```

Two Base-specific fixtures cover blocks `50,033,081..50,042,483`, first indexed on 2026-08-16 with
an authenticated archive endpoint. They assert pool-key decoding and V4 signed-swap invariants. The
window contained 632 pool initialisations and 68,676 swaps; every non-overflow swap had either an
opposite-signed pair or a zero leg. Re-record fixtures only after an intentional decoder or query
change:

```sh
nuthatch check --dir . --update
```

## Layout

```text
nuthatch.toml  chain, contract, ABI, and event selection
abis/          vendored source ABIs
views/         event-derived SQL views
checks/        bounded, recorded SQL invariants
semantic.toml  table semantics for the HTTP and MCP surfaces
schema.json    generated registry schema
```

After changing `nuthatch.toml`, regenerate generated metadata with:

```sh
nuthatch schema --dir .
```
