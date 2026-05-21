# Changelog

All notable changes to the KAT Score API are documented here.

---

## v0.3.0 — KaelAi Shield Phase 1 ✅ COMPLETE
**Released: 2026-05-21**

### Overview

Phase 1 delivers KaelAi Shield — a dedicated DeFi security scoring mode layered
on top of the existing KAT Score engine. Shield mode reweights the five scoring
dimensions for threat detection, introduces a five-tier recommended action system,
checks every wallet against the exploit and trusted registries, and applies a $0.05
PAYG billing charge per Shield query. Phase 1 is production-ready and live.

---

### What Was Built

#### New endpoint parameter: `mode`
`POST /api/v1/score` now accepts `mode=agent` (default, unchanged) or `mode=shield`.

**Shield weight rebalancing:**

| Dimension | Agent Weight | Shield Weight | Rationale |
|---|---|---|---|
| Transaction Legitimacy | 20% | **30%** | Primary threat signal — clean counterparties matter most |
| Behavioral Consistency | 20% | **25%** | Automation patterns distinguish agents from attackers |
| Counterparty Quality | 20% | **20%** | Unchanged |
| Wallet Age & History | 20% | **15%** | Age less relevant for fast-moving exploits |
| Volume Stability | 20% | **10%** | Least predictive of threat in DeFi attack patterns |

#### Five-tier recommended action system (score-driven)

Shield mode replaces the binary proceed/review/decline with a five-tier output
that maps directly to operational response:

| Tier | Score Range | Trigger | Alert Severity |
|---|---|---|---|
| **ALLOW** | 70–100 | No critical flags | `none` |
| **MONITOR** | 50–69 | Minor flags only | `low` |
| **REVIEW** | 25–49 | Multiple flags present | `medium` |
| **FLAG** | 15–24 | Strong threat signals | `high` |
| **BLOCK** | 0–14 | Any score below threshold | `critical` |

Registry overrides always take priority over score bands:
- **Threat registry match** → BLOCK (confidence 0.99, regardless of score)
- **Trusted registry match** → ALLOW (confidence 0.99, all flags suppressed)

Flag-based tier adjustments:
- Critical flag (`tornado_cash_funded`) → escalates to minimum FLAG regardless of score
- 1+ strong flag at MONITOR tier → downgrade to REVIEW
- 2+ strong flags at ALLOW tier → downgrade to MONITOR

#### Phase 1 Shield risk flags (six implemented)

| Flag | Trigger |
|---|---|
| `receive_only_pattern` | No outbound transactions, or inbound/outbound ratio > 20 |
| `zero_known_protocol_ratio` | 0% known protocol interactions with ≥ 3 contract calls |
| `tornado_cash_funded` | Three detection paths — see Tornado Cash Detection section below ✅ Fully functional |
| `value_spike_anomaly` | Value spike ratio ≥ 50× (p95/median) |
| `counterparty_concentration` | Single counterparty accounts for ≥ 70% of volume |
| `failed_tx_anomaly` | ≥ 15% of transactions failed |

#### New Shield response fields

All fields present when `mode=shield`:

```json
{
  "scoring_mode":             "shield",
  "overall_score":            <shield-weighted 0-100>,
  "agent_score":              <original equal-weight score, for reference>,
  "threat_classification":    "confirmed_exploit_wallet | mixer_funded | draining_address | suspicious_pattern | low_protocol_engagement | clean | verified_good_actor",
  "threat_confidence":        0.0–1.0,
  "shield_flags":             ["receive_only_pattern", ...],
  "registry_checked":         true,
  "registry_match":           { "matched": true, "registry_type": "threat|trusted", ... } | null,
  "alert_severity":           "critical | high | medium | low | none",
  "recommended_action":       "block | flag | review | monitor | allow",
  "recommended_action_label": "<plain-English explanation>",
  "shield_data": {
    "shield_score":           <int>,
    "shield_weights_applied": { ... },
    "tier_thresholds":        { ... },
    ...
  }
}
```

#### Exploit & Trusted Registry (`exploit_registry` table)

New `registry_type` column (`threat` | `trusted`) enables dual-purpose use
of a single registry table. Threat entries trigger BLOCK; trusted entries
trigger ALLOW with all behavioral flags suppressed.

