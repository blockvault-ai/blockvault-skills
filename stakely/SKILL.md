---
name: stakely
description: Earn staking rewards on ETH (StakeWise vault) using Stakely's validator infrastructure. Use this whenever the user wants to stake ETH, unstake, withdraw, check staking balance, or compare APYs. Non-custodial — funds and keys stay in the user's BlockVault wallet.
metadata:
  tool: bash,sign_transaction
  category: tools
  enabled: true
  homepage: https://app.stakely.io/staking-api
  secrets:
    - key: STAKELY_API_KEY
      description: "API key generated at https://app.stakely.io/staking-api (X-API-KEY header)."
---

# Stakely

Stake ETH with audited validators via StakeWise vaults powered by Stakely. Help users grow their holdings, track yield, and claim rewards — all non-custodial through BlockVault.

## Instructions

- **Get the address first.** Call `run_js` → `get_assets` with `{"hasBalance": true}` to obtain the user's real ETH address before any API call.
- **Use `bash` for API calls.** Execute `curl` commands to interact with Stakely endpoints — never invent responses.
- Detect the user's language and reply in that language.
- Render results as Markdown tables, never raw JSON.
- Lead with **APY and expected earnings**, not technical fields.

## Supported networks

| Asset | Chain | Product | BlockVault blockchain name |
|-------|-------|---------|---------------------------|
| ETH | Ethereum mainnet | StakeWise vault | `ethereum` |
| ETH | Hoodi testnet | StakeWise vault | `hoodi` |

Use `X-NETWORK: 1` for mainnet, `X-NETWORK: 560048` for Hoodi testnet.

## API endpoints

Base URL: `https://staking-api.stakely.io`
Prefix: `/api/v1/ethereum/stakewise`

The secret placeholder `{{STAKELY_API_KEY}}` is resolved automatically — write it verbatim in headers. Never echo it back to the user.

```bash
# GET vault info (APY, vault address)
curl -sS "https://staking-api.stakely.io/api/v1/ethereum/stakewise/vault" \
  -H "X-API-KEY: {{STAKELY_API_KEY}}" \
  -H "X-NETWORK: 560048"

# GET stake balance for an address
curl -sS "https://staking-api.stakely.io/api/v1/ethereum/stakewise/stake-balance/{address}" \
  -H "X-API-KEY: {{STAKELY_API_KEY}}" \
  -H "X-NETWORK: 560048"

# GET exited balance (exit queue status)
curl -sS "https://staking-api.stakely.io/api/v1/ethereum/stakewise/exited-balance/{address}" \
  -H "X-API-KEY: {{STAKELY_API_KEY}}" \
  -H "X-NETWORK: 560048"

# GET historic actions for address
curl -sS "https://staking-api.stakely.io/api/v1/ethereum/stakewise/historic/{address}" \
  -H "X-API-KEY: {{STAKELY_API_KEY}}" \
  -H "X-NETWORK: 560048"

# POST stake ETH
curl -sS -X POST "https://staking-api.stakely.io/api/v1/ethereum/stakewise/action/stake" \
  -H "X-API-KEY: {{STAKELY_API_KEY}}" \
  -H "X-NETWORK: 560048" \
  -H "Content-Type: application/json" \
  -d '{"address": "<USER_ADDRESS>", "amount": <AMOUNT_IN_ETH>}'

# POST unstake (enters exit queue)
curl -sS -X POST "https://staking-api.stakely.io/api/v1/ethereum/stakewise/action/unstake" \
  -H "X-API-KEY: {{STAKELY_API_KEY}}" \
  -H "X-NETWORK: 560048" \
  -H "Content-Type: application/json" \
  -d '{"address": "<USER_ADDRESS>", "amount": <AMOUNT_IN_ETH>}'

# POST withdraw (claim matured exit)
curl -sS -X POST "https://staking-api.stakely.io/api/v1/ethereum/stakewise/action/withdraw" \
  -H "X-API-KEY: {{STAKELY_API_KEY}}" \
  -H "X-NETWORK: 560048" \
  -H "Content-Type: application/json" \
  -d '{"address": "<USER_ADDRESS>"}'
```

