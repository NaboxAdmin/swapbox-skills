
name: swapbox-price
description: "Use this skill to query token prices against USDT trading pairs across 30+ blockchains via the SwapBox API. Supports any token on any supported chain — get real-time exchange rates, find supported chains, and discover available tokens."
license: Apache-2.0
metadata:
  author: swapbox
  version: "1.0.0"
  homepage: "https://swapbox.io"
---
# SwapBox Price Query Skill

Query real-time token/USDT exchange rates across 30+ blockchains using SwapBox API.

## Skill Routing

**Use this skill when the user:**

- Asks for a token price (e.g., "ETH price", "price of SOL", "BNB/USDT price")
- Asks about exchange rates or trading pairs
- Wants to know how much a token is worth in USDT
- Asks what chains or tokens are supported
- Asks "ETH/USDT price on Ethereum"

**Do NOT use this skill when the user:**

- Wants to execute a swap/trade (same-chain or cross-chain) → use `swapbox-swap`
- Asks about wallet balances, transaction history, or gas fees

## Keyword Glossary


| Chinese       | English          | Context                       |
| ------------- | ---------------- | ----------------------------- |
| price         | price            | Token price query             |
| trading pair  | trading pair     | Token/USDT pair               |
| exchange rate | exchange rate    | Conversion rate               |
| slippage      | slippage         | Price deviation tolerance     |
| chain         | chain/blockchain | Blockchain network            |
| token / coin  | token            | Cryptocurrency token          |
| best route    | best route       | Optimal swap channel          |
| cross-chain   | cross-chain      | Between different blockchains |
| same-chain    | same-chain       | Within the same blockchain    |

## Operation Flow

### Step 1: Identify Intent

Parse the user's request to extract:

- **chain** — Which blockchain (e.g., Ethereum, BSC, Solana). If not specified, ask the user.
- **token** — Which token to query (e.g., ETH, BNB, SOL). If not specified, ask the user.
- **amount** — How many tokens to price (default: 1).

### Step 2: Query Supported Chains

Fetch the list of supported chains to validate the user's chain selection.

```bash
curl -s 'https://swapbox.io/api/config/chains'
```

Response structure:

```json
{
  "code": 0,
  "msg": "Success",
  "data": [
    {
      "chain": "Ethereum",
      "chainName": "Ethereum",
      "chainId": 101,
      "nativeId": 1,
      "prefix": "0x",
      "chainType": 2,
      "swap": 1,
      "bridge": 1,
      "status": 1,
      "mainAsset": {
        "symbol": "ETH",
        "chainId": 101,
        "assetId": 1,
        "contractAddress": "",
        "decimals": 18
      }
    }
  ]
}
```

Use the `chain` field to match the user's request (case-insensitive). Only chains with `status: 1` and `swap: 1` support price queries.

### Step 3: Query Token List

Fetch the token list for the matched chain to find the source token and USDT.

```bash
curl -s -X POST 'https://swapbox.io/api/asset/query' \
  -H 'Content-Type: application/json' \
  -d '{"chain":"Ethereum"}'
```

Response structure:

```json
{
  "code": 0,
  "msg": "Success",
  "data": [
    {
      "symbol": "ETH",
      "chain": "Ethereum",
      "chainId": 101,
      "assetId": 1,
      "contractAddress": "",
      "decimals": 18,
      "usdPrice": 2174.00
    },
    {
      "symbol": "USDT",
      "chain": "Ethereum",
      "chainId": 101,
      "assetId": 0,
      "contractAddress": "0xdac17f958d2ee523a2206206994597c13d831ec7",
      "decimals": 6
    }
  ]
}
```

From this response, extract:

- **Source token**: Match by `symbol` (case-insensitive) to get `chainId`, `assetId`, `contractAddress`
- **USDT**: Find the entry where `symbol` is "USDT" to get its `chainId`, `assetId`, `contractAddress`

**IMPORTANT**: If USDT is not found on this chain, the price query is not supported. Inform the user.

### Step 4: Get Price Quote

Construct the route request to get the exchange rate.

