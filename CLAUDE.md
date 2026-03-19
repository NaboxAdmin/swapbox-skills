# CLAUDE.md

This file provides guidance to Claude Code when working with this repository.

## Project Overview

This is a **Claude Code plugin** — a collection of SwapBox skills for multi-chain DeFi operations. The project provides skills for token price queries and swap execution (same-chain and cross-chain) across 30+ blockchains via the SwapBox API (https://swapbox.io).

Unlike CLI-based skill collections, SwapBox skills work by instructing the AI agent to directly call HTTP APIs using `curl` or equivalent — no binary installation required.

## Architecture

- **skills/** — 2 SwapBox skill definitions (each is a `SKILL.md` with YAML frontmatter + API reference)
- **skills/swapbox-price/** — Token price query skill with full API reference
- **skills/swapbox-swap/** — Swap skill (same-chain & cross-chain)

## Available Skills

| Skill | Purpose | When to Use |
|-------|---------|-------------|
| swapbox-price | Multi-chain token/USDT price query | User asks about token prices, exchange rates, or trading pairs on any supported chain |
| swapbox-swap | Swap (same-chain & cross-chain) | User wants to swap tokens on the same chain, or swap/move assets across chains |

## Key API Endpoints

All requests go to `https://swapbox.io/api`:

- `GET /config/chains` — Supported chain list
- `POST /asset/query` — Token list per chain
- `POST /swap/route` — Swap route and price quote
