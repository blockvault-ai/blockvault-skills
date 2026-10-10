---
name: uniswap-pools
description: Discovers and ranks the best Uniswap liquidity pools, then opens and manages V2/V3/V4 LP positions. Use when the user wants to provide/add liquidity, earn yield from LP, open/manage a position, claim fees, or invest in pools. Finds top pools autonomously (no token/chain needed from the user) and funds both sides of the pair before minting.
metadata:
  category: defi
  enabled: true
---

# Uniswap Liquidity Pools

Discover and rank pools, then open and manage V2/V3/V4 LP positions non-custodial through BlockVault.


## Reference files

Discover the full reference for this skill in the `reference` folder. Each file is a self-contained reference for a specific topic. Load it with `run_js` → `read_skill_reference` (see below).

Call `run_js` with:
- **function**: "read_skill_reference"
- **data**: `{"skill":"uniswap-pools","file":"<name>"}`

To list all available files, omit `file`:

Call `run_js` with:
- **function**: "read_skill_reference"
- **data**: `{"skill":"uniswap-pools"}`


Determine which reference files to load based on the workflow step:
- is related to discovering pools → load `endpoints` + `fields`
- is related to opening a position → load `endpoints` + `fields` + the protocol-specific file (`v2` / `v3` / `v4`)
- is related to managing positions → load `endpoints` + `fields` + the protocol-specific file (`v2` / `v3` / `v4`)
- is related to claiming fees → load `endpoints` + `fields` + the protocol-specific file (`v3` / `v4`)

## Workflow

### Step 1: Read the wallet

Call `run_js` with:
- **function**: "supported_blockchains"
- **data**: `{}`

Note the usable chains (only these can hold a position).

Call `run_js` with:
- **function**: "get_assets"
- **data**: `{"hasBalance": true}`

The `symbol`, `blockchain`, and `balance` of each asset with a non-zero balance is used to fund the LP position. Stop if the user has no assets on any supported chain.

Checklist:
- **Wallet address** — the `address` field of an asset on the target chain.
- **Native gas** — the `symbol` of the native token on each chain and its `balance`. 

Stop if the user has no native gas on any supported chain.


### Step 2: Discover pools

Call the endpoint with `chainId` = the chain of the user's assets and `limit` = 50. 
The API returns a ranked list of pools with

```bash
curl -sS "https://402.blockvault.ai/api/v1/uniswap/pools/recommend?chainId=<CHAIN_ID>&limit=<N>" -o pools.json
```

Emit this exact artifact fence: 

````markdown
  ```artifact
  {"artifact":"<artifact_path>","template":"uniswap-pools:pool-list"}
  ```
````
Make a recommendation: <your recommendation> (score <score>, APY <apy>%, risk <risk>).
Ask the user which pool they want to invest in (by number).



### Step 3: Fund both sides

**3a. Resolve the pool.** From the chosen pool object, note `token0Address`, `token1Address`, `token0Decimals`, `token1Decimals`, `poolReferenceIdentifier`, `fee`, and `protocol`.

**3b. Decide the amounts.** Ask the user how much they want to invest in USD. The `independentToken.amount` is in decimal units of that token (not USD) — for a stablecoin side, "X USD" = `"X"`. The API computes the other side's amount from the pool ratio; it does NOT auto-swap.

**3c. Fund what's missing.** Check which of the two tokens the wallet holds:

- **Both tokens present** → done, continue to Step 4.
- **One token missing** → swap half of the *investment amount* into the missing token with `blockswap`, then continue to Step 4.
- **Neither token present** → fund both sides with `blockswap`, then continue to Step 4.
- **Not enough native gas** → fund gas first with `blockswap`.

To fund or swap, load the `blockswap` skill first:

Call `load_skill` with:
- **skill_names**: `["blockswap"]`

Then follow its instructions to swap/bridge the missing side or top up native gas.

If a token is not yet registered in `get_assets`, register it first:

Call `run_js` with:
- **function**: "add_asset"
- **data**: `{"symbol":"<TOKEN_SYMBOL>","address":"<token0Address|token1Address>","decimals":<token0Decimals|token1Decimals>,"blockchain":"<chain>"}`

Do not proceed to Step 4 until both sides are funded and the wallet has enough gas.

### Step 4: Confirm the protocol with the user

**Ask the user which protocol they want**. 

After the user confirms the protocol, load the appropriate reference file protocol:
  Call `run_js` with:
  - **function**: "read_skill_reference"
  - **data**: `{"skill":"uniswap-pools","file":"<protocol>"}`

Present the trade-off in well structured sentences (V3/V4 = concentrated, higher APY, out-of-range risk; V2 = full-range, simpler, no range risk). If the pool is V2-only, skip the question and use V2.

### Step 5: Open the position
Based on the protocol, follow the appropriate workflow.

## Manage positions
Based on  the protocol, follow the appropriate workflow.

### Withdraw / exit
Based on  the protocol, follow the appropriate workflow.