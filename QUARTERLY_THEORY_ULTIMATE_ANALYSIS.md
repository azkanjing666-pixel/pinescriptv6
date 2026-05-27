# Quarterly Theory Ultimate - Complete Reverse Engineering

## Overview
This is the **actual production indicator** used on TradingView. It's a comprehensive system combining:
- **SSMT (Sequential Smart Money Technique)** divergence detection
- **QCISD (Quarterly Cycle Institutional Supply/Demand)** levels
- **Defining Range (DFR)** analysis
- **True Opens (TO)** tracking across multiple timeframes
- **Advanced Price Structure Points (PSP)**
- **Triad/Dyad correlation analysis**

---

## Core Architecture

### 1. **TYPE DEFINITIONS (UDTs)**

#### **QCISD_Level** - Core divergence level object
```
type QCISD_Level
    line    l                  // Visual line on chart
    label   lbl                // Label for the level
    int     dir                // Direction: 1=bullish, -1=bearish
    bool    is_hp              // High-Priority (aligned with bias)
    bool    confirmed          // Price has closed through level
    string  cycle_name         // "Micro", "90m", "Daily", etc.
    string  pair_name          // Correlated asset name
```
**Purpose:** Stores divergence levels from SSMT analysis. When correlated assets diverge in their highs/lows, this creates a level.

#### **DFR_Item** - Defining Range data
```
type DFR_Item
    line    ln_high, ln_low, ln_eq    // High/Low/Equilibrium lines
    line[]  ln_ext                     // Extension level lines
    label[] lbl_ext                    // Extension labels
    int     q1_start_t, gate_open_t    // Q1 timing
    float   dfr_high, dfr_low, dfr_eq  // Actual prices
```
**Purpose:** Tracks Q1 (first quarter) defining range with extension levels.

#### **TO_Level** - True Open tracking
```
type TO_Level
    line    l                  // Horizontal line
    label   lbl                // Label showing time/price
    float   price              // Opening price
    bool    active             // Currently active period
    string  name               // "TYO", "TQO", "TMO", etc.
```
**Purpose:** Tracks True Opens at different timeframe boundaries.

---

### 2. **THE SEVEN TIMEFRAME CYCLES**

| Cycle | Duration | Q1 Length | Gate Opens | Use Case |
|-------|----------|-----------|-----------|----------|
| **Micro** | 22.5 min | 22.5 min | 7.5 min (1/3) | Intraday scalping |
| **90m** | 90 min | 90 min | 30 min (1/3) | Day trading |
| **Daily** | 6 hours | 6 hours | 2 hours (1/3) | Swing trading |
| **Weekly** | 1 week | | | Position trading |
| **Monthly** | 1 month | | | Intermediate |
| **Quarterly** | 3 months | | | Macro analysis |
| **Yearly** | 1 year | | | Long-term |

**Each cycle has:** SSMT detection (Normal + Hidden divergence), QCISD levels, and True Opens.

---

### 3. **SSMT (SEQUENTIAL SMT) ENGINE**

#### **14-Slot SSMT Array per Timeframe**
```
int SSMT_N_BEAR1    = 0   // Normal bear div (prev-prev session)
int SSMT_N_BEAR0    = 1   // Normal bear div (current)
int SSMT_N_BULL1    = 2   // Normal bull div (prev-prev)
int SSMT_N_BULL0    = 3   // Normal bull div (current)
int SSMT_H_BEAR0    = 4   // Hidden bear div
int SSMT_H_BULL0    = 5   // Hidden bull div
int SSMT_N_BEAR_P2  = 6   // Normal bear (secondary pair)
int SSMT_N_BULL_P2  = 7   // Normal bull (secondary pair)
int SSMT_H_BEAR_P2  = 8   // Hidden bear (secondary pair)
int SSMT_H_BULL_P2  = 9   // Hidden bull (secondary pair)
int SSMT_ACT_N_BEAR = 10  // Any active normal bear
int SSMT_ACT_N_BULL = 11  // Any active normal bull
int SSMT_ACT_H_BEAR = 12  // Any active hidden bear
int SSMT_ACT_H_BULL = 13  // Any active hidden bull
```

#### **Normal Divergence Detection (f_ssmt)**
```
BEARISH DIVERGENCE:
  ah2 >= ah1 AND bh2 <= bh1  OR
  ah2 <= ah1 AND bh2 >= bh1
  (Previous-prev high >= Prev high) AND (Secondary doesn't confirm)

BULLISH DIVERGENCE:
  al2 <= al1 AND bl2 >= bl1  OR
  al2 >= al1 AND bl2 <= bl1
  (Previous-prev low <= Prev low) AND (Secondary doesn't confirm)
```