**Current registry state (11 entries):**

Threat entries (BLOCK on match):

| Incident | Date | Amount | Wallets |
|---|---|---|---|
| Drift Protocol Exploit | 2026-04-01 | $285M | 4 wallets |
| Kelp DAO Exploit | 2026-04-18 | $292M | 1 wallet (Tornado Cash funded) |

Trusted entries (ALLOW on match):

| Entity | Role |
|---|---|
| vitalik.eth `0xd8dA…96045` | verified_public |
| Vitalik Buterin 2018 `0xab58…ec9b` | verified_public |
| Buterin Known Wallet `0x2208…3A9D` | verified_public |
| Uniswap V3 Router `0xE592…1564` | verified_contract |
| Aave V3 Pool `0x8787…4E2` | verified_contract |
| Coinbase Hot Wallet `0xA9D1…3E43` | verified_exchange |

#### New database tables

| Table | Purpose |
|---|---|
| `exploit_registry` | Threat and trusted wallet registry with `registry_type` column |
| `monitored_contracts` | Risk-rated contract address watchlist (Phase 2 population) |
| `shield_alerts` | Persistent alert log for BLOCK/FLAG/REVIEW events |

New columns on `kat_scores`: `scoring_mode VARCHAR(20)`, `shield_data JSONB`.

#### Shield PAYG billing

Shield queries are billed at **$0.05/query** applied on top of the base query
charge. Visible in the response `billing` block as `shield_query_charge: 0.05`
and `monthly_billing_amount_after_shield`. Spend limit gates apply post-charge.

---

### Tornado Cash Detection ✅ Complete
**Implemented: 2026-05-21**

The `tornado_cash_funded` flag is **fully functional** with three active detection paths.

#### Detection paths (any one fires the flag)

1. **Registry path** — wallet manually tagged in `exploit_registry` with
   `funding_source = 'tornado_cash'`. Highest confidence; used for confirmed incident
   wallets such as the Kelp DAO exploit.

2. **Live contract scan** *(new)* — on every fresh Shield query, all unique
   `from_address` / `to_address` values in the wallet's 50-transaction sample are
   checked against the `tornado_cash_contracts` table. Match → `mixer_contact_detected`
   injected into `behavioral_features` → flag fires. Redis-cached per chain (1hr TTL)
   so the lookup adds no meaningful latency.

3. **Indirect funding trace** *(Phase 2)* — detect wallets funded by known TC withdrawal
   addresses one hop away. Not yet implemented.

#### `tornado_cash_contracts` table — 35 entries seeded across 5 chains

| Chain | Entries | Source | Verification status |
|---|---|---|---|
| ETH | 23 | OFAC SDN list, Aug 8 2022 + Etherscan labels | ✅ 21 OFAC-verified, 2 needs_review |
| BSC | 5 | Community-sourced | ⚠️ All needs_review |
| Polygon | 4 | Community-sourced | ⚠️ All needs_review |
| Arbitrum | 2 | Community-sourced | ⚠️ All needs_review |
| Optimism | 1 | Community-sourced | ⚠️ All needs_review |

ETH coverage: 0.1/1/10/100 ETH pools, 100/1k/10k/100k DAI pools,
100/1k/5k/50k USDC pools, cDAI pools, TC Router, TC Proxy, TC Nova.

All non-ETH entries carry `needs_review = TRUE` in the database and must be
validated against current Etherscan labels before being relied upon in production
enforcement decisions.

#### New service: `app/services/tornado_cash.py`
- `get_tc_address_set(chain, db)` — DB query with 1hr Redis cache per chain
- `check_tornado_cash_interaction(counterparty_addrs, chain, db)` — returns
  `(detected: bool, matched_address: str | None)`
- `invalidate_tc_cache(chain)` — bust cache after registry updates

#### Behavioral feature added
`all_counterparty_addresses` added to `behavioral_features` JSONB — full deduplicated
set of all from/to counterparty addresses in the 50-tx sample (previously capped
at 15 samples). Used by Shield TC detection and available for future flag work.

---

### Live Tier Confirmation

