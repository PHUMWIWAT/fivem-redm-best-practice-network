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

## Install

```bash
npx skills add PHUMWIWAT/fivem-redm-best-practice-network --skill fivem-redm-performance-network -g -a grok
```

ลงทุก agent ที่เครื่องเจอ (Claude / Cursor / Grok / …):

```bash
npx skills add PHUMWIWAT/fivem-redm-best-practice-network --skill fivem-redm-performance-network -g
```

แค่ในโปรเจกต์นั้น (เช่น resource DX):

```bash
npx skills add PHUMWIWAT/fivem-redm-best-practice-network --skill fivem-redm-performance-network -a grok
```

Grok วางไฟล์ที่ `~/.grok/skills/` (`-g`) หรือ `.grok/skills/` ใน repo แล้วมี slash `/fivem-redm-performance-network`

คัดลอกมือได้เช่นกัน: โฟลเดอร์นี้ทั้งก้อนไปที่ `%USERPROFILE%\.grok\skills\fivem-redm-performance-network\` (ต้องมี `SKILL.md` + `references/`).

## Not this skill

- Native signatures → `fivem-natives-skill`
- General RedM/FiveM implementation / OneSync security encyclopedia → `b3sty-skill` (if installed)
