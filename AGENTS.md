# SwapBox Skills — Agent Instructions

This is a **SwapBox skill collection** providing 2 skills for multi-chain DeFi operations: token price queries and swap execution (same-chain and cross-chain) across 30+ blockchains.

All skills interact directly with SwapBox HTTP APIs (https://swapbox.io). For swap execution, the skill returns tx data for external signing and broadcasting by the user or their backend service.

## Available Skills

| Skill | Purpose | When to Use |
|-------|---------|-------------|
| swapbox-price | Multi-chain token/USDT price query | User asks about token prices, exchange rates, trading pairs; user asks "ETH 在 Ethereum 上多少钱", "查询 BSC 上 CAKE 的 USDT 价格", "what's the price of SOL", "how much is 1 BNB in USDT" |
| swapbox-swap | Swap execution (same-chain & cross-chain) | User wants to swap/trade/buy/sell tokens (same-chain) or move/swap assets across chains (cross-chain) |

## Architecture

- **skills/** — 2 SwapBox skill definitions (each is a `SKILL.md` with YAML frontmatter + API reference)
- **skills/swapbox-price/** — Token price query skill (active)
- **skills/swapbox-swap/** — Swap skill (same-chain & cross-chain; returns tx data for external signing)

## Skill Discovery

Each skill in `skills/` contains a `SKILL.md` with:

- YAML frontmatter (name, description, metadata)
- Full API reference with parameters and response schemas
- Step-by-step operation flow with curl examples
- Edge cases and error handling

## API Base URL

All API calls use the base URL: `https://swapbox.io/api`

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/config/chains` | GET | List all supported blockchains (includes rpcUrl for broadcast) |
| `/asset/query` | POST | Query token list for a specific chain |
| `/swap/route` | POST | Get swap/bridge route with price quote |
| `/swap/tx/encode` | POST | Get txData for signing (after route selection) |
| `/swap/tx/save` | POST | Save order before broadcast |
| `/swap/tx/hash/update` | POST | Register txHash after broadcast |
