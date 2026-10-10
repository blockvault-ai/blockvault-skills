# Uniswap LP — Field reference

- `chainId`: `1` Ethereum, `137` Polygon, `8453` Base.
- `amount`: **string in decimal units** (e.g. `"100.5"`), converted to wei server-side. 1 USDT = `"1"`, 1 WPOL = `"1"`. For "X USD", use the stablecoin side with `"X"`. Never pass wei or a tiny fraction.
- `fee` (bps): `100`=0.01%, `500`=0.05%, `3000`=0.3%, `10000`=1%. Display as `fee/10000`%.
- `LPAmount` = `{ tokenAddress, amount }` (decimal). Add `decimals` only for tokens outside the curated registry.
- `tickBounds` = `{ tickLower, tickUpper }` — integers, multiples of `tickSpacing`, straddling `currentTick`. **`priceBounds` is BROKEN upstream** (returns a `tickPrice` validation error) — never send it to `create` directly. The `prepare` endpoint accepts `priceBounds`/`rangePct` and converts them to `tickBounds` server-side.
- **Range control (user-facing, on `prepare`):** pass exactly one of `rangePct` (±X% around the current price), `priceBounds` (`{ minPrice, maxPrice }` absolute prices), or `tickBounds` (raw ticks). Omit all three for the server-recommended range from `/pools/recommend`.
- `currentTick` and `tickSpacing` come from `/pools/recommend` (fields `currentTick` / `tickSpacing` on the pool object). Do NOT guess them. They are informational only — the recommended `tickBounds` is already computed server-side.
- `action` (on `check_approval`) is **uppercase**: `CREATE`, `INCREASE`, `DECREASE`, `MIGRATE`. The backend normalizes case, but send uppercase to be safe.
- `poolReference` = the `poolReferenceIdentifier` from `/pools/recommend` for that exact pair + fee tier.
- `token0Address`/`token1Address` are already in the pool's **canonical order** (token0 < token1 by address). Pass them to `existingPool`/`poolParameters` exactly as returned — never swap or reorder them, or the mint computes wrong amounts.
- `token0Decimals`/`token1Decimals` = decimals of each side, from `/pools/recommend`. Use them for `add_asset` and amount conversion.
- `walletAddress` = the wallet `address` from `get_assets`, never a token address.
- `independentToken` = the side you provide (`LPAmount`).
- `nftTokenId`: required for V3/V4 increase/decrease, omitted for V2.
- `liquidityPercentageToDecrease`: integer 1–100.
- Mutating endpoints return a signable `transactions[]` envelope (approvals first). Auto-signed+broadcast; if not, use `run_js` → `sign_transaction` per entry.
- Token addresses come from `/pools/recommend` (`token0Address`/`token1Address`) — never hardcode or guess them.