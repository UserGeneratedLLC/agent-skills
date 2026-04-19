---
name: abtest
description: Implement AB test attributes in game code. Use when the user asks to add an AB test, make a value configurable via AB tests, add remote configurability, or wire up ABTests.GetAttribute / GetJobAttribute calls.
---

# Implementing AB Tests

This skill guides you through adding AB test attributes to the codebase. See the API Reference at the end for method signatures and events.

## 1. Decide: Player-Level vs Job-Level

| Scope | API | Assigned by | Use when |
|-------|-----|-------------|----------|
| **Player** | `GetAttribute` / `GetAttributeAsync` | Per-player (UserId) | Different players should see different values: tutorial thresholds, UI variants, per-player multipliers |
| **Job** | `GetJobAttribute` / `GetJobAttributeAsync` | Per-server (JobId) | All players on a server share the same value: event parameters, feature toggles, wave configs, spawn rates |

Job-level tests use a `JOBID-` prefix internally -- you don't need to add this yourself; backend handles it based on test config.

## 2. Name the Attribute Key

Use dot-separated hierarchical names: `{System}.{Feature}.{Parameter}`

### Naming rules

- PascalCase each segment
- Boolean toggles: end with `.Enabled` (e.g., `Eco.DiversityBonus.Enabled`)
- Numeric tunables: name the parameter directly (e.g., `Event.Money.StormDuration`)
- String variants: name the choice (e.g., `FTU.TutorialGroup`)

### Real examples from the codebase

```
-- Job-level (server-wide)
Event.Money.StormDuration          -- number (seconds)
Event.FireAndIce.MutationChance    -- number (0-1)
Wave.MinGapStudsMin                -- number
GenRate.ShowOverhead               -- boolean
AB.HideCarryStand                  -- boolean
LuckyBlock.NaturalSpawn.Enabled    -- boolean

-- Player-level (per-player)
FTU.SessionCountThreshold          -- number
Camera.ZoomMultiplier              -- number
Eco.DiversityBonus.Enabled         -- boolean
FriendRequest.Enabled              -- boolean
Steal.Common                       -- number (weight)
```

## 3. Sync vs Async

| Variant | Yields? | Use in |
|---------|---------|--------|
| `GetAttribute` / `GetJobAttribute` | No | Gameplay loops, event handlers, UI updates, anywhere yielding is unsafe |
| `GetAttributeAsync` / `GetJobAttributeAsync` | Yes | `playerAdded`, join flows, `init()` where you need the value before proceeding |

**Rule:** Default to sync. Only use async when you explicitly need to block until the value is available (e.g., deciding whether to teleport a player on join).

## 4. Implementation Workflow

### Step 1: Add the import

```luau
local ABTests = require(ReplicatedStorage.UserGenerated.ABTests)
```

Place it in the `-- Imports` section of the service/controller.

### Step 2: Replace hardcoded values

Find the hardcoded value and replace it with an ABTests call. **Always use the current hardcoded value as the default** so behavior is unchanged without backend config.

Before:
```luau
local STORM_DURATION = 30
```

After:
```luau
local stormDuration = ABTests.GetJobAttribute("Event.Money.StormDuration", 30)
```

### Step 3: Add reactive updates (client only, if needed)

If the value can change at runtime via live config updates, listen for changes. Which event depends on scope:

- **Player-level:** `ABTests.PlayerUpdated`
- **Job-level:** `ABTests.JobUpdated`

### Step 4: Add condition callbacks (if needed)

If the test needs custom targeting conditions (beyond what backend can do with built-in conditions), add a callback to `ABTestConditions.server.luau`. See section 6.

## 5. Code Templates

Pick a template from [templates.md](templates.md) matching your scope (player vs job) and context. Covers: server service (sync/async), client controller (player-level with reactive updates, job-level, blocking init), and event module with multiple tunables.

## 6. Adding Condition Callbacks

Condition callbacks let the backend target tests to specific server/player states. Add them to `src/ServerScriptService/UGApp/ABTestConditions.server.luau`.

### Structure

The file exports a module table of callbacks. All callbacks are auto-registered at the bottom via a loop:

```luau
for name, func in pairs(module) do
    if type(name) == "string" and type(func) == "function" then
        ServerABTests.RegisterConditionCallback(name, func)
    end
end
```

### Template for a new callback

```luau
--[[
    Description of what this condition checks.
    @param player Required/Optional
    @param arg1 Description
    @return true if condition is met
]]
function module.MyConditionName(
    player: Player?,
    arg1: number,
    arg2: number?
): boolean
    assert(player)
    Asserts.Player(player)
    Asserts.IntegerNonNegative(arg1)
    if arg2 ~= nil then
        Asserts.IntegerNonNegative(arg2)
    end
    -- logic here
    return result
end
```

### Existing callbacks (for reference)

| Callback | Purpose |
|----------|---------|
| `TestRecentPurchases` | Purchase count in last N days within [min, max] |
| `TestRobuxSpend` | Total Robux spent in last N days within [min, max] |
| `TestSessionCount` | Lifetime session count within [min, max] |
| `TestPlaceId` | `game.PlaceId` in allowed list |
| `IsPrivateServer` | `game.PrivateServerId ~= ""` |
| `IsPrivateServerOwner` | Private server + player is owner |
| `TestServerSize` | Player count within [min, max] |
| `TestPlaceVersion` | `game.PlaceVersion` meets minimum per-place |
| `IsFTUServer` | `TPService.isFTU()` |

### Key rules

- Type signature: `(player: Player?, ...any) -> boolean`
- Always `assert(player)` if the condition requires a player
- Validate all arguments with `Asserts` at the top
- For range checks: `min` inclusive, `max` inclusive and optional
- Player conditions can access profile data via `Profiles.GetAsync(player, true)`
- Server conditions (no player needed) still receive `player: Player?` parameter

## 7. Best Practices

1. **Default = current behavior.** The default value in `GetAttribute`/`GetJobAttribute` must match the current hardcoded value. This ensures no behavior change without backend config.

2. **Validate returned types.** Backend can send unexpected types. Guard booleans and numbers:
   ```luau
   local val = ABTests.GetJobAttribute("Key", true)
   if type(val) ~= "boolean" then val = true end
   ```

3. **Don't yield in hot paths.** Use sync `GetAttribute`/`GetJobAttribute` in gameplay loops, event handlers, and render callbacks. These return the default if not loaded yet.

4. **Comment the ABTest attributes.** At the top of the file or above the function, document what attributes are read and their types/defaults:
   ```luau
   -- ABTest Attributes:
   -- - Camera.ZoomMultiplier: number (default 1.3)
   ```

5. **Backend handles test configuration.** You implement the attribute reads. Backend configures groups, percentages, conditions, and scheduling. Once your code is merged, backend sets up the test.

6. **One attribute per tunable.** Don't pack multiple values into a single table attribute when they could be independent. `Event.Money.SpawnRate` and `Event.Money.MaxCap` are better than `Event.Money.Config = {spawnRate=0.5, maxCap=100}`.

## 8. API Reference

See [api-reference.md](api-reference.md) for the full API — method signatures for `GetAttribute`/`GetJobAttribute` (sync and async), system state accessors, events (`Loaded`, `PlayerUpdated`, `JobUpdated`), supported default-value types, and notes on client/server semantics.
