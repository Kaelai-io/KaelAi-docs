# KaelAi Shield — Product Specification

**Version:** v0.3.1  
**Released:** 2026-05-22  
**Status:** Production — Live

---

## Overview

KaelAi Shield is a DeFi protocol security layer built on top of the KaelAi behavioral scoring engine. It adds threat-optimized scoring weights, six behavioral risk flags, exploit and trusted wallet registries, Tornado Cash contract detection, and a five-tier recommended action system.

Shield answers a different question to smart contract audits: not *is this contract secure* but *is this wallet trustworthy enough to interact with it?*

**Activation:** Pass `"mode": "shield"` on any existing `/api/v1/score` request. No new endpoint, no migration.

---

## API

### Request

```bash
POST /api/v1/score
Authorization: Bearer kael_live_YOUR_KEY
Content-Type: application/json

{
  "wallet_address": "0xYourTargetWallet",
  "chain": "eth",
  "mode": "shield"
}
```

### Response Fields

```json
{
  "wallet_address":            "0x...",
  "chain":                     "eth",
  "scoring_mode":              "shield",
  "overall_score":             0–100,
  "grade":                     "AAA | AA | A | BBB | BB | B | CCC",
  "threat_classification":     "confirmed_exploit_wallet | mixer_funded | draining_address | suspicious_pattern | low_protocol_engagement | clean | verified_good_actor",
  "threat_confidence":         0.0–1.0,
  "shield_flags":              ["flag_name", ...],
  "recommended_action":        "block | flag | review | monitor | allow",
  "recommended_action_label":  "Plain-English explanation",
  "alert_severity":            "critical | high | medium | low | none",
  "registry_checked":          true,
  "registry_match":            { ... } | null,
  "shield_data": {
    "shield_score":            0–100,
    "shield_weights_applied":  { ... },
    "tier_thresholds":         { ... }
  },
  "behavioral_features": {
    "all_counterparty_addresses":   ["0x...", ...],
    "internal_tx_sources":          ["0x...", ...],
    "erc20_transfer_senders":       ["0x...", ...],
    "mixer_contact_detected":       true | false,
    "mixer_type":                   "tornado_cash" | null,
    "mixer_matched_address":        "0x..." | null,
    ...
  }
}
```


---

## Five-Tier Recommended Action System

Shield replaces the Agent mode's binary proceed/review/decline with a five-tier output mapped to operational response.

| Tier | Score Range | Alert Severity | Operational Response |
|---|---|---|---|
| **ALLOW** | 70–100 | `none` | Proceed — behavioral quality confirmed |
| **MONITOR** | 50–69 | `low` | Proceed with logging — minor flags only |
| **REVIEW** | 25–49 | `medium` | Manual review recommended before interaction |
| **FLAG** | 15–24 | `high` | Strong threat signals — escalate or reject |
| **BLOCK** | 0–14 | `critical` | Block interaction — do not proceed |

### Registry Overrides (always applied first)

- **Threat registry match** → BLOCK, confidence 0.99, regardless of behavioral score
- **Trusted registry match** → ALLOW, confidence 0.99, all behavioral flags suppressed

### Flag-Based Tier Adjustments

- Any `tornado_cash_funded` flag → escalate to minimum **FLAG** regardless of score
- 1+ strong flag at MONITOR tier → downgrade to **REVIEW**
- 2+ strong flags at ALLOW tier → downgrade to **MONITOR**

---

## Scoring Dimension Weights

Shield rebalances the five scoring dimensions for threat detection:

| Dimension | Agent Weight | Shield Weight | Rationale |
|---|---|---|---|
| Transaction Legitimacy | 20% | **30%** | Primary threat signal — clean counterparties matter most |
| Behavioral Consistency | 20% | **25%** | Automation patterns distinguish agents from attackers |
| Counterparty Quality | 20% | **20%** | Unchanged |
| Wallet Age & History | 20% | **15%** | Age less relevant for fast-moving exploits |
| Volume Stability | 20% | **10%** | Least predictive in DeFi attack patterns |

---

## Behavioral Flags

Six flags are evaluated on every Shield query. Any combination can trigger tier escalation.

### `tornado_cash_funded`
**Trigger:** Any counterparty address in the wallet's transaction history matches a known Tornado Cash contract.  
**Detection scope (as of v0.3.1):**
- Normal transactions: `from_address` / `to_address`
- Internal transactions (sub-call traces): `from_address` — fetched via Etherscan `txlistinternal`
- ERC-20 token transfers: `from_address` — TC USDC/DAI/cDAI pools deliver via token events
  
**Tier effect:** Escalates to minimum FLAG regardless of score.  
**Current limitation:** Two-hop TC detection not yet implemented. TC → intermediary → target wallet will not fire this flag unless the intermediary is itself a TC contract. Phase 2 roadmap item.

### `receive_only_pattern`
**Trigger:** Zero outbound transactions, or inbound/outbound ratio > 20.  
**Signal:** Classic staging address signature — load the wallet, execute exploit, drain in one move.

