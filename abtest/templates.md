# abtest Code Templates

Code templates for common AB test integration points. Pick the one that matches the scope (player vs job) and context (server service, client controller, event module) of your feature.

## Server service -- sync (most common)

For values read during gameplay. No yielding.

```luau
-- In a service function
local function processReward(player: Player)
    local multiplier = ABTests.GetJobAttribute("Reward.Multiplier", 1)
    giveReward(player, BASE_REWARD * multiplier)
end
```

## Server service -- async (join flow)

For values needed before a player-specific decision on join.

```luau
function MyService.playerAdded(player: Player)
    local threshold = ABTests.GetAttributeAsync(player, "FTU.SpeedThreshold", 200)
    if player.Data.CurrentSpeed >= threshold then
        return
    end
    -- player qualifies for FTU flow
end
```

## Client controller -- player-level with reactive updates

```luau
local ABTests = require(ReplicatedStorage.UserGenerated.ABTests)
local Players = game:GetService("Players")

local LocalPlayer = Players.LocalPlayer

local MyController = {}

local function applySettings()
    local enabled = ABTests.GetAttribute(LocalPlayer, "Feature.Enabled", false)
    if type(enabled) ~= "boolean" then
        enabled = false
    end
    -- apply the value
end

function MyController.init()
    -- Apply once loaded
    if ABTests.IsLoaded() then
        applySettings()
    else
        task.spawn(function()
            ABTests.Loaded:Wait()
            applySettings()
        end)
    end

    -- React to live updates
    ABTests.PlayerUpdated:Connect(function(updatedPlayer)
        if updatedPlayer == LocalPlayer then
            applySettings()
        end
    end)
end

return MyController
```

## Client controller -- job-level with reactive updates

```luau
local ABTests = require(ReplicatedStorage.UserGenerated.ABTests)

local MyController = {}

local function applySettings()
    local showFeature = ABTests.GetJobAttribute("AB.ShowFeature", true)
    if type(showFeature) ~= "boolean" then
        showFeature = true
    end
    -- apply the value
end

function MyController.init()
    if ABTests.IsLoaded() then
        applySettings()
    else
        task.spawn(function()
            ABTests.Loaded:Wait()
            applySettings()
        end)
    end

    ABTests.JobUpdated:Connect(applySettings)
end

return MyController
```

## Client controller -- async in init (blocking)

When the controller cannot proceed without the value. Use sparingly -- this yields init.

```luau
function MyController.init()
    local enabled = ABTests.GetJobAttributeAsync("FTU.HideTower", true)
    if not enabled then
        return
    end
    -- setup tower UI
end
```

## Event module -- job-level tunables

Events typically read many job-level attributes for all their parameters:

```luau
local function run(duration: number, _context: any?)
    local messageDelay = ABTests.GetJobAttribute("Event.MyEvent.MessageDelay", 3)
    local messageText = ABTests.GetJobAttribute(
        "Event.MyEvent.Message",
        "My Event has started!"
    )
    local spawnRate = ABTests.GetJobAttribute("Event.MyEvent.SpawnRate", 0.5)
    local maxItems = ABTests.GetJobAttribute("Event.MyEvent.MaxItemCap", 100)

    task.delay(messageDelay, function()
        Popup.GlobalMessage(messageText)
    end)

    -- use spawnRate, maxItems in event logic...

    return function()
        -- cleanup
    end
end
```
