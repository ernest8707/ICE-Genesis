# ICE Genesis Blueprint

## 1. Product Vision

ICE Genesis is a multi-timeframe institutional trading assistant for TradingView.

Its purpose is to:

- Determine higher-timeframe directional bias.
- Track lower-timeframe setup formation.
- Validate sniper entries on the 15-minute, 5-minute, and 1-minute charts.
- Block or downgrade setups that conflict with higher-timeframe bias.
- Explain what happened, what is happening, what is missing, and what the trader should wait for.
- Present all information through adaptive desktop, laptop, and mobile dashboard layouts.

## 2. Core Trading Principle

**Higher timeframes determine direction. Lower timeframes determine entry.**

### Bullish sniper sequence

1. Higher-timeframe bias is bullish.
2. Sell-side liquidity is swept.
3. Bullish displacement confirms.
4. Bullish CHoCH or BOS confirms.
5. A fresh bullish order block or bullish FVG exists.
6. Price retraces into discount or OTE.
7. The execution timeframe confirms.
8. Decision Engine approves LONG.

### Bearish sniper sequence

1. Higher-timeframe bias is bearish.
2. Buy-side liquidity is swept.
3. Bearish displacement confirms.
4. Bearish CHoCH or BOS confirms.
5. A fresh bearish order block or bearish FVG exists.
6. Price retraces into premium or OTE.
7. The execution timeframe confirms.
8. Decision Engine approves SHORT.

## 3. Default Timeframe Roles

All roles are user-configurable.

| Role | Default |
|---|---|
| Macro Bias | 1D |
| Primary Bias | 4H |
| Session Bias | 1H |
| Setup Timeframe | 15m |
| Confirmation Timeframe | 5m |
| Precision Entry Timeframe | 1m |

## 4. System Architecture

```text
Price Data
    ↓
Market Structure Engine
    ↓
Liquidity Engine
    ↓
Displacement Engine
    ↓
Order Block Engine
    ↓
FVG Engine
    ↓
OTE / Premium-Discount Engine
    ↓
Multi-Timeframe Alignment Engine
    ↓
Confluence Engine
    ↓
Decision Engine
    ↓
Adaptive Dashboard
    ↓
ICE Assistant / Alerts / Trade Planner
```

## 5. Engine Contracts

### 5.1 Market Structure Engine

**Purpose:** Detect confirmed internal and external structure.

**Outputs**
- Internal trend
- External trend
- Bullish/Bearish BOS
- Bullish/Bearish CHoCH
- Broken swing price
- Event timeframe
- Event confirmation state

**Rules**
- Confirm swings before publishing.
- Prevent duplicate events from the same swing.
- External structure takes priority when events overlap.
- Render dashed horizontal lines anchored to the broken swing.
- Show only tiny `BOS` or `CHoCH` text.

### 5.2 Liquidity Engine

**Purpose:** Track institutional liquidity pools and sweeps.

**Outputs**
- BSL / SSL
- EQH / EQL
- PDH / PDL
- PWH / PWL
- Sweep direction and confirmation

**States**
- Untouched
- Swept
- Retested
- Expired

### 5.3 Displacement Engine

**Purpose:** Confirm impulsive institutional movement.

**Outputs**
- Bullish displacement
- Bearish displacement
- Strength: Weak / Medium / Strong
- Confirmation timestamp

### 5.4 Order Block Engine

**Purpose:** Detect only institutional-quality order blocks.

**Required sequence**
- Liquidity sweep
- Displacement
- BOS or CHoCH
- Last opposite candle before displacement
- Optional FVG overlap
- Unmitigated state

**Outputs**
- Direction
- Upper/lower boundary
- Origin candle
- Fresh / Mitigated / Invalidated
- Creation time
- Quality score
- Timeframe

### 5.5 Fair Value Gap Engine

**Purpose:** Track active imbalances.

**Outputs**
- Bullish FVG
- Bearish FVG
- IFVG
- Balanced Price Range
- Open / Partial / Filled / Inverted
- Upper/lower boundary
- Timeframe

### 5.6 OTE / Premium-Discount Engine

**Purpose:** Determine institutional entry location.

**Outputs**
- External high/low
- Equilibrium
- 61.8%, 70.5%, 79%
- Premium / Discount / Equilibrium
- Inside OTE: true/false

### 5.7 Multi-Timeframe Alignment Engine

**Purpose:** Align higher-timeframe bias with lower-timeframe execution.

**Inputs**
- Six configurable timeframe roles
- Trend/structure state from each timeframe
- Required timeframe toggles
- Countertrend permission
- Countertrend score penalty
- Minimum alignment score

