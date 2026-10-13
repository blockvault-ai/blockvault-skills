# BlockSwap — Native gas management

Every transaction needs native gas on its source chain. A token balance is not enough. This file is the single source of truth for detecting and funding gas.

## Contents
- Identify the native gas token
- Detect chains with gas
- Fund gas
- Checklist

### Never spend 100% of native gas

When `token_in` is the native gas token, the swap/bridge itself consumes gas on the source chain. Never quote the full balance — leave a buffer for the transaction fee.

## Identify the native gas token

The native gas token is the entry in `get_balances` whose `contractAddress` is `null`. Do NOT hardcode a symbol — read it from the wallet.

Call `get_balances` with:
- **chain_id**: Number, Optional. Omit (or `0`) to read ALL chains.

Find the entry on the source chain whose `contractAddress` is `null`. Its `symbol` is the native gas token.

## Detect chains with gas

Call `get_balances` with:
- **chain_id**: Number, Required. The chain id to check.

If the native gas entry's `balance` is `"0"` (or the entry is missing), the chain has no native gas.

## Fund gas — three cases

### Case 1 — has some gas + holds a non-native token

Same-chain swap a slice of the non-native token into the native gas token:
   `src_id` = source chain,
   `token_in` = non-native token symbol
   `token_out` = native gas token symbol.
   `amount` = the amount of the non-native token to swap (decimal units).

   ```bash
   # same-chain swap
   curl -s -X POST "https://402.blockvault.ai/api/v1/blockswap/quote" \
   -H "Content-Type: application/json" \
   -d '{"chain":"<src_id>","token_in":"<SYMBOL>","token_out":"<SYMBOL>","amount":"<dec>","slippage":0.005,"swapper":"<addr>","sign":false}'
   ```

### Case 2 — zero gas

A same-chain swap is impossible (you cannot pay its own fee). Fund from another chain that HAS gas.

1. Identify a source chain with native gas and a non-native token the wallet holds.
2. Quote a cross-chain bridge from that chain into the target chain's native gas token:
   `src_id` = source chain with gas
   `dst_id` = target chain with zero gas
   `token_in` = non-native token symbol on source chain
   `token_out` = native gas token symbol on target chain
   ```bash
   # cross-chain bridge
   curl -s -X POST "https://402.blockvault.ai/api/v1/blockswap/quote" \
   -H "Content-Type: application/json" \
   -d '{"chain":"<src_id>","to_chain":"<dst_id>","token_in":"<SYMBOL>","token_out":"<SYMBOL>","amount":"<dec>","slippage":0.005,"swapper":"<addr>","sign":false}'
   ```

### Case 3 — no chain has gas

Notify the user and stop. Do not attempt any transaction.

## Checklist

Copy and track:

- [ ] Identify native gas token via `get_balances` (`contractAddress: null`)
- [ ] Check gas balance via `get_balances` (not just non-zero token balances)
- [ ] If zero gas, pick the funding case (1/2/3) above
- [ ] If `token_in` is native gas, leave a fee buffer (never spend 100%)
- [ ] Re-check `get_balances` after each funding hop

Bridges spend gas on BOTH ends (send + claim). Keep native gas on the destination too if a follow-up step needs it.
