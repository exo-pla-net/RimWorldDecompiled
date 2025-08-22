# Code Validation for Permanent Scarring Mechanics Documentation

This file validates the accuracy of the permanent scarring mechanics documentation by showing the actual code that implements each feature.

## 1. Base Permanent Injury Chance Calculation

**Documentation Claims**: Base chance is 2% × Part Factor × Component Factor × Severity Factor

**Actual Code** (`HediffComp_GetsPermanent.PreFinalizeInjury()`):
```csharp
float num = 0.02f * parent.Part.def.permanentInjuryChanceFactor * Props.becomePermanentChanceFactor;
if (!parent.Part.def.delicate)
{
    num *= HealthTuning.BecomePermanentChanceFactorBySeverityCurve.Evaluate(parent.Severity);
}
if (Rand.Chance(num))
{
    // Make permanent logic...
}
```

✅ **VALIDATED**: Formula matches exactly.

## 2. Severity Curve Configuration

**Documentation Claims**: Severity 4→0×, Severity 14→1×, linear between

**Actual Code** (`HealthTuning.cs`):
```csharp
public static readonly SimpleCurve BecomePermanentChanceFactorBySeverityCurve = new SimpleCurve
{
    new CurvePoint(4f, 0f),
    new CurvePoint(14f, 1f)
};
```

✅ **VALIDATED**: Curve points match exactly.

## 3. Permanent Damage Threshold System

**Documentation Claims**: Non-delicate parts set threshold = Rand.Range(1f, severity/2f)

**Actual Code** (`HediffComp_GetsPermanent.PreFinalizeInjury()`):
```csharp
if (parent.Part.def.delicate)
{
    IsPermanent = true;
}
else
{
    permanentDamageThreshold = Rand.Range(1f, parent.Severity / 2f);
}
```

✅ **VALIDATED**: Logic matches exactly.

## 4. Healing Threshold Mechanics

**Documentation Claims**: Healing stops at threshold and makes injury permanent

**Actual Code** (`HediffComp_GetsPermanent.CompPostInjuryHeal()`):
```csharp
if (!(permanentDamageThreshold >= 9999f) && !IsPermanent && 
    parent.Severity <= permanentDamageThreshold && 
    parent.Severity >= permanentDamageThreshold - amount)
{
    parent.Severity = permanentDamageThreshold;
    IsPermanent = true;
    base.Pawn.health.Notify_HediffChanged(parent);
}
```

✅ **VALIDATED**: Threshold logic matches exactly.

## 5. Pain Category System

**Documentation Claims**: 4 categories with specific weights and values

**Actual Code** (`HealthTuning.cs`):
```csharp
public static readonly PainCategoryWeighted[] InjuryPainCategories = new PainCategoryWeighted[4]
{
    new PainCategoryWeighted(PainCategory.Painless, 0.5f),
    new PainCategoryWeighted(PainCategory.LowPain, 0.2f),
    new PainCategoryWeighted(PainCategory.MediumPain, 0.2f),
    new PainCategoryWeighted(PainCategory.HighPain, 0.1f)
};
```

**Pain Category Values** (`PainCategory.cs`):
```csharp
public enum PainCategory
{
    Painless = 0,
    LowPain = 1,
    MediumPain = 3,
    HighPain = 6
}
```

✅ **VALIDATED**: All categories and weights match.

## 6. Pain Calculation for Permanent Injuries

**Documentation Claims**: Pain = Severity × averagePainPerSeverityPermanent × PainFactor

**Actual Code** (`Hediff_Injury.PainOffset`):
```csharp
HediffComp_GetsPermanent hediffComp_GetsPermanent = this.TryGetComp<HediffComp_GetsPermanent>();
if (hediffComp_GetsPermanent != null && hediffComp_GetsPermanent.IsPermanent)
{
    return Severity * def.injuryProps.averagePainPerSeverityPermanent * hediffComp_GetsPermanent.PainFactor;
}
return Severity * def.injuryProps.painPerSeverity;
```

