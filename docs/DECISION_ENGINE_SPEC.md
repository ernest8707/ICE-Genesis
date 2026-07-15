# ICE Genesis Decision Engine 2.0 Specification

## Purpose

The Decision Engine is the single source of truth for every user-facing trade decision in ICE Genesis. It consumes facts published by the core engines and produces one synchronized decision used by the dashboard, trade box, alerts, and future statistics.

The Decision Engine does not detect market structure, liquidity, order blocks, fair value gaps, OTE, sessions, or higher-timeframe bias. It only interprets the state those engines publish.

---

## Inputs

### Market Structure
- Direction: `bullish`, `bearish`, or `neutral`
- Event: `BOS`, `CHoCH`, or `none`
- Scope: `internal` or `external`
- Confirmed: `true` or `false`

### Liquidity
- Bullish prerequisite: sell-side liquidity swept
- Bearish prerequisite: buy-side liquidity swept
- Sweep source: `internal`, `external`, `PDH`, `PDL`, `PWH`, `PWL`, `EQH`, or `EQL`
- Confirmed: `true` or `false`

### Displacement
- Direction: `bullish`, `bearish`, or `none`
- Strength: `weak`, `medium`, or `strong`
- Confirmed: `true` or `false`

### Order Block
- Direction: `bullish`, `bearish`, or `none`
- State: `fresh`, `mitigated`, `invalidated`, or `none`
- Upper price
- Lower price
- FVG overlap: `true` or `false`

### Fair Value Gap
- Direction: `bullish`, `bearish`, or `none`
- State: `active`, `partially filled`, `filled`, or `none`
- Upper price
- Lower price

### OTE
- Directional alignment: `bullish`, `bearish`, or `none`
- Price location: `inside`, `above`, `below`, or `unavailable`
- 61.8% level
- 79% level

### Higher-Timeframe Bias
- Bias: `bullish`, `bearish`, or `mixed`
- Aligned with setup: `true` or `false`

### Session
- Session: `Asia`, `London`, `New York`, or `inactive`
- Session filter passed: `true` or `false`

---

## Bullish Evidence Rules

Bullish evidence is true when:

- Liquidity: sell-side liquidity was swept.
- Displacement: bullish displacement is confirmed.
- Structure: bullish BOS or bullish CHoCH is confirmed.
- Order block: a fresh bullish order block exists.
- FVG: an active bullish FVG exists.
- OTE: price is inside bullish OTE.
- HTF: higher-timeframe bias is bullish.
- Session: an allowed session is active.

## Bearish Evidence Rules

Bearish evidence is true when:

- Liquidity: buy-side liquidity was swept.
- Displacement: bearish displacement is confirmed.
- Structure: bearish BOS or bearish CHoCH is confirmed.
- Order block: a fresh bearish order block exists.
- FVG: an active bearish FVG exists.
- OTE: price is inside bearish OTE.
- HTF: higher-timeframe bias is bearish.
- Session: an allowed session is active.

---

## Default Scoring Weights

| Condition | Weight |
|---|---:|
| Liquidity sweep | 20 |
| Displacement | 20 |
| Structure confirmation | 20 |
| Fresh order block | 15 |
| Active FVG | 10 |
| OTE alignment | 10 |
| HTF alignment | 5 |
| Session | 0 initially |

Total: 100 points.

Session remains informational in the first implementation so it cannot distort the base score. It may become configurable later.

Bullish and bearish scores must be calculated independently.

---

## Mandatory Conditions

A high score alone must never create a trade signal.

### Mandatory for BUY
- Bullish structure confirmed
- Bullish displacement confirmed
- Fresh bullish order block exists
- Sell-side liquidity sweep confirmed when liquidity gating is enabled

### Mandatory for SELL
- Bearish structure confirmed
- Bearish displacement confirmed
- Fresh bearish order block exists
- Buy-side liquidity sweep confirmed when liquidity gating is enabled

FVG, OTE, HTF alignment, and session improve quality but are optional unless the user enables strict mode.

---

## Recommendation States

### `BUY`
All mandatory bullish conditions pass, bullish score meets the minimum threshold, bullish score exceeds bearish score by the minimum separation, and no bearish conflict invalidates the setup.

### `SELL`
All mandatory bearish conditions pass, bearish score meets the minimum threshold, bearish score exceeds bullish score by the minimum separation, and no bullish conflict invalidates the setup.

### `WAIT FOR LIQUIDITY`
Directional structure exists, but the required liquidity sweep has not been confirmed.

### `WAIT FOR DISPLACEMENT`
Liquidity is confirmed, but directional displacement is missing.

### `WAIT FOR STRUCTURE`
Liquidity and displacement exist, but BOS or CHoCH has not confirmed the direction.

### `WAIT FOR ORDER BLOCK`
Structure is confirmed, but no fresh directional order block exists.

### `WAIT FOR FVG`
Mandatory conditions pass, but strict FVG mode is enabled and no active directional FVG exists.

### `WAIT FOR OTE`
The setup is directionally valid, but price has not entered the OTE zone.

### `WAIT FOR HTF`
The setup is otherwise valid, but strict HTF alignment is enabled and higher-timeframe bias is not aligned.

