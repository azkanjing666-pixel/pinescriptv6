# Quarterly Theory - TIME CYCLE ENGINE (Core Foundation)

## Critical Understanding

**Everything in Quarterly Theory is built on ONE thing: Correct Time Cycle Detection**

If your cycles are wrong:
- ❌ SSMT divergences trigger at wrong times
- ❌ QCISD levels form in wrong places
- ❌ DFR tracking fails
- ❌ True Opens mismatch
- ❌ Entire system collapses

The **time cycle engine** is NOT a feature. It's the **foundation**.

---

## The Seven Cycles (Nested)

```
YEARLY (1 year)
  └─ QUARTERLY (3 months)
      └─ MONTHLY (1 month)
          └─ WEEKLY (1 week)
              └─ DAILY (6 hours)
                  └─ 90-MINUTE (90 minutes)
                      └─ MICRO (22.5 minutes)
```

Each cycle contains **4 quarters (phases)**.

---

## How Cycles Actually Work

### **MICRO Cycle (22.5 minutes total)**
```
Timeline:
Q1: 0-5.625 min    (first quarter)
Q2: 5.625-11.25 min
Q3: 11.25-16.875 min
Q4: 16.875-22.5 min (completes, new cycle starts)

Then repeats: 22.5-45 min is new Micro cycle
```

**In Code:**
```pinescript
ms_Micro = 1350000  // 22.5 * 60 * 1000 milliseconds
ms_Into_Micro = int((time - dayStart) % ms_Micro)
qMicro = int(math.floor(ms_Into_Micro / msMicro)) + 1
// Result: qMicro = 1, 2, 3, or 4
```

---

### **90-MINUTE Cycle**
```
Timeline:
Q1: 0-22.5 min     (first 1/4)
Q2: 22.5-45 min    (second 1/4)
Q3: 45-67.5 min    (third 1/4)
Q4: 67.5-90 min    (completes, new cycle starts)

Then repeats: 90-180 min is new 90m cycle
```

**In Code:**
```pinescript
ms_90m = 90 * 60 * 1000  // milliseconds
int minsIn6H = minsFromDayStart % 360  // 360 min = 6 hours
q90 = int(math.floor(minsIn6H / 90)) + 1
// Result: q90 = 1, 2, 3, or 4
```

---

### **DAILY Cycle (6-hour periods)**
```
Session structure (ET timezone):
Q1: 00:00-06:00 (Midnight to 6am)
Q2: 06:00-12:00 (London/NY AM opens)
Q3: 12:00-18:00 (NY close to 6pm)
Q4: 18:00-00:00 (Asia/Evening)

Then repeats next day
```

**Key:** Day starts at **18:00 ET (6pm)**, not midnight!

```pinescript
dayStartToday = timestamp(tz, yy, mo, dom, 18, 0)
dayStart = time >= dayStartToday ? dayStartToday : (dayStartToday - msDay)

minsFromDayStart = int(((time - dayStart) % msDay) / (60 * 1000))
qDay = int(math.floor(minsFromDayStart / 360)) + 1  // 360 min per quarter
// Result: qDay = 1, 2, 3, or 4
```

---

### **WEEKLY Cycle**
```
Week structure (Monday = 1):
Q1: Monday (day 2 of week)
Q2: Tuesday (day 3)
Q3: Wednesday-Thursday (days 4-5)
Q4: Friday-Sunday (days 6-1)

Reset every Monday
```

```pinescript
qWeek = dow_c == dayofweek.sunday ? 1 : dow_c == dayofweek.monday ? 2 : ...
// Result: qWeek = 1-7 (each day of week)

// On Monday of new week: new_week = true
```

---

### **MONTHLY Cycle**
```
4 Quarters within month:
Q1: Days 1-7 (first week + 1 day)
Q2: Days 8-14
Q3: Days 15-21
Q4: Days 22-end

ISO week-based calculation for accuracy
```

```pinescript
curr_week = weekofyear(dayStart + msDay, tz)
weeks_in_year = (d_jan1 == dayofweek.thursday or ...) ? 53 : 52

qMonth = weeks_in_year == 53 ? 
  (curr_week == 1 ? 5 : ((curr_week - 2) % 4) + 1) : 
  ((curr_week - 1) % 4) + 1
// Result: qMonth = 1, 2, 3, 4, or 5
```

