---
name: usd1-por
description: Query USD1 stablecoin on-chain data — total supply across 10 chains, CCIP pool balances, and Proof of Reserves oracle status
user-invocable: true
---

# USD1 Proof of Reserves

Query USD1 stablecoin supply and reserve data directly from on-chain sources.
Use `curl` + JSON-RPC (no extra tooling required). `cast` (Foundry) also works if available.

## Key Contracts & Addresses

### Proof of Reserves Oracle (Ethereum mainnet)
- **Proxy**: `0x691b74146cdba162449012aa32d3cbf5df77d4c4`
- **Underlying DataFeedsCache**: `0x402641d87cBF09B57AF4161fAac37f67EF711ddB`
- **Decimals**: 18
- **Status**: Chainlink Keystone feed — check `latestBundleTimestamp()` first; returns 0 if no data has been written yet

| Function | Selector | Returns |
|----------|----------|---------|
| `latestBundle()` | `0xa928c096` | `bytes` — ABI-encoded `(uint256 timestamp, uint256 reserves)` |
| `latestBundleTimestamp()` | `0xe8c4be30` | `uint256` — unix timestamp of last update |
| `bundleDecimals()` | `0x9d91348d` | `uint8[]` — decimals for each bundle field |
| `description()` | `0x7284e416` | `string` |

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
| Ethereum | `https://ethereum.publicnode.com` |
| BNB Chain | `https://bsc.publicnode.com` |
| Plume | `https://rpc.plume.org` |
| AB Core | `https://rpc.core.ab.org` |
| Monad | `https://rpc.monad.xyz` |
| Mantle | `https://rpc.mantle.xyz` |
| Morph L2 | `https://rpc.morphl2.io` |
| Tron | `https://api.trongrid.io` |
| Solana | `https://solana-rpc.publicnode.com` |
| Aptos | `https://fullnode.mainnet.aptoslabs.com` |

## Querying Reserves (Oracle)

### Check if oracle has data
```bash
# latestBundleTimestamp() — returns 0 if no data written yet
curl -s https://ethereum.publicnode.com -X POST -H "Content-Type: application/json" -d '{
  "jsonrpc":"2.0","method":"eth_call",
  "params":[{"to":"0x691b74146cdba162449012aa32d3cbf5df77d4c4","data":"0xe8c4be30"},"latest"],
  "id":1
}' | python3 -c "
import sys, json
d = json.load(sys.stdin)
ts = int(d['result'], 16)
if ts == 0:
    print('Oracle has no data yet')
else:
    import datetime
    print('Last updated:', datetime.datetime.utcfromtimestamp(ts).isoformat(), 'UTC')
"
```

### Read reserves from oracle (when data is available)
```bash
# latestBundle() returns ABI-encoded bytes: (uint256 timestamp, uint256 reserves)
RESULT=$(curl -s https://ethereum.publicnode.com -X POST -H "Content-Type: application/json" -d '{
  "jsonrpc":"2.0","method":"eth_call",
  "params":[{"to":"0x691b74146cdba162449012aa32d3cbf5df77d4c4","data":"0xa928c096"},"latest"],
  "id":1
}')

# Decode with python (strip ABI bytes wrapper, then parse two uint256 values)
echo "$RESULT" | python3 -c "
import sys, json
d = json.load(sys.stdin)
raw = d.get('result', '')
if not raw or raw == '0x':
    print('No data'); exit()
# ABI-encoded bytes: offset(32) + length(32) + data(64+)
data = bytes.fromhex(raw[2:])
offset = int.from_bytes(data[0:32], 'big')
length = int.from_bytes(data[32:64], 'big')
payload = data[64:64+length]
timestamp = int.from_bytes(payload[0:32], 'big')
reserves  = int.from_bytes(payload[32:64], 'big') / 1e18
import datetime
print(f'Reserves: \${reserves:,.2f}')
print(f'Updated:  {datetime.datetime.utcfromtimestamp(timestamp).isoformat()} UTC')
"
```