#### **Hidden Divergence Detection (f_hidden_ssmt)**
```
Uses CLOSING prices instead of swing highs/lows:
  clsmax >= clsmax1 AND cclsmax <= cclsmax1  (Hidden Bear)
  clsmin >= clsmin1 AND cclsmin <= cclsmin1  (Hidden Bull)
```

**Meaning:**
- **Normal Divergence** = Price makes new extreme but secondary doesn't → **Reversal coming**
- **Hidden Divergence** = Price doesn't make new extreme but secondary does → **Continuation**

---

### 4. **QCISD (Quarterly Cycle Institutional Levels)**

These are the actual support/resistance levels created by SSMT divergences.

**How QCISD Levels Are Created:**
1. When SSMT detects a bearish divergence → Create a **bearish QCISD level** at the recent high
2. When SSMT detects a bullish divergence → Create a **bullish QCISD level** at the recent low
3. Level shows as **DASHED** (pending) until price closes through it
4. Once confirmed (price closes through) → Shows as **SOLID**

**Level Lifecycle:**
- **Pending (Dashed):** Level identified but not yet broken
- **Confirmed (Solid):** Price closed through the level
- **Retest (Color change):** Price touches level again after confirmation
- **Invalidated (Deleted):** Price action moved away 3+ bars

---

### 5. **DEFINING RANGE (DFR) - THE CRITICAL Q1 ANALYSIS**

**Key Concept:** Every new quarter starts with a Q1 (first quarter/third). This period DEFINES the range and tone for the entire cycle.

**DFR Mechanics:**
```
Q1 Start → Gate Opens (after 1/3 elapse) → Gate Closes (Q1 ends)

Micro:   0.00 min → 7.5 min gate → 22.5 min close
90m:     0:00    → 30 min gate   → 90 min close
Daily:   6 hours → 2 hour gate   → 6 hour close
```

**What Gets Tracked:**
- **High/Low** formed during gated period
- **Equilibrium (EQ)** = (High + Low) / 2
- **Extensions** at multiples:
  - +1, +2, +2.5, +4 (above high)
  - -1, -2, -2.5, -4 (below low)

**Why It Matters:**
- Price respects DFR boundaries
- Extensions are liquidity targets
- DFR reset signals new cycle momentum

---

### 6. **TRIAD/DYAD CORRELATION SYSTEM**

**Triad = 3 correlated assets**
**Dyad = 2 correlated assets**

**How It Works:**
```
Main Asset (What you're trading): SPY
Secondary:                         IVY (or QQQ)
Tertiary:                          VIX (or DXY)

Compare their SSMT signals:
- All 3 agree on direction = HIGH PROBABILITY
- 2 agree, 1 disagrees = DIVERGENCE WARNING
- All 3 disagree = MAJOR REVERSAL LIKELY
```

**Inversion Option:**
- Some correlations are **inverse** (move opposite)
- Indicator can flip secondary/tertiary to show inverted correlation
- Example: VIX often moves opposite to equities

---

### 7. **TRUE OPENS (TO) TRACKING**

Marks where price opened at important cycle boundaries.

**TO Locations:**
| Timeframe | What It Tracks | Trigger |
|-----------|---------------|---------|
| **Yearly** | First Monday of April | Annual reset |
| **Quarterly** | 4th Sunday of Q start | Q reset |
| **Monthly** | 2nd Monday of month | M reset |
| **Weekly** | Monday 18:00 ET | Week reset |
| **Daily** | 00:00 ET | Day reset |
| **Session** | 4 times/day (AO/LO/NYO AM/PM) | Session opens |
| **90m** | Every 90 min on cycle | 90m reset |

**Why Traders Use Them:**
- Price gravitates back to True Opens
- Acts as magnet support/resistance
- Institutional traders anchor positions here

---

### 8. **AUTO BIAS SYSTEM**

Automatically determines if market is bullish or bearish based on **closing relative to higher timeframe True Opens**.

```
Micro → Compare close to 90m + Session TO
90m   → Compare close to Session + Daily TO
Daily → Compare close to Weekly TO
Weekly → Compare close to Monthly TO
Monthly → Compare close to Quarterly TO
Quarterly → Compare close to Yearly TO

If close > all comparisons = BULLISH BIAS
If close < all comparisons = BEARISH BIAS
If mixed = NEUTRAL BIAS
```