---

### **QUARTERLY Cycle (3 months)**
```
Year structure:
Q1: Jan, Feb, Mar  (month 1-3)
Q2: Apr, May, Jun  (month 4-6)
Q3: Jul, Aug, Sep  (month 7-9)
Q4: Oct, Nov, Dec  (month 10-12)

Reset first day of each quarter
```

```pinescript
q2Month = ((mo_c - 1) % 3) + 1  // converts month 1-12 to quarter 1-3
is_quarter1 = q2Month == 1
is_quarter2 = q2Month == 2
is_quarter3 = q2Month == 3
is_quarter4 = q2Month == 4

qYear = mo_c <= 3 ? 1 : mo_c <= 6 ? 2 : mo_c <= 9 ? 3 : 4
// Result: qYear = 1, 2, 3, or 4
```

---

### **YEARLY Cycle**
```
Year structure:
Q1: Jan, Feb, Mar, Apr  (first 4 months / first quarter of year)
Q2: May, Jun, Jul, Aug  (second quarter of year)
Q3: Sep, Oct, Nov       (third quarter of year)
Q4: Dec                 (fourth quarter of year)

Reset January 1st
```

```pinescript
is_year1 = qYear == 1
is_year2 = qYear == 2
is_year3 = qYear == 3
is_year4 = qYear == 4
```

---

## The Critical Time Variables (from code)

```pinescript
// DAY START (18:00 ET, not midnight!)
dayStartToday    = timestamp(tz, yy, mo, dom, dayStartHour, 0)
dayStart         = time >= dayStartToday ? dayStartToday : (dayStartToday - msDay)

// MINUTES FROM DAY START
minsFromDayStart = int(((time - dayStart) % msDay) / (60 * 1000))  // 0..1439

// YEAR/MONTH/DAY (from dayStart, not from "now")
yy_c  = year(dayStart, tz)
mo_c  = month(dayStart, tz)
dom_c = dayofmonth(dayStart, tz)
dow_c = dayofweek(dayStart, tz)

// WEEK (ISO week number)
curr_week = weekofyear(dayStart + msDay, tz)

// CYCLE CALCULATIONS
qYear = mo_c <= 3 ? 1 : mo_c <= 6 ? 2 : mo_c <= 9 ? 3 : 4
q2Month = ((mo_c - 1) % 3) + 1
qMonth = ... (ISO week calculation)
qWeek = dayofweek calculation
qDay = int(math.floor(minsFromDayStart / 360)) + 1
q90 = int(math.floor((minsFromDayStart % 360) / 90)) + 1
qMicro = int(math.floor((time - dayStart) % ms90m / msMicro)) + 1
```

---

## Change Detection (New Cycle Signals)

```pinescript
// ta.change() detects when value changes from previous bar
new_year = ta.change(qYear)    // true when qYear changes
new_quart = ta.change(q2Month) // true when quarter changes
new_month = ta.change(qMonth)  // true when month changes
new_week = ta.change(qWeek)    // true when week changes
new_daily = ta.change(qDay)    // true when day changes
new_90m = ta.change(q90)       // true when 90m changes
new_micro = ta.change(qMicro)  // true when micro changes
```

**Usage:**
```pinescript
if new_daily != 0  // On day change
    // Reset daily tracking
    // Create new DFR
    // Record True Open
    // Start new cycle logic
```

---

## The Session Index System

Instead of individual booleans, the code uses **arrays of 64 combinations** for Micro/90m:

```pinescript
// 4 daily phases × 4 90m per day × 4 micro per 90m = 64 micro sub-periods
arr_sesm = array.from(
  is_sesa1, is_sesa2, is_sesa3, is_sesa4,  // Day 1, Micro 1-4
  is_sesb1, is_sesb2, is_sesb3, is_sesb4,  // Day 2, Micro 1-4
  is_sesc1, is_sesc2, is_sesc3, is_sesc4,  // Day 3, Micro 1-4
  is_sesd1, is_sesd2, is_sesd3, is_sesd4,  // Day 4, Micro 1-4
  ...
)

// Find which one is TRUE
val_sesm = array.indexof(arr_sesm, true)  // Returns 0-63

// Detect change
newm = val_sesm != val_sesm[1]  // New micro period
```

