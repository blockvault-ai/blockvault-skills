# Uniswap Pools — JMESPath queries (Background)

The pool recommendation response is a JSON object with a top-level `pools` array. Each pool has: `token0`, `token1`, `token0Address`, `token1Address`, `token0Decimals`, `token1Decimals`, `protocol`, `fee`, `poolReferenceIdentifier`, `tickBounds`, `apy`, `risk`, `score`, `reason`.

## Present the top pools to the user

After `bash -o pools.json`, run `text_editor` with `command: "query"` to extract a compact list of the top 5 pools:

- **path**: `artifacts/pools.json`
- **expression**: `pools[:5].[token0, token1, apy, score, risk]`

This returns an array of `[token0, token1, apy, score, risk]` tuples for the top 5 pools.

## Pick the best pool

To select the single best pool (highest score), use:

- **expression**: `pools[0].[token0, token1, token0Address, token1Address, token0Decimals, token1Decimals, protocol, fee, poolReferenceIdentifier, tickBounds, apy, risk, score]`

This returns the full field set of the top-ranked pool, which you need to open the position in Step 4.

## Filter by risk

To list only Low/Medium risk pools (the `risk` field is `"Low risk"`, `"Medium risk"`, or `"Higher risk"`):

- **expression**: `pools[?risk == 'Low risk' || risk == 'Medium risk'].[token0, token1, apy, score, risk]`