### `PREPARING`
Several conditions are aligned, but the setup has not reached entry readiness.

### `NO TRADE — CONFLICT`
Bullish and bearish evidence are both strong, or structure and higher-timeframe direction materially conflict.

### `WAIT`
No valid directional sequence currently exists.

---

## Conflict Rules

The engine must return `NO TRADE — CONFLICT` when any enabled rule is true:

- Bullish and bearish scores are both above the trade threshold.
- Bullish and bearish scores differ by less than the minimum separation.
- Structure is bullish while the active fresh order block is bearish.
- Structure is bearish while the active fresh order block is bullish.
- Strict HTF mode is enabled and HTF bias directly opposes the setup.
- A setup's active order block becomes invalidated.

Default minimum score separation: 15 points.

---

## Grades

| Score | Grade |
|---|---|
| 95–100 | A+ |
| 90–94 | A |
| 85–89 | B+ |
| 80–84 | B |
| 70–79 | C |
| Below 70 | PASS |

The grade is descriptive only. It does not override mandatory conditions.

---

## Setup Lifecycle

The engine publishes one current stage:

1. `LIQUIDITY`
2. `DISPLACEMENT`
3. `STRUCTURE`
4. `ORDER BLOCK`
5. `FVG`
6. `OTE`
7. `ENTRY READY`
8. `INVALIDATED`

The current stage is the first required step that has not yet passed. If all enabled steps pass, the stage is `ENTRY READY`.

---

## Missing Conditions

The engine publishes up to three missing conditions in priority order:

1. Liquidity
2. Displacement
3. Structure
4. Order Block
5. FVG
6. OTE
7. HTF Alignment
8. Session

Example:

- `Liquidity Sweep`
- `OTE Retracement`
- `HTF Alignment`

---

## Decision Output Contract

The Decision Engine must publish these values:

- `decisionDirection`: `bullish`, `bearish`, or `neutral`
- `recommendation`: one recommendation state listed above
- `bullishScore`: 0–100
- `bearishScore`: 0–100
- `confidence`: the winning directional score, or the larger score while waiting
- `grade`
- `lifecycleStage`
- `missingCondition1`
- `missingCondition2`
- `missingCondition3`
- `decisionReason`
- `entryPrice`
- `stopPrice`
- `target1Price`
- `target2Price`
- `riskReward`
- `entryReady`: `true` or `false`

The dashboard, trade box, and alerts must read only from this output contract.

---

## Trade Box Rules

The trade box must never independently calculate its own bias or confidence.

- Show a green BUY box only when `recommendation == BUY`.
- Show a red SELL box only when `recommendation == SELL`.
- Hide the trade box for all WAIT, PREPARING, and NO TRADE states.
- Display the same confidence and grade shown in the dashboard.

Entry, stop, and targets are planning references, not financial advice.

---

## Dashboard Rules

The dashboard must show:

### Market Context
- HTF bias
- Market state
- Order flow
- Premium/discount

### Setup Checklist
- Liquidity
- Displacement
- Structure
- Order block
- FVG
- OTE
- HTF alignment
- Session

### ICE Decision
- Recommendation
- Confidence
- Grade
- Lifecycle stage
- Next required condition

The dashboard and trade box must never disagree.

---

## Alert Rules

Alerts must be generated only from Decision Engine outputs.

Initial alerts:

- `ICE BUY READY`
- `ICE SELL READY`
- `ICE SETUP PREPARING`
- `ICE SETUP INVALIDATED`

No module-specific event may directly trigger a final trade-ready alert.

---

## Non-Repainting Requirements

- Use only confirmed pivots for structure inputs.
- Do not revise a published BOS or CHoCH after confirmation.
- Do not use future bars to create current decisions.
- A trade-ready state may disappear only when current price action invalidates or mitigates the setup according to configured rules.

---

## Initial User Inputs

- Minimum score for BUY/SELL: default `80`
- Minimum bullish/bearish separation: default `15`
- Require liquidity: default `true`
- Require FVG: default `false`
- Require OTE: default `false`
- Require HTF alignment: default `false`
- Require active session: default `false`
- Strict conflict protection: default `true`

---

## Acceptance Criteria for Issue #3

- [ ] Bullish and bearish scores are independent.
- [ ] Scores use the documented weights.
- [ ] Mandatory conditions override score.
- [ ] Dashboard and trade box always agree.
- [ ] Trade box is hidden during WAIT and NO TRADE states.
- [ ] Missing conditions display in priority order.
- [ ] Recommendation explains the next required step.
- [ ] Conflicting bullish and bearish setups produce NO TRADE.
- [ ] Grade exactly matches the documented scale.
- [ ] Existing chart objects remain functional.
- [ ] TradingView compiles without errors.
- [ ] Tested on META 1H, MSFT 1H, SPY 15M, and QQQ 15M.

---

## Deferred Work

The following are intentionally outside the first Decision Engine sprint:

- Natural-language ICE Narrator paragraphs
- Backtesting statistics
- Historical setup database
- Machine learning
- Automated order execution
- Multi-symbol scanning
- User-configurable scoring templates

These should be added only after the base decision contract is stable.
