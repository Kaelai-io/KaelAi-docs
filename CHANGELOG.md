# Changelog

All notable changes to the KAT Score API are documented here.

---

## v0.3.2 — Shield Empty Wallet Handling & DeFi Language Cleanup
**Released: 2026-05-23 | Updated: 2026-05-26**

### Overview

Patch release correcting four behavioural issues in Shield mode scoring.
The most impactful fix changes how Shield handles wallets with no transaction
history: previously these returned BLOCK at score 0, which was incorrect —
an unknown wallet is not the same as a dangerous one. All four fixes are live.

---

### What Changed

#### Fix 10 — Trusted registry lookup now runs in Agent mode (not just Shield)

**Applies to:** `api/v1/endpoints/score.py`

**Root cause:** The `lookup_registry()` call in `score.py` was placed inside the
`if mode == "shield":` branch, meaning Agent mode never performed a registry lookup.
Wallets with a `registry_type = trusted` entry (such as `vitalik.eth`) were therefore
scored purely on behavioral data in Agent mode — producing a low CCC score and
`decline` recommendation rather than the expected trusted-registry override.

**Confirmed broken behaviour:** `0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045`
(vitalik.eth, `trusted` registry entry) returned **score 18 / CCC / decline** in Agent
mode despite being in the trusted registry. `registry_checked` was absent from the
Agent mode response entirely.

**Fix:** The `registry_match = await lookup_registry(...)` call was moved **above**
the `if mode == "shield":` block so it executes for every mode. An Agent-mode
override block was added immediately after:

- **Trusted registry match in Agent mode:** Sets `recommended_action: "proceed"`,
  `confidence: 0.99`, `risk_flag: "trusted_registry_match"`, and attaches
  `registry_match` + `registry_checked: True` to the response. Action label names
  the entity and confirms registry override.
- No change to the behavioral score or dimension scores — the registry override
  changes the action and confidence only, not the underlying scoring.

The Shield path is unaffected — it uses the same `registry_match` variable now
resolved before both branches execute (no duplicate DB query).

**Confirmed fix:** `0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045` rescored in Agent
mode post-deploy:

| Field | Before | After |
|---|---|---|
| `recommended_action` | `decline` | `proceed` |
| `confidence` | 0.75 | 0.99 |
| `risk_flag` | `suspicious` | `trusted_registry_match` |
| `registry_checked` | absent | `true` |
| `registry_match.registry_type` | absent | `trusted` |
| `recommended_action_label` | *behavioral decline reason* | `TRUSTED REGISTRY MATCH — vitalik.eth. Verified good actor. Ethereum co-founder. Publicly disclosed. ENS: vitalik.eth. Safe to proceed.` |

**Files changed:** `api/v1/endpoints/score.py`

---

#### Fix 11 — Threat registry override (BLOCK) now applies in Agent mode

**Applies to:** `api/v1/endpoints/score.py`

**Root cause:** Same as Fix 10 — `lookup_registry()` was Shield-only. Wallets with a
`registry_type = threat` entry were scored behaviorally in Agent mode, returning
`low_agent_fit / review` rather than a BLOCK override. A confirmed exploit wallet was
treated as merely a poor agent-commerce fit rather than a blocked threat.

**Confirmed broken behaviour:** `0xD3FEEd5DA83D8e8c449d6CB96ff1eb06ED1cF6C7`
(Drift Protocol Exploit attacker, `threat` registry entry, $285M) returned
**score 22 / CCC / review / low_agent_fit** in Agent mode. No registry data in response.

**Fix:** The Agent-mode override block introduced in Fix 10 also handles threat entries:

- **Threat registry match in Agent mode:** Sets `recommended_action: "block"`,
  `confidence: 0.99`, `risk_flag: "threat_registry_match"`, and attaches
  `registry_match` + `registry_checked: True` to the response. Action label
  names the incident, amount (using the same sub-million formatter from Fix 9),
  and instructs immediate block.
- Behavioral score and dimensions are preserved in the response for audit purposes —
  only the action, confidence, and risk flag are overridden.

**Design intent:** A known exploit wallet should return BLOCK in both Agent and Shield
mode. The registry is the highest-confidence signal available and must override
behavioral scoring regardless of mode. An exploit wallet that scores behaviorally
ambiguous should still be blocked — the registry exists precisely for this case.

**Confirmed fix:** `0xD3FEEd5DA83D8e8c449d6CB96ff1eb06ED1cF6C7` rescored in Agent
mode post-deploy:

| Field | Before | After |
|---|---|---|
| `recommended_action` | `review` | `block` |
| `confidence` | 0.65 | 0.99 |
| `risk_flag` | `low_agent_fit` | `threat_registry_match` |
| `registry_checked` | absent | `true` |
| `registry_match.registry_type` | absent | `threat` |
| `registry_match.incident_name` | absent | `Drift Protocol Exploit` |
| `recommended_action_label` | *low agent fit review reason* | `REGISTRY MATCH — confirmed attacker wallet from Drift Protocol Exploit ($285.0M). Block immediately.` |

**Files changed:** `api/v1/endpoints/score.py`

---

#### Fix 7 — Shield mode now uses Wallet Trust Score branding throughout

