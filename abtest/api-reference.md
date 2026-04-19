# abtest API Reference

Full reference for `ABTests` module methods, events, and supported value types.

## Setup

Works on both client and server:

```luau
local ABTests = require(ReplicatedStorage.UserGenerated.ABTests)
```

## Player Attributes

| Method | Yields? | Description |
|--------|---------|-------------|
| `ABTests.GetAttribute(player, key, default)` | No | Returns current value or default if not loaded yet |
| `ABTests.GetAttributeAsync(player, key, default)` | Yes | Yields until attribute is loaded, then returns value or default |

Client can only read `LocalPlayer`'s attributes. Other players always return default.

## Job Attributes

Job attributes use the first player in the server (or `LocalPlayer` on client). All players on a server share the same value.

| Method | Yields? | Description |
|--------|---------|-------------|
| `ABTests.GetJobAttribute(key, default)` | No | Returns current value or default if not loaded yet |
| `ABTests.GetJobAttributeAsync(key, default)` | Yes | Yields until attribute is loaded, then returns value or default |

## System State

| API | Description |
|-----|-------------|
| `ABTests.IsLoaded()` | Returns `true` if the system has loaded initial values |

## Events

| Event | Fires when | Payload |
|-------|-----------|---------|
| `ABTests.Loaded` | System loads initial values | *(none)* |
| `ABTests.PlayerUpdated` | A player's attributes change at runtime | `(player: Player)` |
| `ABTests.JobUpdated` | Job-level attributes change at runtime | *(none)* |

## Default Values

Default values can be any Luau type:

| Type | Example |
|------|---------|
| String | `ABTests.GetAttribute(player, "Variant", "Control")` |
| Number | `ABTests.GetAttribute(player, "DamageMultiplier", 1.5)` |
| Boolean | `ABTests.GetAttribute(player, "NewFeature", false)` |
| Table | `ABTests.GetAttribute(player, "ShopPrices", { sword = 100, shield = 50 })` |

## Notes

- **Async vs sync:** Async methods yield until loaded. Use async for join flows, sync for gameplay loops and event handlers.
- **Client limitation:** Client can only read `LocalPlayer`'s attributes. Other players always return default.
- **How it works:** You implement the attribute reads in your code. Backend handles all test configuration (groups, percentages, player/server scoping, scheduling). Once your code is merged, backend configures the test.
