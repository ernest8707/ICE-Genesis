# ICE Genesis Architecture

## Purpose

ICE Genesis is a single-file Pine Script v6 indicator organized as a pipeline of cooperating engines. Each engine has one primary responsibility and publishes state that downstream engines consume.

The active TradingView source remains:

```text
src/ICE_Genesis_v1_0.pine
```

Git history—not versioned filenames—tracks prior builds.

## Design principles

1. **Confirmed data first.** Structure and higher-timeframe values must use confirmed pivots or completed candles wherever practical.
2. **Detection is separate from rendering.** Engines determine facts; visual sections draw those facts.
3. **One source of truth.** Downstream modules consume published engine state instead of recalculating structure independently.
4. **No duplicate events.** A confirmed swing may produce no more than one structure-break event.
5. **Controlled chart objects.** Lines, labels, and boxes must respect configurable history limits and Pine object limits.
6. **Incremental development.** One engine is changed and tested at a time before merging into `main`.

## Runtime pipeline

```text
Price and volume
      |
      v
Current-timeframe context + HTF bias
      |
      v
Swing Registry
      |
      +--------------------+
      |                    |
      v                    v
Liquidity Engine     Displacement Engine
      |                    |
      +---------+----------+
                |
                v
Market Structure Classification
(BOS / CHoCH, internal / external, trend state)
                |
                +--------------------+
                |                    |
                v                    v
Order Block Engine             FVG Engine
                |                    |
                +---------+----------+
                          |
                          v
                  Confluence / ICE Score
                          |
                          v
                Trade Plan + Dashboard
                          |
                          v
                        Alerts
```

## Current source layout

The present codebase is divided into these sections:

1. Inputs
2. Colors
3. Current-timeframe EMA context
4. Higher-timeframe bias
5. Session and displacement
6. Market Structure Engine — swing registry
7. Liquidity Engine
8. Market Structure Engine — classification and state
9. Order Block / FVG candidates
10. Score and qualification
11. Market-structure rendering
12. Order Block / FVG rendering
13. Trade plan
14. Optional HTF levels
15. Dashboard
16. Alerts

The target architecture keeps the single Pine file but clarifies the contracts between these sections.

---

# Engine contracts

## 1. Context Engine

### Responsibility

Calculate current-timeframe EMA context, completed-candle higher-timeframe bias, trading-session state, ATR, and basic candle metrics.

### Inputs

- Price and volume series
- 20, 50, and 200 EMA settings
- Primary and confirmation timeframes
- Session definitions
- ATR and candle-body thresholds

### Published outputs

```text
ema20, ema50, ema200
bullBias, bearBias, biasText
inSession, sessionAllowed, activeSessionText
atr, bodyPct
```

### Rules

- HTF requests should use completed candles.
- Context outputs must not create structure events.
- Session state is a filter, not a market-direction signal.

---

## 2. Swing Registry

### Responsibility

Detect, confirm, store, and identify internal and external swing highs and lows.

### Inputs

```text
internalLen
externalLen
high, low
```

### Published outputs

```text
internalSwingHighPrice
internalSwingHighBar
internalSwingLowPrice
internalSwingLowBar
externalSwingHighPrice
externalSwingHighBar
externalSwingLowPrice
externalSwingLowBar
```

### Required state

Each stored swing needs:

```text
price
origin bar index
confirmed status
consumed/broken status
```

### Rules

- Pivots become valid only after confirmation.
- Internal and external registries remain separate.
- A new confirmed pivot resets the consumed flag for that side.
- A consumed swing cannot produce another BOS or CHoCH.

---

## 3. Liquidity Engine

### Responsibility

Track liquidity pools and publish confirmed sweep events.

### Liquidity types

- Internal BSL / SSL
- External BSL / SSL
- EQH / EQL
- Previous-day high / low
- Previous-week high / low

### Inputs

- Confirmed swing registry
- Equal-level tolerance
- Sweep confirmation mode
- Maximum sweep-to-break window

### Published outputs

```text
bullishLiquiditySweep
bearishLiquiditySweep
lastBullishSweepBar
lastBearishSweepBar
lastLiquidityType
lastLiquidityPrice
```

Direction convention:

```text
Bullish setup liquidity event = sell-side liquidity swept
Bearish setup liquidity event = buy-side liquidity swept
```

### Rules

