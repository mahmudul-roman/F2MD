# Trust-Aware Misbehavior Detection Apps — Implementation Documentation

## Overview

Two new misbehavior detection applications were created by extending the existing
`ThresholdApp` and `BehavioralApp` in the F2MD (Framework for Misbehavior Detection)
Veins simulation framework. Both apps augment the standard detection pipeline with a
**trust value** `T ∈ [0, 1]` that represents how trustworthy a road network zone is at
a given position and simulation time.

> **Current state:** `getTrustValue()` returns a static value (currently `0.5`).
> Replace the body of that function in each `.cc` file with a real trust field table
> lookup when the trust infrastructure is ready.

---

## Files Created

| File | Location |
|---|---|
| `TrustAwareThresholdApp.h` | `src/veins/modules/application/f2md/mdApplications/` |
| `TrustAwareThresholdApp.cc` | `src/veins/modules/application/f2md/mdApplications/` |
| `TrustAwareBehavioralApp.h` | `src/veins/modules/application/f2md/mdApplications/` |
| `TrustAwareBehavioralApp.cc` | `src/veins/modules/application/f2md/mdApplications/` |

---

## Files Modified

| File | Change |
|---|---|
| `mdEnumTypes/MdAppTypes.h` | Added `TrustAwareThresholdApp = 6`, `TrustAwareBehavioralApp = 7` |
| `F2MDParameters.h` | Added 8 new trust parameters with defaults |
| `F2MDVeinsApp.h` | Added `#include` for both new apps + 4 instance members |
| `F2MDVeinsApp.cc` | Reads 8 trust params from ini; added cases in `setMDApp()` |
| `JosephVeinsApp.ned` | Declared all 8 trust parameters so OMNeT++ resolves them |
| `omnetpp.ini` | Set `appTypeV1=6`, `appTypeV2=7`; added full trust parameter block |

---

## New Enum Values (`MdAppTypes.h`)

```cpp
enum App {
    ThresholdApp            = 0,
    AggrigationApp          = 1,
    BehavioralApp           = 2,
    CooperativeApp          = 3,
    ExperiApp               = 4,
    MachineLearningApp      = 5,
    TrustAwareThresholdApp  = 6,   // NEW
    TrustAwareBehavioralApp = 7,   // NEW
    SIZE_OF_ENUM
};
```

---

## New Parameters (`F2MDParameters.h` + `JosephVeinsApp.ned`)

| Parameter | Type | Default | Description |
|---|---|---|---|
| `trustOptionV1` | `int` | `0` | Option for TrustAwareThresholdApp (0/1/2) |
| `trustOptionV2` | `int` | `0` | Option for TrustAwareBehavioralApp (0/1/2/3) |
| `trustLiftFactor` | `double` | `0.3` | ThresholdApp Option A: max threshold lift at T=0 |
| `lowTrustThreshold` | `double` | `0.2` | ThresholdApp Option C: below this → always report |
| `highTrustThreshold` | `double` | `0.8` | ThresholdApp Option C: above this → always pass |
| `trustAlpha` | `double` | `0.5` | BehavioralApp Option A: blend weight (0=trust only, 1=checks only) |
| `trustBeta` | `double` | `1.0` | BehavioralApp Option B: accumulation amplification strength |
| `trustMinReportThreshold` | `double` | `2.0` | BehavioralApp Option C: report threshold when T=0 |

---

## TrustAwareThresholdApp

### Inheritance
`MDApplication` → `TrustAwareThresholdApp`

### Constructor
```cpp
TrustAwareThresholdApp(int version, double Threshold,
    int trustOption, double trustLiftFactor,
    double lowTrustThreshold, double highTrustThreshold);
```

### Key Method
```cpp
bool CheckNodeForReport(unsigned long myPseudonym,
    BasicSafetyMessage* bsm, BsmCheck* bsmCheck,
    NodeTable* detectedNodes);
```

### Trust Value Placeholder
```cpp
double getTrustValue(veins::Coord senderPos, double currentTime);
// Currently returns 0.5 — replace with real lookup
```

### Option A — LiftedThreshold (`trustOption = 0`)

Raises the comparison threshold in low-trust areas so check scores that
would barely pass normally are caught.

