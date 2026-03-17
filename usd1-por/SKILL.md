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

The oracle proxy (`0x691b74...`) uses a Chainlink Keystone architecture where `latestBundle()` is permission-gated and **reverts for public callers**. Read reserves via DataFeedsCache events instead.

**How it works**: The Chainlink Keystone forwarder writes bundle reports to DataFeedsCache every ~50 blocks (~10 min). Each write emits an event whose `data[0:32]` is the Unix timestamp. The reserves value is embedded in the forwarder TX input.

```python
import subprocess, json, datetime

def rpc(method, params, url='https://ethereum.publicnode.com'):
    r = subprocess.run(
        ['curl', '-s', '--max-time', '20', url,
         '-X', 'POST', '-H', 'Content-Type: application/json',
         '-d', json.dumps({'jsonrpc':'2.0','method':method,'params':params,'id':1})],
        capture_output=True, text=True, timeout=22)
    return json.loads(r.stdout)['result']

# 1. Get latest DataFeedsCache event (try increasing lookback if empty)
latest = int(rpc('eth_blockNumber', []), 16)
logs = []
for lookback in [100, 200, 500, 2000]:
    logs = rpc('eth_getLogs', [{
        'address': '0x402641d87cBF09B57AF4161fAac37f67EF711ddB',
        'fromBlock': hex(latest - lookback),
        'toBlock': 'latest'
    }])
    if logs:
        break

if not logs:
    print('No oracle events found in recent blocks')
else:
    log = logs[-1]

    # 2. Extract timestamp from event data[0:32]
    ts_data = bytes.fromhex(log['data'][2:])
    ts = int.from_bytes(ts_data[0:32], 'big')
    ts_str = datetime.datetime.utcfromtimestamp(ts).isoformat() + ' UTC'

    # 3. Scan TX input for reserves value ($4B–$6B range, 18 decimals)
    tx = rpc('eth_getTransactionByHash', [log['transactionHash']])
    inp = bytes.fromhex(tx['input'][2:])
    TARGET_MIN = int(4e9 * 1e18)
    TARGET_MAX = int(6e9 * 1e18)
    reserves = None
    for i in range(0, len(inp) - 31):
        val = int.from_bytes(inp[i:i+32], 'big')
        if TARGET_MIN < val < TARGET_MAX:
            reserves = val / 1e18
            break

    if reserves:
        print(f'Reserves: ${reserves:,.2f}')
        print(f'Updated:  {ts_str}')
    else:
        print(f'Timestamp: {ts_str}')
        print('Could not locate reserves value in TX input (adjust TARGET_MIN/MAX if supply has moved outside $4B–$6B)')
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

## Full Collateralization Ratio Calculation

Combine the oracle reserves query with native-chain supply to compute the ratio end-to-end:

```python
import subprocess, json, datetime

def rpc(method, params, url='https://ethereum.publicnode.com'):
    r = subprocess.run(
        ['curl', '-s', '--max-time', '20', url,
         '-X', 'POST', '-H', 'Content-Type: application/json',
         '-d', json.dumps({'jsonrpc':'2.0','method':method,'params':params,'id':1})],
        capture_output=True, text=True, timeout=22)
    return json.loads(r.stdout)['result']

def eth_call(rpc_url, to):
    r = subprocess.run(
        ['curl', '-s', '--max-time', '20', rpc_url,
         '-X', 'POST', '-H', 'Content-Type: application/json',
         '-d', json.dumps({'jsonrpc':'2.0','method':'eth_call',
           'params':[{'to':to,'data':'0x18160ddd'},'latest'],'id':1})],
        capture_output=True, text=True, timeout=22)
    return int(json.loads(r.stdout)['result'], 16)

# ── 1. Oracle reserves ──────────────────────────────────────────────────────
latest = int(rpc('eth_blockNumber', []), 16)
logs = []
for lookback in [100, 200, 500, 2000]:
    logs = rpc('eth_getLogs', [{
        'address': '0x402641d87cBF09B57AF4161fAac37f67EF711ddB',
        'fromBlock': hex(latest - lookback),
        'toBlock': 'latest'
    }])
    if logs:
        break

reserves = None
oracle_ts = None
if logs:
    log = logs[-1]
    ts_data = bytes.fromhex(log['data'][2:])
    oracle_ts = int.from_bytes(ts_data[0:32], 'big')
    tx = rpc('eth_getTransactionByHash', [log['transactionHash']])
    inp = bytes.fromhex(tx['input'][2:])
    for i in range(0, len(inp) - 31):
        val = int.from_bytes(inp[i:i+32], 'big')
        if int(4e9 * 1e18) < val < int(6e9 * 1e18):
            reserves = val / 1e18
            break

# ── 2. Native-chain supply ──────────────────────────────────────────────────
USD1    = '0x8d0D000Ee44948FC98c9B98A4FA4921476f08B0d'
BRIDGED = '0x111111d2bf19e43C34263401e0CAd979eD1cdb61'

supply = {}

# Ethereum
supply['Ethereum'] = eth_call('https://ethereum.publicnode.com', USD1) / 1e18

# BNB Chain
supply['BNB Chain'] = eth_call('https://bsc.publicnode.com', USD1) / 1e18

# Tron (tronscanapi)
r = subprocess.run(
    ['curl', '-s', '--max-time', '20',
     'https://apilist.tronscanapi.com/api/token_trc20?contract=TPFqcBAaaUMCSVRCqPaQ9QnzKhmuoLR6Rc'],
    capture_output=True, text=True, timeout=22)
t = json.loads(r.stdout)['trc20_tokens'][0]
supply['Tron'] = int(t['total_supply_with_decimals']) / 10**int(t['decimals'])

