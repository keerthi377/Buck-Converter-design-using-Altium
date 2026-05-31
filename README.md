# Buck Converter Power Module
### Based on Texas Instruments LMR36520FADDAR

## 1. Project Overview

This project is a **step-down (buck) DC-DC converter** that accepts a wide range of DC input voltages and produces a stable, regulated **12V output at up to 2A continuous current**. It is designed as a compact PCB module for integration into a larger system.

The design uses the **TI LMR36520FADDAR** — a fully integrated synchronous buck converter with internal high-side and low-side MOSFETs and internal compensation. No external MOSFETs or compensation network is required.

| Parameter              | Value                                      |
|------------------------|--------------------------------------------|
| Input voltage range    | 4.2V – 65V                                 |
| Output voltage         | 12V (set by R1/R2 feedback divider)        |
| Output current         | 2A continuous                              |
| Control mode           | Peak current mode (internal compensation)  |
| Topology               | Synchronous buck (both FETs internal)      |
| Internal reference     | 1.0V                                       |
| Package                | HSOIC-8 with exposed thermal pad (EP)      |
| Internal LDO output    | 5V (VCC pin — internal use only)           |


---

## 2. Basic Operating Principle

A buck converter steps voltage **down** by rapidly switching the input through an inductor. The ratio of switch ON-time to the total period is the **duty cycle (D)**:

```
Vout = D × Vin        →        D = Vout / Vin
```

**Example:** 12V out from 24V in → D = 12/24 = 0.5 (switch ON 50% of the time)

### Two phases each switching cycle:

```
Phase 1 — ON (High-side MOSFET closed):
  VIN ──► [High-side FET ON] ──► L1 ──► Vout
  Inductor current ramps UP. Energy stored in magnetic field.

Phase 2 — OFF (Low-side MOSFET closed):
  GND ──► [Low-side FET ON] ──► L1 ──► Vout
  Inductor releases energy. Current ramps DOWN to sustain output.
```

The output capacitors (C4, C5) smooth the ripple into stable DC.

### Feedback regulation loop:

```
Vout ──► [R1/R2 divider] ──► FB pin ──► Error amplifier
                                    compares to 1V Vref
                                              │
                                    adjusts duty cycle
                                              │
                                    Vout held constant 
```

If Vout droops (load increases) → FB drops below 1V → duty cycle increases → more energy delivered → Vout recovers. The entire correction happens within a few microseconds.

---

## 3. Key Concepts — LDO, VCC, and Bootstrap

### 3.1 Internal LDO and the VCC Pin

The LMR36520 contains an **internal 5V LDO regulator** that powers its own control circuits — the error amplifier, PWM logic, gate drivers, and protection circuits.

**Why is this needed?**
The converter accepts input voltages from 4.2V to 65V. The internal circuits cannot operate across that entire range directly — they need a stable, low voltage. The LDO solves this by producing a fixed 5V internally regardless of what VIN is.

```
VIN (4.2V–65V) ──► [Internal LDO] ──► 5V (VCC pin)
                                         │
                              Powers internal gate drivers,
                              PWM logic, error amplifier,
                              protection circuits
```

**VCC pin rules — CRITICAL:**

| ✅ Allowed uses of VCC                         | ❌ Do NOT connect to VCC              |
|------------------------------------------------|---------------------------------------|
| Pull-up resistor for the PG (power-good) pin   | Microcontrollers                      |
| Small bias circuits (< 1mA total)              | Relays or solenoids                   |
| C6 bypass capacitor (mandatory, 1µF to GND)    | Sensors or LED indicators             |
|                                                | Any external load > a few milliamps   |

> The internal LDO is not designed to source significant current. Overloading VCC will cause the IC to malfunction or enter thermal shutdown.

---

### 3.2 Bootstrap Circuit (BOOT pin + C3)

The bootstrap circuit solves a fundamental problem in buck converters: **how do you turn ON a high-side MOSFET whose source terminal is floating at the switching node (SW)?**