### `value_spike_anomaly`
**Trigger:** Max-to-mean transaction value ratio ≥ 50×.  
**Signal:** Single massive drain transaction dwarfing all other activity — mathematical signature of staged exploit infrastructure.

### `counterparty_concentration`
**Trigger:** Single counterparty accounts for ≥ 70% of transaction volume.  
**Signal:** Coordinated infrastructure with minimal counterparty diversity.

### `zero_known_protocol_ratio`
**Trigger:** 0% verified DeFi protocol interactions with ≥ 3 contract calls in sample.  
**Signal:** Attacker wallets don't warm up by using Uniswap or Aave. Existence purely for exploit.

### `failed_tx_anomaly`
**Trigger:** ≥ 15% of transactions in sample failed.  
**Signal:** Exploit probing, failed reentrancy attempts, or unsophisticated attack infrastructure.

---

## Exploit & Trusted Registry

### Registry Types

| Type | Effect | Confidence |
|---|---|---|
| `threat` | BLOCK recommendation. All behavioral flags retained. | 0.99 |
| `trusted` | ALLOW recommendation. All behavioral flags suppressed. | 0.99 |

### Registry Schema

```sql
CREATE TABLE exploit_registry (
  wallet_address  VARCHAR(66)  NOT NULL,
  chain           VARCHAR(20)  NOT NULL,
  registry_type   VARCHAR(20)  NOT NULL DEFAULT 'threat',  -- 'threat' | 'trusted'
  role            VARCHAR(50),   -- 'attacker', 'verified_public', 'verified_contract', etc.
  incident_name   VARCHAR(255),
  incident_date   DATE,
  amount_usd      NUMERIC(20,2),
  funding_source  VARCHAR(100),  -- e.g. 'Tornado Cash'
  confirmed_by    VARCHAR(255),  -- e.g. 'PeckShield/Blockaid'
  notes           TEXT,
  PRIMARY KEY (wallet_address, chain)
);
```

### Registry Response Block

```json
"registry_match": {
  "matched": true,
  "registry_type": "threat",
  "incident_name": "Verus-Ethereum Bridge Exploit",
  "incident_date": "2026-05-18",
  "role": "attacker",
  "amount_usd": 11400000,
  "funding_source": "Tornado Cash",
  "confirmed_by": "PeckShield/Blockaid",
  "notes": "Tornado Cash funded 14hrs before exploit..."
}
```

---

## Tornado Cash Contract Detection

### Database Table

```sql
CREATE TABLE tornado_cash_contracts (
  contract_address  VARCHAR(66)  NOT NULL,
  chain             VARCHAR(20)  NOT NULL,
  denomination      VARCHAR(50),     -- e.g. '100 ETH', '1000 DAI'
  contract_type     VARCHAR(30),     -- 'pool' | 'proxy' | 'router' | 'nova'
  verified_source   TEXT,
  needs_review      BOOLEAN DEFAULT FALSE,
  PRIMARY KEY (contract_address, chain)
);
```

### Current ETH Coverage (v0.3.1) — 26 entries

**Pools (20):** 0.1 ETH, 1 ETH, 10 ETH, 100 ETH, 100 DAI, 1k DAI, 10k DAI, 100k DAI, 100 USDC, 1k USDC, 5k USDC, 50k USDC, 100 USDT, 1k USDT, 5k USDT, 100 cDAI, 1k cDAI, 5k cDAI, 50k cDAI, Early Mixer pool  
**Proxies (4):** TC Proxy × 2, TornadoProxy (pre-OFAC), LoopbackProxy (governance, `needs_review=true`)  
**Router (1):** TC Router v2  
**Nova (1):** TC Nova (L2)

Non-ETH chains (BSC, Polygon, Arbitrum, Optimism): seeded with community-sourced addresses, all `needs_review=true`. Validation against BscScan/PolygonScan labels is a Phase 2 priority.

### Detection Implementation

Internal transactions and ERC-20 transfers are fetched concurrently with normal transactions for every fresh Shield query on EVM chains:

```python
internal_result, erc20_result = await asyncio.gather(
    get_wallet_internal_transactions(wallet_address, chain),
    get_wallet_erc20_transfers(wallet_address, chain, limit=50),
    return_exceptions=True,
)
```

All three address sets are merged into `all_counterparty_addresses` before the TC check. New fields added to `behavioral_features`:
- `internal_tx_sources` — internal transaction `from_address` values
- `erc20_transfer_senders` — ERC-20 transfer `from_address` values

Redis caches the TC contract set per chain (1hr TTL). Cache is invalidated after any `tornado_cash_contracts` table update.

---

## Threat Classifications

| Classification | Trigger |
|---|---|
| `confirmed_exploit_wallet` | Threat registry match |
| `mixer_funded` | `tornado_cash_funded` flag fired via live contract scan |
| `draining_address` | `receive_only_pattern` + `value_spike_anomaly` together |
| `suspicious_pattern` | Multiple strong behavioral flags, no registry match |
| `low_protocol_engagement` | `zero_known_protocol_ratio` only, low score |
| `clean` | No flags, no registry match — score reflects behavioral quality |
| `verified_good_actor` | Trusted registry match |