```
effectiveThreshold = Threshold + (1 - T) * trustLiftFactor

T = 1.0  →  effectiveThreshold = Threshold          (no change)
T = 0.0  →  effectiveThreshold = Threshold + trustLiftFactor  (strictest)
```

All 17 check scores (proximity, range, speed, position, Kalman, etc.) are
compared against `effectiveThreshold` instead of the fixed `Threshold`.

**When to use:** You want trust to uniformly tighten all checks in
suspicious zones without changing the decision logic.

---

### Option B — TrustAsCheck (`trustOption = 1`)

The trust score `T` is treated as one additional check value alongside the
existing plausibility/consistency scores. If `T <= Threshold`, the sender
is reported — exactly as if any other check had failed.

```
if (T <= Threshold) → checkFailed = true
```

**When to use:** You want trust to be architecturally equal to any other
plausibility check — one failed check out of many triggers a report.

---

### Option C — TrustGate (`trustOption = 2`)

Trust acts as a hard gate before standard checks run.

```
T < lowTrustThreshold  → report unconditionally (dangerous zone)
T > highTrustThreshold → suppress report (highly trusted zone)
otherwise              → run normal threshold checks
```

**When to use:** You have high confidence in the trust field and want it
to override individual check results in extreme zones.

---

## TrustAwareBehavioralApp

### Inheritance
`MDApplication` → `TrustAwareBehavioralApp`

### Constructor
```cpp
TrustAwareBehavioralApp(int version, double Threshold,
    int trustOption, double trustAlpha,
    double trustBeta, double trustMinReportThreshold);
```

### Key Method
```cpp
bool CheckNodeForReport(unsigned long myPseudonym,
    BasicSafetyMessage* bsm, BsmCheck* bsmCheck,
    NodeTable* detectedNodes);
```

### Trust Value Placeholder
```cpp
double getTrustValue(veins::Coord senderPos, double currentTime);
// Currently returns 0.5 — replace with real lookup
```

### Background: How BehavioralApp Works

The base BehavioralApp accumulates a per-sender `TimeOut` counter:

```
minFactor = min(all check scores)               // ∈ (-∞, 1]
TMOadd    = 0.0005 * exp(10 * (1 - minFactor))  // exponential accumulation
TimeOut  += TMOadd
TimeOut  -= 1.0                                 // decay per beacon
if TimeOut > 5.5066 → report sender
```

The scale mismatch problem: `TimeOut` can grow to large values, but
`T ∈ [0, 1]`. The four options solve this in different ways.

---

### Option A — BlendMinFactor (`trustOption = 0`) ✅ Recommended

Blends `minFactor` and `T` in the same `[0, 1]` space **before** the
exponential. This fully eliminates the scale mismatch.

```
effectiveMinFactor = trustAlpha * minFactor + (1 - trustAlpha) * T
TMOadd = 0.0005 * exp(10 * (1 - effectiveMinFactor))

trustAlpha = 0.5  →  equal weight between checks and trust
trustAlpha = 1.0  →  checks only (same as plain BehavioralApp)
trustAlpha = 0.0  →  trust only
```

- Low trust (T→0) boosts `TMOadd` even if checks are clean.
- Report threshold stays at 5.5066 (unchanged).

**When to use:** Default choice. Cleanest mathematical formulation.

---

### Option B — TrustMultiplier (`trustOption = 1`)

Trust amplifies how fast evidence accumulates. Checks must still detect
something for trust to have any effect.

```
trustMultiplier = 1 + trustBeta * (1 - T)
TMOadd = 0.0005 * exp(10 * (1 - minFactor)) * trustMultiplier

T = 1.0  →  multiplier = 1.0   (no change)
T = 0.0  →  multiplier = 1 + trustBeta  (e.g. 2.0 at trustBeta=1)
```

**When to use:** You want trust to act as a risk amplifier on top of
existing check evidence, not as an independent signal.

---

### Option C — AdjustedThreshold (`trustOption = 2`)

Lowers the reporting bar in low-trust areas without changing how evidence
accumulates.