**The problem without bootstrap:**
```
To turn ON a MOSFET:  Vgate must be > Vsource + Vth
High-side MOSFET source = SW node = swings between 0V and VIN

When SW = VIN (48V), you need Vgate > 48V + 4V = 52V
But the IC only has VIN 48V) available.
→ IC cannot generate a voltage above VIN on its own.
→ High-side MOSFET cannot fully turn ON.
→ Converter does not work.
```

**How bootstrap solves it:**
```
Step 1 — Charging phase (when SW = 0V, low-side FET ON):
  VCC (5V) ──► [Internal diode] ──► C3 (100nF) ──► SW (0V)
  C3 charges to ≈ 5V

Step 2 — Driving phase (when SW rises to VIN):
  SW node rises to VIN (e.g. 48V)
  C3 bottom plate is now at 48V
  C3 top plate (BOOT pin) rises to 48V + 5V = 53V
  → Gate driver uses this 29V to fully switch ON the high-side MOSFET ✓
```

C3 acts as a **floating charge reservoir** that lifts itself above VIN each cycle to provide the gate drive energy the high-side switch needs.

**C3 requirements:**
- Value: **100nF** (as in this design)
- Must be a high-quality **ceramic (MLCC) X5R or X7R**
- Placed as close as physically possible to BOOT (pin 7) and SW (pin 8)
- A poor C3 placement causes high-side gate drive failure and converter malfunction

---

## 4. Component Description with Part Numbers

### IC — LMR36520FADDAR (Texas Instruments)

Fully integrated synchronous buck converter. Contains high-side FET, low-side FET, gate drivers, error amplifier, PWM comparator, internal LDO, soft-start, OCP, thermal shutdown, and UVLO — all in one HSOIC-8 package.

---

### C6 — VCC Bypass Capacitor

| Parameter       | Value                  |
|-----------------|------------------------|
| **Value**       | 1µF                    |
| **Mfr. Part**   | KGM21AR71C105JU        |
| **Package**     | 0402 MLCC              |
| **Voltage**     | 16V rated (C0G/NP0 or X5R) |
| **Function**    | Decouples the internal 5V LDO output on the VCC pin. Filters noise from the gate drive circuits. Required by the datasheet — do not omit. Place as close as possible to pin 6 and PGND. |

---

### L1 — Power Inductor

| Parameter          | Value                    |
|--------------------|--------------------------|
| **Value**          | 33µH                     |
| **Mfr. Part**      | MSS1278-333MLD (Bourns)  |
| **Package**        | 12.5 × 12.5mm SMD shielded |
| **Isat**           | Check datasheet — must be ≥ 6.5A |
| **DCR**            | As low as possible        |
| **Function**       | Core energy storage element. During ON phase, stores energy as magnetic flux and ramps current up. During OFF phase, releases that energy to sustain output current. The 33µH value sets the ripple current at the chosen switching frequency. |

> **Why shielded?** The MSS1278 is a shielded construction. This contains the magnetic field within the core, greatly reducing EMI radiated to nearby circuits.

---

### C1, C2 — Input Bypass Capacitors

| Parameter       | Value                    |
|-----------------|--------------------------|
| **Value**       | 4.7µF each (9.4µF total) |
| **Mfr. Part**   | KGM43JR72A475KV          |
| **Package**     | 1206 MLCC                |
| **Voltage**     | 100V rated (J = ±5% tol, R7 = X7R) |
| **Function**    | Placed directly at VIN (pin 2) and PGND (pin 1). Provide instantaneous charge during each switching transition and suppress high-frequency noise from propagating back to the input supply. The 100V rating gives wide margin for high input voltages. |

---

### C4, C5 — Output Filter Capacitors

| Parameter       | Value                              |
|-----------------|------------------------------------|
| **Value**       | 22µF each (44µF total in parallel) |
| **Mfr. Part**   | MBAST32MSB7226KPNA18               |
| **Package**     | 1210 MLCC                          |
| **Function**    | Smooth the switching current ripple from L1 into a stable DC output voltage. Two capacitors in parallel halve the effective ESR and improve ripple filtering. Note: MLCC capacitance de-rates with DC bias voltage — a 22µF cap may measure ~12–15µF at 12V DC bias, which is why two are used in parallel. |