The following tiers were confirmed against real Ethereum wallets on 2026-05-21:

| Tier | Wallet | Shield Score | Action | Notes |
|---|---|---|---|---|
| BLOCK | Drift `0xD3FE…F6C7` | 16 | 🔴 BLOCK | Threat registry match — $285M exploit |
| BLOCK | Drift `0xAa84…57C1` | 22 | 🔴 BLOCK | Threat registry match — $285M exploit |
| BLOCK | Kelp `0x4966…75E` | 18 | 🔴 BLOCK | Threat registry + `tornado_cash_funded` flag |
| FLAG | Binance cold `0xBE0E…` | 16 | 🟠 FLAG | Score 15–24, `receive_only_pattern` + `counterparty_concentration` |
| FLAG | Ethereum Foundation `0xde0B…` | 16 | 🟠 FLAG | Score 15–24, strong flags |
| REVIEW | Coinbase 3 `0x5038…` | 38 | 🟡 REVIEW | Score 25–49, `counterparty_concentration` |
| REVIEW | Binance 1 `0x3f5C…` | 27 | 🟡 REVIEW | Score 25–49, multiple flags |
| ALLOW | vitalik.eth `0xd8dA…` | 16 | 🟢 ALLOW | Trusted registry match, flags suppressed |
| ALLOW | Vitalik 2018 `0xab58…` | 16 | 🟢 ALLOW | Trusted registry match, flags suppressed |
| ALLOW | Buterin wallet `0x2208…` | 17 | 🟢 ALLOW | Trusted registry match, flags suppressed |

**MONITOR tier (50–69):** Code path is implemented and correct. Live confirmation
pending against a personal active DeFi wallet with sustained outbound protocol
interactions. See note below on conservative scoring behavior.

---

### Note on Conservative Scoring Behavior (By Design)

Shield mode is intentionally conservative. In testing across 20+ real Ethereum
wallets, the majority score below 50 — including well-known legitimate addresses
such as the Ethereum Foundation treasury, Binance cold wallets, and Coinbase
hot wallets. This is expected and correct for a security tool.

**Why wallets score low in Shield mode:**
- Most public Ethereum wallets are heavily inbound-skewed in any 50-tx window
  (donations, airdrops, unsolicited sends), triggering `receive_only_pattern`
- Exchange and foundation addresses have highly concentrated counterparty volume,
  triggering `counterparty_concentration`
- Low `known_protocol_ratio` is common even for legitimate wallets because most
  interactions are with unverified or new contracts not yet in the Etherscan label set

**The correct operational interpretation:** ALLOW should be rare. A Shield score
of 70+ indicates a wallet with consistent outbound DeFi engagement across verified
protocols, stable volumes, and no anomalous patterns — a high bar that genuine
autonomous agent wallets should clear but that most human-operated or custodial
addresses will not. For known legitimate entities that score low behaviorally,
the trusted registry provides the override path (ALLOW at 0.99 confidence).

MONITOR and ALLOW tiers exist for wallets that earn them through behavioral quality,
not as defaults. This is the right design for a DeFi security product.

---

## Phase 2 — Priorities

### 1. Etherscan Entity Label Integration (wallet-level)
**Priority: High**

Automatically recognise named entities without manual registry entries.
On Shield queries, call the Etherscan address tag API for the wallet address
itself. A returned label (e.g. "Binance: Hot Wallet", "Uniswap V3: Router")
sets `threat_classification = verified_entity` and `recommended_action = allow`
with confidence 0.85 (lower than manual registry at 0.99). Cache labels in Redis
for 7 days. Surface as `etherscan_entity_label` in `shield_data`.

Affected: `shield_scorer.py`, `exploit_registry.py`, `score.py`, `score.py` model.

### 2. Tornado Cash Contract Validation — BSC / Polygon / Arbitrum / Optimism
**Priority: High**

The `tornado_cash_contracts` table was seeded with community-sourced addresses
for BSC (5), Polygon (4), Arbitrum (2), and Optimism (1). All carry
`needs_review = TRUE`. Before these chains are relied upon in production:

