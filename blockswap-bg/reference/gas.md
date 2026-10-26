# BlockSwap — Native gas management

Every transaction needs native gas on its source chain. A token balance is not enough. This file is the single source of truth for detecting and funding gas.

## Contents
- Identify the native gas token
- Detect chains with gas
- Fund gas
- Checklist

### Never spend 100% of native gas

When `token_in` is the native gas token, the swap/bridge itself consumes gas on the source chain. Never quote the full balance — leave at least 2× the native entry's `estimatedTxFee` as a buffer.

## Identify the native gas token

The native gas token is the entry in `get_balances` whose `contractAddress` is `null`. Do NOT hardcode a symbol — read it from the wallet.

Call `get_balances` with:
- **chain_id**: Number, Optional. Omit (or `0`) to read ALL chains.

Find the entry on the source chain whose `contractAddress` is `null`. Its `symbol` is the native gas token.

## Detect chains with gas

Call `get_balances` with:
- **chain_id**: Number, Required. The chain id to check.

Read the native entry's `hasGasForTx`. It is computed by the tool from the live gas price (`gasPriceGwei`) and a conservative fee for a heavy DeFi transaction (`estimatedTxFee`, already padded for gas spikes and rollup L1 fees); `affordableTxs` is how many such transactions the balance pays for.

- `hasGasForTx: true` → the chain HAS enough gas. Proceed, even if the balance looks small (L2 fees are a few cents).
- `hasGasForTx: false` (or the entry is missing) → the chain has no usable gas; pick a funding case below.

Never estimate gas costs yourself and never call a balance "insufficient" while `hasGasForTx` is `true`. If a transaction fails with `errorCode` `INSUFFICIENT_GAS` (`insufficient funds for intrinsic transaction cost`), that error is authoritative: the chain has no usable gas. Do NOT retry any transaction on that chain — not even a smaller swap to buy gas, it needs gas too. Fund it from another chain (Case 2) or stop (Case 3).

## Fund gas — three cases

### Case 1 — has gas (`hasGasForTx: true`) but few `affordableTxs` + holds a non-native token

Optional top-up before a multi-transaction step. Same-chain swap a slice of the non-native token into the native gas token:
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

### Case 2 — target chain has `hasGasForTx: false`, another chain has `hasGasForTx: true`

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

### Case 3 — no chain has `hasGasForTx: true`

The wallet can only be funded from outside. Do not attempt any transaction (swap, bridge, claim, approval) and do not call `notify_user`. Call `end_task` with:

- **status**: `"blocked"`
- **reason**: e.g. `"no native gas on any supported chain"`
- **summary**: which chains you checked and what the user must fund (native token per chain, wallet address).

The bot is paused and the user is notified once.

## Checklist

Copy and track:

- [ ] Identify native gas token via `get_balances` (`contractAddress: null`)
- [ ] Check `hasGasForTx` on the native entry via `get_balances` (not just non-zero token balances)
- [ ] If `hasGasForTx` is `false`, pick the funding case (2/3) above
- [ ] After an `INSUFFICIENT_GAS` error, never retry on that chain
- [ ] If `token_in` is native gas, leave a fee buffer (never spend 100%)
- [ ] Re-check `get_balances` after each funding hop

Bridges spend gas on BOTH ends (send + claim). Keep native gas on the destination too if a follow-up step needs it.
