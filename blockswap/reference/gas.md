# BlockSwap — Native gas management

Every transaction needs native gas on its source chain. A token balance is not enough. This file is the single source of truth for detecting and funding gas.

## Contents
- Identify the native gas token
- Detect chains with gas
- Fund gas
- Checklist

### Never spend 100% of native gas

When `token_in` is the native gas token, the swap/bridge itself consumes gas on the source chain. Never quote the full balance — leave a buffer for the transaction fee.

1. Call `estimate_fee` to size the fee:
   - **function**: "estimate_fee"
   - **data**: `{"symbol":"<native>","blockchain":"<chain>"}`
2. Quote `amount` = full balance **minus** the fee buffer (and a small safety margin).
3. For a bridge, keep gas on the source chain to pay the send leg, and on the destination to pay the claim leg.

## Identify the native gas token

The native gas token is the asset with `type: "MainCurrency"` on that chain. Do NOT hardcode a symbol — read it from the wallet.

Call `run_js` with:
   - **function**: "get_assets"
   - **data**: `{}`

Find the asset on the source chain whose `type` is `"MainCurrency"`. Its `symbol` is the native gas token.

## Detect chains with gas

Call `run_js` with:
   - **function**: "get_balance"
   - **data**: `{"symbol":"<native>","blockchain":"<chain>"}`

If `amount` is `"0"` (or the asset is missing), the chain has no native gas.

## Fund gas — three cases

### Case 1 — has some gas + holds a non-native token

Same-chain swap a slice of the non-native token into the native gas token
`src_id` = source chain, 
`token_in` = non-native token 
`token_out` = native gas token.

```bash
# same-chain swap
curl -s -X POST "https://402.blockvault.ai/api/v1/blockswap/quote" \
  -H "Content-Type: application/json" \
  -d '{"chain":"<src_id>","token_in":"<SYMBOL>","token_out":"<SYMBOL>","amount":"<dec>","slippage":0.005,"swapper":"<addr>","sign":false}'
```

### Case 2 — zero gas

A same-chain swap is impossible (you cannot pay its own fee). Fund from another chain that HAS gas.

1. Identify a source chain with native gas and a non-native token the user holds.
2. Quote a cross-chain bridge from that chain into the target chain's native gas token:
   `src_id` = source chain with gas
   `dst_id` = target chain with zero gas
   `symbol_in` = non-native token on source chain
   `symbol_out` = native gas token on target chain
   ```bash
   # cross-chain bridge
   curl -s -X POST "https://402.blockvault.ai/api/v1/blockswap/quote" \
   -H "Content-Type: application/json" \
   -d '{"chain":"<src_id>","to_chain":"<dst_id>","token_in":"<SYMBOL>","token_out":"<SYMBOL>","amount":"<dec>","slippage":0.005,"swapper":"<addr>","sign":false}'
   ```


### Case 3 — no chain has gas

Tell the user and stop. Do not attempt any transaction.

## Checklist

Copy and track:

- [ ] Identify native gas token via `get_assets` (`type: "MainCurrency"`)
- [ ] Check gas balance via `get_balance` (not `get_assets {hasBalance:true}`)
- [ ] If zero gas, pick the funding case (1/2/3) above
- [ ] If `token_in` is native gas, leave a fee buffer (never spend 100%)
- [ ] Confirm the full funding route with the user before executing
- [ ] Re-check `get_balance` after each funding hop

Bridges spend gas on BOTH ends (send + claim). Keep native gas on the destination too if a follow-up step needs it.