- Validate each address against current Etherscan / BscScan / PolygonScan labels
- Cross-reference against the Chainalysis and TRM public TC address datasets
- Update `verified_source` and set `needs_review = FALSE` on confirmed entries
- Remove any entries that cannot be verified

Run `invalidate_tc_cache(chain)` after any table updates to bust the 1hr Redis cache.

### 3. Indirect Tornado Cash Funding Trace
**Priority: Medium**

Detection path 3 (not yet implemented): detect wallets funded by known TC
withdrawal addresses one hop away. Implementation approach:

- On Shield query, check if any of the wallet's inbound `from_address` values
  is a known TC withdrawal address (i.e., an address that previously received
  a TC withdrawal)
- Requires a `tc_withdrawal_addresses` table or integration with a third-party
  blockchain analytics API (Chainalysis, TRM, Nansen)
- Flag with lower confidence than direct interaction (0.70 vs 0.99)
- Surface as `tornado_cash_indirect` sub-flag within `tornado_cash_funded`

### 4. Advanced Shield Flags
**Priority: High**

Extend Phase 1 flags with:
- `flash_loan_pattern` — single-block borrow/repay cycle (DeFi exploit signature)
- `bridge_hop_chain` — rapid cross-chain movement consistent with laundering
- `newly_funded_wallet` — wallet funded < 24h before first exploit interaction
- `contract_deployer_anomaly` — deployed contract immediately drained funds
- `dust_attack_sender` — wallet that sends sub-cent amounts to probe addresses
- `governance_manipulation` — flash-loan-funded governance vote pattern

### 3. Shield Pro Tier Launch
**Priority: Medium**

Productise Shield as a distinct paid add-on:
- Shield Pro subscription: $49/mo (included 1,000 Shield queries/mo)
- Overage at $0.05/query (same as current PAYG)
- Shield Pro unlocks: bulk registry lookup, shield_alerts webhook delivery,
  monitored_contracts watchlist API, 30-day alert history, CSV export
- Gate Shield Pro features behind `tier = shield_pro` on ApiKey model
- Stripe product + price IDs to add to config

---

*For questions or feedback: [hello@kaelai.io](mailto:hello@kaelai.io)*

## v0.2.0 — April 2026

### Added

- **Wallet type classification** — every score response now includes a `wallet_type` field classifying the wallet as one of:
  - `autonomous_agent` — consistent automated behaviour, agent-compatible
  - `likely_exchange` — high-volume hub patterns typical of centralised exchanges
  - `smart_contract` — contract address detected
  - `retail_wallet` — human-operated wallet with irregular patterns
  - `insufficient_history` — fewer than 10 transactions; score is provisional
  - `unknown` — insufficient signals to classify

- **`scope_warning` field** — when the wallet type materially affects score interpretation, a plain-English warning is included in the response explaining how to interpret the score for that wallet type (e.g. exchange wallets score differently by design)

- **Taostats API integration for Bittensor / TAO scoring** — replaced limited RPC block-scan fallback with the full Taostats API data pipeline:
  - Transfer history (up to 50 transactions) via `api/transfer/v1`
  - Account data including balance, wallet age, and subnet staking positions via `api/account/latest/v1`
  - Alpha balances (per-subnet staking positions) extracted from account response for validators operating across multiple subnets
  - Stake portfolio and miner registration where available on current API plan
  - Falls back to Substrate RPC if API key is not configured

- **Extended TAO behavioral features** — TAO scoring now surfaces:
  - `nonce` — total lifetime extrinsics (proxy: total transfer count via Taostats pagination)
  - `subnet_count` — number of subnets the wallet holds alpha positions on
  - `stake_total` — total TAO staked across all positions
  - `is_delegate` — delegate/hotkey status inferred from account data
  - `inbound_count` / `outbound_count` — directional transfer breakdown
  - `volume_total` — sum of all transfer amounts in TAO

- **Bittensor scoring context updated** — Claude Haiku scoring prompt now includes explicit thresholds: nonce > 1000 is a strong positive signal; subnet_count > 0 indicates active participation; stake_total > 1 TAO indicates meaningful network investment; is_delegate = true is a strong positive signal