**Notes:**
- Replace `{address}` in GET paths with the user's actual `0x...` address.
- `amount` is a **number in ETH units** (NOT wei, NOT a string). Examples: `0.1`, `0.2`, `1.5`.
- For mainnet change `X-NETWORK: 560048` → `X-NETWORK: 1`.
- **NEVER call the prefix alone** (`/api/v1/ethereum/stakewise`) — always append a specific endpoint.

### Action response format

All POST `action/*` endpoints return:

```json
{
  "unsigned_tx_hash_hex": "0x...",
  "serialized_tx_hex": "...",
  "payload": {
    "type": "0x0",
    "nonce": "0x...",
    "gasLimit": "0x...",
    "to": "0x...",
    "value": "0x...",
    "data": "0x...",
    "gasPrice": "0x..."
  }
}
```

**Only the `payload` object matters for signing.** Ignore `serialized_tx_hex` and `unsigned_tx_hash_hex`. When calling `sign_transaction`, **spread the payload fields flat** at the top level — do NOT nest them inside a `payload` key.

## Address resolution

Before any action, get the user's address:

Call `run_js` with:
- **function**: `"get_assets"`
- **data**: `{"hasBalance": true}`

Pick the ETH asset on the target chain and note its `address`. Use this exact address everywhere.

## stake flow
Follow all steps silently, DO NO OMIT ANY STEP.

1. Get address and confirm balance:

   Call `run_js` with:
   - **function**: `"get_assets"`
   - **data**: `{"hasBalance": true}`

2. Get vault APY and tell user the expected yield:

   ```bash
   curl -sS "https://staking-api.stakely.io/api/v1/ethereum/stakewise/vault" \
     -H "X-API-KEY: {{STAKELY_API_KEY}}" \
     -H "X-NETWORK: 560048"
   ```

   Parse response and tell user: "Staking X ETH at ~Y% APY (~Z ETH/year). Proceed?"

3. On user confirmation, build the stake transaction by calling the stake endpoint use the address from step 1 and the amount specified by the user:

   ```bash
   curl -sS -X POST "https://staking-api.stakely.io/api/v1/ethereum/stakewise/action/stake" \
     -H "X-API-KEY: {{STAKELY_API_KEY}}" \
     -H "X-NETWORK: 560048" \
     -H "Content-Type: application/json" \
     -d '{"address": "<USER_ADDRESS>", "amount": <AMOUNT_IN_ETH>}'
   ```

4. Make a summary of the transaction details and ask for confirmation before proceeding. 
if the user confirms, take the `payload` object from step 3 response and sign it using `sign_transaction`:

   Call `sign_transaction` with the payload fields spread at top level:
   - **to**: `"<to from payload>"`
   - **value**: `"<value from payload>"`
   - **data**: `"<data from payload>"`
   - **nonce**: `"<nonce from payload>"`
   - **gasLimit**: `"<gasLimit from payload>"`
   - **gasPrice**: `"<gasPrice from payload>"` (for legacy tx)
   - **blockchain**: `"<chain>"`
   - **fromAddress**: `"<address>"`
   - **broadcast**: `true`
   - **description**: `"Stake X ETH via Stakely"`

   **Only include fields that exist in the payload.** Do NOT pass `null` or undefined values.


## Check earnings flow

1. Get staked amount + pending rewards:

   ```bash
   curl -sS "https://staking-api.stakely.io/api/v1/ethereum/stakewise/stake-balance/{address}" \
     -H "X-API-KEY: {{STAKELY_API_KEY}}" \
     -H "X-NETWORK: 560048"
   ```

2. Get current APY:

   ```bash
   curl -sS "https://staking-api.stakely.io/api/v1/ethereum/stakewise/vault" \
     -H "X-API-KEY: {{STAKELY_API_KEY}}" \
     -H "X-NETWORK: 560048"
   ```

3. Show: staked amount, APY, estimated monthly/annual rewards, claimable balance
4. If claimable > 0, suggest claiming

## Unstake / Withdraw flow

1. Resolve address (same as earn flow step 1)