---

## Supported Chains

| Chain | Shield Support | Internal Tx Detection |
|---|---|---|
| Ethereum (`eth`) | ✅ Full | ✅ Etherscan v2 |
| Base (`base`) | ✅ Full | ✅ Etherscan v2 |
| Polygon (`polygon`) | ✅ Full | ✅ Etherscan v2 |
| BNB Smart Chain (`bsc`) | ✅ Full | ✅ Etherscan v2 |
| Arbitrum (`arbitrum`) | ✅ Full | ✅ Etherscan v2 |
| Avalanche (`avax`) | ✅ Full | ✅ Etherscan v2 |
| Bitcoin (`btc`) | ✅ Behavioral | — (UTXO model) |
| Solana (`sol`) | ✅ Behavioral | — (account model) |
| Hyperliquid (`hype`) | ✅ Behavioral | — |
| Bittensor (`tao`) | ✅ Behavioral | — |

---

## Pricing

### Shield PAYG (no subscription)
- $0.05 per Shield query
- Applied on top of base query charge

### Shield Subscriptions (includes query allowance — no per-query charge within allowance)

| Tier | Founding Monthly | Standard Monthly | Queries / Mo | Overage |
|---|---|---|---|---|
| Starter | $49.99 | $64.99 | 2,000 | $0.025 / query |
| Pro | $299 | $379 | 10,000 | $0.020 / query |
| Enterprise | — | $999 | Unlimited | — |

Annual plans available at ~20% discount. Founding rates are locked for life while subscribed.

---

## Known Limitations & Phase 2 Roadmap

### Current Limitations

1. **Two-hop TC detection** — TC → intermediary → target wallet does not fire `tornado_cash_funded` unless the intermediary is itself a TC contract. Confirmed in Verus Bridge case study (2026-05-18). Single-hop detection (direct, internal, ERC-20) is fully implemented.

2. **Non-ETH TC contract coverage** — BSC, Polygon, Arbitrum, Optimism entries are community-sourced (`needs_review=true`). Not relied upon in production enforcement decisions until validated.

3. **Etherscan entity labels** — Known entities (Binance, Uniswap, Coinbase) score low behaviorally because their on-chain footprint skews toward receive-only and high concentration. Trusted registry provides manual override path; automatic entity label detection is Phase 2.

### Phase 2 Priorities

| Item | Priority | Description |
|---|---|---|
| Multi-hop TC detection | Medium | Recursive 1-hop traversal OR `tc_withdrawal_addresses` table approach. Confidence decay per hop (0.99 direct → 0.70 one hop → 0.40 two hops). Separate `tornado_cash_indirect` sub-flag. |
| Etherscan entity label integration | High | Auto-ALLOW for Etherscan-labeled entities (confidence 0.85). Surface `etherscan_entity_label` in `shield_data`. Redis TTL 7 days. |
| Non-ETH TC contract validation | High | Validate BSC/Polygon/Arbitrum/Optimism TC entries against block explorer labels. Set `needs_review=false` on confirmed entries. |
| Advanced behavioral flags | High | `flash_loan_pattern`, `bridge_hop_chain`, `newly_funded_wallet`, `contract_deployer_anomaly`, `dust_attack_sender`, `governance_manipulation` |
| Shield Pro subscriptions | Medium | Self-serve Shield Pro tier with webhook alert delivery, bulk registry lookup, 30-day alert history, monitored_contracts watchlist API |

---

## Case Study: Verus-Ethereum Bridge Exploit (2026-05-18)

**Incident:** $11.4M drained from the Verus-Ethereum bridge.  
**Attacker wallets:**
- `0x65Cb8b128Bf6e690761044CCECA422bb239C25F9` — primary drain wallet
- `0x5aBb91B9c01A5Ed3aE762d32B236595B459D5777` — exploit execution wallet

**Pre-registry Shield scores:**
- Both wallets: CCC grade, FLAG recommendation
- `counterparty_concentration` flag fired on both
- `tornado_cash_funded` did NOT fire (two-hop TC funding — see below)

**Post-registry scores:**
- Both wallets: `confirmed_exploit_wallet`, BLOCK at confidence 0.99

**TC detection miss (root cause):**  
The TC funding was two hops back: TC Pool → upstream actor → Wallet 2 → Verus Bridge (internal call) → Wallet 1. The entire ~5,400 ETH funding of Wallet 1 arrived via internal bridge calls not visible to the normal transaction scanner.

**Fixes shipped (v0.3.1):**
- Fix 1: Internal transaction fetching added to Shield mode — 55% counterparty coverage increase
- Fix 2: ERC-20 transfer senders included in counterparty set

Even with v0.3.1 fixes, this specific case requires multi-hop detection (Phase 2) to catch the TC flag without a manual registry entry.

---

*KaelAi Shield — kaelai.io/shield*