**Applies to:** `services/shield_scorer.py`, `shield/index.html`, `docs/index.html`, `docs/shield-spec.md`

Shield mode API responses and all public Shield documentation now use **Wallet Trust Score** terminology consistently. KAT Score (Known Agent Trust Score) is the Agent mode product identity and is not appropriate in Shield mode, which serves DeFi protocol security teams with no interest in agent commerce.

**Changes made:**

1. **`services/shield_scorer.py`** — Module docstring updated. All `SHIELD_SCOPE_WARNINGS` entries that referenced "KAT Shield Score" or "KAT Shield" updated to "Wallet Trust Score". The `agent_score` field removed from both code paths in `build_shield_result()` — Shield API responses no longer include the equal-weight agent reference score.

2. **`shield/index.html`** — `agent_score` removed from the JSON response example. Page now shows a clean Shield response with no agent-mode fields visible to DeFi protocol customers.

3. **`docs/index.html`** — Mode comparison table: "KAT Score 0-100" updated to "Wallet Trust Score 0-100". Shield response schema table: `agent_score` row removed. JSON response example: `agent_score` field removed.

4. **`docs/shield-spec.md`** — `agent_score` field removed from response fields JSON block and accompanying description paragraph.

**Live confirmation:** Wallet `0x47ac0Fb4F2D84898e4D9E7b4DaB3C24507a6D503` rescored post-deploy — response contains no `agent_score` field and no KAT or agent references in any response field value. Wallet Trust Score 76/100, ALLOW.

---

#### Fix 1 — Empty wallet handling: BLOCK → REVIEW with `insufficient_data` classification

**Root cause:** Wallets returning zero transactions were assigned dimension scores
of 0 across all five dimensions by the base scorer. After Shield reweighting, the
resulting shield score of 0 fell below the BLOCK threshold (< 15), triggering an
automatic BLOCK regardless of whether any threat signals were present. Additionally,
the `receive_only_pattern` flag was firing for zero-transaction wallets on the
logic that `outbound_count == 0` — technically true but meaningless with no data.

**Fix:** Two-part change in `shield_scorer.py`:

1. `compute_shield_flags()` — added zero-transaction guard. When `transaction_count`
   is 0, the engine skips all pattern-based flags (there are no patterns to detect).
   Only registry-based signals (`tornado_cash_funded` via `mixer_contact_detected`)
   can fire for wallets with zero transactions.

2. `build_shield_result()` — added an early insufficient-data intercept. When both
   `transactions_analysed == 0` and `behavioral_features.transaction_count == 0`,
   and no threat registry match exists, the normal scoring pipeline is bypassed.
   The wallet receives:
   - `threat_classification: insufficient_data`
   - `recommended_action: review`
   - Plain-English reasoning: *"No transaction history found. Cannot assess behavioral
     patterns. Recommend manual review before interaction. This wallet shows no threat
     signals — it is unknown, not dangerous."*

**Confirmed:** TAO wallet `5Dw8w2ec6b4CDG5cXtUPk1qRqjHpAqQb8B1eS1Mu3vUKyfWB` and
BTC wallet `3NxkswbBJo163PycajjoECWiLSTHSHaHvi` — both previously BLOCK/0,
now correctly return REVIEW/30 with `insufficient_data`.

**Files changed:** `services/shield_scorer.py`

---

#### Fix 2 — Score floor for zero-transaction wallets: 0 → 30

**Root cause:** All-zero dimension scores produced a shield score of 0, which
sat in the BLOCK band (0–14) and communicated to consumers that the wallet was
extremely high risk — semantically incorrect for a wallet that simply has no history.

**Fix:** In the insufficient-data intercept added by Fix 1, `shield_score` is
hard-set to **30** (lower-middle of the REVIEW band, 25–49). This accurately
represents uncertainty without implying danger. `confidence` is set to 0.20 and
`threat_confidence` to 0.25 — both clearly signalling low assessment confidence.

The score floor applies only when the insufficient-data condition is met (zero
transactions, no threat signals). It does not affect wallets with actual data
that score legitimately low.

**Files changed:** `services/shield_scorer.py`

---

#### Fix 3 — `block_reason` field added to Shield API response

A new `block_reason` field is now included in Shield responses when
`recommended_action` is `block`. It does not appear for any other action tier.

**Three possible values:**

| Value | Trigger |
|---|---|
| `threat_detected` | Threat registry match, or `tornado_cash_funded` critical flag |
| `critical_flags` | Multiple high-confidence threat flags firing simultaneously |
| `insufficient_data` | Zero transaction history reached BLOCK (defensive; should not occur after Fix 1) |

`block_reason` is present in both the top-level response and in the `shield_data`
JSONB block. This gives downstream protocol security teams the context to
distinguish between a genuinely dangerous wallet and an edge case.

**Files changed:** `services/shield_scorer.py`

---

#### Fix 4 — Agent language removed from Shield mode reasoning and response labels

**Root cause:** Shield mode is used by DeFi protocols screening real user wallets,
not agent commerce platforms. However, the base scoring system prompt, risk flag
labels, scope warnings, and reasoning prefix rules all used agent-specific language
("autonomous agent", "agent commerce fit", "agent-to-agent commerce").

**Fix:** Four changes:

