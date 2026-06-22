# Control logic — external interface & mode header

This document captures the **exact** control behaviour agreed for the MK4 Golf PWM fan controller.

## External 3-wire connector (J_EXT)

| Pin | Net | Direction | Function |
|---|---|---|---|
| 1 | VCC | in | Control-rail supply and PWM enable (battery voltage, 9–16 V) |
| 2 | PWM | out | PWM signal that drives the high-side gate stage |
| 3 | GND | — | Ground return |

## Main jumper (JP_MAIN)

The main jumper decides whether **VCC** comes from the host or is held permanently live on the board.

| JP_MAIN | VCC source | Control authority |
|---|---|---|
| **Fitted** | Host system drives VCC | Host enables the board (VCC) and uses the PWM line |
| **Removed** | Internal +12 V rail bridged onto VCC (permanently live) | Local `J_MODE` header selects the mode |

## Mode header (J_MODE) — 3-pin, Option 1

`J_MODE` is a 3-pin header. The **centre pin is local GND**. You ground **one** outer pin to choose the mode (option (b): no neutral state — one outer pin is always grounded in standalone use).

```
   J_MODE
   ┌───┬───┬───┐
   │ 1 │ G │ 2 │      G = centre = local GND
   └───┴───┴───┘
     │       │
  FULL      PWM
  SPEED     MODE
```

| Grounded pin | Mode | 555 state | Gate / PWM line | Fan |
|---|---|---|---|---|
| **pin 1 → GND** | **FULL SPEED** | Held in **RESET** (interlock — cannot oscillate) | Forced **on** (gate pulled to drive all FETs closed), PWM line = VCC (100 %) | Max |
| **pin 2 → GND** | **PWM** | RESET **released** → oscillates ~25 kHz | Driven by 555 via gate stage | Variable (pot) |

### Interlock (added per request)

When **pin 1 (full speed)** is grounded, the same signal **forces the 555 into RESET**. This guarantees the oscillator output cannot fight the forced-high gate line — no contention, no shoot-through risk from two sources driving the gate node.

### Why RESET instead of grounding pin 1 of the 555

The earlier revision grounded the **555's own ground pin (pin 1)** to gate operation. Removing that ground left the 555 with **no reference**, so its output floated to an undefined state (the bug you observed).

**This design fixes that:** the 555's **pin 1 is hardwired to GND permanently**. Enable/disable is done through **RESET (pin 4)**, which is the part's intended enable input:
- RESET low (≤ ~0.7 V) → output forced low, oscillator stopped.
- RESET high (released, pulled up) → 555 runs.

The mode header therefore switches a **logic/RESET node**, never the chip's ground — so a floating-output condition is impossible.

## Default / fail-safe states

| Condition | Result |
|---|---|
| Gate drive lost (any reason) | `R_gs` 10 kΩ pulldown holds FETs **OFF** (fan off — fail-safe) |
| `J_MODE` left open in standalone (neither pin grounded) | 555 RESET pulled to a defined level by `R_pull`; define this per your wiring (see note) |
| Main jumper fitted, host VCC absent | Board unpowered → fan off |

> **Note on the open-header state:** with option (b) one outer pin is expected to be grounded in standalone use. If you want a *defined* state when neither is grounded, the pull network biases RESET to the **PWM-disabled (off)** condition by default; confirm if you'd prefer it to default to PWM-on instead.