```bash
curl -s -X POST 'https://swapbox.io/api/swap/route' \
  -H 'Content-Type: application/json' \
  -H 'language: en' \
  -H 'origin: https://swapbox.io' \
  -H 'referer: https://swapbox.io/' \
  -d '{
    "platform": "",
    "dex": "",
    "channel": "",
    "fromChain": "{SOURCE_CHAIN}",
    "toChain": "{SOURCE_CHAIN}",
    "fromAddress": "0x0000000000000000000000000000000000000001",
    "toAddress": "0x0000000000000000000000000000000000000001",
    "btcPubKey": "",
    "chainId": {SOURCE_TOKEN_CHAIN_ID},
    "assetId": {SOURCE_TOKEN_ASSET_ID},
    "contractAddress": "{SOURCE_TOKEN_CONTRACT_ADDRESS}",
    "swapChainId": {USDT_CHAIN_ID},
    "swapAssetId": {USDT_ASSET_ID},
    "swapContractAddress": "{USDT_CONTRACT_ADDRESS}",
    "amount": "{AMOUNT}",
    "slippage": "0.5",
    "smartSlippage": false,
    "fee": "",
    "isAntiMev": false
  }'
```

**Parameter mapping from token list:**


| Route Parameter       | Source                        | Description                                      |
| --------------------- | ----------------------------- | ------------------------------------------------ |
| `fromChain`           | chains API`chain` field       | Chain name, e.g. "Ethereum"                      |
| `toChain`             | Same as`fromChain`            | Same chain for price query                       |
| `fromAddress`         | Fixed placeholder             | Use`0x0000000000000000000000000000000000000001`  |
| `toAddress`           | Fixed placeholder             | Use`0x0000000000000000000000000000000000000001`  |
| `chainId`             | Source token`chainId`         | e.g. 101 for Ethereum                            |
| `assetId`             | Source token`assetId`         | e.g. 1 for ETH                                   |
| `contractAddress`     | Source token`contractAddress` | Empty string`""` for native tokens               |
| `swapChainId`         | USDT`chainId`                 | e.g. 101                                         |
| `swapAssetId`         | USDT`assetId`                 | e.g. 0                                           |
| `swapContractAddress` | USDT`contractAddress`         | e.g.`0xdac17f958d2ee523a2206206994597c13d831ec7` |
| `amount`              | User specified or`"1"`        | Query amount                                     |
| `slippage`            | Fixed`"0.5"`                  | 0.5% slippage                                    |

**IMPORTANT for non-EVM chains** (e.g., BTC, Solana, TRON):

- For BTC: Use a valid BTC address format as `fromAddress`/`toAddress`, e.g. `bc1qw508d6qejxtdg4y5r3zarvary0c5xw7kv8f3t4`
- For Solana: Use a valid Solana address
- For TRON: Use a valid TRON address starting with `T`
- Check `prefix` field from chains API to determine address format

### Step 5: Parse and Display Results

Response structure:

```json
{
  "code": 0,
  "msg": "Success",
  "data": [
    {
      "dex": "OKX",
      "channel": "OKX",
      "best": 1,
      "fromTokenAmount": "1",
      "toTokenAmount": "2182.158811",
      "minReceiveAmount": "2171.248016945",
      "priceImpact": "0",
      "fee": "10.910794055",
      "feeSymbol": "USDT",
      "swapType": 1
    },
    {
      "dex": "MetaPath",
      "channel": "Bridgers",
      "best": 0,
      "fromTokenAmount": "1",
      "toTokenAmount": "2170.637364",
      "swapType": 1
    }
  ]
}
```

**Processing rules:**

1. Filter for the route where `best == 1`
2. Take the `toTokenAmount` field — this is the price (how many USDT you get per token)
3. If `data` is empty or `null`, the trading pair is not supported

**Display format:**

```
1 {TOKEN} = {toTokenAmount} USDT (via {dex})

Chain: {chain}
Best Route: {dex} ({channel})
Price Impact: {priceImpact}%
Fee: {fee} {feeSymbol}
```

For multi-token queries, display as a table.

## Complete Example

**User**: "ETH/USDT price on Ethereum"

**Agent workflow:**

1. Parse: chain = "Ethereum", token = "ETH", amount = 1
2. Call `/api/config/chains` → confirm Ethereum is supported (chainId=101)
3. Call `/api/asset/query` with `{"chain":"Ethereum"}` → find ETH (chainId=101, assetId=1, contractAddress="") and USDT (chainId=101, assetId=0, contractAddress="0xdac17f958d2ee523a2206206994597c13d831ec7")
4. Call `/api/swap/route` with the mapped parameters
5. Filter `best==1` → `toTokenAmount = "2182.158811"`
6. Display: "1 ETH = 2182.16 USDT (via OKX)"