- **Extended chain support** — added five additional chains:
  - Bitcoin (`btc`)
  - Solana (`sol`)
  - Avalanche (`avax`)
  - Hyperliquid (`hype`)
  - Bittensor / TAO (`tao`)

- **Chain stats endpoint** — `GET /api/v1/chain-stats` returns per-chain scoring statistics including query volume, average score, and score distribution

- **Pay-as-you-go pricing** — `$0.005 per query` top-up model now available; free tier remains 100 queries on signup with no expiry

### Changed

- TAO scoring: `wallet_age_days` now derived from Taostats `created_on_date` field (accurate calendar date) rather than estimated from block number
- TAO scoring: subnet registrations now pulled from `alpha_balances` array in account response, capturing all subnets where the coldkey holds staking positions — significantly more complete than the miner registration endpoint alone
- `force_refresh` parameter is now a **query parameter** (`?force_refresh=true`), not a request body field
- README and QUICKSTART updated to reflect 10-chain support, wallet type classification, scope warnings, and current pricing

### Fixed

- Taostats API authentication: corrected from `Authorization: Bearer <key>` to `Authorization: <key>` (Taostats uses raw key, no Bearer prefix)
- Taostats transfer pagination: removed `page=0` parameter (Taostats uses 1-based pagination; `page=0` returns empty results)

---

## v0.1.0 — March 2026

Initial beta release.

### Added

- **KAT Score endpoint** — `POST /api/v1/score` returns a full Known Agent Trust Score for any EVM wallet address
- **Five-dimension scoring engine** — each wallet is scored across five independent dimensions (0–20 each):
  - Behavioral Consistency
  - Transaction Legitimacy
  - Wallet Age & History
  - Counterparty Quality
  - Volume Stability
- **Bond-style grade** — overall score mapped to AAA / AA / A / BBB / BB / B / CCC rating
- **Plain-English reasoning** — every response includes an AI-generated explanation of the score
- **Redis caching** — 24-hour score cache; average cached response time 9ms
- **Score history endpoint** — `GET /api/v1/score/{address}/history` returns full scoring history for a wallet, newest first
- **Score trend endpoint** — `GET /api/v1/score/{address}/trend` returns trend direction (improving / stable / declining) over a configurable window
- **Force refresh** — `?force_refresh=true` query parameter bypasses cache and triggers a fresh on-chain analysis
- **Self-serve API key registration** — `POST /api/v1/keys/register` generates a `kael_live_` key instantly, no waitlist
- **Public stats endpoint** — `GET /api/v1/stats` returns live usage statistics (cached 5 minutes)
- **Multi-chain support** — ETH, Base, Polygon, BSC, Arbitrum
- **Rate limiting** — registration endpoint rate-limited to 3 attempts/IP/hour

---

*For questions or feedback: [hello@kaelai.io](mailto:hello@kaelai.io)*


## Phase 2 — Planned

### Etherscan Entity Label Integration (wallet-level)
**Priority: High**

Currently the exploit_registry requires manual entries for known good actors.
Etherscan exposes entity labels for well-known addresses (exchanges, protocols,
foundations, bridges) via their address tag endpoint. Phase 2 should integrate
this at the wallet level so named entities are automatically recognised without
requiring registry entries.

**Implementation notes:**
- On Shield mode queries, call Etherscan address tag API for the wallet address itself
  (not just the contracts it interacts with — which is already implemented in contract_labels.py)
- If a label is returned (e.g. "Binance: Hot Wallet", "Uniswap V3: Router", "Coinbase 10"),
  treat it as a soft trusted signal: set threat_classification = verified_entity,
  recommended_action = allow, and surface the Etherscan label in the response
- Confidence for auto-labelled entities should be lower than manual registry entries
  (suggest 0.85 vs 0.99 for manual) to reflect that Etherscan labels can lag or be wrong
- Cache entity labels in Redis with a longer TTL (7 days) to avoid hammering the API
- Add etherscan_entity_label field to shield_data response block
- Fallback gracefully if Etherscan returns no label — normal scoring continues

**Affected files:** services/shield_scorer.py, services/exploit_registry.py,
api/v1/endpoints/score.py, models/score.py (add etherscan_entity_label column to shield_data JSONB)
