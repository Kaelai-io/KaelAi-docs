```
██╗  ██╗ █████╗ ███████╗██╗
██║ ██╔╝██╔══██╗██╔════╝██║
█████╔╝ ███████║█████╗  ██║
██╔═██╗ ██╔══██║██╔══╝  ██║
██║  ██╗██║  ██║███████╗███████╗
╚═╝  ╚═╝╚═╝  ╚═╝╚══════╝╚══════╝
```

# KaelAi — Wallet Trust Infrastructure for the Agent Economy

KaelAi provides behavioral wallet scoring across 10 chains for two distinct use cases:

| Product | Use Case | Endpoint |
|---|---|---|
| **KaelAi Agent** | Trust scoring for autonomous AI agent wallets — x402, Bittensor, agentic commerce | [kaelai.io/agent](https://kaelai.io/agent) |
| **KaelAi Shield** | Threat detection for DeFi protocols — exploit detection, TC funded wallets, registry checks | [kaelai.io/shield](https://kaelai.io/shield) |

![API Status](https://img.shields.io/badge/API-Live-00c896?style=flat-square)
![Version](https://img.shields.io/badge/version-v0.3.1-6b7f8a?style=flat-square)
![License](https://img.shields.io/badge/license-MIT-6b7f8a?style=flat-square)

---

## KaelAi Agent

Score autonomous AI agent wallets on-chain and return a Known Agent Trust (KAT) Score from 0–100. Built for Bittensor subnet operators, x402 protocol developers, and agent infrastructure builders.

### Quick Start

```bash
curl -X POST https://kaelai.io/api/v1/score \
  -H "Authorization: Bearer kael_live_YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"wallet_address": "0xYourAgentWalletAddress", "chain": "eth"}'
```

### Example Agent Response

```json
{
  "wallet_address": "0xd8da6bf26964af9d7eed9e03e53415d37aa96045",
  "overall_score": 78,
  "grade": "A",
  "confidence": 0.91,
  "chain": "eth",
  "wallet_type": "autonomous_agent",
  "scope_warning": null,
  "dimension_scores": {
    "behavioral_consistency": 17,
    "transaction_legitimacy": 15,
    "wallet_age_and_history": 18,
    "counterparty_quality": 13,
    "volume_stability": 15
  },
  "reasoning": "Established wallet with 2+ years of consistent on-chain activity..."
}
```

### Grade Scale

| Grade | Score Range | Trust Level |
|---|---|---|
| AAA | 95–100 | Highest trust — long-established, consistent agent |
| AA | 88–94 | Very high trust |
| A | 80–87 | High trust — solid on-chain history |
| BBB | 70–79 | Good trust — minor flags |
| BB | 60–69 | Moderate trust — limited history or minor anomalies |
| B | 50–59 | Below average — proceed with caution |
| CCC | 0–49 | Low trust — significant risk signals |

---

## KaelAi Shield

DeFi protocol security layer. Shield adds threat-optimized scoring weights, behavioral risk flags, exploit registry lookups, and a five-tier recommended action system to the same behavioral engine.

Pass `"mode": "shield"` on any existing API call — no new endpoint, no migration.

```bash
curl -X POST https://kaelai.io/api/v1/score \
  -H "Authorization: Bearer kael_live_YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"wallet_address": "0xSuspiciousWallet", "chain": "eth", "mode": "shield"}'
```

### Example Shield Response

```json
{
  "wallet_address": "0x...",
  "scoring_mode": "shield",
  "overall_score": 19,
  "grade": "CCC",
  "threat_classification": "confirmed_exploit_wallet",
  "threat_confidence": 0.99,
  "shield_flags": ["counterparty_concentration"],
  "recommended_action": "block",
  "recommended_action_label": "Block — confirmed threat registry match",
  "alert_severity": "critical",
  "registry_checked": true,
  "registry_match": {
    "matched": true,
    "registry_type": "threat",
    "incident_name": "Verus-Ethereum Bridge Exploit",
    "incident_date": "2026-05-18",
    "amount_usd": 11400000
  }
}
```

### Five-Tier Recommended Action System

| Tier | Score Range | Trigger | Alert Severity |
|---|---|---|---|
| **ALLOW** | 70–100 | No critical flags, no registry match | `none` |
| **MONITOR** | 50–69 | Minor flags only | `low` |
| **REVIEW** | 25–49 | Multiple behavioral flags | `medium` |
| **FLAG** | 15–24 | Strong threat signals | `high` |
| **BLOCK** | 0–14, or any registry match | Critical flags or confirmed attacker | `critical` |

Registry overrides are always applied first:
- **Threat registry match** → BLOCK at confidence 0.99, regardless of score
- **Trusted registry match** → ALLOW at confidence 0.99, all behavioral flags suppressed

### Shield Behavioral Flags

| Flag | Trigger Condition |
|---|---|
| `tornado_cash_funded` | Counterparty address matches known TC contract (direct, internal tx, or ERC-20 transfer source) |
| `receive_only_pattern` | No outbound transactions, or inbound/outbound ratio > 20 |
| `value_spike_anomaly` | Max-to-mean transaction value ratio ≥ 50× |
| `counterparty_concentration` | Single counterparty accounts for ≥ 70% of volume |
| `zero_known_protocol_ratio` | 0% verified DeFi protocol interactions with ≥ 3 contract calls |
| `failed_tx_anomaly` | ≥ 15% of transactions failed |

### Shield Weight Rebalancing vs Agent

| Dimension | Agent Weight | Shield Weight | Rationale |
|---|---|---|---|
| Transaction Legitimacy | 20% | **30%** | Primary threat signal |
| Behavioral Consistency | 20% | **25%** | Automation patterns distinguish agents from attackers |
| Counterparty Quality | 20% | **20%** | Unchanged |
| Wallet Age & History | 20% | **15%** | Age less relevant for fast-moving exploits |
| Volume Stability | 20% | **10%** | Least predictive in DeFi attack patterns |

---

## Supported Chains — Both Products

| Chain | `chain` value | Notes |
|---|---|---|
| Ethereum | `eth` | Full EVM support incl. internal tx + ERC-20 analysis |
| Bitcoin | `btc` | UTXO behavioral analysis |
| Solana | `sol` | SPL token + program interaction scoring |
| Base | `base` | Full EVM support |
| BNB Smart Chain | `bsc` | Full EVM support |
| Polygon | `polygon` | Full EVM support |
| Arbitrum | `arbitrum` | Full EVM support |
| Avalanche | `avax` | Full EVM support |
| Hyperliquid | `hype` | Perp + vault behavioral analysis |
| Bittensor | `tao` | Coldkey/hotkey, subnet staking, delegation scoring |

**Bittensor note:** For counterparty trust assessment, scoring the **hotkey** address returns richer behavioral data and higher confidence scores. Hotkeys contain all operational activity — weight setting, miner scoring, validation extrinsics. Both coldkey and hotkey are supported.

---

## API Endpoints

| Method | Path | Description |
|---|---|---|
| `POST` | `/api/v1/score` | Score a wallet — Agent or Shield mode |
| `GET` | `/api/v1/score/{address}/history` | Full score history for a wallet |
| `GET` | `/api/v1/score/{address}/trend` | Trend direction over last N scores |
| `POST` | `/api/v1/keys/register` | Self-serve API key registration |
| `GET` | `/api/v1/stats` | Public usage statistics |
| `GET` | `/api/v1/chain-stats` | Per-chain scoring statistics |
| `GET` | `/api/v1/billing/agent-slots` | Agent founding member slot availability |
| `GET` | `/api/v1/billing/shield-slots` | Shield founding member slot availability |

---

## Authentication

```
Authorization: Bearer kael_live_YOUR_KEY
```

Get your key instantly — free tier, no credit card: [kaelai.io](https://kaelai.io)

---

## Pricing

### KaelAi Agent

| Tier | Price | Queries |
|---|---|---|
| Free | $0 | 100 queries, never expires |
| Pay As You Go | $0.01 / query | Top up, no subscription |
| Builder (Founding) | $24.99 / mo | 3,000 queries / mo · rate locked for life |
| Builder (Standard) | $29.99 / mo | 3,000 queries / mo |
| Scale (Founding) | $129.99 / mo | 15,000 queries / mo · rate locked for life |
| Scale (Standard) | $149 / mo | 15,000 queries / mo |

Annual plans available at up to 20% discount. → [kaelai.io/agent](https://kaelai.io/agent)

### KaelAi Shield

| Tier | Price | Queries | Notes |
|---|---|---|---|
| PAYG | $0.05 / query | Pay per query | No subscription required |
| Starter (Founding) | $49.99 / mo | 2,000 queries / mo | Rate locked for life |
| Pro (Founding) | $299 / mo | 10,000 queries / mo | Rate locked for life |
| Enterprise | $999 / mo | Unlimited queries / mo | Custom SLA |

Annual plans available at up to 20% discount. → [kaelai.io/shield](https://kaelai.io/shield)

---

## Documentation

Full API reference: **[kaelai.io/docs](https://kaelai.io/docs)**

Shield product spec: **[docs/shield-spec.md](docs/shield-spec.md)**

Changelog: **[CHANGELOG.md](CHANGELOG.md)**

---

## Links

- 🤖 **KaelAi Agent:** [kaelai.io/agent](https://kaelai.io/agent)
- 🛡️ **KaelAi Shield:** [kaelai.io/shield](https://kaelai.io/shield)
- 📖 **Docs:** [kaelai.io/docs](https://kaelai.io/docs)
- 📝 **Blog:** [kaelai.io/blog](https://kaelai.io/blog)
- 📬 **Contact:** [hello@kaelai.io](mailto:hello@kaelai.io)
- 🐦 **X / Twitter:** [@Kaelai_](https://x.com/Kaelai_)

---

*Built for the agent economy — [kaelai.io](https://kaelai.io)*