- Wick Rejection requires a trade through the level and a close back inside.
- Any Trade Through requires only penetration.
- Equal levels remain active until swept or removed by history limits.
- Structure filters consume liquidity events but do not redraw or redefine them.

---

## 4. Displacement Engine

### Responsibility

Identify meaningful bullish and bearish impulsive candles.

### Inputs

- ATR
- Candle body size
- Body percentage of range
- Optional FVG requirement

### Published outputs

```text
bullDisplacement
bearDisplacement
recentBullDisplacement
recentBearDisplacement
lastDisplacementBar
```

### Rules

A displacement candle should satisfy:

```text
correct candle direction
body >= ATR multiple
body percentage >= threshold
optional same-direction FVG
```

Dashboard memory may retain a recent event, but structure qualification must use a clearly defined event window.

---

## 5. Market Structure Engine 2.0

### Responsibility

Classify confirmed breaks of internal and external swings, maintain trend state, prevent duplicates, and publish structure events to all downstream engines.

### Inputs

- Swing Registry outputs
- Liquidity Engine outputs
- Displacement Engine outputs
- Close-versus-wick break setting
- Sweep and displacement requirements

### Event types

```text
Internal Bullish BOS
Internal Bearish BOS
Internal Bullish CHoCH
Internal Bearish CHoCH
External Bullish BOS
External Bearish BOS
External Bullish CHoCH
External Bearish CHoCH
```

### Break definition

```text
Close mode:
  bullish break = close > swing high
  bearish break = close < swing low

Trade-through mode:
  bullish break = high > swing high
  bearish break = low < swing low
```

### Classification contract

```text
Break in the same direction as current structure state -> BOS
Break opposite the current structure state -> CHoCH
First accepted break when state is neutral -> establishes direction
```

### Published outputs

```text
internalTrendState   // -1 bearish, 0 neutral, +1 bullish
externalTrendState   // -1 bearish, 0 neutral, +1 bullish
structureEvent       // true only on event bar
structureDirection   // -1 or +1
structureClass       // BOS or CHoCH
structureScope       // Internal or External
structurePrice
structureSwingBar
structureBreakBar
lastStructureText
```

### Precedence

When internal and external events occur on the same bar:

```text
External event has dashboard and downstream precedence.
Both may be rendered if enabled.
```

### Acceptance gates

An event is accepted only when all enabled gates pass:

```text
swing not already consumed
valid break through swing
liquidity sweep inside configured window, when required
displacement confirmation, when required
```

### Duplicate prevention

After an event is accepted:

```text
mark that swing consumed
publish one event pulse
wait for a newly confirmed pivot before that side can trigger again
```

### Rendering contract

Rendering reads published event data and does not reclassify the event.

Each structure visual contains:

```text
dashed horizontal line
line begins at original swing bar
line reaches break bar plus configured forward extension
tiny BOS or CHoCH text on the line
bullish and bearish direction colors
separate internal and external visual strength
bounded history
```

---

## 6. Order Block Engine

### Responsibility

Create institutional-quality order-block candidates from accepted structure sequences.

### Inputs

- Accepted structure event
- Relevant liquidity sweep
- Displacement event
- Search window
- Optional FVG overlap requirement

### Candidate rule

```text
Bullish OB = last bearish candle in the qualifying leg before bullish displacement and accepted bullish structure break.
Bearish OB = last bullish candle in the qualifying leg before bearish displacement and accepted bearish structure break.
```

### Published outputs

```text
activeOBDirection
activeOBTop
activeOBBottom
activeOBCreationBar
activeOBState       // Active, Mitigated, Invalid
activeOBQuality
```

### Rules

- OB search must remain inside the qualifying price leg.
- Creation candle cannot mitigate its own zone.
- Mitigation mode is Touch, Midpoint, or Full.
- Active state and rendering state remain separate.

---

## 7. Fair Value Gap Engine

### Responsibility

Detect, track, and publish active bullish and bearish imbalances.

### Inputs

- Three-candle price relationship
- Displacement direction
- Optional setup association

### Published outputs

```text
activeFVGDirection
activeFVGTop
activeFVGBottom
activeFVGCreationBar
activeFVGState      // Active, Filled, Invalid
```

### Rules

- Bullish FVG: current low above high two bars earlier.
- Bearish FVG: current high below low two bars earlier.
- Creation candle cannot fill its own FVG.
- Rendering uses a dashed border to distinguish FVGs from OBs.

---

## 8. Confluence / ICE Score Engine