```
effectiveReportThreshold = 5.5066 * T + trustMinReportThreshold * (1 - T)

T = 1.0  →  threshold = 5.5066  (normal)
T = 0.0  →  threshold = trustMinReportThreshold  (e.g. 2.0)
```

**When to use:** You want trust to make it easier to act on already-
accumulated suspicion, but not change the accumulation dynamics.

---

### Option D — DecayModifier (`trustOption = 3`)

Trust controls how fast accumulated suspicion decays. In low-trust areas,
suspicion lingers indefinitely.

```
decayAmount = 1.0 * T
TimeOut -= decayAmount  (instead of fixed -1)

T = 1.0  →  full decay (-1 per beacon, same as normal)
T = 0.0  →  no decay   (counter never decreases)
```

**When to use:** You want suspicious zones to cause long-term memory of
bad behaviour even after checks return to normal.

---

## Detection Flow (with trust integration)

```
Receive BSM from sender
        │
        ├── ExperiChecks::CheckBSM()
        │       Produces BsmCheck with scores for all 17 checks
        │
        ├── V1: TrustAwareThresholdApp::CheckNodeForReport()
        │       1. getTrustValue(senderPos, simTime)   ← T ∈ [0,1]
        │       2. Apply selected option (A / B / C)
        │       3. Return true if sender should be reported
        │
        └── V2: TrustAwareBehavioralApp::CheckNodeForReport()
                1. getTrustValue(senderPos, simTime)   ← T ∈ [0,1]
                2. calculateMinFactor(bsmCheck)
                3. Apply selected option (A / B / C / D)
                4. Update TimeOut counter for sender
                5. Return true if TimeOut exceeds report threshold
```

---

## `omnetpp.ini` Configuration

```ini
# App type selection
*.node[*].appl.appTypeV1 = 6    # TrustAwareThresholdApp
*.node[*].appl.appTypeV2 = 7    # TrustAwareBehavioralApp

# Option selection
*.node[*].appl.trustOptionV1 = 0   # 0=LiftedThreshold, 1=TrustAsCheck, 2=TrustGate
*.node[*].appl.trustOptionV2 = 0   # 0=BlendMinFactor, 1=TrustMultiplier,
                                    # 2=AdjustedThreshold, 3=DecayModifier

# ThresholdApp parameters
*.node[*].appl.trustLiftFactor     = 0.3
*.node[*].appl.lowTrustThreshold   = 0.2
*.node[*].appl.highTrustThreshold  = 0.8

# BehavioralApp parameters
*.node[*].appl.trustAlpha               = 0.5
*.node[*].appl.trustBeta                = 1.0
*.node[*].appl.trustMinReportThreshold  = 2.0
```

---

## How to Plug In Real Trust Values

Replace the placeholder body in **both** `.cc` files:

**`TrustAwareThresholdApp.cc`:**
```cpp
double TrustAwareThresholdApp::getTrustValue(veins::Coord senderPos, double currentTime)
{
    return trustFieldTable.lookup(senderPos, currentTime);
}
```

**`TrustAwareBehavioralApp.cc`:**
```cpp
double TrustAwareBehavioralApp::getTrustValue(veins::Coord senderPos, double currentTime)
{
    return trustFieldTable.lookup(senderPos, currentTime);
}
```

The function signature accepts the sender's claimed position (from the BSM)
and the current simulation time, matching the expected trust table interface.

---

## Instance Members Added to `F2MDVeinsApp.h`

```cpp
TrustAwareThresholdApp  TrustThreV1 = TrustAwareThresholdApp(1, 0.28125, 0, 0.3, 0.2, 0.8);
TrustAwareThresholdApp  TrustThreV2 = TrustAwareThresholdApp(2, 0.28125, 0, 0.3, 0.2, 0.8);
TrustAwareBehavioralApp TrustBehaV1 = TrustAwareBehavioralApp(1, 0.5, 0, 0.5, 1.0, 2.0);
TrustAwareBehavioralApp TrustBehaV2 = TrustAwareBehavioralApp(2, 0.5, 0, 0.5, 1.0, 2.0);
```

These are re-initialised inside `setMDApp()` (called at stage 1) with the
actual parameter values loaded from the `.ini` file.

---

*Generated from F2MD simulation framework extension session.*