2. Enter exit queue:

   ```bash
   curl -sS -X POST "https://staking-api.stakely.io/api/v1/ethereum/stakewise/action/unstake" \
     -H "X-API-KEY: {{STAKELY_API_KEY}}" \
     -H "X-NETWORK: 560048" \
     -H "Content-Type: application/json" \
     -d '{"address": "<USER_ADDRESS>", "amount": <AMOUNT_IN_ETH>}'
   ```

3. Sign & broadcast — same as earn flow step 4:

   Call `sign_transaction` with the payload fields spread flat:
   - **to**, **value**, **data**, **nonce**, **gasLimit**, **gasPrice** from the unstake response payload
   - **blockchain**: `"<chain>"`
   - **fromAddress**: `"<address>"`
   - **broadcast**: `true`
   - **description**: `"Unstake X ETH via Stakely"`

   **Only include fields that exist in the payload.** Do NOT pass `null` or undefined values.

4. For withdrawals after exit completes:

   ```bash
   curl -sS -X POST "https://staking-api.stakely.io/api/v1/ethereum/stakewise/action/withdraw" \
     -H "X-API-KEY: {{STAKELY_API_KEY}}" \
     -H "X-NETWORK: 560048" \
     -H "Content-Type: application/json" \
     -d '{"address": "<USER_ADDRESS>"}'
   ```

## Rendering

**Stake balance:**

```markdown
## ETH Staking on {{ chain }}

| | |
|---|---|
| 🏦 Address | `{{ address[:6] }}…{{ address[-4:] }}` |
| 💰 Staked | **{{ staked }} {{ symbol }}** |
| 📈 APY | {{ apy }}% |
| 🎁 Pending rewards | {{ rewards }} {{ symbol }} |
| ⏳ Exit queue | {{ exit_queue or "—" }} |
| ✅ Claimable | {{ claimable or "—" }} |

> Est. monthly: ~{{ monthly }} {{ symbol }} · Annual: ~{{ annual }} {{ symbol }}
```

**Yield summary** (multiple positions):

Write the positions to an artifact file, then emit an ```artifact fenced block so the app renders them as a swipeable card deck. The template `stakely:positions` renders `data.positions`.

Call `text_editor` to save the positions as JSON:

- **command**: `"create"`
- **path**: `"artifacts/stakely-positions.json"`
- **file_text**: a JSON object `{"positions": [...]}` — one object per position with:
  - `symbol` — asset symbol (e.g. `ETH`).
  - `chain` — display name (e.g. `Ethereum`).
  - `apy` — numeric APY.
  - `staked` — staked amount.
  - `rewards` — pending/claimable rewards.
  - `monthly` — estimated monthly rewards.

Then emit:

````markdown
```artifact
{"artifact":"artifacts/stakely-positions.json","template":"stakely:positions"}
```
````

**Total est. annual:** ~{{ totals.annual }} USD

**Historic actions:**

Write the actions to an artifact file, then emit an ```artifact fenced block so the app renders them as a swipeable card deck. The template `stakely:history` renders `data.actions`.

Call `text_editor` to save the actions as JSON:

- **command**: `"create"`
- **path**: `"artifacts/stakely-history.json"`
- **file_text**: a JSON object `{"actions": [...]}` — one object per action with:
  - `type` — `stake`, `unstake`, or `withdraw`.
  - `amount` — numeric amount.
  - `symbol` — asset symbol.
  - `date` — ISO 8601.
  - `txHash` — transaction hash (optional).

Then emit:

````markdown
```artifact
{"artifact":"artifacts/stakely-history.json","template":"stakely:history","context":{"explorer":"<explorer base URL>"}}
```
````

## Constraints

- **Non-custodial.** Never request private keys or mnemonics — signing goes through `sign_transaction`.
- **Use real addresses.** Always get the user's address from `get_assets` — never invent or assume one.
- **Never echo `{{STAKELY_API_KEY}}`** to the user.
- **Warn about unbonding.** When unstaking, mention the exit queue delay before the user confirms.
- **Stick to listed endpoints.** If the user asks about an unsupported chain, point them to `https://docs.stakely.io/staking-api/supported-networks`.
