# zSkyridingBar Refactor & Improvement Plan

This document outlines recommended architectural and logical improvements for the `zSkyridingBar` codebase to enhance maintainability, performance, and readability.

## 1. Refactor Monolithic Ability Logic
**Target:** `UpdateStaticChargeAndWhirlingSurge` (Lines ~1050-1280)

**Issues:**
*   **Complexity:** Handles three distinct states (Static Charge stacks, Lightning Rush cooldown, Whirling Surge cooldown) in one 230-line block.
*   **Code Duplication:** The "Reverse Fill" and "Shine" animation logic is repeated for both Lightning Rush and Whirling Surge.
*   **Readability:** Deeply nested conditionals make the priority logic hard to follow.

**Proposed Solution:**
Implement a prioritized orchestrator calling focused helper functions:
```lua
function zSkyridingBar:UpdateAbilityWidget()
    if not speedAbilityFrame then return end
    
    -- Priority 1: Active Buff Stacks
    if self:UpdateStaticChargeState() then return end
    
    -- Priority 2: Lightning Rush Cooldown
    if self:UpdateLightningRushState() then return end
    
    -- Priority 3: Whirling Surge Cooldown
    self:UpdateWhirlingSurgeState()
end
```
*   Create a shared helper `TriggerAbilityShine(icon)` to unify animation code.
*   Create a shared helper `UpdateCooldownOverlay(startTime, duration)` to handle the reverse-fill math once.

## 2. Generic Frame Construction
**Target:** `CreateSpeedBarFrame`, `CreateChargesBarFrame`, `CreateSecondWindFrame`

**Issues:**
*   High boilerplate duplication for basic frame setup (Movable, Strata, Scaling).
*   Inconsistent border creation logic.

**Proposed Solution:**
Create a `zSkyridingBar:CreateBaseFrame(name, parent, width, height)` helper that handles:
*   Standard AceDB-linked positioning.
*   Registration for `createMoveableFrameHeader`.
*   Standard strata and scale application.

## 3. Event Handler Modularization
**Target:** `eventFrame:SetScript("OnEvent", ...)`

**Issues:**
*   A single anonymous function dispatches all events, making it difficult to trace specific logic flows.

**Proposed Solution:**
Move logic into named methods:
```lua
function zSkyridingBar:OnUnitPowerUpdate(unit, powerType) ... end
function zSkyridingBar:OnSpellCastSucceeded(unit, castGUID, spellID) ... end
```
The dispatcher becomes a simple lookup:
```lua
eventFrame:SetScript("OnEvent", function(_, event, ...)
    if zSkyridingBar[event] then
        zSkyridingBar[event](zSkyridingBar, ...)
    end
end)
```

## 4. Performance Optimizations
*   **Variable Caching:** Localize more frequently used API calls (like `C_Spell.GetSpellCooldown`) outside the `UpdateTracking` loop.
*   **Update Throttling:** Some UI elements (like Second Wind) do not need to update at 20fps (`TICK_RATE`). Move them to event-driven updates or a slower timer.
*   **Animation Efficiency:** Modify `AnimateStatusBar` to reuse a single `OnUpdate` script reference rather than redefining a closure every time an animation starts.
