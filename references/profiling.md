# Measuring resmon, profiler, and network events

Open this file when diagnosing hitch, FPS, resmon numbers, event spam, or when writing the Verification section of a report. Measure before changing code. Never invent a millisecond or byte count.

Official docs:

- [Fact sheet (hitch, profiler, resmon)](https://docs.fivem.net/docs/scripting-manual/introduction/fact-sheet/)
- [Using the profiler](https://docs.fivem.net/docs/scripting-manual/debugging/using-profiler/)
- [Client console commands](https://docs.fivem.net/docs/client-manual/console-commands/) (`resmon`, `neteventlog`, `netgraph`)

FiveM and RedM share these tools. `resmon` / `neteventlog` / `profiler` are developer commands on the client.

---

## Which tool answers which question

| Question | Tool | Side |
|---|---|---|
| Which **resource** is eating client frame time / memory? | `resmon` | Client F8 |
| Which resource is eating **server** tick time? | Server `resmon` (recent artifacts) or `profiler record` on the **server console** | Server |
| Which **thread / file:line** inside that resource? | `profiler` | Same side as the hitch |
| Which **net events** fire, direction, size in bytes? | `neteventlog` | Client F8 (dev mode) |
| Packets/bytes per second, ping, packet loss | `netgraph`, `cl_drawperf` | Client |
| Load-correlated tick over hours | txAdmin performance chart | Server |
| NUI/JS cost | `nui_devtools` | Client |

Client `resmon` cannot prove a server hitch. Server profiler cannot prove a client FPS drop. Profile the side that is slow.

---

## Hitch types

**Script hitch** — a resource's Lua/native work overran the tick. `Wait(0)` pools, sync SQL, huge loops. Fix with `lua-hotpaths.md`.

**Network thread hitch** — too many or too large packets ( `-1` broadcasts, event floods, fat state bags). Raising `Wait` on an unrelated loop does **not** fix this. Fix with `fivem-networking.md` and confirm with `neteventlog`.

A resource at `0.00ms` on resmon can still flood the net. A quiet `neteventlog` can still hitch from `GetGamePool` every frame. Optimize the whole system.

---

## Developer mode (client)

`resmon`, `neteventlog`, `profiler` on the **client** are developer commands. Production clients often print `Access denied for command resmon`.

Enable developer mode by either:

1. Launch FiveM/RedM with `+set moo 31337` on the shortcut, or
2. Use a non-production update channel (Beta / Latest). Those channels can be unstable.

Same requirement on RedM.

---

## resmon

Client F8:

```text
resmon true
resmon false
```

`resmon 1` / `resmon 0` is the same toggle used in many consoles.

What it shows: per-resource **CPU time (msec)** and **memory**. Use it to find **which** resource is expensive, not which line.

How to read:

- Idle (standing still, UI closed) vs peak (open inventory, in a zone, shooting). Capture both.
- Constant high msec → a `Wait(0)` loop or per-frame native/NUI spam.
- Spike on interaction → cost of that action (encode, SQL, pool scan). That can still be Critical if the action is common.
- Memory climbing without bound → leak (forgotten tables, NUI listeners, entity handles).

Community rule of thumb (not an engine guarantee): a resource sitting above ~0.10–0.25 ms **at idle** deserves a look. Do not treat that band as a hard spec. Compare against **this server's** baseline.

Server: on recent FXServer artifacts, `resmon 1` in the live console (txAdmin) or after `svgui` on a raw server. If the command is missing, use the server profiler instead.

resmon does **not** show event payload bytes. Use `neteventlog` for that.

---

## Profiler

Works on client F8 **and** the server console. Use latest client + latest artifacts. Chrome (or a Chromium tracing UI / Perfetto) is required to view saved traces. If you `profiler view` without saving, keep the game/server running.

```text
profiler record 500
profiler status
profiler view
profiler saveJSON filename.json
```

500 frames is a good first capture. `profiler status` shows whether recording is still active.

- Client `profiler view` can open Chrome itself.
- Server `profiler view` prints a URL — paste it into Chrome.
- `profiler saveJSON` writes next to the server run script (or the client app data). Load in Chrome DevTools → Performance → Load profile, or Perfetto.

How to read:

1. Yellow **CPU time** graph — spikes are hitches / frame drops.
2. Zoom a spike.
3. Open the **resource tick** band.
4. Hover a thread: msec, **file path, line range**.

That file:line is the hotspot. Do not optimize a different loop because it “looks expensive”.

---

## Network event log

Client F8 (dev mode):

```text
neteventlog true
neteventlog false
```

Shows direction (Server → Client / Client → Server), **event name**, and **payload size in bytes**.

Hunt for:

- The same event every frame
- Huge S→C payloads (then consider latent or less data)
- Fan-out that matches player count (likely `-1`)
- Fat state-bag updates (often logged by community event-log resources; bags also hit `stateBag` rate limiters)

Reliable Network Event Overflow while joining or playing → this log first, then kill the spamming event.

Related:

| Command | What |
|---|---|
| `netgraph true` | Live in/out packets and bytes, ping, routing delay |
| `net_statsFile metrics.csv` | CSV of ping / packets / bytes (client app data dir) |
| `cl_drawperf true` | On-screen FPS, ping, packet loss, CPU/GPU |

`netgraph` proves bandwidth; `neteventlog` names the event.

---

## txAdmin and server logs

Use txAdmin's performance chart for tick time over hours (leaks, restart-correlated stalls, player-count scaling). A 500-frame profiler snapshot can miss a hitch that happens every 10 minutes.

Server console hitch warnings name a resource and a msec overrun. Profile **that** resource on the **server**, then open the file:line the profiler gives.

---

## Measurement protocol

1. **Baseline** the failing scenario (idle + the action that lags) with resmon and, if net-related, `neteventlog`. Screenshot or write down numbers.
2. Change **one class** of issue (one loop, one event, one broadcast).
3. Remeasure the **same** scenario.
4. If the number did not move, revert and pick a different hotspot.
5. Record whether each figure is **measured** or **estimated**.

Static estimate when the user cannot run tools (see SKILL.md report format):

```text
Event:
Frequency:        (from Wait / GetGameTimer / estimate)
Payload:          (from neteventlog, or estimate from table shape)
Recipients:       (1 / nearby N / -1 = player count)
Bytes/sec/player:
Total bytes/sec:
```

Prefix every unmeasured number with `estimate`. Never present an estimate as a profiler result.

---

## Verification checklist

After an optimization:

- [ ] Client resmon idle and peak captured (or explicitly unavailable)
- [ ] Server hitch / server resmon / server profiler if the complaint was server-side
- [ ] `neteventlog` no longer shows the killed spam (if the change was network)
- [ ] Gameplay, sync, permissions, NUI, exports still work
- [ ] Report labels measured vs estimated
- [ ] Remaining bottlenecks listed instead of claiming the resource is “fully optimized”