---

### C3 — Bootstrap Capacitor

| Parameter       | Value               |
|-----------------|---------------------|
| **Value**       | 100nF               |
| **Mfr. Part**   | 530L104KT16         |
| **Package**     | 0402 MLCC           |
| **Function**    | Connected between BOOT (pin 7) and SW (pin 8). Provides the floating high-side gate drive voltage. |

---

### R1 — Lower Feedback Resistor

| Parameter       | Value                  |
|-----------------|------------------------|
| **Value**       | 9.09kΩ                 |
| **Mfr. Part**   | RE0402BRE079K09L       |
| **Package**     | 0402                   |
| **Tolerance**   | 0.1%                   |
| **Function**    | Lower resistor of the output voltage divider. Connected from FB pin (pin 5) to GND. Together with R2, programs the output voltage to 12V. High precision (0.1%) minimises output voltage error. |

---

### R2 — Upper Feedback Resistor

| Parameter       | Value       |
|-----------------|-------------|
| **Value**       | 100kΩ       |
| **Package**     | 0402        |
| **Function**    | Upper resistor of the output voltage divider. Connected from Vout to FB pin.|

---

### R3 — EN Pull-up Resistor

| Parameter       | Value       |
|-----------------|-------------|
| **Value**       | 100kΩ       |
| **Package**     | 0402        |
| **Function**    | Pulls the EN pin (pin 3) to VIN so the converter starts automatically when power is applied. Without this resistor, EN could float and the IC would not start reliably. If remote enable/disable control is needed, connect a logic signal or open-drain output here instead. |

---

## 5. IC Pin Functions

### LMR36520FADDAR — Pin Map

```
           ┌─────────────────┐
  PGND  1 ─┤                 ├─ 8  SW  ──► to L1
   VIN  2 ─┤  LMR36520       ├─ 7  BOOT ◄── C3 ──► SW
    EN  3 ─┤  FADDAR         ├─ 6  VCC  ──► C6 ──► GND
    PG  4 ─┤                 ├─ 5  FB   ◄── R1/R2 divider
           │       EP/PAD    │
           └────────┬────────┘
                    │
                  PGND (thermal + electrical)
```

### Detailed Pin Descriptions

| Pin | Name  | Type    | Description |
|-----|-------|---------|-------------|
| 1   | PGND  | Ground  | Power and analog ground reference. All voltages inside the IC are measured relative to this pin. Connect directly to the ground plane with short, wide, low-resistance traces. The input capacitors (C1, C2) and output capacitors (C4, C5) GND terminals must return to this pin without sharing trace resistance with other grounds. |
| 2   | VIN   | Power   | DC input supply. The raw input voltage enters here. Connect bypass capacitors C1 and C2 with the shortest possible traces from this pin to PGND. Absolute maximum: 38V. |
| 3   | EN    | Analog  | Enable control input. Logic HIGH → converter ON. Logic LOW → converter OFF and outputs discharge. Pulled to VIN via R3 (100kΩ) for always-on operation. Can be driven by an external logic signal for remote ON/OFF control. |
| 4   | PG    | Analog  | Power-Good open-drain flag. The internal open-drain transistor releases (PG goes HIGH via pull-up) when Vout is within regulation. Pulls LOW when Vout is out of regulation, during soft-start, or when EN is low.  If used, pull up to VCC (5V) through a resistor (e.g. 100kΩ) — do not pull up to VIN directly. |
| 5   | FB    | Analog  | Feedback voltage input. Connect to the midpoint of the R1/R2 voltage divider. The internal error amplifier regulates this pin to exactly **1.0V**.|
| 6   | VCC   | Power   | Output of the internal 5V LDO regulator. Supplies the IC's own gate drivers and control logic. Do not use to power external circuits except tiny loads < 1mA (e.g. PG pull-up resistor). |
| 7   | BOOT  | Power   | Bootstrap supply for the high-side gate driver. A 100nF capacitor (C3) must be connected from BOOT to SW. This capacitor charges through an internal diode when SW is low, then floats above VIN when SW rises, providing the above-VIN gate drive voltage needed to fully switch on the high-side MOSFET. |
| 8   | SW    | Power   | Switching node. This is where the internal high-side and low-side MOSFETs connect together, and where L1 connects. Swings between 0V and VIN at the switching frequency. This node has very high dV/dt. |
| EP  | PAD   | Thermal | Exposed thermal pad on the underside of the IC package.  This is the primary — and almost exclusive — heat dissipation path of the IC. Without a properly soldered EP, the IC will overheat and shut down at moderate loads. It is not a functional electrical pin (electrical characteristics are unspecified), but it must be connected to PGND. |