### Responsibility

Score the current setup from published engine states without redetecting market structure.

### Inputs

- Liquidity sweep
- Displacement
- Structure event/state
- Active OB
- Active FVG
- OTE state
- HTF alignment
- Premium/discount
- Session alignment

### Default scoring model

| Condition | Points |
|---|---:|
| Liquidity sweep | 15 |
| Strong displacement | 15 |
| BOS / CHoCH | 15 |
| Valid order block | 15 |
| Active FVG | 10 |
| OTE | 10 |
| HTF alignment | 10 |
| Premium / discount | 5 |
| Session | 5 |

### Published outputs

```text
setupDirection
setupScore
setupGrade
setupQualified
setupReason
```

---

## 9. Trade Plan Engine

### Responsibility

Convert an already-qualified setup into an optional visual planning aid.

### Inputs

- Active OB / FVG
- Entry mode
- ATR stop buffer
- Risk/reward target

### Published outputs

```text
plannedEntry
plannedStop
plannedTarget
plannedRR
```

This engine must not create a setup by itself.

---

## 10. Dashboard Engine

### Responsibility

Display published engine state clearly and explain why the current setup is accepted, developing, or rejected.

### Inputs

Dashboard reads only published outputs from other engines.

### Required sections

```text
Market Context
Setup Checklist
Trade Decision
Optional Trade Plan
```

### Market Structure display contract

```text
Direction: Bullish or Bearish
Class: BOS or CHoCH
Scope: Internal or External
Status: Confirmed
```

Example:

```text
Structure
Bullish CHoCH
External · Confirmed
```

### Rules

- External structure takes precedence when events conflict on the same bar.
- “Waiting” should be used only when no useful state exists.
- Where possible, show active price ranges and states rather than generic labels.

---

## 11. Alert Engine

### Responsibility

Expose stable alert conditions derived from published events.

### Initial alerts

```text
Internal bullish/bearish BOS
Internal bullish/bearish CHoCH
External bullish/bearish BOS
External bullish/bearish CHoCH
Qualified bullish/bearish setup
Order block mitigation
FVG fill
```

Alerts must use the same accepted event pulse as the dashboard and rendering logic.

---

# State ownership

To avoid conflicting logic, each piece of state has one owner:

| State | Owner |
|---|---|
| Confirmed swings | Swing Registry |
| Liquidity pools and sweeps | Liquidity Engine |
| Displacement | Displacement Engine |
| BOS / CHoCH and trend state | Market Structure Engine |
| OB boundaries and mitigation | Order Block Engine |
| FVG boundaries and fill state | FVG Engine |
| Setup score and grade | Confluence Engine |
| Entry / stop / target | Trade Plan Engine |
| Presentation | Rendering and Dashboard sections |

No downstream engine should overwrite upstream state.

# Development workflow

For each GitHub issue:

1. Define or update the engine contract in this document.
2. Create or switch to the issue branch.
3. Change only the required engine and its direct integrations.
4. Compile in TradingView.
5. Test on the agreed symbols and timeframes.
6. Commit the tested change.
7. Open a pull request linked to the issue.
8. Merge only after acceptance criteria pass.

# Market Structure Engine 2.0 acceptance criteria

Issue #1 is complete only when all boxes below pass:

- [ ] Internal and external pivots confirm without repainting after confirmation.
- [ ] Bullish and bearish BOS classify correctly.
- [ ] Bullish and bearish CHoCH classify correctly.
- [ ] Each swing produces no more than one event.
- [ ] Optional close-break mode works.
- [ ] Optional liquidity gate works.
- [ ] Optional displacement gate works.
- [ ] External events take precedence in dashboard state.
- [ ] Dashed lines begin at the broken swing.
- [ ] Tiny BOS/CHoCH text is readable and does not use label boxes.
- [ ] Structure history respects configured limits.
- [ ] Order blocks, FVGs, liquidity, OTE, sessions, and alerts still compile.
- [ ] Tested on META 1H, MSFT 1H, SPY 15m, and QQQ 15m.

# Planned issue sequence

```text
#1 Market Structure Engine 2.0
#2 Institutional Liquidity Engine
#3 Displacement Engine
#4 Institutional Order Block Engine
#5 Fair Value Gap / IFVG Engine
#6 OTE and Premium/Discount Engine
#7 ICE Score Engine
#8 Dashboard and Narrative Engine
#9 Performance and publication readiness
```