1. **`SHIELD_SCORING_SYSTEM_PROMPT`** added to `scorer.py` — separate from the
   existing `SCORING_SYSTEM_PROMPT` (agent mode). Shield prompt instructs the LLM
   to score for DeFi legitimacy and risk profile, not agent commerce fit. Explicitly
   prohibits the terms "agent", "autonomous agent", "agent commerce", "agent wallet"
   in the generated reasoning.

2. **Reasoning prefix rules** in `scorer.py` — when `mode == "shield"` and
   `overall_score < 40`, the required prefix phrases use DeFi language:
   - `"This wallet exhibits red flags consistent with malicious or high-risk DeFi activity"`
   - `"This wallet shows a low DeFi activity profile but no suspicious signals"`

3. **`SHIELD_RISK_FLAG_LABELS`** added to `shield_scorer.py` — replaces
   `_RISK_FLAG_LABELS` in Shield responses. `low_agent_fit` is remapped to
   `low_activity_profile` with DeFi-appropriate wording throughout.

4. **`SHIELD_SCOPE_WARNINGS`** added to `shield_scorer.py` — replaces
   `_SCOPE_WARNINGS` in Shield responses. All references to "agent wallets" and
   "agent commerce" replaced with DeFi-appropriate equivalents.

`build_shield_result()` applies all Shield-specific label overrides on every
response, ensuring no agent language escapes into Shield mode output regardless
of what the base scorer returns.

**Files changed:** `services/shield_scorer.py`, `services/scorer.py`

---

#### Fix 5 — Trailing agent language in type_note removed from Shield mode reasoning

**Root cause:** `_classify_wallet_type()` in `scorer.py` returns a `(wallet_type, type_note)`
tuple. For `likely_exchange` wallets (and others), `type_note` contained agent-specific
language: *"KAT Score is optimised for autonomous agent wallets — exchange wallets are
expected to score differently."* This string was appended to the LLM-generated reasoning
before Shield post-processing ran, meaning it appeared verbatim in Shield mode responses
despite Fix 4 cleaning up all other agent language.

**Fix:** Added `_SHIELD_TYPE_NOTES` dict to `scorer.py` mapping each `wallet_type` to a
DeFi-appropriate replacement string. In `compute_kat_score()`, when `mode == "shield"`,
`type_note` is overridden with the Shield version before it is appended to reasoning:

```python
if mode == "shield" and type_note is not None:
    type_note = _SHIELD_TYPE_NOTES.get(wallet_type, type_note)
```

Replacement string for `likely_exchange` (and `retail_wallet`):
> *"KaelAi Shield is optimised for DeFi protocol security screening — results reflect
> on-chain behavioral patterns relevant to DeFi risk assessment."*

**Confirmed:** SOL wallet `5bi727G5HTtSWGzE5z4LCzWRHXvtbf1BdmdoecUfT6ww` rescored —
agent language strings (`autonomous agent`, `agent commerce`, `agent wallet`,
`KAT Score is optimised for autonomous agent wallets`) confirmed absent from all
Shield mode response fields.

**Files changed:** `services/scorer.py`

---

#### Fix 6 — Meta-commentary and developer notes prohibited from Shield mode reasoning

**Root cause:** The LLM generating Shield mode reasoning occasionally produced sentences
that read as internal commentary about the scoring system rather than behavioral
observations about the wallet — e.g. noting that patterns are "characteristic of a test
wallet" (inferring real-world identity) or editorializing about scoring methodology or
tool limitations. These are not on-chain behavioral observations and do not belong in
a production API response consumed by protocol security teams.

**Fix:** Added an explicit prohibition block to `SHIELD_SCORING_SYSTEM_PROMPT` in
`scorer.py`, directly before the JSON return instruction:

> *"The reasoning field must contain ONLY on-chain behavioral observations about this
> specific wallet. Do not include: meta-commentary about the scoring system, methodology,
> or tool capabilities; statements about limitations of the scoring approach; references
> to what the wallet 'is' in the real world (e.g. 'this is a test wallet'); internal
> notes about data quality or sample size constraints. Describe what was observed
> on-chain. Do not editorialize about the tool or the assessment process."*

**Files changed:** `services/scorer.py`

---

#### Fix 8 — Shield mode review bot notifications use Wallet Trust Score branding

**Applies to:** `services/telegram.py`

The Telegram review bot notification formatter (`_format_score_alert`) used a single
template for both Agent and Shield mode responses. In Shield mode this produced two
branding errors:

1. **`KAT Score:` label** — Shield mode notifications displayed `KAT Score: 8/100`
   instead of `Wallet Trust Score: 8/100`. KAT Score is Agent mode product identity
   and must not appear in Shield mode output.

2. **`Type: autonomous_agent` line** — the `wallet_type` field from the base scorer
   was displayed verbatim. In Shield mode `autonomous_agent` is not a meaningful
   or appropriate label — wallet type classification is an Agent mode concept.

**Fix:** `_format_score_alert()` now reads `scoring_mode` from the score dict and
branches on `is_shield = scoring_mode == "shield"`:

- **Score label:** `"Wallet Trust Score"` when Shield, `"KAT Score"` when Agent.
- **Type line:** Omitted entirely in Shield mode. Agent mode retains it as
  `Wallet Type:` (previously `Type:` — corrected label also applied).