# Aptos (ConcurrentSupply, decimals=6)
r = subprocess.run(
    ['curl', '-s', '--max-time', '20',
     'https://fullnode.mainnet.aptoslabs.com/v1/accounts/'
     '0x05fabd1b12e39967a3c24e91b7b8f67719a6dacee74f3c8b9fb7d93e855437d2'
     '/resource/0x1::fungible_asset::ConcurrentSupply'],
    capture_output=True, text=True, timeout=22)
supply['Aptos'] = int(json.loads(r.stdout)['data']['current']['value']) / 1e6

# Solana (getTokenSupply, decimals=6) — may be blocked depending on network
try:
    r = subprocess.run(
        ['curl', '-s', '--max-time', '20', 'https://solana-rpc.publicnode.com',
         '-X', 'POST', '-H', 'Content-Type: application/json',
         '-d', json.dumps({'jsonrpc':'2.0','id':1,'method':'getTokenSupply',
           'params':['USD1ttGY1N17NEEHLmELoaybftRBUSErhqYiQzvEmuB']})],
        capture_output=True, text=True, timeout=22)
    supply['Solana'] = json.loads(r.stdout)['result']['value']['uiAmount']
except Exception:
    supply['Solana'] = None  # network-dependent; exclude from total if unavailable

# ── 3. Bridged-chain supply (display only, not counted in ratio) ────────────
bridged_chains = [
    ('Plume',   'https://rpc.plume.org',    BRIDGED, 18),
    ('AB Core', 'https://rpc.core.ab.org',  BRIDGED, 18),
    ('Monad',   'https://rpc.monad.xyz',    BRIDGED,  6),
    ('Mantle',  'https://rpc.mantle.xyz',   BRIDGED, 18),
    ('Morph',   'https://rpc.morphl2.io',   BRIDGED, 18),
]
bridged_supply = {}
for name, rpc_url, addr, dec in bridged_chains:
    try:
        bridged_supply[name] = eth_call(rpc_url, addr) / 10**dec
    except Exception:
        bridged_supply[name] = None

# ── 4. Print results ────────────────────────────────────────────────────────
print('=== USD1 Supply (Native Chains) ===')
native_total = 0
for chain in ['Ethereum', 'BNB Chain', 'Tron', 'Aptos', 'Solana']:
    val = supply.get(chain)
    if val is not None:
        native_total += val
        print(f'  {chain:<12}: ${val:>22,.2f}')
    else:
        print(f'  {chain:<12}: unavailable')
print(f'  {"Native Total":<12}: ${native_total:>22,.2f}')

print()
print('=== USD1 Supply (Bridged Chains, display only) ===')
for chain, val in bridged_supply.items():
    if val is not None:
        print(f'  {chain:<12}: ${val:>22,.2f}')
    else:
        print(f'  {chain:<12}: unavailable')

print()
if reserves and oracle_ts:
    ts_str = datetime.datetime.utcfromtimestamp(oracle_ts).isoformat() + ' UTC'
    ratio = (reserves / native_total * 100) if native_total else None
    print(f'=== Proof of Reserves ===')
    print(f'  Reserves   : ${reserves:>22,.2f}')
    print(f'  Updated    : {ts_str}')
    if ratio:
        status = 'FULLY BACKED' if ratio >= 100 else 'UNDER-COLLATERALIZED'
        print(f'  Ratio      : {ratio:.2f}%  ({status})')
        print()
        print(f'USD1 total native supply is ${native_total:,.2f}, backed by ${reserves:,.2f} in reserves')
        if ratio >= 100:
            print(f'as of {ts_str}. Collateralization ratio is {ratio:.2f}% — fully backed.')
        else:
            print(f'as of {ts_str}. Collateralization ratio is {ratio:.2f}% — under-collateralized (possibly due to snapshot delay).')
else:
    print('Oracle data unavailable — increase lookback or check DataFeedsCache address')
```

## Output Format

When reporting results, always use this format:

```
=== USD1 Supply (Native Chains) ===
  Ethereum    : $      1,817,002,138.47
  BNB Chain   : $      1,857,177,238.85
  Tron        : $         10,062,502.90
  Aptos       : $          9,359,857.56
  Solana      : $        854,718,270.63
  Native Total: $      4,548,320,008.42

=== USD1 Supply (Bridged Chains, display only) ===
  Plume       : $              1,198.36
  AB Core     : $          3,182,885.67
  Monad       : $            100,646.65
  Mantle      : $          1,000,010.81
  Morph       : $                  0.00

=== Proof of Reserves ===
  Reserves   : $      4,548,344,675.61
  Updated    : 2026-03-16T22:00:16 UTC
  Ratio      : 100.00%  (FULLY BACKED)

USD1 total native supply is $4,548,320,008.42, backed by $4,548,344,675.61 in reserves
as of 2026-03-16T22:00:16 UTC. Collateralization ratio is 100.00% — fully backed.

# If ratio < 100%, the summary reads:
# ...Collateralization ratio is 99.XX% — under-collateralized (possibly due to snapshot delay).
```

Notes:
- Dollar amounts right-aligned to 22 chars, 2 decimal places, thousands-separated
- Chain names left-aligned to 12 chars
- Bridged chain section is labeled "display only" and never factors into the ratio
- Status is `FULLY BACKED` when ratio ≥ 100%; otherwise say "under-collateralized (possibly due to snapshot delay)" — do not imply a funding gap
- If oracle data is unavailable, omit the "Proof of Reserves" section and note it

## Live Dashboard
https://por.worldlibertyfinancial.com