**Used For:**
- Coloring QCISD levels (green=bullish, red=bearish)
- Filtering false signals
- Alignment with higher timeframe bias

---

### 9. **ADVANCED PSP (Price Structure Points)**

Detects **divergences between main asset and correlates** at pivot points.

**Bullish PSP Signal:**
- Main asset makes pivot LOW
- Main asset candle is bullish (close > open)
- Secondary asset candle is bearish (close < open)
- **Interpretation:** Main asset bottoming while secondary weakening = Bullish setup

**Bearish PSP Signal:**
- Main asset makes pivot HIGH
- Main asset candle is bearish
- Secondary asset candle is bullish
- **Interpretation:** Main asset topping while secondary strengthening = Bearish setup

---

### 10. **SSMT STATUS BAR**

Shows at-a-glance which cycles have active Normal SSMT.

**Colors:**
- **Bullish (Green):** Active bullish SSMT
- **Bearish (Red):** Active bearish SSMT
- **Sandwich/Conflict (Gray):** Both bullish AND bearish active = Indecision

**Cycles Shown:** Y, Q, M, W, D, 90m, Mic

---

## Key Formulas & Logic

### **f_main_process** - Core SSMT calculation
```
Tracks for each session:
  hmax = highest high of session
  lmin = lowest low of session
  clsmax = highest close of session
  clsmin = lowest close of session
  
On new session:
  Store previous session's extremes
  Reset current session tracking
  
Returns 18 values tracking 3 bars:
  [current, previous, previous-previous] for highs/lows/closes
```

### **Divergence Logic**
```
N_BEAR0 = (ah1 >= ah0 AND bh1 <= bh0) OR (ah1 <= ah0 AND bh1 >= bh0)
          "Current session high:" vs "Secondary current"
          If moving opposite = divergence signal

N_BULL0 = (al1 <= al0 AND bl1 >= bl0) OR (al1 >= al0 AND bl1 <= bl0)
          "Current session low:" vs "Secondary current"
          If moving opposite = divergence signal
```

### **Level Confirmation**
```
Bearish Level:
  Created at bearish divergence high
  Pending if close > lvl
  Confirmed when close < lvl
  Invalidated after 3+ bars close < lvl

Bullish Level:
  Created at bullish divergence low
  Pending if close < lvl
  Confirmed when close > lvl
  Invalidated after 3+ bars close > lvl
```

---

## Alert System

**4 Customizable Alert Slots:**
Each with 2 control conditions (CON1 + CON3)

```
Both conditions must be TRUE to trigger:
  CON1 = First SSMT selection (e.g., "W Bull SSMT")
  CON3 = Second SSMT selection (e.g., "90m Bull SSMT")

Example: 
  CON1 = "W Bull SSMT" AND CON3 = "90m Bull SSMT"
  Alert only fires when Weekly AND 90-min both show bullish
  = Higher probability = Less noise
```

---

## Implementation Priority

### **Tier 1 - Core (Must Have)**
1. SSMT divergence detection (Normal only)
2. QCISD level creation
3. 2 timeframes (Daily + Weekly) working correctly

### **Tier 2 - Extended**
4. All 7 timeframes
5. Hidden SSMT divergence
6. Triad correlation

### **Tier 3 - Advanced**
7. DFR with extensions
8. True Opens tracking
9. PSP detection
10. Auto Bias system

---

## Session Timing (ET - America/New_York)

**Asia:** 18:00 - 06:00
**London:** 06:00 - 12:00
**NY AM:** 12:00 - 18:00 (includes US market)
**NY PM:** 18:00+ (PM trading)

**Important Times:**
- Asia Open: 19:30 ET
- London Open: 01:30 ET
- NY Open (9:30): 07:30 ET (after premarket opens 04:00)
- NY Close: 13:00 ET (16:00 market time)

---

## Next Steps

1. **Extract Core SSMT Engine** - Make it work with 1 timeframe (Daily)
2. **Add QCISD Level Creation** - Visualize divergence points
3. **Build Status Tracking** - Know when signals are active
4. **Add Correlation** - Compare 2 assets
5. **Scale to All 7 Timeframes** - Replicate full power

This is a **professional-grade institutional indicator**. The code is clean, the logic is sound, and every component serves a purpose.

Ready to build? 🚀
