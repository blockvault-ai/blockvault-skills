# BlockSwap — Route planning (multi-hop)

When the goal spans multiple chains, plan the route as a sequence of quotes, not one quote. Each hop is still a single `/quote` — you chain them because the goal spans multiple chains and gas must be funded along the way.

## Trace the route before executing

For each hop, note: source chain, token, destination chain, and whether the source has native gas (detect it via `reference/gas.md`).

The examples below use Polygon/Ethereum/Base for concreteness — the same pattern applies to any chains in `supported_blockchains`. Substitute the actual chains from the user's wallet.

## Example 1 — everything on Base

User wants everything on Base, holds USDC on Ethereum (no ETH gas) and POL on Polygon:

1. **Fund Ethereum gas first.** Bridge POL from Polygon → ETH on Ethereum (`to_chain` = ethereum). Polygon has POL gas, so this hop works.
2. **Bridge USDC** Ethereum → Base (`to_chain` = base). Ethereum now has ETH gas from step 1.
3. **Optionally sweep the leftover ETH** Ethereum → Base if the user wants zero dust.

## Example 2 — fund gas to run a transaction

User wants to open a Uniswap LP position on Base, but has no ETH gas on Base. They hold POL on Polygon (with gas) and USDC on Ethereum (with gas):

1. **Fund Base gas.** Bridge POL from Polygon → ETH on Base (`to_chain` = base). Polygon has POL gas, so this hop works.
2. **Bridge USDC** Ethereum → Base (`to_chain` = base). Ethereum has ETH gas, so this hop works.
3. **Now open the LP position** on Base — both tokens and gas are present.

## Rules

- **Gas is the ordering constraint.** A hop whose source chain has no gas must be preceded by a hop that funds that gas. Trace the chain of gas dependencies first, then execute in that order. Detect and fund gas via `reference/gas.md`.
- **Each hop is one `/quote`.** Never split a single hop into manual sub-swaps.
- **Confirm the full route with the user** before executing any hop (list the hops and their order).
- **Re-check `get_balance` after each hop** — balances and gas change after every bridge.