# Common Mistakes

## Requiring from non-server code

Analytics is server-only. Requiring it from ReplicatedStorage shared modules or client code will error.

```luau
-- WRONG: shared module in ReplicatedStorage could be required by client
-- ReplicatedStorage/Shared/SomeModule.luau
local Analytics = require(ServerScriptService.UserGenerated.Analytics)

-- CORRECT: only require from ServerScriptService or ServerStorage modules
-- ServerScriptService/Server/Services/SomeService.luau
local ServerScriptService = game:GetService("ServerScriptService")
local Analytics = require(ServerScriptService.UserGenerated.Analytics)
```

## Requiring server scripts instead of ModuleScripts

Only ModuleScripts (`.luau`, `init.luau`) can be required. Server scripts (`.server.luau`) and client scripts (`.client.luau`) cannot.

```luau
-- WRONG: .server.luau cannot be required
local GameHandler = require(ServerScriptService.ServerMain.init) -- init.server.luau = ERROR

-- CORRECT: require the ModuleScript
local GameHandler = require(ServerScriptService.ServerMain.GameHandler) -- init.luau = OK
```

## Creating new tracking instead of using existing state

Always search the codebase first. If data is already tracked somewhere, use it.

```luau
-- WRONG: creating redundant state
local nightIndex = 0
local function setNight()
    nightIndex += 1
end

-- CORRECT: use what the game already tracks
results["Shared.TimeString"] = GetValue(runtime:FindFirstChild("TimeString"), "")
```

## Putting state data in events instead of attribute providers

If data would be useful context for any event, it's state and belongs in the attribute provider.

```luau
-- WRONG: health is state, not event-specific
Analytics:LogPlayerEvent(player, "PlayerTookDamage", {
    DamageAmount = damage,
    HealthAfter = humanoid.Health,
    DamageSource = source,
})

-- CORRECT: health goes in BuildGeneralAttributesAsync, event only has event-specific data
-- In attribute provider:
results["Shared.Health"] = humanoid.Health

-- In event:
Analytics:LogPlayerEvent(player, "PlayerTookDamage", {
    DamageAmount = damage,
    DamageSource = source,
})
```

## Ignoring lobby vs game server

Game-specific state (round duration, wave number, etc.) may not exist on the lobby server. Always guard.

```luau
-- WRONG: errors on lobby server where RoundStartTime doesn't exist
results["Shared.RoundDuration"] = os.clock() - GameHandler.RoundStartTime

-- CORRECT: check existence first
if GameHandler.RoundStartTime then
    results["Shared.RoundDuration"] = math.round(os.clock() - GameHandler.RoundStartTime)
end
```

## Over-using task.spawn

Only use `task.spawn` when multiple steps could error. Simple analytics calls don't need it.

```luau
-- WRONG: unnecessary wrapper
task.spawn(function()
    Analytics:IncrementPlayerAttribute(player, "TotalDigEvents", 1)
end)

-- CORRECT: call directly
Analytics:IncrementPlayerAttribute(player, "TotalDigEvents", 1)
```

`task.spawn` IS appropriate when you need to compute multiple values that could fail:

```luau
-- CORRECT: complex logic that could error at multiple points
task.spawn(function()
    local itemInstance = Assets.DigItems:FindFirstChild(itemObject.itemName)
    local itemRarity = itemInstance and itemInstance:GetAttribute("Rarity") or "Unknown"
    local pickupDepth = 0
    if player.Character and player.Character:GetAttribute("InDigZone") then
        local hrp = player.Character:FindFirstChild("HumanoidRootPart")
        if hrp and digZone.DIG_ZONE_FOLDER then
            local terrainBox = digZone.DIG_ZONE_FOLDER.TerrainBoundingBox
            local startPos = (terrainBox.CFrame * CFrame.new(0, terrainBox.Size.Y / 2, 0)).Position
            local progressDirection = -terrainBox.CFrame.UpVector
            pickupDepth = math.max(0, utils.getDirectedDistance(startPos, hrp.Position, progressDirection))
        end
    end
    Analytics:LogPlayerEvent(player, "PlayerItemPickup", {
        ItemName = itemObject.itemName,
        ItemRarity = itemRarity,
        PickupDepth = pickupDepth,
    })
end)
```

## Tracking unique players incorrectly

Use a set keyed by UserId, not a count of current players.

```luau
-- WRONG: counts current players, not unique players who joined
handler.PlayersStarted = #game.Players:GetPlayers()

-- CORRECT: track unique players via set
local UniquePlayerIds: {[number]: true} = {}
local UniquePlayerCount = 0

function handler.playerAdded(player: Player)
    if not UniquePlayerIds[player.UserId] then
        UniquePlayerIds[player.UserId] = true
        UniquePlayerCount += 1
        handler.PlayersStarted = UniquePlayerCount
    end
end
```

## Unpacking structured data into flat event keys

Events accept nested tables natively. Pass item data, trade payloads, reward structs directly.

```luau
-- WRONG: manually unpacking into flat keys
Analytics:LogPlayerEvent(player, "PlayerItemPickup", {
    ItemName = itemData.Name,
    ItemRarity = itemData.Rarity,
    ItemLevel = itemData.Level,
})

-- CORRECT: pass the struct directly
Analytics:LogPlayerEvent(player, "PlayerItemPickup", {
    Item = itemData,
})
```

This also applies to trade events, reward structs, inventory snapshots, etc. The analytics backend handles nested tables.

## Using unverified attributes

Every attribute you reference must be verified by searching the codebase first. If you can't find where it's set, don't use it.

Verification steps:
1. Search for the attribute name across `src/`
2. Confirm it's set/updated somewhere (look for `SetAttribute`, `.Value =`, or assignment)
3. Confirm the type (ValueBase needs `.Value`, Instance Attribute needs `:GetAttribute()`)
4. Confirm availability (lobby server vs game server)