- **Action banners:** Shield mode gets five-tier banners (BLOCK / FLAG / REVIEW /
  MONITOR / ALLOW) matching the Shield recommended action vocabulary. Agent mode
  retains the existing three-tier banners (DECLINE / REVIEW / PROCEED).
- **Risk lines:** Shield mode uses DeFi-appropriate risk labels
  (`low_activity_profile` instead of `low_agent_fit`).
- **Shield flags line:** Added to Shield mode notifications when flags are present
  (e.g. `Flags: counterparty_concentration`). Not shown in Agent mode.

**Confirmed:** Rescore of WUSD/GLOVE exploit wallet
`0x88329a09428778f62bc0c8baac0997864e5a57f8` — Shield mode Telegram notification
shows `Wallet Trust Score: 10/100`, no `Type:` line, `Flags: counterparty_concentration`.

**Files changed:** `services/telegram.py`

---

#### Fix 9 — Registry match amount formatter handles sub-million amounts correctly

**Applies to:** `services/shield_scorer.py`

The `recommended_action_label` for registry-matched wallets formatted the exploit
amount using `${amount/1e6:.0f}M` regardless of the actual amount value. For
sub-million amounts (e.g. `$207,000`) this produced `$0M` — incorrect and
misleading in the API response and all downstream Telegram notifications.

**Fix:** The amount formatter now applies a threshold:

```python
if amount >= 1_000_000:
    amount_s = f"${amount/1e6:.1f}M"   # e.g. $285.0M, $3.1M
else:
    amount_s = f"${amount/1000:.0f}K"   # e.g. $207K, $850K
```

**Before:** `REGISTRY MATCH — confirmed attacker wallet from WUSD GLOVE Exploit ($0M). Block immediately.`
**After:** `REGISTRY MATCH — confirmed attacker wallet from WUSD GLOVE Exploit ($207K). Block immediately.`

Applied to both the top-level `recommended_action_label` and the `shield_data`
block label, which share the same formatter function.

**Confirmed:** WUSD/GLOVE exploit wallet `0x88329a09428778f62bc0c8baac0997864e5a57f8`
now returns `$207K` in `recommended_action_label` and `shield_recommended_action_label`.

**Files changed:** `services/shield_scorer.py`

### Organic ALLOW Validation — Binance Hot Wallet

**Date confirmed: 2026-05-23**

The ALLOW tier (score ≥ 70, no critical flags) was confirmed reachable organically —
without a trusted registry entry — against a high-volume production wallet.

**Wallet:** Binance Hot Wallet `0x47ac0Fb4F2D84898e4D9E7b4DaB3C24507a6D503`

| Field | Value |
|---|---|
| Shield Score | **76 / 100** |
| Grade | **A** |
| Tier | **🟢 ALLOW** |
| Threat Classification | `clean` |
| Threat Confidence | 0.76 |
| Registry Match | **None** — organic result only |
| Shield Flags | `counterparty_concentration` (minor; does not downgrade ALLOW tier) |
| Wallet Age | 327 days |
| Confidence | 0.78 |

**Behavioral signals that drove ALLOW:**
- 327-day transaction history, 41 unique counterparties
- Verified protocol interactions: DAI, MakerDAO, USDC, USDT, Aave, Balancer, 1inch
- Zero failed transactions
- High counterparty diversity consistent with legitimate high-volume DeFi activity

**Significance:** This is the first confirmation that the ALLOW tier is reachable through
behavioral quality alone, with no registry assistance. The `counterparty_concentration`
flag fired (single counterparty ≥ 70% of volume — expected for exchange hot wallets) but
at score 76 this sits within the single-flag tolerance at ALLOW tier and does not trigger
a downgrade. The behavioral floor for organic ALLOW is: long wallet history (6+ months),
verified protocol interactions, zero failed transactions, and diverse counterparties.

**Comparison — wallets tested in the same session:**

| Wallet | Score | Tier | Registry |
|---|---|---|---|
| vitalik.eth `0xd8dA…` | 16 | 🟢 ALLOW | Trusted override |
| Research wallet `0x7E5F…` | 6 | 🔴 BLOCK | None |
| Buterin 2018 `0xAb58…` | 15 | 🟢 ALLOW | Trusted override |
| **Binance hot `0x47ac…`** | **76** | **🟢 ALLOW** | **None — organic** |

The two Buterin addresses confirm the trusted registry override path is working at 0.99
confidence. The research wallet (receive-only, zero known protocols, 3 flags) confirms
BLOCK fires correctly for wallets with strongly anomalous behavioral patterns even in the
absence of a registry match.

---

### Live Tier Confirmation (v0.3.2)

Rescored against the same three personal investor wallets used in pre-fix testing:

| Wallet | Chain | Pre-fix | Post-fix | Tier |
|---|---|---|---|---|
| `5Dw8w2ec6b4CDG5cXtUPk1qRqjHpAqQb8B1eS1Mu3vUKyfWB` | TAO | 0 / BLOCK / `suspicious_pattern` | **30 / REVIEW / `insufficient_data`** | 🟠 REVIEW |
| `3NxkswbBJo163PycajjoECWiLSTHSHaHvi` | BTC | 0 / BLOCK / `suspicious_pattern` | **30 / REVIEW / `insufficient_data`** | 🟠 REVIEW |
| `0xafab46e2ea59f9a55317ad2244c89001a3654200` | ETH | 54 / MONITOR / `clean` | **56 / MONITOR / `clean`** | 🟡 MONITOR |

