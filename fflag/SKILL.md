---
name: fflag
description: Create and set up FastFlags (feature flags / remote config) in the codebase. Use when the user says /fflag, asks to add a fastflag, feature flag, fflag, replicated flag, private flag, or remote config value.
---

# FastFlag Setup

Create new FastFlags with correct placement, key naming, assertions, and consumption wiring.

## Workflow

### Step 1: Gather Requirements

Ask the user (use AskQuestion when available):

1. **What is the flag for?** (feature toggle, numeric tuning, config map, etc.)
2. **Does the client need it?** Determines Replicated vs Private.
3. **Default value?** What the flag returns before the DataStore loads or if no value is set.

If the user already provided this context (e.g., as part of a feature request), infer the answers and skip asking.

### Step 2: Determine Type and Placement

```
Does the client need this value?
├─ YES → FastFlags.Replicated
│   ├─ New system / multiple flags? → New file in Shared/Flags/<SystemName>Flags.luau
│   ├─ Fits an existing flags file? → Add to that file
│   └─ Tied to existing config module? → Add to that config module
│
└─ NO → FastFlags.Private (server-only)
    ├─ Used by one service/module? → Define inline in that module
    └─ Used by multiple server modules? → Define in the most related service, require from there
```

**Critical:** Each key can only be registered ONCE per runtime. Never define the same flag in two places. Shared modules under ReplicatedStorage are safe because `require` caches results.

### Step 3: Choose the Key Path

Keys use dot-separated hierarchy. Match existing patterns:

| Prefix | Use for | Examples |
|--------|---------|----------|
| `Game.<System>.<Flag>` | Gameplay features | `Game.LimitedShop.Enabled2`, `Game.Geyser.PadNerfedSpeed` |
| `Trading.<Flag>` | Trading system | `Trading.SendTradeRequestsDisabled` |
| `Event.<Event>.<Flag>` | Event-scoped config | `Event.FireAndIce.MutationWeight` |
| `Tower.<Flag>` | Tower system | `Tower.CooldownDuration`, `Tower.RitualEnabled` |
| `Announcements.<Feature>.<Flag>` | Announcement toggles | `Announcements.DivineObtain.Enabled` |
| `UserGenerated.<System>.<Flag>` | Platform/infra | `UserGenerated.Analytics.PlayerEventChances2` |

