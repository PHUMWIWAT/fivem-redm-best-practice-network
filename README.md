# fivem-redm-performance-network

Grok skill for optimizing **FiveM / RedM resmon** and **network event bytes**.

Agent workflow lives in `SKILL.md`. Deep facts live in `references/` and are loaded only when needed:

| File | Contents |
|---|---|
| `references/profiling.md` | resmon, profiler, `neteventlog`, hitch types, how to measure |
| `references/fivem-networking.md` | Trigger\*Event, latent, msgpack, routing/`-1`, state bags |
| `references/lua-hotpaths.md` | Wait staging, natives, pools, NUI, SQL, allocations |

## Use

- Slash: `/fivem-redm-performance-network`
- TUI: `/skills fivem-redm-performance-network`
- Auto: Grok loads it when you ask to optimize resmon, hitches, `TriggerServerEvent` spam, payload size, or `Wait(0)` loops

ภาษาไทย: ใช้ skill นี้ตอนให้ agent ไล่ลด resmon หรือลดปริมาณ event/bytes บนเน็ตเวิร์ก FiveM/RedM — อย่าให้โหลดไฟล์ references ทุกไฟล์ทุกงาน

## Install (Grok user skill)

Copy this folder to:

```text
%USERPROFILE%\.grok\skills\fivem-redm-performance-network\
```

Required files: `SKILL.md`, `references/*.md`. Skills reload from disk within a few seconds.

This repo is the source of truth. After editing here, copy again to `~\.grok\skills\`.

## Not this skill

- Native signatures → `fivem-natives-skill`
- General RedM/FiveM implementation / OneSync security encyclopedia → `b3sty-skill` (if installed)
