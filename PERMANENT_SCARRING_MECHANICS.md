# RimWorld Permanent Scarring Mechanics Documentation

This document describes the complete mechanics of how injuries become permanent scars in RimWorld, based on the decompiled source code.

## Overview

RimWorld's permanent injury system allows wounds to become permanent scars that cannot heal naturally. These scars persist as a permanent reminder of significant trauma, affecting the character's appearance, pain levels, and overall health mechanics.

## Core Components

### 1. HediffComp_GetsPermanent Component

The `HediffComp_GetsPermanent` component is attached to injuries that can potentially become permanent. It manages:

- **Permanent State**: Whether the injury is currently permanent (`IsPermanent` property)
- **Permanent Damage Threshold**: The severity level at which the injury becomes permanent
- **Pain Category**: Random pain category assigned when injury becomes permanent

#### Key Fields:
```csharp
public float permanentDamageThreshold = 9999f;  // Default "never permanent" value
public bool isPermanentInt;                     // Internal permanent state
private PainCategory painCategory;              // Pain level (Painless, LowPain, MediumPain, HighPain)
```

### 2. Permanent Injury Calculation

#### Base Chance Formula
When an injury is finalized, the chance of becoming permanent is calculated as:

```
Permanent Chance = 0.02 × Part Factor × Component Factor × Severity Factor
```

Where:
- **Base Chance**: 2% (0.02f from `HealthTuning.BecomePermanentBaseChance`)
- **Part Factor**: `bodyPart.def.permanentInjuryChanceFactor` (varies by body part)
- **Component Factor**: `Props.becomePermanentChanceFactor` (from injury hediff definition)
- **Severity Factor**: Evaluated from `HealthTuning.BecomePermanentChanceFactorBySeverityCurve`

#### Severity Curve
The severity factor follows this curve:
- Severity 4 or less: 0× multiplier (no permanent injuries)
- Severity 14 or more: 1× multiplier (full chance)
- Between 4-14: Linear interpolation

#### Special Cases for Delicate Parts
For delicate body parts (eyes, brain, etc.):
- **No severity curve applied** - uses full base chance
- **Immediately becomes permanent** when chance succeeds
- No permanent damage threshold is set

#### Non-Delicate Parts Process
For non-delicate parts that succeed the permanent chance roll:
1. A permanent damage threshold is set: `Rand.Range(1f, currentSeverity / 2f)`
2. The injury is not immediately permanent
3. During healing, if severity reaches the threshold, it becomes permanent

### 3. Healing and Permanent Threshold

When an injury heals via `Heal()` method:

```csharp
public override void CompPostInjuryHeal(float amount)
{
    // Check if injury should become permanent during healing
    if (!(permanentDamageThreshold >= 9999f) && 
        !IsPermanent && 
        parent.Severity <= permanentDamageThreshold && 
        parent.Severity >= permanentDamageThreshold - amount)
    {
        parent.Severity = permanentDamageThreshold;  // Stop at threshold
        IsPermanent = true;                          // Make permanent
        base.Pawn.health.Notify_HediffChanged(parent);
    }
}
```

This means:
- Injuries heal down to their permanent threshold and stop
- The remaining severity becomes a permanent scar
- Further natural healing is prevented

### 4. Pain Category System

When an injury becomes permanent, it's assigned a random pain category:

| Pain Category | Numeric Value | Weight | Description |
|---------------|---------------|--------|-------------|
| Painless      | 0             | 0.5    | 50% chance - No pain |
| LowPain       | 1             | 0.2    | 20% chance - Minor discomfort |
| MediumPain    | 3             | 0.2    | 20% chance - Moderate pain |
| HighPain      | 6             | 0.1    | 10% chance - Severe pain |

The pain factor is used in calculating the injury's pain contribution:
```csharp
Pain = Severity × averagePainPerSeverityPermanent × PainFactor
```

### 5. Instant Permanent Injuries

