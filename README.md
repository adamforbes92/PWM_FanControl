# PWM_FanControl

PWM Fan Controller for **MK4 Golf** — 12 V automotive, **high-side** switching, rated **300 W (~25 A)**, built around a **555 timer**.

Designed for **heatsink-free** operation using 4× parallel 40 V P-channel MOSFETs, with full automotive protection (load-dump, reverse-battery, inrush, flyback, gate clamp).

---

## Quick spec

| Parameter | Value |
|---|---|
| Supply | 12 V nominal (automotive, 9–16 V) |
| Load | 300 W max (~25 A @ 12 V) |
| Switch side | High-side (P-channel) |
| Switch | 4× SiR464DP-class, 40 V, ~3 mΩ, PowerPAK-SO8 (no heatsink) |
| PWM source | TLC555 (CMOS 555), ~25 kHz |
| Duty range | 0–100 % (diode-steered pot) |
| Gate driver | **TC4420** (6 A) — sinks gate low for ON; `R_pu` pulls to rail for OFF |
| Enable | 555 **RESET** (pin 4); chip GND (pin 1) permanently grounded |

## External 3-wire connector (J_EXT)

| Pin | Net | Function |
|---|---|---|
| 1 | VCC | Control-rail supply / PWM enable (battery voltage) |
| 2 | PWM | PWM output to gate stage (fan speed line) |
| 3 | GND | Ground return |

## Operating modes

### Main jumper FITTED (installed in vehicle)
- Host applies **VCC** → board runs → **PWM** line drives the fan.
- Host controls enable via VCC and uses the PWM line normally.

### Main jumper REMOVED (standalone / bench)
- **VCC is held permanently live** locally (bridged to the internal +12 V rail).
- The **3-pin mode header (J_MODE)** then selects behaviour by grounding **one** of its outer pins (centre pin = local GND):

| Action | Mode | What happens |
|---|---|---|
| Ground **J_MODE pin 1** | **FULL SPEED** | Gate forced on → all FETs closed → PWM line driven to 100 % (VCC). 555 simultaneously held in RESET (interlock). |
| Ground **J_MODE pin 2** | **PWM** | 555 RESET released → 555 oscillates → PWM on the PWM pin. |

> **Option 1 mapping:** pin 1 = full speed, pin 2 = PWM.
> The 555's ground (pin 1) is **hardwired to GND at all times** — the previous "remove jumper → 555 loses ground → output floats" failure cannot recur, because the header now switches a **RESET/enable** node, not the chip's ground.

## Gate drive (TC4420, high-side P-channel — option B)

The high-side P-FET gate must swing relative to the **+12 V rail**: ~rail for OFF, ~rail−10 V for ON.

- **OFF (default / fail-safe):** `R_pu` (4.7 k) pulls the gate up to the rail, and `R_gs` (10 k) holds Vgs→0 if drive is lost → FETs OFF.
- **ON:** the **TC4420** actively **sinks** the gate node low (through the per-FET gate resistors) to enrich Vgs and turn the P-FETs on.
- A small **level-shift NPN** conditions the ground-referenced 555/RESET logic into the TC4420 input domain.
- **Gate Zener (12 V)** clamps Vgs within the ±20 V rating.

This replaces the earlier discrete push-pull totem pole (MMBT3904/3906) — those parts remain in the BOM as **DNF alternates** for anyone building without the driver IC.

## Protection summary

| Feature | Implementation |
|---|---|
| Overcurrent | 30 A blade fuse (F1) |
| Reverse battery | Ideal-diode controller (LM74700-Q1 + N-FET) — near-lossless vs 25 W Schottky |
| Load dump (ISO 7637-2) | 5KP24A high-power TVS + series choke + bulk caps |
| Inrush | NTC (SL22 2R5) in series, relay/MOSFET bypass after ~200 ms |
| Bulk energy / ripple | 4× 2200 µF 25 V low-ESR + 470 µF input |
| Flyback / freewheel | SK54 Schottky (drain→fan) + diode across motor |
| Switching spike / EMI | RC snubber (10 Ω + 100 nF) on drain |
| Gate (Vgs) protection | 12 V Zener clamp |
| Gate fail-safe | 10 kΩ pulldown → FETs default OFF if drive lost |

## Files

| File | Description |
|---|---|
| `hardware/PWM_FanControl.easyeda.json` | **EasyEDA-importable** schematic (File → Open → import this JSON) |
| `hardware/PWM_FanControl.net` | SPICE-style netlist — **authoritative wiring reference** |
| `hardware/netlist.txt` | Human-readable netlist (nets + connections) |
| `hardware/BOM.csv` | Bill of materials |
| `docs/DESIGN_NOTES.md` | Topology rationale, component math, thermal budget, cautions |
| `docs/control-logic.md` | Detailed truth table & control-circuit explanation |

## Importing into EasyEDA

1. Open [EasyEDA Std Edition](https://easyeda.com/editor).
2. **File → Open → Open from Local…** and select `hardware/PWM_FanControl.easyeda.json`.
3. The schematic loads as a component placement list keyed by net name.
4. **Verify every connection against `hardware/PWM_FanControl.net`** (the source of truth), assign footprints (suggested in the BOM), then route the PCB.

> The JSON is a **best-effort** export using a simplified component+net-label representation, not full coordinate geometry. It may not render as a fully routed sheet — treat it as a starting canvas and rely on the netlist for wiring.

## ⚠️ Disclaimer

This is a **design aid / starting point**, not a verified production design. Before building:
- Validate the load-dump strategy against your alternator (an unsuppressed load dump can exceed 40 V; 40 V FETs rely on the TVS clamping the rail).
- Confirm fan stall/inrush current and size F1, the NTC and flyback diode accordingly.
- Bench-test gate-drive edges before trusting the design.
- Verify all parts against current datasheets and availability.
