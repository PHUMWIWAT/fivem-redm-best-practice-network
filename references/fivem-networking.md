# FiveM / RedM network events, serialization, and routing

Open this file when the task touches `Trigger*Event`, latent events, payload size, broadcasts, state bags, callbacks, or bandwidth. Lua is the source of truth; JS/C# names are listed once.

Official docs:

- [Triggering events](https://docs.fivem.net/docs/scripting-manual/working-with-events/triggering-events/)
- [Listening for events](https://docs.fivem.net/docs/scripting-manual/working-with-events/listening-for-events/)
- [State bags](https://docs.fivem.net/docs/scripting-manual/networking/state-bags/)
- [Network and local IDs](https://docs.fivem.net/docs/scripting-manual/networking/ids/)
- [Server commands](https://docs.fivem.net/docs/server-manual/server-commands/) (rate limiters, `sv_enableNetEventReassembly`)
- [Routing buckets](https://docs.fivem.net/docs/cookbook/2020/11/27/routing-buckets-split-game-state/)

FiveM and RedM share this event model. Do not assume GTA native names on RedM.

---

## Event API map

| Direction | Lua | JS | Notes |
|---|---|---|---|
| Local (same VM) | `TriggerEvent(name, ...)` | `emit(name, ...)` | Never crosses the network |
| Client → server | `TriggerServerEvent(name, ...)` | `emitNet(name, ...)` | Small / latency-sensitive |
| Client → server (large) | `TriggerLatentServerEvent(name, bps, ...)` | `TriggerLatentServerEvent(name, bps, ...)` | Does not monopolize the net channel |
| Server → one/all clients | `TriggerClientEvent(name, target, ...)` | `emitNet(name, target, ...)` | `target = -1` is every connected client |
| Server → client (large) | `TriggerLatentClientEvent(name, target, bps, ...)` | `TriggerLatentClientEvent(name, target, bps, ...)` | Same `-1` rule; `bps` is **per target** |

Register a handler that may be triggered over the network:

```lua
RegisterNetEvent('resource:action', function(...)
    -- Lua/JS: on the server, `source` is the triggering player
end)
```

`RegisterServerEvent` is the older alias. `AddEventHandler` listens locally and does **not** by itself mark the event as networked — use `RegisterNetEvent` (optionally with a separate `AddEventHandler`) when the other side must be able to fire it.

JS: `on` is local; `onNet` is networked **and** also runs for local emits of the same name.

Classify every call site:

- local vs C→S vs S→C
- one recipient vs nearby set vs routing bucket vs `-1`
- one-shot action vs continuous state sync
- inside a loop or not

---

## What a net event actually costs

A networked event is not free. Cost is roughly:

```text
(event name + msgpack payload)  ×  frequency  ×  recipients
```

Plus handler CPU on every recipient (and on the server for C→S).

`Wait(0)` is **not** 60 Hz. It yields one frame. Label any Hz figure as **estimate** unless `neteventlog` or a profiler captured it.

### Estimate table (label as estimate)

Event `resource:update`, ~1.8 KB payload, 20 times/sec:

| Recipients | Traffic |
|---|---|
| 1 player | ~36 KB/s |
| 100 players (`-1`) | ~3.6 MB/s |
| 500 players (`-1`) | ~18 MB/s |

Frame-loop `TriggerServerEvent` with `Wait(0)`:

```text
1 player    ≈ tens of events/sec
100 players ≈ thousands of C→S events/sec
```

That pattern is Critical even when each payload is tiny. The name string is sent every time too.

### Engine rate limiters

FXServer token-bucket limiters (defaults; operators can raise them). Spam is not only “slow” — it drops or kicks:

| Limiter | Default rate / burst | Meaning |
|---|---|---|
| `netEvent` | 50 / 200 | Script net events |
| `netEventFlood` | 75 / 300 | Flood path |
| `stateBag` | 75 / 125 | State bag updates |
| `stateBagFlood` | 150 / 175 | Bag flood path |
| `stateBagSize` | 131072 / 262144 | Bag payload bytes |

Clients can be dropped with **Reliable Network Event Overflow** when the reliable channel is flooded. Fix the sender; do not tell players to “just reconnect”.

---

## Serialization

Net events pack arguments with **msgpack**, not JSON.

- `json.encode` / `json.decode` is an extra Lua CPU + string allocation **on top of** the net pack. Flag it inside loops and inside handlers.
- Encode once when state changes, then send the cached buffer/table. Do not encode the same unchanged table every tick.
- Functions, userdata, and circular tables do not serialize. Entity **handles** are process-local — send `NetworkGetNetworkIdFromEntity(entity)` and recover with `NetworkGetEntityFromNetworkId(netId)` plus `DoesEntityExist`.
- Prefer primitives and small tables over nested inventory/job/player blobs.
- Vector3 and numbers pack cheaply; giant string keys and deep copies do not.

Bad: send the world because the handler only needs coords.

```lua
TriggerServerEvent('player:update', {
    name = player.name,
    job = player.job,
    inventory = player.inventory,
    weapons = player.weapons,
    coords = player.coords,
    health = player.health,
})
```

Good: send the field the receiver uses.

```lua
TriggerServerEvent('player:updatePosition', player.coords)
```

---

## Latent events

Use latent events for **large** transfers (multiple KB and up): clothing packs, full config dumps, bulk UI catalogs, screenshot-sized blobs.

They take `bps` (bytes per second). `bps` of `-1` or `0` uses the default **25000**. `bps` applies **per target**. `TriggerLatentClientEvent(..., -1, bps, ...)` therefore sends about `bps * player_count` bytes per second from the server.

```lua
TriggerLatentClientEvent('resource:bigDump', targetPlayer, 50000, hugeTable)
TriggerLatentServerEvent('resource:bigUpload', 50000, hugeTable)
```

Rules:

- Do **not** use latent for small latency-sensitive gameplay (shots, door toggles, inventory give).
- Extremely large `bps` (~10,000,000+) can still stall because the runtime tries to send everything at once.
- Large non-latent S→C payloads block that client's game net channel and can **timeout** the client. That is the whole point of latent.
- Reassembly of split events needs `sv_enableNetEventReassembly` (default `true`). Pending-event caps: `sv_netEventReassemblyMaxPendingEvents` (default 100).

---

## Routing and broadcast

Always ask: **who actually needs this data?**

```text
source player
  → nearby / in-scope players
    → same routing bucket
      → players with a specific job/state
        → every connected client (-1)
```

Pick the smallest valid audience.

```lua
-- Bad: every client, including other buckets / other sides of the map
TriggerClientEvent('resource:update', -1, data)

-- Good: the one player who must see it
TriggerClientEvent('resource:update', targetPlayer, data)
```

For “nearby”: compute recipients on the **server** from server-known positions or OneSync scope. Never let a client supply the recipient list.

Routing buckets partition the world. `-1` still delivers to players in other instances. Scope messages to the bucket (or to the players you already moved into it).

OneSync only streams entities in scope. Broadcasting full entity state to `-1` does not make far-away clients “own” that entity; it just wastes bandwidth. Prefer entity/player state bags for replicated visible state.

---

## State bags vs events

| Use | Tool |
|---|---|
| Replicated visible state (fuel, cuffed, outfit id) | State bag + `AddStateBagChangeHandler` |
| One-shot action (give item, open door, play emote) | Net event |
| Large blob | Latent event, not a bag value |

Lua:

```lua
Entity(vehicle).state.fuel = 80          -- owner or server
Player(source).state.cuffed = true
LocalPlayer.state.busy = true            -- local player
GlobalState.moneyEnabled = true          -- server-set, client-read
```

Replication defaults:

- Server writes **are** replicated.
- Client writes **are not**, unless `state:set(key, value, true)`.
- `sv_stateBagStrictMode true` → only the server may write bags.

Override per key:

```lua
Entity(vehicle).state:set('clone', 600, false)   -- server, local only
Entity(enemy).state:set('taskAck', 'guard', true) -- client, replicate
```

Shallow bags: a nested assign does **not** replicate.

```lua
-- will not replicate
Entity(x).state.x.y = 'b'

-- do this instead
Entity(x).state['x:y'] = 'b'
```

Every get deserializes the full bag value. Do not read `Entity(x).state.big` twice per tick.

Do not store huge tables on bags. Use a latent event. Fat bag updates hit `stateBag` / `stateBagSize` limiters and can hitch the network thread.

Prefer handlers over polling:

```lua
AddStateBagChangeHandler('fuel', nil, function(bagName, key, value, _unused, replicated)
    local entity = GetEntityFromStateBagName(bagName)
    if entity == 0 then return end
    -- react
end)
```

---

## Delta, batch, throttle

**Delta:** send the change, not the whole inventory.

```lua
-- Bad
TriggerServerEvent('inventory:update', entireInventory)

-- Good
TriggerServerEvent('inventory:itemChanged', { item = itemName, amount = amount })
```

**Batch:** many small updates in a short window → one array. Batching must not add unacceptable gameplay latency.

**Throttle:** data that is not frame-critical (position HUD, idle stats) should not fire every tick.

```lua
local lastUpdate = 0
CreateThread(function()
    while true do
        local now = GetGameTimer()
        if now - lastUpdate >= 250 then
            lastUpdate = now
            TriggerServerEvent('resource:update', GetEntityCoords(PlayerPedId()))
        end
        Wait(50)
    end
end)
```

Do not throttle shots, locksteps, or anything where late packets change who wins.

---

## Local events vs network vs NUI vs callbacks

Same client, two scripts, no server authority needed:

```lua
TriggerEvent('resource:stateChanged', data)  -- or an export
```

Not `TriggerServerEvent`. Round-tripping through the server to talk to yourself costs a net event both ways.

`SendNUIMessage` is **client ↔ CEF**. It does not use the game net channel, but it still costs JSON/CPU and will show up in **resmon**. Send on state change only. See `lua-hotpaths.md`.

`lib.callback` / paired request-response events are still **two** net messages. Apply the same payload, throttle, and validation rules. Client-side callback cooldowns (ox_lib) reduce spam; they are not a substitute for server validation.

---

## Security while optimizing

Never drop server checks to save bytes. Never let the client choose who gets a broadcast.

Every public C→S handler must still validate:

```text
source, permissions, item, amount, price, state,
distance, ownership, routing bucket, cooldowns
```

Capture `local src = source` before any yield. Re-validate after awaits.

Treat client-provided handles and net IDs as untrusted: existence, type, model, owner, bucket, distance.

---

## Anti-patterns

| Pattern | Why it hurts |
|---|---|
| `TriggerServerEvent` / `TriggerClientEvent` inside `Wait(0)` | Event flood, overflow kicks, handler CPU on every peer |
| `TriggerClientEvent(..., -1, ...)` for one player's UI | Bandwidth × player count |
| Whole player/inventory table every few ms | Payload × freq × recipients |
| `json.encode` every tick then send | Extra CPU **and** a fat string on the wire if you send the JSON |
| State bag = entire config dump | Bag size/flood limiters, network hitch |
| Latent for a 20-byte shot event | Added latency, no benefit |
| Send entity handles across the network | Invalid on the other side |
| Client supplies the recipient list | Exploit + extra broadcasts |

Preferred pipeline:

```text
Gameplay state changes
  → local processing
    → sync only if another machine needs it
      → smallest audience
        → smallest payload (delta)
```