---

## 6. Output Voltage Setting

Output voltage is set by the R1/R2 resistor divider connected between Vout, FB, and GND:

```
Vout = Vref × (1 + R2/R1)

Where Vref = 1.0V (internal reference)
```

**This design — 12V output:**
```
Vout = 1.0 × (1 + 100,000 / 9,090)
     = 1.0 × (1 + 11.00)
     = 12.0V  ✓
```

**Other common output voltages:**

| Vout  | R2      | R1       | Notes                       |
|-------|---------|----------|-----------------------------|
| 3.3V  | 23.2kΩ  | 10kΩ     | Logic supply                |
| 5.0V  | 40.2kΩ  | 10kΩ     | USB / logic supply          |
| 12.0V | 100kΩ   | 9.09kΩ   | **This design**             |
| 15.0V | 140kΩ   | 10kΩ     | Op-amp supply               |
| 24.0V | 230kΩ   | 10kΩ     | Industrial actuator supply  |

> Use **0.1% or 1% tolerance** resistors for the feedback divider. 5% resistors will give ±5% output voltage error — unacceptable for most loads.



## 7. Protection Features

| Protection            | Trigger condition                      | Behaviour                                        |
|-----------------------|----------------------------------------|--------------------------------------------------|
| **Overcurrent (OCP)** | Peak inductor current exceeds limit    | Cycle-by-cycle current limiting; hiccup mode on sustained short circuit |
| **Thermal shutdown**  | Junction temperature ≈ 150°C           | IC shuts off all switching; restarts automatically when cooled |
| **UVLO**              | VIN drops below ~3.8V                 | Clean shutdown; prevents erratic operation at low input |
| **Soft-start**        | Power-up sequence                      | Internal ramp limits inrush current and prevents output overshoot on startup |
| **Power-Good (PG)**   | Vout out of regulation or EN = low     | Open-drain output pulls low to signal upstream controller or sequencer |

---

## 9. Efficiency Notes

Expected efficiency at 12V out, 5A, from 24V in: **~93–95%**

| Loss source              | Approximate share |
|--------------------------|-------------------|
| MOSFET switching loss    | ~40%              |
| MOSFET conduction loss   | ~30%              |
| Inductor DCR (winding)   | ~20%              |
| Gate drive + quiescent   | ~10%              |

At 2A full load, the IC dissipates approximately **0.5–1W** of heat. This exits almost entirely through the **EP thermal pad** into the PCB ground plane. Thermal resistance from junction to ambient (θJA) with a good thermal pad layout is approximately 20–35°C/W — meaning 1W of dissipation raises the die approximately 20–35°C above ambient.

> Ensure adequate copper area on the ground plane under the IC. Adding a small heatsink copper pour on the back of the board below the IC significantly reduces thermal resistance.

---
 
## 10. Design Tools & References

| Resource                  | Link / Details                                                  |
|---------------------------|-----------------------------------------------------------------|
| IC Datasheet              | TI LMR36520 — https://www.ti.com/product/LMR36520              |
| Component calculator      | TI WEBENCH Power Designer — https://webench.ti.com             |
| Inductor datasheet        | Bourns MSS1278-333MLD — https://www.bourns.com                 |
| Schematic / PCB tool      | Altium Designer                                                 |
| Input capacitor datasheet | KGM43JR72A475KV — Kyocera AVX                                  |
| Output capacitor          | MBAST32MSB7226KPNA18 — Vishay / Murata (verify series)         |
| Feedback resistor         | RE0402BRE079K09L — Yageo 0.1% precision series                 |