Rules:
- No overlapping hierarchy (can't have both `Game.Saves` as a value AND `Game.Saves.Enabled` as nested)
- Use PascalCase for each segment
- Be specific -- `Game.Geyser.PassiveSpawnDelay` not `Game.SpawnDelay`

### Step 4: Select the Assertion

Pick from this table based on the value type:

| Value type | Assertion | Notes |
|------------|-----------|-------|
| On/off toggle | `Asserts.Boolean` | |
| Text / ID | `Asserts.String` | |
| Any positive number | `Asserts.FinitePositive` | Rates, multipliers, durations |
| Non-negative number | `Asserts.FiniteNonNegative` | Weights, chances (allows 0) |
| Percentage 0-1 | `Asserts.Range(0, 1)` | |
| Percentage 0-100 | `Asserts.Range(0, 100)` | |
| Positive integer | `Asserts.IntegerPositive` | Counts, levels, IDs |
| Non-negative integer | `Asserts.IntegerNonNegative` | |
| Integer in range | `Asserts.IntegerRange(a, b)` | |
| Enum-like string | `Asserts.Set({"a", "b", "c"})` | |
| Optional value | `Asserts.Optional(inner)` | Value or nil |
| String-keyed map | `Asserts.Map(Asserts.String, valueAssert)` | Config dictionaries |
| Array | `Asserts.Array(valueAssert)` | Lists |
| Structured object | `Asserts.Table({ key = assert })` | Complex configs |
| Multiple valid types | `Asserts.AnyOf(a, b, ...)` | |

For complex table values, define the assert as a local variable above the flag (see MysteryMerchantDynamicStockFlags for a full example).

### Step 5: Write the Flag

Use the appropriate template below.

---

## Templates

### Shared Replicated Flags File

Create at `src/ReplicatedStorage/Shared/Flags/<SystemName>Flags.luau`:

```luau
--!strict
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local FastFlags = require(ReplicatedStorage.UserGenerated.FastFlags)
local Asserts = require(ReplicatedStorage.UserGenerated.Lang.Asserts)

local FeatureEnabled = FastFlags.Replicated(
	"Game.System.FeatureEnabled",
	Asserts.Boolean,
	true
)

local TuningValue = FastFlags.Replicated(
	"Game.System.TuningValue",
	Asserts.FinitePositive,
	5.0
)

return table.freeze({
	FeatureEnabled = FeatureEnabled,
	TuningValue = TuningValue,
})
```

### Inline Private Flags (in a service)

Place under a `-- FFlags` or `-- Feature Flags` section comment, before Constants:

```luau
-- FFlags
local FeatureDisabled = FastFlags.Private("System.FeatureDisabled", Asserts.Boolean, false)
```

Requires at top of the service:

```luau
local FastFlags = require(ReplicatedStorage.UserGenerated.FastFlags)
local Asserts = require(ReplicatedStorage.UserGenerated.Lang.Asserts)
```

### Adding to an Existing Flags File

1. Add the local variable with `FastFlags.Replicated(...)` or `FastFlags.Private(...)`
2. Add it to the return table

---

## Consumption Patterns

### Get (non-yielding, use in gameplay)

```luau
if MyFlags.FeatureEnabled:Get() then
	-- feature is on
end

local rate = MyFlags.TuningValue:Get()
```

### GetAsync (yielding, use at startup)

```luau
local value = MyFlags.FeatureEnabled:GetAsync()
```

Only use when you MUST have the real DataStore value before continuing.

### Changed (react to live updates)

```luau
MyFlags.TuningValue.Changed:Connect(function(current, previous)
	updateSomething(current)
end)
```

### Loaded (wait for initial load)

```luau
if not FastFlags.IsLoaded() then
	FastFlags.Loaded:Wait()
end
```

Or per-flag:

```luau
MyFlags.FeatureEnabled.Loaded:Wait()
```

### Typical controller/service pattern

```luau
local MyFlags = require(ReplicatedStorage.Shared.Flags.MyFlags)

function MyService.init()
	-- Use :Get() in gameplay, react with .Changed
	if MyFlags.FeatureEnabled:Get() then
		setupFeature()
	end

	MyFlags.FeatureEnabled.Changed:Connect(function(current)
		if current then
			setupFeature()
		else
			teardownFeature()
		end
	end)
end
```

---

## Checklist

Before finishing, verify:

- [ ] Key path follows existing naming conventions (check Step 3)
- [ ] Assertion matches the value type (check Step 4)
- [ ] Default value passes the assertion
- [ ] Flag is only registered once per runtime (no duplicate definitions)
- [ ] Replicated flags are in a shared module under ReplicatedStorage
- [ ] Private flags used by multiple modules live in one place, required by others
- [ ] `FastFlags` and `Asserts` imports are present in the file
- [ ] Flag is added to the frozen return table (for shared flag files)
- [ ] Consumption code uses `:Get()` (not `:GetAsync()`) in gameplay paths

## API Reference

See [api-reference.md](api-reference.md) for the full API: `FastFlags.Private`/`FastFlags.Replicated` constructors, the single-instantiation rule and placement guidance, system state (`IsLoaded`, `Loaded`), validation (`IsA`, `Assert`), the `Value` API (`:Get()`, `:GetAsync()`, `.Changed`, `.Loaded`, `.Key`, `.DefaultValue`), default-value semantics, key-path → JSON mapping, supported value types, and live-edit delivery via https://fflag.ug.xyz/.

For the full Asserts API, read the workspace's asserts rule at `.cursor/rules/usergenerated/asserts.mdc` (the exact location may vary by project layout; search for `asserts.mdc` if not found at that path).
