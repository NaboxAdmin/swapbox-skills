name: swapbox-swap
description: "Execute on-chain token swaps via the SwapBox DEX aggregator. Swap any token pair on the same blockchain with optimal routing. Returns txData and approval txData for external signing."
license: Apache-2.0
metadata:
  author: swapbox
  version: "1.0.0"
  homepage: "https://swapbox.io"
---
# SwapBox On-Chain Swap Skill

Execute token swaps on the same blockchain using SwapBox's DEX aggregator. Full flow: quote → approve-check (if ERC-20) → encode → save → (user signs & broadcasts) → hash update.

## Skill Routing

**Use this skill when the user:**

- Wants to swap/trade/buy/sell tokens on the same chain
- Asks "swap 10 USDC for ETH", "swap BNB for USDT"
- Wants a swap quote and then execute the trade

**Do NOT use this skill:**

- For price queries only → use `swapbox-price`
- (This skill supports both same-chain and cross-chain execution)

## Complete Swap Flow

### Step 1: Get Route Quote

Call `/api/swap/route` with the **user's real wallet address** (not placeholder):

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
    "fromChain": "BSC",
    "toChain": "BSC",
    "fromAddress": "0xD06A23cC78e13CFCf52fA34232e449E13573F5FE",
    "toAddress": "0xD06A23cC78e13CFCf52fA34232e449E13573F5FE",
    "btcPubKey": "",
    "chainId": 102,
    "assetId": 1,
    "contractAddress": "",
    "swapChainId": 102,
    "swapAssetId": 0,
    "swapContractAddress": "0x55d398326f99059ff775485246999027b3197955",
    "amount": "0.001",
    "slippage": "0.5",
    "smartSlippage": false,
    "fee": "",
    "isAntiMev": false
  }'
```

From the response, select the route where `best == 1`. Extract: `orderId`, `dex`, `channel`, `fee`, `minReceiveAmount`, `toTokenAmount`, `approveAddress`.

**Parameter mapping** (from `/api/asset/query` and `/api/config/chains`):


| Route Param                                         | Source                               |
| --------------------------------------------------- | ------------------------------------ |
| `fromChain`, `toChain`                              | Chain name (e.g., "BSC", "Ethereum") |
| `fromAddress`, `toAddress`                          | User's wallet address                |
| `chainId`, `assetId`, `contractAddress`             | Source token from asset query        |
| `swapChainId`, `swapAssetId`, `swapContractAddress` | Target token from asset query        |
| `amount`                                            | User-specified amount                |
| `slippage`                                          | User-specified or default "0.5"      |

### Step 2: Check Approval via `/swap/approve` (ERC-20 Only)

**If `contractAddress` is NOT empty** (selling an ERC-20 token), call `/api/swap/approve` to check whether approval is required and, if needed, get the approval txData.

**Parameter mapping (MUST NOT hardcode)**:


| Approve Param     | Source                                                                                        |
| ----------------- | --------------------------------------------------------------------------------------------- |
| `dex`             | Selected route from`/swap/route` (`best == 1`) → `dex`                                       |
| `channel`         | Selected route from`/swap/route` (`best == 1`) → `channel`                                   |
| `fromChain`       | Same as`/swap/route` request `fromChain`                                                      |
| `fromAddress`     | User wallet address (same as`/swap/route` request `fromAddress`)                              |
| `chainId`         | Source token object from`/asset/query` → `chainId` (same as `/swap/route` request `chainId`) |
| `assetId`         | Source token object from`/asset/query` → `assetId`                                           |
| `amount`          | User input amount (same unit as`/swap/route` request `amount`)                                |
| `contractAddress` | Source token object from`/asset/query` → `contractAddress` (ERC-20 contract)                 |
| `approveAddress`  | Selected route from`/swap/route` (`best == 1`) → `approveAddress`                            |

```bash
curl --location 'https://swapbox.io/api/swap/approve' \
  --header 'Content-Type: application/json' \
  --data '{
    "dex": "<dex from route>",
    "channel": "<channel from route>",
    "fromChain": "<fromChain>",
    "fromAddress": "<user wallet>",
    "chainId": <chainId>,
    "assetId": <assetId>,
    "amount": 1,
    "contractAddress": "<contractAddress>",
    "approveAddress": "<approveAddress from route>"
  }'