### With cast (if Foundry installed)
```bash
# Check timestamp
cast call 0x691b74146cdba162449012aa32d3cbf5df77d4c4 "latestBundleTimestamp()(uint256)" \
  --rpc-url https://ethereum.publicnode.com

# Read bundle and decode
BUNDLE=$(cast call 0x691b74146cdba162449012aa32d3cbf5df77d4c4 "latestBundle()(bytes)" \
  --rpc-url https://ethereum.publicnode.com)
cast abi-decode "x()(uint256,uint256)" "$BUNDLE"
# divide second value by 1e18 to get USD reserves
```

## Querying USD1 Supply

### EVM 链 — curl (totalSupply selector: `0x18160ddd`)

```python
import subprocess, json

def eth_call(rpc, to):
    r = subprocess.run(['curl','-s', rpc, '-X','POST','-H','Content-Type: application/json',
      '-d', json.dumps({'jsonrpc':'2.0','method':'eth_call',
        'params':[{'to':to,'data':'0x18160ddd'},'latest'],'id':1})],
      capture_output=True, text=True, timeout=15)
    return json.loads(r.stdout)

USD1    = '0x8d0D000Ee44948FC98c9B98A4FA4921476f08B0d'
BRIDGED = '0x111111d2bf19e43C34263401e0CAd979eD1cdb61'

chains = [
    # (name, rpc, address, decimals)
    ('Ethereum', 'https://ethereum.publicnode.com', USD1,    18),
    ('BNB Chain','https://bsc.publicnode.com',      USD1,    18),
    ('Plume',    'https://rpc.plume.org',           BRIDGED, 18),
    ('AB Core',  'https://rpc.core.ab.org',         BRIDGED, 18),
    ('Monad',    'https://rpc.monad.xyz',           BRIDGED,  6),
    ('Mantle',   'https://rpc.mantle.xyz',          BRIDGED, 18),
    ('Morph',    'https://rpc.morphl2.io',          BRIDGED, 18),
]

for name, rpc, addr, dec in chains:
    d = eth_call(rpc, addr)
    val = int(d['result'], 16) / 10**dec
    print(f'{name:<12}: {val:>22,.2f}')
```

### Solana
```bash
# 需要可访问 Solana RPC 的网络环境
curl -s https://solana-rpc.publicnode.com -X POST -H "Content-Type: application/json" -d '{
  "jsonrpc":"2.0","id":1,"method":"getTokenSupply",
  "params":["USD1ttGY1N17NEEHLmELoaybftRBUSErhqYiQzvEmuB"]
}' | python3 -c "import sys,json; d=json.load(sys.stdin); print('Solana:', f\"{d['result']['value']['uiAmount']:,.2f}\")"
```

### Tron
```bash
# 使用 Tronscan API — total_supply_with_decimals 字段，decimals=18
curl -s "https://apilist.tronscanapi.com/api/token_trc20?contract=TPFqcBAaaUMCSVRCqPaQ9QnzKhmuoLR6Rc" \
  | python3 -c "
import sys, json
t = json.load(sys.stdin)['trc20_tokens'][0]
print('Tron:', f\"{int(t['total_supply_with_decimals']) / 10**int(t['decimals']):,.2f}\")
"
```

### Aptos
```bash
# 必须用 ConcurrentSupply，不是 Supply（后者路径不存在）
# decimals = 6，取 data.current.value
curl -s "https://fullnode.mainnet.aptoslabs.com/v1/accounts/0x05fabd1b12e39967a3c24e91b7b8f67719a6dacee74f3c8b9fb7d93e855437d2/resource/0x1::fungible_asset::ConcurrentSupply" \
  | python3 -c "
import sys, json
d = json.load(sys.stdin)
print('Aptos:', f\"{int(d['data']['current']['value']) / 1e6:,.2f}\")
"
```

## Supply Accounting Logic

**Total native supply = Ethereum + BNB + Tron + Solana + Aptos**

Bridged chain supply (Plume, AB Core, Monad, Mantle, Morph) is **excluded** — CCIP locks native USD1 on source chains and mints equivalent on destination chains; counting both would double-count.

**Collateralization ratio = Reserves / Total native supply × 100%**

A ratio ≥ 100% means USD1 is fully backed. If the oracle has no data, report supply only and note that reserve data is not yet available.

## Live Dashboard
https://por.worldlibertyfinancial.com
