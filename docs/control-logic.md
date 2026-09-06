# Control logic — jumper, trigger inputs & signal chain

Net-level description of the control behaviour, verified against the schematic in `hardware/PWMFanController.epro`.

## External interface

### Power pads

| Pad | Net | Direction | Function |
|---|---|---|---|
| VCC | VCC | in | Supply: OEM "low speed" request (jumper fitted) or permanent battery (jumper removed) |
| PWM | PWM_OUT | out | High-side switched output to fan + |
| GND | GND | — | Ground return |

### EXT_CNTL (JST-XH, 2-pin) — trigger inputs, active when grounded

| Pin | Net | Grounded → |
|---|---|---|
| 1 | SHUTDOWN | SG3525 released from shutdown → PWM output runs (soft-started, pot-set duty) |
| 2 | FULLSPEED | TC4420 input forced low via R6 (0 Ω) → gates driven → 100 % output |

### EXT_CNTL_JMP (2-pin header)

Shorts **SHUTDOWN → GND** when fitted. That is its only function.

## Truth table

| EXT_CNTL_JMP | EXT_CNTL 1 (SHUTDOWN) | EXT_CNTL 2 (FULLSPEED) | SG3525 | Output |
|---|---|---|---|---|
| Fitted | — | open | Running | PWM at pot duty, ~30 kHz, soft-started on power-up |
| Removed | open | open | **Shutdown** (pin 10 pulled to 5.1 V by R2 10 k) | Off — gates held at rail |
| Removed | grounded | open | Running | PWM at pot duty, soft-started |
| Removed | open | grounded | Shutdown | **100 % on** (gate stage driven directly) |
| Removed | grounded | grounded | Running | 100 % on (FULLSPEED dominates the shared node) |

## Signal chain (net by net)

1. **SHUTDOWN** — SG3525 pin 10, pulled up to VREF (5.1 V) through **R2 10 k**, filtered by **C8 1 µF**. High = shutdown (outputs off, soft-start cap discharged). Grounding it (jumper or JST pin 1) lets the SG3525 run. Default state is therefore *off*.
2. **WIPER / POT** — VREF (pin 16, internal 5.1 V regulator) feeds the **CALIBRATION** trimmer (3362P, 10 kΩ) across VREF–GND; the wiper drives the non-inverting error-amp input (pin 2). COMP (pin 9) is strapped to the inverting input (pin 1) → unity-gain follower → wiper voltage sets duty directly.
3. **Oscillator** — RT (pin 6) = R4 10 k to GND; CT (pin 5) = C7 4.7 nF with the discharge pin (7) strapped across → f_osc ≈ 30 kHz.
4. **Soft-start** — C9 100 µF on pin 8; internal ~50 µA source ramps duty over ~5 s each time shutdown is released.
5. **OUTA / OUTB** (pins 11/14, each f/2) — diode-OR'd through **D3/D1** (1N4148WS) into a common node loaded by **R5 1 k** to GND, recombining into a single ~30 kHz PWM stream (**SG3525_OUT**, via **R1 1 k**; TP_SG3).
6. **SG3525_OUT → Q4** (2N7002). Output high → Q4 on → **TC4420_IN** pulled low.
7. **TC4420_IN** — pulled up to VCC by **R10 1 k (0.5 W)**; also tied to the **FULLSPEED** trigger through **R6 0 Ω**. Low = fan on. (TP_TC)
8. **TC4420 (U2, non-inverting)** — input low → output (**GATE**) low → V_GS ≈ −VCC → Q1–Q3 conduct. Input high → GATE at rail → off. (TP_GATE)
9. **GATE** — pulled to VCC by **R7 10 k** (fail-safe off), fanned out through **R8/R9/R11 10 Ω** to the three 120P03 gates.
10. **PWM_OUT** — Q1–Q3 drains, **D4** (MBR60100DC) flyback to GND, and the green **PWM** LED via R12 10 k.

## Indicators

| LED | Wiring | Meaning |
|---|---|---|
| FULL (red) | VCC → R3 10 k → LED → GND | VCC present (OEM request active / battery live) |
| PWM (green) | PWM_OUT → R12 10 k → LED → GND | Output switching; brightness tracks duty |

## Fail-safe states

| Condition | Result |
|---|---|
| No trigger grounded, jumper out | SG3525 in shutdown, Q4 off, TC4420_IN = VCC, GATE = VCC → **fan off** |
| Gate drive lost (U2 failure/unpowered) | R7 10 k holds GATE at rail → fan off |
| Level shifter lost (Q4 open) | R10 holds TC4420_IN high → fan off |
| Trigger wiring severed | Input floats to its pull-up → fan off (SHUTDOWN) / PWM unaffected (FULLSPEED open) |

## ⚠ Trigger input voltage domains

The two JST inputs are **not** equally protected:

- **SHUTDOWN** idles at **5.1 V** (R2 to VREF). The SG3525 shutdown pin is not rated for 12 V — a floating/faulted ECU output imposing battery voltage can damage U1.
- **FULLSPEED** connects through 0 Ω to TC4420_IN, which idles at **VCC (12–14 V)** via R10 and carries the full PWM waveform whenever the board is running. Anything attached to this pin must tolerate 12 V switching at ~30 kHz, and any external clamp below VCC will partially turn the FETs on.

See `DESIGN_NOTES.md` → *Trigger input protection (proposal)* for the recommended fix before driving these pins from an ECU (the schematic already carries the note "Add zener to protect 12 V on triggers").