---

## Key Rules for Cycle Logic

### **1. Day Starts at 18:00 ET, Not 00:00**
This is institutional convention (Asia opens at that time).

### **2. Every Cycle Has 4 Quarters**
- Each quarter = 1/4 of the cycle time
- Q1 always comes first
- On Q1 start = new cycle signal

### **3. Cycles Are Nested**
- Micro completes, starts new 90m
- 90m completes, starts new Daily
- Daily completes, starts new Weekly
- etc.

### **4. Boundaries Are Precise**
```
Don't use: "approximately when day changes"
Use: timestamp calculations + millisecond-level accuracy
```

### **5. Time Zone Matters**
All calculations use **"America/New_York"** timezone.
Must be consistent throughout.

---

## Implementation Checklist

### **Phase 1: Core Time Variables**
- [ ] Set `dayStartHour = 18` (6pm ET)
- [ ] Calculate `dayStart` correctly (not midnight!)
- [ ] Calculate `minsFromDayStart` 
- [ ] Extract `yy_c`, `mo_c`, `dom_c`, `dow_c` from `dayStart`

### **Phase 2: Quarterly Calculations**
- [ ] `qYear` = 1-4 based on month
- [ ] `q2Month` = 1-3 (which month in quarter)
- [ ] `qMonth` = 1-5 (ISO week based)
- [ ] `qWeek` = 1-7 (day of week)
- [ ] `qDay` = 1-4 (period of day)
- [ ] `q90` = 1-4 (90-min period)
- [ ] `qMicro` = 1-4 (micro period)

### **Phase 3: Change Detection**
- [ ] Implement all 7 `ta.change()` calls
- [ ] Create `new_year`, `new_quart`, `new_month`, `new_week`, `new_daily`, `new_90m`, `new_micro`
- [ ] Test that they trigger **exactly** when cycles change

### **Phase 4: Session Indices**
- [ ] Build session arrays for all timeframes
- [ ] Calculate `val_ses*` indices
- [ ] Verify `val_ses* != val_ses*[1]` triggers on change

### **Phase 5: Testing**
- [ ] Print current cycles every bar
- [ ] Watch bar-by-bar as cycles change
- [ ] Verify timing matches expected boundaries
- [ ] Confirm each cycle advances 1→2→3→4→1

---

## Code Structure (What You Need)

```pinescript
//@version=6
indicator("Time Cycle Engine", overlay=true)

// SETTINGS
string tz = "America/New_York"
int dayStartHour = 18

// TIME CONSTANTS
int msDay  = 24 * 60 * 60 * 1000
int ms90m  = 90 * 60 * 1000
int msMicro = 1350000  // 22.5 min

// CYCLE DETECTION (every bar)
int dayStart = na
int yy_c = na, mo_c = na, dom_c = na, dow_c = na
int qYear = na, q2Month = na, qMonth = na
int qWeek = na, qDay = na, q90 = na, qMicro = na

// Calculate all cycles
if barstate.isnew
    // dayStart calculation
    // All qYear, q2Month, etc. calculations
    // All change detection

// PLOT/DISPLAY
// Show current cycles so you can verify
plotchar(bar_index, char=str.tostring(qDay) + str.tostring(q90) + str.tostring(qMicro))
```

---

## Why This Matters

**Every SSMT divergence** depends on knowing EXACTLY which session/cycle you're in.

**Every QCISD level** anchors to cycle boundaries.

**Every DFR** resets on Q1 start.

**Every True Open** marks cycle boundaries.

**Get the time cycles wrong = Everything fails.**

This is why the original code spends 20% of its logic on **just cycle detection**.

---

## Next: Build This First

Create a simple indicator that:
1. Shows current cycle values: `[qYear][q2Month][qMonth][qWeek][qDay][q90][qMicro]`
2. Displays on every bar
3. Verify they match the boundaries you expect
4. Watch day/week/month changes happen in real-time

**Only once cycles are 100% correct** should you build anything else.

This is the foundation. Everything else is built on top.