ETH wallet confirmed stable at MONITOR. No flags fired on any wallet.
No agent language present in any response field.

---

## v0.3.1 — Shield TC Detection Hardening
**Released: 2026-05-22**

### Overview

Patch release improving Tornado Cash detection coverage in Shield mode.
Two blind spots in the counterparty address pipeline are closed (internal txs,
ERC-20 transfer senders), three missing TC contracts are added to the ETH registry,
and a confirmed single-hop limitation from the Verus Bridge case study is documented
as a Phase 2 design item.

---

### What Changed

#### Fix 1 — Internal transaction sources now included in TC scan (Shield mode, EVM only)

**Root cause:** The scoring engine fetched only normal transactions via Moralis
`/{address}`. Internal transactions (sub-call traces) were never fetched. TC pool
withdrawals routed through intermediate contracts arrive exclusively as internal
calls — entirely invisible to the previous TC scanner.

**Fix:** Added `get_wallet_internal_transactions()` to `services/moralis.py`.
On every fresh Shield query against an EVM chain, internal txs are fetched via
Etherscan v2 `txlistinternal`, all `from_address` values are merged into the
counterparty set, and the augmented set is checked against `tornado_cash_contracts`.

New fields added to `behavioral_features` JSONB for transparency:
- `internal_tx_sources` — list of unique internal tx `from_address` values
- `erc20_transfer_senders` — list of unique ERC-20 transfer `from_address` values

Performance: Both fetches run concurrently via `asyncio.gather`. Graceful
degradation — Shield scoring continues normally if either fetch fails.
Applies only in Shield mode; Agent scoring is unaffected.

**Files changed:** `services/moralis.py`, `api/v1/endpoints/score.py`

#### Fix 2 — ERC-20 transfer senders now included in TC scan (Shield mode, EVM only)

**Root cause:** `get_wallet_erc20_transfers()` was already available in
`services/moralis.py` but its `from_address` values were never merged into
`all_counterparty_addresses` before the TC check. TC USDC/DAI/cDAI pools
deliver funds exclusively via ERC-20 Transfer events (not raw ETH sends) —
these were entirely missed.

**Fix:** Shield post-processing in `api/v1/endpoints/score.py` now fetches
ERC-20 transfers (limit=50) concurrently alongside internal txs, merges all
`from_address` values into the counterparty set, and stores them in
`erc20_transfer_senders` within `behavioral_features`.

**Files changed:** `api/v1/endpoints/score.py`

#### Fix 3 — Three missing TC contracts added to ETH registry

**Audit finding:** The original `tornado_cash_contracts` table was seeded from
the OFAC SDN list (August 8 2022) plus two Etherscan-labeled entries (router,
nova). Three confirmed TC contracts deployed before the OFAC list were missing:

| Address | Contract Name | Type | Source | needs_review |
|---|---|---|---|---|
| `0x94a1b5cdb22c43faab4abeb5c74999895464ddaf` | Mixer | pool | etherscan_source — deployed by known TC proxy | false |
| `0x905b63fff465b9ffbf41dea908ceb12478ec7601` | TornadoProxy | proxy | etherscan_source — pre-OFAC SDN deployment | false |
| `0x5efda50f22d34f262c29268506c5fa42cb56a1ce` | LoopbackProxy | proxy | etherscan_source — TC governance infrastructure | true |

ETH table now contains **26 entries** (23 verified, 3 needs_review).

TC Redis cache invalidated after update. New contracts active immediately on
next Shield query (cache miss or force_refresh).

**Entries with existing `needs_review=true` (pre-existing, not changed):**
- `0xaeb0a1a9c4c9f7f3e8c5f3bdc7a2a5c9a4c4b65a` (1000 USDT pool) — verify on BscScan
- `0x80c7f18fde57c8d0eeafc0e65d7a27e8a00c0e7c` (5000 USDT pool) — verify on BscScan

**Files changed:** `tornado_cash_contracts` table (DB), Redis TC cache invalidated

---

### Verus Bridge Case Study — Detection Limit Documented

**Incident:** Verus-Ethereum Bridge exploit (2026-05-18, $11.4M)
- Wallet 1: `0x65Cb8b128Bf6e690761044CCECA422bb239C25F9`
- Wallet 2: `0x5aBb91B9c01A5Ed3aE762d32B236595B459D5777`

**Finding:** Even with Fixes 1 and 2 applied, `tornado_cash_funded` did not fire
for either wallet. Root cause: the TC funding is **two hops back** from these wallets.

Confirmed funding chain:
```
TC Pool → [unknown upstream actor] → Wallet 2 → Verus Bridge (0x71518580) → Wallet 1
```

Fixes 1 and 2 close the single-hop gap: they would detect TC if `0x71518580`
(the Verus Bridge) or `0x00000011f84b9aa48e5f8aa8b9897600006289be` (Uniswap
V2DutchOrderReactor) were TC contracts. They are not — the TC interaction
happened upstream of Wallet 2, not directly visible from Wallet 1's transaction
history at any depth reachable via Etherscan internal tx scan.

