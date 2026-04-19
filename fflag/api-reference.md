# fflag API Reference

Full reference for the `FastFlags` module: constructors, value API, system state, key paths, JSON mapping, and live-edit delivery.

## Setup

Works on both client and server:

```luau
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local FastFlags = require(ReplicatedStorage.UserGenerated.FastFlags)
local Asserts = require(ReplicatedStorage.UserGenerated.Lang.Asserts)
```

Flags must be registered on the server first. The server sets up replication, then clients can access replicated flags.

## Creating Flags

| Constructor | Visibility | Description |
|-------------|------------|-------------|
| `FastFlags.Private(key, assertion, default)` | Server-only | Cannot be seen or accessed on the client |
| `FastFlags.Replicated(key, assertion, default)` | Client + Server | Replicates to all clients. Server must register first. |

Both return a `Value` object (see Value API below).

## Single Instantiation Rule

Each key can only be registered **once per runtime**. Calling `Private` or `Replicated` with the same key twice on the same side (client or server) throws `ConflictingKeys` -- a hard, unrecoverable error.

Client and server are separate runtimes, so a `Replicated` flag is registered once on the server and once on the client (expected). But within the same runtime, a key must appear exactly once.

**Placement guidance:**

- **Replicated flags:** Place in a shared module under `ReplicatedStorage`. Both sides require the same module; `require` caching ensures single registration.
- **Private flags (one module):** Define directly in that module.
- **Private flags (multiple modules):** Define in a single location (the most related service). Other modules require it from there. Avoid cyclic dependencies -- extract into a dedicated flags module if needed.

## System State

| API | Description |
|-----|-------------|
| `FastFlags.IsLoaded()` | Returns `true` if initial values have loaded from the DataStore |
| `FastFlags.Loaded` | Event. Fires when the system loads. Use `:Wait()` or `:Connect()`. |

## Validation

| API | Description |
|-----|-------------|
| `FastFlags.IsA(value)` | Returns `true` if the value is a `FastFlags.Value` |
| `FastFlags.Assert(value)` | Asserts the value is a `FastFlags.Value`, returns it or errors |

## Value API

Once you have a flag value from `Private` or `Replicated`:

| Method / Property | Yields? | Description |
|-------------------|---------|-------------|
| `value:Get()` | No | Returns current value, or default if not loaded yet |
| `value:GetAsync()` | Yes | Yields until loaded from DataStore, then returns |
| `value:IsLoaded()` | No | Returns `true` if this flag has loaded |
| `value.Changed` | -- | Event `(current, previous)`. Fires on live config changes. |
| `value.Loaded` | -- | Event. Fires once when load attempt completes (even if key doesn't exist in DataStore). |
| `value.Key` | -- | The key string used at registration |
| `value.DefaultValue` | -- | The default value provided at registration |

**When to use which:**

- `:Get()` -- gameplay loops, event handlers, UI updates, anywhere yielding is unsafe
- `:GetAsync()` -- server startup, join flows, anywhere you need the real stored value before proceeding

## Default Values

The default is used when:

- The flag hasn't loaded from the DataStore yet
- The stored value fails assertion validation (falls back to default, logs a warning)
- No value exists in the DataStore

Default values can be any storable Luau type: string, number, boolean, or table. Tables are deep-copied and frozen.

## Key Paths and JSON Structure

Keys use dot notation that maps to nested JSON in the backend:

```luau
FastFlags.Private("UserGenerated.Saves.AutoSavePeriod", Asserts.FinitePositive, 600)
```

Maps to:

```json
{
  "UserGenerated": {
    "Saves": {
      "AutoSavePeriod": 600
    }
  }
}
```

**Supported value types:**

| Type | Lua | JSON |
|------|-----|------|
| Boolean | `true`/`false` | `true`/`false` |
| Number | `123`, `1.5` | `123`, `1.5` |
| String | `"text"` | `"text"` |
| Null | `nil` | `null` |
| Array | `{1, 2, 3}` | `[1, 2, 3]` |
| Object | `{a=1, b=2}` | `{"a": 1, "b": 2}` |

**Key conflicts:** Keys with hierarchical overlap error. You cannot have both a value and nested object at the same path (e.g., `UserGenerated.Saves` as a value AND `UserGenerated.Saves.Enabled` as nested).

## Editing Flags

Edit flags at **https://fflag.ug.xyz/**

| Method | Delivery | Use when |
|--------|----------|----------|
| **Immediate Push** (MessagingService) | Broadcast to all running servers within seconds | Urgent changes |
| **Slow Push** (DataStore) | Servers poll every ~15 minutes | Non-urgent, or when MessagingService quota is a concern |

## Assertions

For the full Asserts API, read the workspace's asserts rule at `.cursor/rules/usergenerated/asserts.mdc` (the exact location may vary by project layout; search for `asserts.mdc` if not found at that path).