✅ **VALIDATED**: Pain calculation matches exactly.

## 7. Permanent Injuries Don't Bleed

**Documentation Claims**: Permanent injuries have BleedRate of 0

**Actual Code** (`Hediff_Injury.BleedRate`):
```csharp
if (base.Part.def.IsSolid(base.Part, pawn.health.hediffSet.hediffs) || 
    this.IsTended() || 
    this.IsPermanent())
{
    return 0f;
}
```

✅ **VALIDATED**: IsPermanent() causes 0 bleed rate.

## 8. Permanent Injuries Don't Affect Health Summary

**Documentation Claims**: Permanent injuries return 0 for health impact

**Actual Code** (`Hediff_Injury.SummaryHealthPercentImpact`):
```csharp
if (this.IsPermanent() || !Visible)
{
    return 0f;
}
return Severity / (75f * pawn.HealthScale);
```

✅ **VALIDATED**: IsPermanent() causes 0 health impact.

## 9. Instant Permanent Injury Logic

**Documentation Claims**: InstantPermanentInjury bypasses chance calculation

**Actual Code** (`DamageWorker_AddInjury.FinalizeAndAddInjury()`):
```csharp
if (dinfo.InstantPermanentInjury)
{
    HediffComp_GetsPermanent hediffComp_GetsPermanent = hediff_Injury.TryGetComp<HediffComp_GetsPermanent>();
    if (hediffComp_GetsPermanent != null)
    {
        hediffComp_GetsPermanent.IsPermanent = true;
    }
}
```

✅ **VALIDATED**: Instant permanent bypasses all chance logic.

## 10. Age-Related Permanent Injury Generation

**Documentation Claims**: 15% chance per age period for humanlike, 3% for animals

**Actual Code** (`AgeInjuryUtility.GenerateRandomOldAgeInjuries()`):
```csharp
float chance = (pawn.RaceProps.Humanlike ? 0.15f : 0.03f);
for (float num4 = num2; num4 < Mathf.Min(pawn.ageTracker.AgeBiologicalYears, b); num4 += num2)
{
    if (Rand.Chance(chance))
    {
        num3++;
    }
}
```

**Severity Range**:
```csharp
hediff_Injury.Severity = Rand.RangeInclusive(2, 6);
hediff_Injury.TryGetComp<HediffComp_GetsPermanent>().IsPermanent = true;
```

✅ **VALIDATED**: Age mechanics and severity range (2-6) match exactly.

## 11. Heal Permanent Wounds Component

**Documentation Claims**: 15-30 day cycle, heals random permanent/chronic conditions

**Actual Code** (`HediffComp_HealPermanentWounds`):
```csharp
private void ResetTicksToHeal()
{
    ticksToHeal = Rand.Range(15, 30) * 60000;  // 15-30 days
}

private void TryHealRandomPermanentWound()
{
    if (base.Pawn.health.hediffSet.hediffs.Where((Hediff hd) => hd.IsPermanent() || hd.def.chronic).TryRandomElement(out var result))
    {
        HealthUtility.CureHediff(result);
        // Show notification...
    }
}
```

✅ **VALIDATED**: Cycle time and healing target selection match exactly.

## 12. IsPermanent Extension Method

**Documentation Claims**: Extension method checks for HediffComp_GetsPermanent component

**Actual Code** (`HediffUtility.cs`):
```csharp
public static bool IsPermanent(this Hediff hd)
{
    HediffWithComps hediffWithComps = hd as HediffWithComps;
    if (hediffWithComps == null)
    {
        return false;
    }
    return hediffWithComps.TryGetComp<HediffComp_GetsPermanent>()?.IsPermanent ?? false;
}
```

✅ **VALIDATED**: Extension method implementation matches exactly.

## Overall Validation Result

✅ **ALL CLAIMS VALIDATED**: Every major mechanism described in the documentation has been verified against the actual source code. The documentation accurately reflects the implementation of permanent scarring mechanics in RimWorld.