**For this case specifically:** Both wallets are correctly BLOCKED via the exploit
registry (manually tagged with `funding_source = 'Tornado Cash'` from PeckShield
intelligence). The registry path is working as designed and is the correct control
for confirmed attacker wallets.

**Limitation confirmed:** Single-hop TC detection (Fixes 1 + 2) cannot catch
multi-hop laundering chains without recursive chain traversal. Documented below
as a Phase 2 design item.

---

### Phase 2 Addition: Multi-Hop TC Detection

**Priority: Medium | Status: Backlog | Complexity: High**

Detect TC funding that is two or more hops removed from the scored wallet.

**Problem this solves:** Sophisticated attacker wallets route TC withdrawals
through one or more intermediary addresses before reaching the exploit wallet.
Single-hop detection (Fixes 1 + 2) sees only direct TC contract interactions.
A wallet funded TC → intermediary → target will not trigger the flag even with
all inbound addresses checked, unless the intermediary itself is a TC contract.

**Confirmed case:** Verus Bridge exploit — TC funded an upstream actor who then
triggered the bridge exploit, funnelling proceeds to the attacker wallets through
the bridge contract. Single-hop detection cannot reach the TC interaction.

**Proposed approach:**

Option A — Recursive internal tx traversal (1 additional hop):
- For each `internal_tx_source` address captured in Fix 1, check whether *that*
  address has any direct TC interactions in its own transaction history
- Adds 1 Etherscan API call per unique internal tx source (potentially 3–15 calls)
- Captures the Verus Bridge pattern (TC → Wallet 2 → Bridge → Wallet 1)
- Cache intermediate-hop results in Redis (TTL 24h) to avoid redundant lookups

Option B — TC withdrawal address registry:
- Maintain a `tc_withdrawal_addresses` table of known TC withdrawal destination
  addresses (addresses that previously received a confirmed TC withdrawal)
- Populated via off-chain indexing of TC deposit/withdrawal event logs
- Check inbound `from_address` values against this table instead of recursing
- Higher precision but requires ongoing maintenance of withdrawal address set
- Lower latency than live recursive traversal

**False positive risk:** Multi-hop detection increases false positive rate.
An address that received funds from an address that received funds from TC is
not necessarily TC-funded — intermediaries may be legitimate exchange hot wallets,
bridge contracts, or other high-flow addresses. Must apply confidence decay per hop:
- Direct TC contact: confidence 0.99
- 1 hop removed: confidence 0.70 (flag with lower confidence, no auto-BLOCK)
- 2 hops removed: confidence 0.40 (informational only, separate sub-flag)

**Recommendation:** Implement Option A (recursive 1-hop) first with a separate
`tornado_cash_indirect` flag (distinct from `tornado_cash_funded`). Gate
auto-escalation on confidence ≥ 0.70. Phase 2 design review required before
implementation — false positive rate must be validated against known-clean wallets
before production deployment.

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

### 0. Registry Integrity Monitoring — Trusted Registry Behavioral Re-evaluation
**Priority: PRIORITY | Status: Backlog | Complexity: Medium**

#### Problem

The current trusted registry is a static allowlist — once a wallet is added with
`registry_type = trusted`, it receives a permanent ALLOW override at 0.99 confidence
regardless of any future behavioral change. This creates a reputation poisoning attack
vector: a wallet could accumulate legitimate history, gain trusted status, then pivot
to malicious behaviour with no automatic detection. High-value targets (exchange hot
wallets, bridge contracts, protocol treasuries) are particularly exposed.

#### Design: Three-tier trusted registry

Replace the single `registry_type = trusted` value with a three-tier system:

| Tier | Label | Who it covers | Re-evaluation |
|---|---|---|---|
| **Tier 1** | `trusted_permanent` | Immutable public entities: Vitalik, Ethereum Foundation, OFAC-cleared foundations | Never — identity is fixed |
| **Tier 2** | `trusted_monitored` | Potentially compromisable entities: exchange hot wallets, bridge contracts, protocol treasuries, DAO multisigs | Weekly or monthly background rescore |
| **Tier 3** | `trusted_provisional` | Newly added entities (first 30 days) | Daily rescore; behavioral scoring runs alongside registry override during probation |

**Migration:** All existing `trusted` entries default to `Tier 2 (monitored)` on
migration. Vitalik, Ethereum Foundation, and other immutable public-figure entries
are manually promoted to `Tier 1 (permanent)` post-migration.

#### Implementation: Nightly Celery rescore task

A nightly Celery task (`tasks/registry_monitor.py`) rescores all Tier 2 and Tier 3
trusted registry entries in Shield mode internally:

- **Bypasses the public API endpoint entirely** — no user tokens consumed, no billing
  credits deducted. Calls the internal scoring pipeline (`compute_kat_score` +
  `build_shield_result`) directly, the same path used by real-time contract monitoring.
- Only Moralis and Helius data provider API calls are made (standard chain data fetch).
- Results are stored in a new `registry_monitor_log` table (not `kat_scores`) to keep
  internal monitoring separate from user-facing score history.
- Uses the existing Celery infrastructure (`workers/celery_app.py`) shared with the
  monthly billing task.

**Trigger conditions for automatic status change (Tier 2 + Tier 3):**