## Passing raw IDs instead of resolved item data

When events involve exchanges, trades, or transfers where items are looked up by ID, the actual item data must be captured BEFORE the exchange loop deletes it. Passing raw ID maps is useless to the analytics team.

```luau
-- WRONG: raw bytes codes, nobody can read these
Analytics:LogPlayerEvent(player, "PlayerTradeCompleted", {
    Given = { Slot1 = "abc-123", Slot2 = "def-456" },
    Received = { Slot1 = "ghi-789" },
})

-- CORRECT: resolve IDs to actual item data BEFORE the exchange loop
local givenItems = {}
for slot, bytes in pairs(tradeOffer) do
    givenItems[slot] = StoredBrainrots[bytes] -- capture before nil
end
-- ... exchange loop runs, nils StoredBrainrots[bytes] ...
Analytics:LogPlayerEvent(player, "PlayerTradeCompleted", {
    Given = givenItems,
    Received = receivedItems,
})
```

## Logging death events without a cause

`PlayerDeath` with empty data `{}` is nearly useless. Always include a `Cause` field. Check player attributes set by game systems (e.g. `Slapped`, `InLava`, `Poisoned`) and zone/location info. At minimum distinguish PvP vs Environment.

```luau
-- WRONG: no context about what killed the player
Analytics:LogPlayerEvent(player, "PlayerDeath", {})

-- CORRECT: check game state for cause
Analytics:LogPlayerEvent(player, "PlayerDeath", {
    Cause = if player:GetAttribute("Slapped") then "PvP" else "Environment",
    AtZone = player:GetAttribute("AtZone"),
})
```

## Counter-only tracking without a corresponding event

A counter like `TotalItemsCollected` tells you HOW MANY but not WHAT. Every meaningful collection, pickup, or acquisition needs BOTH a counter AND an event with the item data.

```luau
-- WRONG: only a counter, no way to know WHAT was collected
Analytics:IncrementPlayerAttribute(player, "TotalBrainrotsCollected", 1)

-- CORRECT: counter + event with the item data
Analytics:IncrementPlayerAttribute(player, "TotalBrainrotsCollected", 1)
Analytics:LogPlayerEvent(player, "PlayerBrainrotPickup", {
    Brainrot = brainrotData, -- pass the struct directly
})
```

## Not verifying pre-installed Profiles.luau requires

Profiles.luau may already exist from a package install with placeholder require paths (e.g. `DataModules.ProfileTemplate`, `Server.Services.DataService`). These paths almost never resolve in the actual game. Always verify every `require()` in Profiles.luau points to a real module in the filesystem.

```luau
-- WRONG: blindly trusting existing requires
local ProfileTemplate = require(ServerScriptService.DataModules.ProfileTemplate) -- DOES NOT EXIST
local DataService = require(ServerScriptService.Server.Services.DataService)     -- DOES NOT EXIST

-- CORRECT: verify paths resolve, wire to the actual data module
local DataManager = require(ServerScriptService.Data.DataManager) -- actually exists
```

## Manually defining Profile type instead of using the library's generic

For ProfileStore/ProfileService games, the Profile type MUST use the library's exported generic, not a manually defined struct. A manual struct misses methods, signals, and metadata fields.

```luau
-- WRONG: manual struct misses OnSessionEnd, EndSession, Reconcile, etc.
export type Profile = {
    Data: Data,
    FirstSessionTime: number,
    SessionLoadCount: number,
}

-- CORRECT: use the library's generic type
export type Profile = ProfileStore.Profile<Data>
```

## Using `any` in Bindable event type parameters

The OnAdded/OnReleasing Bindable events must use the actual `Profile` type, not `any`. Using `any` defeats type checking in Setup.server.luau, Profiles.luau, and all downstream listeners.

```luau
-- WRONG: any kills type checking
DataManager.OnAdded = Bindable.new() :: Bindable.Event<Player, any>

-- CORRECT: use the actual Profile type
DataManager.OnAdded = Bindable.new() :: Bindable.Event<Player, Profile>
```

## Firing OnReleasing from multiple locations

OnReleasing must fire from exactly ONE place: the library's release callback (`OnSessionEnd` for ProfileStore, `ListenToRelease` for ProfileService). Do NOT also fire it from `PlayerRemoving`. When `PlayerRemoving` calls `EndSession()`, that triggers `OnSessionEnd` which fires OnReleasing. Adding a second fire in `PlayerRemoving` causes it to fire twice on normal leave.

```luau
-- WRONG: fires OnReleasing twice on normal leave
Players.PlayerRemoving:Connect(function(player)
    DataManager.OnReleasing:Fire(player, profile)  -- fires here
    profile:EndSession()  -- triggers OnSessionEnd which fires it AGAIN
end)

-- CORRECT: only fire in OnSessionEnd
profile.OnSessionEnd:Connect(function()
    if DataManager[player] then
        DataManager.OnReleasing:Fire(player, DataManager[player])
    end
    DataManager[player] = nil
end)
Players.PlayerRemoving:Connect(function(player)
    -- game cleanup, final data writes, then just EndSession
    profile:EndSession()  -- triggers OnSessionEnd -> OnReleasing (once)
end)
```

## Nil'ing the profile in multiple locations

The profile must be removed from the lookup table in exactly ONE place: the library's release callback. If the game nils the profile in both `OnSessionEnd` AND `PlayerRemoving`, this is a data module bug -- fix it. Move the nil to `OnSessionEnd` only, and let `PlayerRemoving` just call `EndSession()`.

Duplicate nil'ing can cause race conditions where `OnReleasing` listeners see a nil profile if `PlayerRemoving` runs the nil before `OnSessionEnd` fires.