Some damage sources can create instant permanent injuries:
- Set via `DamageInfo.InstantPermanentInjury = true`
- Bypasses all chance calculations
- Immediately marks injury as permanent
- Used for specific scenarios like surgical mishaps

### 6. Restrictions and Conditions

Permanent injuries cannot occur if:
- The body part has directly added prosthetics/implants
- The injury type doesn't support permanent injuries (missing `HediffComp_GetsPermanent`)
- The body part has `permanentInjuryChanceFactor` of 0

### 7. Age-Related Permanent Injuries

The `AgeInjuryUtility` generates permanent injuries for older characters:

#### Generation Parameters:
- **Age Threshold**: `lifeExpectancy / 8` (e.g., ~10 years for humans)
- **Chance per Age Period**: 15% for humanlike, 3% for animals
- **Max Age**: 1.5× life expectancy
- **Severity Range**: 2-6 damage for permanent scars

#### Process:
1. Calculate number of injuries based on age and chance
2. Select random outside body parts weighted by coverage
3. 30% chance for amputation, 70% for permanent scar
4. Generate appropriate damage type (Bullet, Scratch, Bite, Stab, Frostbite)

### 8. Permanent Injury Effects

#### Visual and UI Changes:
- **Label**: Uses `permanentLabel` or `instantlyPermanentLabel` from definition
- **Color**: Gray color (`Color(0.72f, 0.72f, 0.72f)`) in health tab
- **Pain Display**: Shows pain category in brackets (e.g., "High Pain")

#### Mechanical Effects:
- **No Health Impact**: Permanent injuries don't affect the health percentage summary
- **No Bleeding**: Permanent injuries never bleed (`BleedRate` returns 0)
- **Pain Contribution**: Uses different pain calculation with pain categories
- **Healing Prevention**: Cannot heal naturally or through tending
- **Merge Prevention**: Cannot merge with other injuries

### 9. Healing Permanent Wounds

The `HediffComp_HealPermanentWounds` component can heal permanent injuries:

#### Healing Mechanism:
- **Cycle Time**: 15-30 days (900,000-1,800,000 ticks)
- **Target Selection**: Random permanent injury or chronic condition
- **Healing Method**: Complete removal via `HealthUtility.CureHediff()`
- **Notification**: Displays message about permanent wound healing

This component is typically found on items like Healer Mechanites or special medical treatments.

## Configuration and Tuning

### Key Constants (from HealthTuning.cs):
```csharp
public const float BecomePermanentBaseChance = 0.02f;  // 2% base chance

// Severity curve points
BecomePermanentChanceFactorBySeverityCurve = new SimpleCurve {
    new CurvePoint(4f, 0f),   // No permanent injuries below severity 4
    new CurvePoint(14f, 1f)   // Full chance at severity 14+
};

// Pain category weights
InjuryPainCategories = new PainCategoryWeighted[] {
    new PainCategoryWeighted(PainCategory.Painless, 0.5f),
    new PainCategoryWeighted(PainCategory.LowPain, 0.2f),
    new PainCategoryWeighted(PainCategory.MediumPain, 0.2f),
    new PainCategoryWeighted(PainCategory.HighPain, 0.1f)
};
```

### Body Part Factors:
Each body part definition includes `permanentInjuryChanceFactor`:
- **0**: Cannot become permanently injured
- **0.1-0.5**: Low chance (extremities)
- **1.0**: Standard chance (torso, limbs)
- **2.0+**: High chance (delicate parts)

## Summary

The permanent scarring system in RimWorld creates a realistic wound progression where:
1. Severe injuries have a chance to leave permanent scars
2. The chance depends on injury severity, body part, and injury type
3. Scars stop healing at a threshold, creating lasting reminders
4. Pain levels vary randomly, affecting long-term quality of life
5. Special treatments can heal even permanent wounds
6. Age naturally accumulates permanent injuries over time

This system encourages careful medical treatment and adds long-term consequences to combat and accidents, making each injury potentially meaningful to a character's story.