Any of the following fires an automatic `trusted → flagged` status change and sends a
Telegram alert for manual review before trusted status can be restored:

| Signal | Threshold |
|---|---|
| Behavioral score drop | Shield score falls below 30 on rescore |
| Critical flag fires | `tornado_cash_funded` detected in counterparty set |
| Protocol engagement collapse | `zero_known_protocol_ratio` fires after ≥ 3 prior clean rescores |
| Inbound pattern shift | `receive_only_pattern` fires after ≥ 2 prior clean rescores |
| Value spike anomaly | `value_spike_anomaly` fires after ≥ 3 prior stable rescores |
| Counterparty concentration spike | `counterparty_concentration` fires with confidence > 0.80 after prior clean rescores |

**Alert format (Telegram):**
```
🚨 TRUSTED REGISTRY ALERT
Wallet: 0x47ac0Fb4F2D84898e4D9E7b4DaB3C24507a6D503
Entity: Binance Hot Wallet (Tier 2 — monitored)
Status change: trusted_monitored → flagged
Trigger: tornado_cash_funded (confidence 0.90)
Previous score: 76 | New score: 12
Action required: Manual review before trusted status restored
```

**Important:** While a Tier 2 wallet is in `flagged` status, it no longer receives
the ALLOW registry override. It falls through to normal behavioral scoring — meaning
it will score on its merits and may receive BLOCK, FLAG, or REVIEW depending on the
behavioral data. This is intentional and correct.

#### Tier 3 probation behaviour

During the 30-day probation period, Tier 3 entries receive the registry ALLOW override
as normal (consistent user experience) but the internal daily rescore runs in parallel.
If any trigger condition fires during probation, the entry is immediately demoted to
`flagged` without ever completing the probation period. After 30 clean daily rescores
with no trigger conditions, the entry is automatically promoted to Tier 2.

#### Schema changes required

```sql
-- extend registry_type enum
ALTER TYPE registry_type ADD VALUE 'trusted_permanent';
ALTER TYPE registry_type ADD VALUE 'trusted_monitored';
ALTER TYPE registry_type ADD VALUE 'trusted_provisional';
ALTER TYPE registry_type ADD VALUE 'flagged';

-- new monitoring log table
CREATE TABLE registry_monitor_log (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    registry_id   UUID REFERENCES exploit_registry(id),
    scored_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    shield_score  INTEGER,
    shield_flags  JSONB,
    trigger_fired BOOLEAN DEFAULT FALSE,
    trigger_reason TEXT,
    prior_status  VARCHAR(40),
    new_status    VARCHAR(40)
);
```

#### Files to create / modify

| File | Change |
|---|---|
| `app/tasks/registry_monitor.py` | New — nightly Celery task |
| `app/models/exploit_registry.py` | Add new `registry_type` enum values + `flagged` status |
| `app/services/exploit_registry.py` | Update `lookup_registry()` to handle new tiers; add `flag_trusted_entry()` |
| `app/services/telegram.py` | Add `alert_trusted_registry_flag()` |
| `app/workers/celery_app.py` | Register nightly beat schedule for registry monitor task |
| `alembic/` | Migration for new enum values + `registry_monitor_log` table |

#### Note on billing / token usage

Internal Celery rescoring does **not** consume API tokens or billing credits.
The nightly task calls `compute_kat_score()` and `build_shield_result()` directly —
bypassing `charge_shield_query()` and all API key authentication entirely. Moralis
and Helius API calls are made as normal (these are infrastructure costs, not
per-user billing). The `registry_monitor_log` insert is the only DB write per rescore.

### 1. Automated Threat Intelligence Ingestion Pipeline
**Priority: PRIORITY | Status: Backlog | Complexity: High**

#### Problem

The KaelAi Shield exploit registry is populated manually — confirmed attacker wallets are only added when a team member reads a security alert and inserts the record by hand. This creates a detection gap between when a threat is publicly known and when the registry protects against it.

The live SquidRouter exploit on 2026-05-24 demonstrated this gap directly: both attacker wallets (`0x9bdc5c3b31ba492fd91cf8900c1bfd9f308d4e38` and `0xa447c9bcb0a0b7a2a88bd68f7c2a59bac25d8817`) were confirmed by Blockaid in a public alert, but scored `insufficient_data / REVIEW` on first query until manually added to the registry. Automated ingestion would have added them within minutes of the alert with no manual intervention.

#### Threat Feed Sources

| Source | Type | Method |
|---|---|---|
| Blockaid | Official alerts | Public alert feed or Twitter account monitoring |
| PeckShield Alert | Twitter/X | RSS or public timeline scraping for wallet address patterns |
| ZachXBT | Telegram + Twitter/X | Telegram channel monitoring + RSS or public timeline scraping |
| SlowMist | Security alerts | RSS feed or website scraping |
| CertiK | Incident reports | RSS feed or website scraping |

#### Implementation

A Celery beat task (`tasks/threat_intel_ingest.py`) running every 15 minutes that:

1. Checks each configured threat feed source for new content published since last poll
2. Parses alert content for EVM/BTC/SOL wallet addresses matching known exploit alert patterns (regex + LLM-assisted classification for ambiguous cases)
3. Extracts metadata: incident name, date, role (`attacker` | `consolidation` | `launderer`), amount (USD), source attribution
4. Inserts matched wallets into `exploit_registry` with `confidence = suspected` (not `confirmed`) and `confirmed_by = <source_name>`
5. Fires a Telegram alert to Brett for each new registry addition

**Alert format (Telegram):**
```
THREAT INTEL INGESTION
New wallet added to exploit registry

Wallet: 0x9bdc5c3b31ba492fd91cf8900c1bfd9f308d4e38
Chain: ETH
Incident: SquidRouter Exploit
Date: 2026-05-24
Role: attacker
Amount: ~$3.07M
Source: Blockaid
Confidence: suspected

Review at kaelai.io/admin to promote to confirmed.
```

#### Confidence Levels

All automated ingestion entries are tagged `confidence = suspected`. Manual promotion to `confirmed` is required after review. This ensures near-real-time protection without compromising registry accuracy — a `suspected` entry still triggers BLOCK at Shield tier, but the alert label surfaces the lower confidence to API consumers.

| Level | Source | Action |
|---|---|---|
| `suspected` | Automated ingestion | BLOCK with `suspected_exploit_wallet` classification |
| `confirmed` | Manual review or Blockaid official confirmation | BLOCK with `confirmed_exploit_wallet` classification |

#### Schema Changes Required

```sql
ALTER TABLE exploit_registry ADD COLUMN confidence VARCHAR(20) DEFAULT 'confirmed';
ALTER TABLE exploit_registry ADD COLUMN auto_ingested BOOLEAN DEFAULT FALSE;
ALTER TABLE exploit_registry ADD COLUMN source_url TEXT;
CREATE INDEX idx_exploit_registry_confidence ON exploit_registry(confidence);
```

#### Files to Create / Modify

| File | Change |
|---|---|
| `app/tasks/threat_intel_ingest.py` | New — 15-minute Celery beat task |
| `app/services/threat_feeds.py` | New — per-source feed polling and wallet extraction |
| `app/services/exploit_registry.py` | Add `ingest_suspected_wallet()` helper |
| `app/services/shield_scorer.py` | Surface `confidence` level in `registry_match` response block |
| `app/services/telegram.py` | Add `alert_threat_intel_ingestion()` |
| `app/workers/celery_app.py` | Register 15-minute beat schedule for ingest task |
| `alembic/` | Migration for `confidence`, `auto_ingested`, `source_url` columns |

#### Twitter/X Monitoring Approach

For PeckShield, ZachXBT, and Blockaid Twitter sources:
- Monitor public timelines via RSS (Nitter RSS proxy or equivalent) for new posts containing EVM address patterns (`0x[0-9a-fA-F]{40}`)
- Filter posts matching known exploit alert keywords: `exploit`, `hack`, `drained`, `attacker`, `stolen`, `bridge attack`, `rug`
- Extract all wallet addresses from matching posts and queue for registry insertion
- Include original tweet URL as `source_url` for audit trail

#### ZachXBT Telegram Monitoring

ZachXBT's Telegram channel is publicly accessible. The existing Telegram bot infrastructure can be extended to monitor it:
- Add the channel to the bot's monitored list
- Parse new messages for EVM/BTC/SOL wallet addresses + exploit keywords
- Same extraction and ingestion pipeline as Twitter sources

---

### 2. Etherscan Entity Label Integration (wallet-level)
**Priority: High**

Automatically recognise named entities without manual registry entries.
On Shield queries, call the Etherscan address tag API for the wallet address
itself. A returned label (e.g. "Binance: Hot Wallet", "Uniswap V3: Router")
sets `threat_classification = verified_entity` and `recommended_action = allow`
with confidence 0.85 (lower than manual registry at 0.99). Cache labels in Redis
for 7 days. Surface as `etherscan_entity_label` in `shield_data`.

Affected: `shield_scorer.py`, `exploit_registry.py`, `score.py`, `score.py` model.

### 3. Tornado Cash Contract Validation — BSC / Polygon / Arbitrum / Optimism
**Priority: High**

The `tornado_cash_contracts` table was seeded with community-sourced addresses
for BSC (5), Polygon (4), Arbitrum (2), and Optimism (1). All carry
`needs_review = TRUE`. Before these chains are relied upon in production:

- Validate each address against current Etherscan / BscScan / PolygonScan labels
- Cross-reference against the Chainalysis and TRM public TC address datasets
- Update `verified_source` and set `needs_review = FALSE` on confirmed entries
- Remove any entries that cannot be verified

Run `invalidate_tc_cache(chain)` after any table updates to bust the 1hr Redis cache.

### 4. Indirect Tornado Cash Funding Trace
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

### 5. Advanced Shield Flags
**Priority: High**

Extend Phase 1 flags with:
- `flash_loan_pattern` — single-block borrow/repay cycle (DeFi exploit signature)
- `bridge_hop_chain` — rapid cross-chain movement consistent with laundering
- `newly_funded_wallet` — wallet funded < 24h before first exploit interaction
- `contract_deployer_anomaly` — deployed contract immediately drained funds
- `dust_attack_sender` — wallet that sends sub-cent amounts to probe addresses
- `governance_manipulation` — flash-loan-funded governance vote pattern

### 6. Shield Pro Tier Launch
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
