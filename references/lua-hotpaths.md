# Lua / native hot paths (resmon CPU)

Open this file when lowering client or server **CPU** time: threads, `Wait`, natives, entity pools, allocations, NUI, player iteration, SQL. Network bytes live in `fivem-networking.md`. How to measure lives in `profiling.md`.

Do not raise every `Wait` blindly. Keep gameplay responsiveness. Verify native names on RedM with the natives skill — GTA pool names are not a RedM guarantee.

---

## Find the threads

Search:

```lua
CreateThread
Citizen.CreateThread
CreateThreadNow
while true do
Wait(0)
Wait(1)
Wait(5)
Wait(10)
```

A loop without `Wait` is a busy loop and stalls the runtime.

`Wait(0)` yields **one frame**. Use it only when the body must run every frame (draw, disable controls, precise aim). Idle work at `Wait(0)` is the usual resmon floor.

---

## Adaptive sleep

Use the longest wait that still feels correct. Values below are starting points, not a spec.

```text
Far / inactive     500–2000 ms
Nearby             100–500 ms
Active interaction 0–50 ms
Draw / controls    0 ms
```

```lua
CreateThread(function()
    while true do
        local sleep = 1000
        if playerNearZone then
            sleep = 250
        end
        if isInteracting then
            sleep = 0
        end
        -- logic
        Wait(sleep)
    end
end)
```

Do not leave a per-frame thread running when the player is far away. Gate by distance or state, then sleep long.

---

## Polling vs events

If the state is already an event, bag change, callback, or command, do not poll it.

```lua
-- Bad
CreateThread(function()
    while true do
        local state = GetSomething()
        if state then
            Update()
        end
        Wait(0)
    end
end)

-- Good
RegisterNetEvent('resource:stateChanged', function(state)
    Update(state)
end)
```

Poll only when the engine has no event (some control/entity checks). Then poll as slowly as the feature allows.

---

## Native calls

Cache values whose lifetime is “this tick” or “until this state changes”. Do not cache values that must stay live (current health during combat, current coords for a frame-perfect check — cache **once per iteration**, not forever).

```lua
-- Bad: same natives several times
if GetPlayerPed(PlayerId()) ~= 0 then
    local ped = GetPlayerPed(PlayerId())
    local coords = GetEntityCoords(GetPlayerPed(PlayerId()))
end

-- Good
local ped = PlayerPedId()
local coords = GetEntityCoords(ped)
```

Hot natives to treat as suspects inside loops: `GetGamePool`, `GetEntityCoords`, `GetEntityHealth`, `GetPlayerPed`, `GetActivePlayers`, `GetPlayers`.

`GetHashKey` in a hot loop is a native; prefer `joaat('name')` or a backtick hash at load time.

Vector distance: `#(a - b)`. Prefer a squared / axis early-out before a full distance when rejecting far entities.

```lua
local dx, dy, dz = a.x - b.x, a.y - b.y, a.z - b.z
local distSq = dx * dx + dy * dy + dz * dz
if distSq < (50.0 * 50.0) then
    -- nearby
end
```

Unknown native signature → natives skill, not guesswork.

---

## Entity scanning

`GetGamePool('CPed')` / `'CVehicle'` / `'CObject'` (FiveM) walking every frame is Critical.

```lua
-- Bad
CreateThread(function()
    while true do
        for _, vehicle in ipairs(GetGamePool('CVehicle')) do
            -- work
        end
        Wait(0)
    end
end)
```

Prefer: proximity / zone membership, event on enter, cached handle while in range, lower-frequency scan, process only relevant entities. RedM pool/native names may differ — verify.

Create **local** (non-networked) objects for cosmetics/previews. Networked entities cost stream and ownership, not just Lua.

---

## Allocations

Inside high-frequency loops, avoid:

- new tables every iteration
- `json.encode` / `json.decode`
- string concat to build messages
- closures allocated per tick
- `SendNUIMessage` of an unchanged blob

```lua
-- Bad: allocates every frame
while true do
    local data = {
        coords = GetEntityCoords(PlayerPedId()),
        health = GetEntityHealth(PlayerPedId()),
    }
    Wait(0)
end
```

Reuse a table, update fields, or don’t build it until something changed.

---

## NUI

`SendNUIMessage` / `SetNuiFocus` are client CPU (CEF), not game-net. They still move resmon.

```lua
-- Bad
while true do
    SendNUIMessage({ action = 'update', data = data })
    Wait(0)
end

-- Good
if oldState ~= newState then
    SendNUIMessage({ action = 'update', data = newState })
    oldState = newState
end
```

Close focus on UI close, player drop, and resource stop. Heavy DOM work belongs in the browser profiler (`nui_devtools`), not more Lua waits.

---

## Server loops

Search server `CreateThread` / `while true` / `GetPlayers()`.

```lua
for _, playerId in ipairs(GetPlayers()) do
```

Ask: does every player need this? every tick? can the set be cached? can it be event-driven? can work be spread across ticks?

Client player iteration: prefer `GetActivePlayers()` over a 0–255 numeric loop ([cookbook](https://docs.fivem.net/docs/cookbook/2019/06/29/get_active_players-the-replacement-for-player-loops/)).

Repeated exports in a hot loop: cache the export function if the project already does that **and** invalidates on resource stop/restart. Stale export references after a restart are a correctness bug.

Yield in long server work (`Wait(0)` every N iterations) so one resource cannot hitch the whole tick.

---

## Database

Never query from a high-frequency loop.

```lua
-- Bad
while true do
    MySQL.query.await('SELECT ...')
    Wait(100)
end
```

```text
event / state change
  → query only if data is stale
    → cache
```

Sync `query.await` on the server tick is a hitch source even at `Wait(100)` if the query is slow. Move heavy SQL off the hot path; do not “fix” it by lowering Wait.

---

## Architecture

Prefer:

```text
Gameplay state
  → state change
    → local work
      → network only if another machine needs it
```

Avoid:

```text
Every frame → collect everything → encode everything → TriggerServerEvent → broadcast -1
```

---

## Priority while editing CPU

1. Frame-level `GetGamePool` / `Wait(0)` with real work / NUI spam
2. Tight native repeats, player iteration, sync SQL
3. Allocations and redundant math
4. Micro-optimizations that resmon will not show
