---
name: usd1-por
description: Query USD1 stablecoin Proof of Reserves — get on-chain reserves from Chainlink oracle, total supply across 10 chains, and collateralization ratio
user-invocable: true
---

# USD1 Proof of Reserves

Query USD1 stablecoin reserve and supply data directly from on-chain sources.
Use `cast` (Foundry) if available; fall back to `curl` + JSON-RPC.

## Key Contracts & Addresses

### Proof of Reserves Oracle (Ethereum mainnet)
- **Contract**: `0x691b74146cdba162449012aa32d3cbf5df77d4c4`
- **Decimals**: 18
- **Functions**:
  - `latestBundle() → bytes` — encoded `(uint256 timestamp, uint256 reserves)`
  - `bundleDecimals() → uint8[]`
  - `latestBundleTimestamp() → uint256`

### USD1 Token Addresses
| Chain | Type | Address | Decimals |
|-------|------|---------|----------|
| Ethereum (1) | native | `0x8d0D000Ee44948FC98c9B98A4FA4921476f08B0d` | 18 |
| BNB Chain (56) | native | `0x8d0D000Ee44948FC98c9B98A4FA4921476f08B0d` | 18 |
| Tron | native | `TPFqcBAaaUMCSVRCqPaQ9QnzKhmuoLR6Rc` | 18 |
| Solana | native | mint: `USD1ttGY1N17NEEHLmELoaybftRBUSErhqYiQzvEmuB` | 6 |
| Aptos | native | metadata: `0x05fabd1b12e39967a3c24e91b7b8f67719a6dacee74f3c8b9fb7d93e855437d2` | 6 |
| Plume | bridged | `0x111111d2bf19e43C34263401e0CAd979eD1cdb61` | 18 |
| AB Core (36888) | bridged | `0x111111d2bf19e43C34263401e0CAd979eD1cdb61` | 18 |
| Monad | bridged | `0x111111d2bf19e43C34263401e0CAd979eD1cdb61` | 6 |
| Mantle | bridged | `0x111111d2bf19e43C34263401e0CAd979eD1cdb61` | 18 |
| Morph L2 | bridged | `0x111111d2bf19e43C34263401e0CAd979eD1cdb61` | 18 |

### CCIP Lock/Release Pool Addresses (native chains)
| Chain | Pool Address |
|-------|-------------|
| Ethereum | `0x36a72eD0096B414521C45E3ddC9ed657d1D9c141` |
| BNB Chain | `0xCe3f7378aE409e1CE0dD6fFA70ab683326b73f04` |
| Solana | `B4PB9qWUW6R18Gbpk5Km8mJ2GoQgCcVpgdqhU7C2n8fh` |
| Aptos | `0x1eb155d08acc900954b6ccee01659b390399ae81ad4c582b73d41374c475caf6` |

## Default RPC Endpoints
| Chain | RPC URL |
|-------|---------|
| Ethereum | `https://rpc.ankr.com/eth` |
| BNB Chain | `https://rpc.ankr.com/bsc` |
| Plume | `https://rpc.plume.org` |
| AB Core | `https://rpc.core.ab.org` |
| Monad | `https://rpc.monad.xyz` |
| Mantle | `https://rpc.mantle.xyz` |
| Morph L2 | `https://rpc.morphl2.io` |
| Tron | `https://api.trongrid.io` |
| Solana | `https://solana-rpc.publicnode.com` |

## Querying with cast (preferred)

### Get Reserves
```bash
# Decode latestBundle bytes → (timestamp, reserves)
BUNDLE=$(cast call 0x691b74146cdba162449012aa32d3cbf5df77d4c4 "latestBundle()(bytes)" \
  --rpc-url https://rpc.ankr.com/eth)
cast abi-decode "f()(uint256,uint256)" "$BUNDLE"
# Output: [timestamp, raw_reserves]  divide raw_reserves by 1e18
```

### Get USD1 totalSupply on EVM chains
```bash
# Ethereum
cast call 0x8d0D000Ee44948FC98c9B98A4FA4921476f08B0d "totalSupply()(uint256)" \
  --rpc-url https://rpc.ankr.com/eth

# BNB Chain
cast call 0x8d0D000Ee44948FC98c9B98A4FA4921476f08B0d "totalSupply()(uint256)" \
  --rpc-url https://rpc.ankr.com/bsc

# Bridged chains (for reference only, excluded from total)
cast call 0x111111d2bf19e43C34263401e0CAd979eD1cdb61 "totalSupply()(uint256)" \
  --rpc-url https://rpc.plume.org
```

### Get CCIP pool balance (USD1 locked for bridging)
```bash
cast call 0x8d0D000Ee44948FC98c9B98A4FA4921476f08B0d \
  "balanceOf(address)(uint256)" 0x36a72eD0096B414521C45E3ddC9ed657d1D9c141 \
  --rpc-url https://rpc.ankr.com/eth
```

## Querying with curl (JSON-RPC fallback)

### Get USD1 totalSupply via eth_call
```bash
# totalSupply() selector = 0x18160ddd
curl -s https://rpc.ankr.com/eth -X POST -H "Content-Type: application/json" -d '{
  "jsonrpc":"2.0","method":"eth_call",
  "params":[{"to":"0x8d0D000Ee44948FC98c9B98A4FA4921476f08B0d","data":"0x18160ddd"},"latest"],
  "id":1
}' | python3 -c "import sys,json; r=json.load(sys.stdin); print(int(r['result'],16)/1e18)"
```

## Querying Non-EVM Chains

### Tron — TRC-20 totalSupply
```bash
curl -s "https://api.trongrid.io/v1/contracts/TPFqcBAaaUMCSVRCqPaQ9QnzKhmuoLR6Rc/tokens?limit=1" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['data'][0]['total_supply_with_decimals'])"
# Divide result by 1e18
```

### Solana — SPL token supply
```bash
curl -s https://solana-rpc.publicnode.com -X POST -H "Content-Type: application/json" -d '{
  "jsonrpc":"2.0","id":1,"method":"getTokenSupply",
  "params":["USD1ttGY1N17NEEHLmELoaybftRBUSErhqYiQzvEmuB"]
}' | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['result']['value']['uiAmount'])"
```

### Aptos — Fungible asset supply
```bash
curl -s "https://fullnode.mainnet.aptoslabs.com/v1/accounts/0x05fabd1b12e39967a3c24e91b7b8f67719a6dacee74f3c8b9fb7d93e855437d2/resource/0x1::fungible_asset::Supply" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print(int(d['data']['current'])/1e6)"
```

## Supply Accounting Logic

**Total native supply = Ethereum + BNB + Tron + Solana + Aptos**

Bridged chain supply (Plume, AB Core, Monad, Mantle, Morph) is **excluded** from the total — CCIP locks native USD1 on source chains and mints equivalent on destination chains, so counting both would double-count.

**Collateralization ratio = Reserves / Total native supply × 100%**

A ratio ≥ 100% means USD1 is fully backed.

## Live Dashboard
https://por.worldlibertyfinancial.com
