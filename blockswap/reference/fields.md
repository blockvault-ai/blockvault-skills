# BlockSwap — Quote fields

| Field | Required | Notes |
|---|---|---|
| `chain` | yes | source chain name (`ethereum`/`base`/`polygon`) or numeric id (from `/blockswap/chains`) |
| `to_chain` | no | omit (= same chain) for a swap; set a different name/id to bridge |
| `token_in` | yes | input symbol or address |
| `token_out` | yes | output symbol or address |
| `amount` | yes | input amount in decimal (exact-input) |
| `slippage` | no | fraction (0.005 = 0.5%). Omit for auto; max 0.1 (10%) |
| `swapper` | yes | wallet address from `get_assets` |
| `sign` | no | `false` = estimate, `true` = signing flow |