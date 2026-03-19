# SwapBox Skills

Skills for AI agents to query token prices and execute swaps (same-chain & cross-chain) across 30+ blockchains via [SwapBox](https://swapbox.io).

## Available Skills

| Skill | Purpose | Status |
|-------|---------|--------|
| **swapbox-price** | Query token/USDT exchange rates on any supported chain | Active |
| **swapbox-swap** | Execute swaps (same-chain & cross-chain) via aggregator | Active |

## Supported Chains

Ethereum, BSC, Polygon, Arbitrum, Optimism, Avalanche, Base, Solana, TRON, BTC, NULS, NERVE, and 20+ more. Full list available via the [chains API](https://swapbox.io/api/config/chains).

## Quick Start

### Cursor

Install from the plugin marketplace or clone this repository into your project:

```bash
git clone https://github.com/AboPlus/swapbox-skills.git
```

Cursor automatically discovers skills from `.cursor-plugin/plugin.json`.

### Claude Code

```bash
claude plugin install github:AboPlus/swapbox-skills
```

Or install from the Claude plugin marketplace.

### Codex CLI

```bash
git clone https://github.com/AboPlus/swapbox-skills ~/.codex/swapbox-skills
mkdir -p ~/.agents/skills
ln -s ~/.codex/swapbox-skills/skills ~/.agents/skills/swapbox-skills
```

See [.codex/INSTALL.md](.codex/INSTALL.md) for detailed instructions.

### OpenCode

Add to your `opencode.json`:

```json
{
  "instructions": [
    "path/to/swapbox-skills/AGENTS.md",
    "path/to/swapbox-skills/skills/swapbox-price/SKILL.md"
  ]
}
```

## Usage Examples

Once installed, just ask your AI agent in natural language:

- "查询 Ethereum 上 ETH 的 USDT 价格"
- "What's the price of BNB on BSC?"
- "SOL 现在值多少 USDT？"
- "Show me the USDT price for MATIC on Polygon"
- "列出所有支持的链"

The AI agent will automatically call the SwapBox API and return formatted results.

## How It Works

SwapBox Skills work by providing AI agents with structured instructions to call SwapBox HTTP APIs directly — no CLI binary or SDK required.

```
User Request → AI Agent reads SKILL.md → Calls SwapBox API → Returns formatted result
```

**Price Query Flow:**

1. `GET /api/config/chains` — Validate the chain
2. `POST /api/asset/query` — Find the source token and USDT on the chain
3. `POST /api/swap/route` — Get the exchange rate from the best route

## Project Structure

```
swapbox-skills/
├── skills/
│   ├── swapbox-price/           # Token price query (Active)
│   │   ├── SKILL.md
│   │   └── references/
│   │       └── api-reference.md
│   ├── swapbox-swap/            # Swap (same-chain & cross-chain)
│   │   └── SKILL.md
├── .cursor-plugin/              # Cursor integration
├── .claude-plugin/              # Claude Code integration
├── .codex/                      # Codex CLI integration
├── .opencode/                   # OpenCode integration
├── AGENTS.md                    # Skill routing & overview
├── CLAUDE.md                    # Claude Code guidance
├── package.json
├── LICENSE                      # Apache-2.0
└── README.md
```

## License

[Apache-2.0](LICENSE)