**Outputs**
- Macro, Primary, and Session bias
- Combined HTF bias
- Setup, Confirmation, and Precision direction
- Alignment score
- Misaligned timeframe
- Alignment state

**States**
- ALIGNED
- BUILDING
- MIXED
- CONFLICT
- COUNTERTREND
- NO TRADE

**Rules**
- A-grade entries require HTF/LTF directional agreement.
- Countertrend trades are blocked by default.
- Mixed or neutral HTF bias cannot produce an A-grade setup.
- No repainting from unconfirmed HTF candles.

### 5.8 Confluence Engine

**Purpose:** Convert engine facts into separate bullish and bearish evidence scores.

**Default weights**
- Liquidity: 20
- Displacement: 20
- Structure: 20
- Order Block: 15
- FVG: 10
- OTE: 10
- HTF alignment: 5

### 5.9 Decision Engine

**Purpose:** Be the single source of truth for all trading decisions.

**Outputs**
- Bias
- Recommendation
- Confidence
- Grade
- Setup stage
- Next step
- Missing conditions
- Conflict reason
- Trade eligibility

**Recommendations**
- BUY / SELL
- WAIT FOR LIQUIDITY
- WAIT FOR DISPLACEMENT
- WAIT FOR STRUCTURE
- WAIT FOR ORDER BLOCK
- WAIT FOR FVG
- WAIT FOR OTE
- WAIT FOR HTF ALIGNMENT
- SETUP BUILDING
- NO TRADE
- NO TRADE — CONFLICT
- COUNTERTREND — BLOCKED

### 5.10 Adaptive Dashboard Engine

**Purpose:** Present the same decision data on every screen size.

**Modes**
- Full: large monitors
- Compact: laptops
- Minimal: phones
- Off: hidden

**Compact mode must show**
- Bias
- Stage
- Liquidity
- Displacement
- Structure
- OB
- FVG
- OTE
- Next step
- Confidence / grade
- Decision

**Minimal mode must show**
- Bias
- Stage
- Next condition
- Score
- Decision

### 5.11 ICE Assistant Engine

**Purpose:** Convert Decision Engine outputs into plain-English guidance.

It answers:
1. What happened?
2. What is happening?
3. What is missing?
4. What should the trader wait for?
5. Why is the setup blocked?

### 5.12 Trade Planner

**Purpose:** Display a planning aid only after Decision Engine approval.

**Outputs**
- Direction
- Entry
- Stop
- TP1
- TP2
- Risk-to-reward
- Invalidation level

No plan should appear while the Decision Engine says WAIT or NO TRADE.

## 6. Setup Grades

| Score | Grade |
|---|---|
| 95–100 | A+ |
| 90–94 | A |
| 85–89 | B+ |
| 80–84 | B |
| 70–79 | C |
| Below 70 | Pass / No Trade |

An A or A+ sniper entry requires HTF alignment, the correct liquidity sweep, displacement, structure confirmation, a fresh OB or active FVG, OTE or premium/discount alignment, and execution-timeframe confirmation.

## 7. Visual Language

- Bullish BOS: green dashed line
- Bearish BOS: red dashed line
- Bullish CHoCH: aqua dashed line
- Bearish CHoCH: orange dashed line
- Bullish OB: green translucent zone, solid border
- Bearish OB: red translucent zone, solid border
- FVG: lighter fill, dashed border
- Liquidity: small subtle labels and lines
- EMA 20: green
- EMA 50: orange
- EMA 200: purple
- Values use color; labels remain neutral
- Minimal clutter is mandatory

## 8. Performance Rules

- Cap all line, label, and box history.
- Reuse table cells.
- Avoid duplicate calculations.
- Use confirmed HTF values.
- Prevent repeated event publication.
- Remove invalidated objects.
- Keep calculations separate from rendering.
- The published TradingView script remains one Pine v6 file.

## 9. Development Workflow

```text
Issue
  ↓
Design specification
  ↓
Feature branch
  ↓
Small compiling patches
  ↓
TradingView test
  ↓
GitHub commit
  ↓
Pull request
  ↓
Merge
```

## 10. Current Build Order

1. Finish Adaptive Dashboard
2. Multi-Timeframe Alignment Engine
3. Integrate alignment into Decision Engine
4. ICE Assistant
5. Trade Planner
6. Alerts
7. Statistics and replay
8. Performance optimization
9. Beta release

## 11. North Star

Every feature must improve at least one of these:

- Signal quality
- Readability
- Discipline
- Decision speed
- Trader education

If it does not improve one of those, it does not belong in ICE Genesis.