```

**Response handling**:

- If `isApproved == true` → approval is NOT required; proceed to Step 3.
- If `isApproved == false` → the API returns the approval transaction **txData** in the response `data` field. Return this `data` as the “approval txData” to the user for signing and broadcasting via their wallet/backend service. Only continue after the approval transaction is confirmed.

**Approval txData `from/to` rule**:

- `from` = user wallet address (same as `fromAddress`)
- `to` = approve contract address (the route `approveAddress`)

**If `contractAddress` is empty** (native token like BNB, ETH): skip this step.

### Step 3: Get Transaction Data (Encode)

```bash
curl -s -X POST 'https://swapbox.io/api/swap/tx/encode' \
  -H 'Content-Type: application/json' \
  -H 'language: en' \
  -H 'origin: https://swapbox.io' \
  -H 'referer: https://swapbox.io/' \
  -d '{
    "platform": "",
    "orderId": "<orderId from route>",
    "dex": "<dex from route>",
    "channel": "<channel from route>",
    "fromChain": "<fromChain>",
    "toChain": "<toChain>",
    "fromAddress": "<user wallet>",
    "toAddress": "<user wallet>",
    "btcPubKey": "",
    "chainId": <chainId>,
    "assetId": <assetId>,
    "contractAddress": "<empty for native>",
    "amount": "<amount>",
    "slippage": "<slippage>",
    "fee": "<fee from route>",
    "minReceiveAmount": "<minReceiveAmount from route>",
    "swapChainId": <swapChainId>,
    "swapAssetId": <swapAssetId>,
    "swapContractAddress": "<target token contract>",
    "path": null,
    "toTokenAmount": "<toTokenAmount from route>",
    "gmgnRoute": null
  }'
```

Response contains `txData` (to, data, value, gasLimit, gasPrice, chainId, etc.) for signing.

### Step 4: Save Order

```bash
curl -s -X POST 'https://swapbox.io/api/swap/tx/save' \
  -H 'Content-Type: application/json' \
  -H 'language: en' \
  -H 'origin: https://swapbox.io' \
  -H 'referer: https://swapbox.io/' \
  -d '{
    "platform": "",
    "orderId": "<orderId>",
    "dex": "<dex>",
    "channel": "<channel>",
    "fromChain": "<fromChain>",
    "toChain": "<toChain>",
    "fromAddress": "<user wallet>",
    "toAddress": "<user wallet>",
    "btcPubKey": "",
    "chainId": <chainId>,
    "assetId": <assetId>,
    "contractAddress": "<contractAddress>",
    "amount": "<amount>",
    "fee": "<fee>",
    "toTokenAmount": "<toTokenAmount>",
    "slippage": "<slippage>",
    "partnerOrderId": "",
    "minReceiveAmount": "<minReceiveAmount>",
    "terminal": "SwapBox-CLI",
    "swapChainId": <swapChainId>,
    "swapAssetId": <swapAssetId>,
    "swapContractAddress": "<swapContractAddress>",
    "swapType": "",
    "blockHeight": ""
  }'
```

### Step 5: Return txData for User Signing & Broadcasting

After `/swap/tx/encode` and `/swap/tx/save`, this skill’s responsibility is complete:\n\n- Approval: if approval is required, return the `/swap/approve` response `data` (approval txData) to the user.\n- Swap main transaction: return the `/swap/tx/encode` response `txData` to the user.\n\nDownstream steps (NOT executed by this skill; for reference only):\n\n1. User or backend service signs the `txData` with a local wallet.\n2. Broadcast via the chain `rpcUrl` using `eth_sendRawTransaction` (or equivalent).\n3. After broadcast succeeds, call `/swap/tx/hash/update` to report `orderId` and `txHash` to SwapBox backend.

## Flow Summary

```
1. POST /swap/route (with user address)     → get orderId, best route, approveAddress
2. [If ERC-20] POST /swap/approve          → check isApproved, optional approve txData
3. POST /swap/tx/encode                    → get swap txData
4. POST /swap/tx/save                      → save order
5. User/backend signs & broadcasts & calls /swap/tx/hash/update
```

## Amount Conversion

- **Route/encode/save**: Use human-readable amount (e.g., `"0.001"` for 0.001 BNB).
- **Approve**: Use the same amount unit as the `/swap/route` request (follow `/swap/approve` API expectation). Do NOT convert to minimal units unless the API explicitly requires it.

## Security Rules

1. **User confirmation** — Before Step 3, display: fromToken, toToken, amount, expected output, fee, minReceiveAmount. Wait for explicit confirmation.
2. **Approval scope** — For approve, use exact swap amount, not unlimited.
3. **Price impact** — If route `priceImpact` > 5%, warn user. If > 10%, strongly warn.

## Edge Cases

- **No route** — `data` empty from route: inform user "No swap route available".
- **Insufficient balance** — If swap later fails on-chain due to insufficient balance, report the error and suggest reducing the amount or adding funds.
- **Approve already done** — If `/swap/approve` returns `isApproved == true`, skip approval.
- **Broadcast failure** — Report RPC error. Do not retry automatically.
- **Chain config missing rpcUrl** — Use public RPC for known chains (e.g., BSC: https://bsc-dataseed.binance.org/).

## Keyword Glossary


| Term             | Meaning          |
| ---------------- | ---------------- |
| swap / exchange  | swap             |
| approve          | approve          |
| slippage         | slippage         |
| contract address | contract address |
| native token     | native token     |
| transaction hash | tx hash          |

## API Reference

See [swapbox-price/references/api-reference.md](../swapbox-price/references/api-reference.md) for full parameter tables for `/swap/route`, `/swap/tx/encode`, `/swap/tx/save`, `/swap/tx/hash/update`.
