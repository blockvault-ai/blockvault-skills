# BlockSwap — Quote fields

| Field | Required | Notes |
|---|---|---|
| `chain` | yes | source chain id as string (from `/blockswap/chains`) |
| `to_chain` | no | omit (= same chain) for a swap; set a different id to bridge |
| `token_in` | yes | input symbol or address |
| `token_out` | yes | output symbol or address |
| `amount` | yes | input amount in decimal (exact-input) |
| `slippage` | no | fraction (0.005 = 0.5%). Default 0.5%, max 5% |
| `swapper` | yes | wallet address from `get_assets` |
| `sign` | no | `false` = estimate, `true` = signing flow |