## Quickstart Examples

### Query ETH price on Ethereum

```bash
# Step 1: Get token list for Ethereum
curl -s -X POST 'https://swapbox.io/api/asset/query' \
  -H 'Content-Type: application/json' \
  -d '{"chain":"Ethereum"}'

# Step 2: Get price quote (ETH → USDT)
curl -s -X POST 'https://swapbox.io/api/swap/route' \
  -H 'Content-Type: application/json' \
  -H 'language: en' \
  -H 'origin: https://swapbox.io' \
  -H 'referer: https://swapbox.io/' \
  -d '{"platform":"","dex":"","channel":"","fromChain":"Ethereum","toChain":"Ethereum","fromAddress":"0x0000000000000000000000000000000000000001","toAddress":"0x0000000000000000000000000000000000000001","btcPubKey":"","chainId":101,"assetId":1,"contractAddress":"","swapChainId":101,"swapAssetId":0,"swapContractAddress":"0xdac17f958d2ee523a2206206994597c13d831ec7","amount":"1","slippage":"0.5","smartSlippage":false,"fee":"","isAntiMev":false}'
```

### Query BNB price on BSC

```bash
# Step 1: Get token list for BSC
curl -s -X POST 'https://swapbox.io/api/asset/query' \
  -H 'Content-Type: application/json' \
  -d '{"chain":"BSC"}'

# Step 2: Find BNB and USDT entries, then call /api/swap/route with their respective chainId, assetId, contractAddress
```

### List all supported chains

```bash
curl -s 'https://swapbox.io/api/config/chains'
```

## Chain Name Reference

Common chain names (use exact value from the `chain` field in API response):


| Chain Name | chainId | Native Token | Address Prefix |
| ---------- | ------- | ------------ | -------------- |
| Ethereum   | 101     | ETH          | 0x             |
| BSC        | 103     | BNB          | 0x             |
| Polygon    | 106     | MATIC/POL    | 0x             |
| Arbitrum   | 111     | ETH          | 0x             |
| Optimism   | 115     | ETH          | 0x             |
| Avalanche  | 110     | AVAX         | 0x             |
| Base       | 129     | ETH          | 0x             |
| Solana     | —      | SOL          | —             |
| TRON       | —      | TRX          | T              |
| BTC        | 201     | BTC          | bc1/1/3        |
| NULS       | 1       | NULS         | NULS/tNULS     |
| NERVE      | 9       | NVT          | NERVE/tNERVE   |

**Note**: Always fetch the latest chain list from `/api/config/chains` as chains may be added or removed. The table above is for quick reference only.

## Edge Cases

### Chain not found

If the user's chain name doesn't match any entry in the chains API:

- Try fuzzy matching (e.g., "eth" → "Ethereum", "bsc" → "BSC", "bnb chain" → "BSC")
- If still no match, list available chains and ask the user to choose

### Token not found

If the token symbol doesn't match any entry in the asset query:

- Try case-insensitive matching
- Try matching by contract address if the user provided one
- If still no match, inform the user: "Token {symbol} not found on {chain}. Available tokens: ..."

### USDT not available on chain

Some chains may not have USDT listed. In this case:

- Check if USDC is available as an alternative stablecoin
- Inform the user that USDT trading pair is not available on this chain

### No route returned

If the `/api/swap/route` response `data` is empty or null:

- The trading pair is not supported or has no liquidity
- Inform the user: "No swap route available for {token}/USDT on {chain}"

### API errors

- `code != 0`: Report the error message from `msg` field
- Network timeout: Retry once, then inform the user of the connectivity issue

## Output Format Rules

### Price display

- Show up to 6 decimal places for prices > 1 USDT
- Show up to 8 decimal places for prices < 1 USDT
- Use comma separators for large numbers (e.g., 72,440.01)
- Always show the source DEX/channel

### Multi-token response

When querying multiple tokens, present as a formatted table:

```
| Token | Price (USDT) | Route | Fee |
|-------|-------------|-------|-----|
| ETH   | 2,182.16    | OKX   | 10.91 USDT |
| BNB   | 605.32      | OKX   | 3.03 USDT  |
```

## Additional Resources

For detailed API parameter tables and response field descriptions, see [references/api-reference.md](references/api-reference.md).
