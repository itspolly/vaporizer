# Safe Vaporizer Design Specification

## Overview

A safe, feature-rich vaporizer with USB-C charging (pass-through capable), Bluetooth connectivity, touchscreen UI, variable power output (1W-15W), and comprehensive hardware safety systems.

**Document Version: 3.13** — Critical fix: MAX17055 fuel gauge LOW-SIDE topology (previous HIGH-SIDE was wrong).

---

## 1. Power Requirements

### 1.1 Coil Power Matrix

| Coil R | Target Power | Required Voltage | Required Current |
|--------|--------------|------------------|------------------|
| 0.6Ω   | 1W           | 0.77V            | 1.29A            |
| 0.6Ω   | 15W          | 3.00V            | 5.00A            |
| 2.0Ω   | 1W           | 1.41V            | 0.71A            |
| 2.0Ω   | 15W          | 5.48V            | 2.74A            |
| 3.0Ω   | 1W           | 1.73V            | 0.58A            |
| 3.0Ω   | 15W          | 6.71V            | 2.24A            |

**Formulas:**
- V = √(P × R)
- I = √(P / R)

### 1.2 Design Parameters

- **Battery**: 1S LiPo, 3.0V (empty) to 4.2V (full), nominal 3.6V
- **Output Voltage Range**: 0.77V to 6.71V
- **Output Current Range**: 0.58A to 5.0A peak coil, **up to 6A battery** (boost efficiency losses)
- **Maximum Continuous Power**: 15W coil + ~1W system = 16W

### 1.3 Battery Current Budget (Critical)

At low battery voltage with boost active:
```
I_batt = P_out / (V_batt × η)
       = 15W / (3.2V × 0.90)
       = 5.2A from battery
```

**This informs:**
- BQ29700 variant selection (must tolerate >5A continuous)
- Trace/connector sizing (14 AWG minimum)
- Battery internal resistance requirements (<50mΩ)

---

## 2. System Architecture

```
                                    ┌─────────────────┐
                                    │   510 COIL      │
                                    │   (0.6-3Ω)      │
                                    └────────┬────────┘
                                             │
                              ┌──────────────┴──────────────┐
                              │      COIL DRIVER            │
                              │  ┌─────────────────────┐    │
                              │  │ Q3: High-Side PMOS  │◄───┼── SAFETY_GATE (inverted)
                              │  │ (Si7461DP, -60V/8A)│    │
                              │  └──────────┬──────────┘    │
                              │             │               │
                              │  ┌──────────▼──────────┐    │
                              │  │ R_SHUNT (5mΩ)       │    │
                              │  └──────────┬──────────┘    │
                              │             │               │
                              │  ┌──────────▼──────────┐    │
                              │  │ Q4: Low-Side NMOS   │◄───┼── PWM from ESP32
                              │  │ (CSD18563Q5A, 60V)  │    │
                              │  └──────────┬──────────┘    │
                              └─────────────┼───────────────┘
                                            │
┌─────────────┐    ┌─────────────┐    ┌─────▼─────┐
│   USB-C     │    │  BQ25896    │    │  BOOST    │
│  CONNECTOR  │───►│  CHARGER    │───►│  TPS61088 │
└─────────────┘    │  + PPATH    │    │  (adj)    │
                   └──────┬──────┘    └─────┬─────┘
                          │                 │
                   ┌──────▼──────┐          │
                   │  BQ29700    │◄─────────┘
                   │  PROTECTION │
                   └──────┬──────┘
                          │
                   ┌──────▼──────┐
                   │   BATTERY   │
                   │  1S 1000mAh │
                   └─────────────┘
```

---

## 3. Safety Logic Block Diagram

```
                           ┌─────────────────────────────────────────────┐
                           │            SAFETY GATE LOGIC                │
                           │                                             │
    ESP32 WDT_PING ───────►│ 555 TIMER ─────────┐                       │
    (GPIO, 100ms pulse)    │ (Watchdog)         │                       │
                           │                    ▼                       │
                           │              ┌───────────┐                 │
    FIRE_BUTTON ──────────►│─────────────►│  5-INPUT  │                 │
    (Active Low)           │              │   AND     │────────────────►│──► SAFETY_GATE
                           │              │   GATE    │                 │    (to TC4426)
    !OC_FAULT ────────────►│─────────────►│  (4x 2in) │                 │
    (From comparator,      │              └───────────┘                 │
     with 10kΩ pull-up)    │                    ▲                       │
                           │                    │                       │
    ESP32 FIRE_EN ────────►│────────────────────┤                       │
    (GPIO, with 10kΩ       │                    │                       │
     pull-DOWN for         │                    │                       │
     safe default)         │                    │                       │
                           │                    │                       │
    T_ALERT ──────────────►│────────────────────┘                       │
    (MCP9808, active LOW,  │     ⚠️ HARDWARE THERMAL INTERLOCK          │
     10kΩ pull-up)         │                                            │
                           └─────────────────────────────────────────────┘

TRUTH TABLE:
┌──────────┬────────────┬──────────┬──────────┬──────────┬─────────────┐
│ 555_OUT  │ FIRE_BTN   │ !OC_FAULT│ FIRE_EN  │ T_ALERT  │ SAFETY_GATE │
├──────────┼────────────┼──────────┼──────────┼──────────┼─────────────┤
│    0     │     X      │    X     │    X     │    X     │      0      │
│    1     │     1      │    X     │    X     │    X     │      0      │
│    1     │     0      │    0     │    X     │    X     │      0      │
│    1     │     0      │    1     │    0     │    X     │      0      │
│    1     │     0      │    1     │    1     │    0     │      0      │ ← THERMAL CUTOFF
│    1     │     0      │    1     │    1     │    1     │      1      │
└──────────┴────────────┴──────────┴──────────┴──────────┴─────────────┘

SAFETY_GATE = 555_OUT AND !FIRE_BTN AND !OC_FAULT AND FIRE_EN AND T_ALERT

⚠️ HARDWARE THERMAL PROTECTION (MCU-Independent):
  T_ALERT HIGH → Temperature < T_CRIT (85°C) → Normal operation
  T_ALERT LOW  → Temperature ≥ T_CRIT (85°C) → IMMEDIATE hardware cutoff

CRITICAL: TC4426 (inverting driver) converts:
  SAFETY_GATE HIGH → Q3_GATE LOW → PMOS ON (fires)
  SAFETY_GATE LOW  → Q3_GATE HIGH (V_BOOST) → PMOS OFF (safe)
```

---

## 3.1 Ground Reference Terminology

**⚠️ IMPORTANT: This design has multiple ground references due to LOW-SIDE battery protection.**

The BQ29705 protection FETs are in the **ground return path**, which creates two distinct ground references:

```
                    LOAD SIDE                           BATTERY SIDE
                    ─────────                           ────────────
                         │                                   │
                    System GND                           Battery-
                   (Q1:Source)                          (Q2:Source)
                         │                                   │
                         └──── Q1 ──── Q2 ────┬──── R_SENSE_FG ────┘
                               (CHG)   (DIS)  │         5mΩ
                                              │
                                          GND_SENSE
                                        (for FuelGauge)
```

| Term | Location | KiCad Symbol | Used By |
|------|----------|--------------|---------|
| **System GND** / **Sys GND** | Q1:Source (load side) | `GND` power symbol | ESP32, BQ25896, TPS61088, LDO, all ICs, all bypass caps |
| **GND_SENSE** | Q2:Source (between FETs and sense resistor) | Net label `GND_SENSE` | MAX17055 CSP pin only |
| **BAT_NEG** / **Battery-** | Battery negative terminal | Net label `BAT_NEG` | MAX17055 CSN pin, BQ29705 VSS, J3 pad 2 |

**In KiCad:**
- Use the standard **`GND` power symbol** for System GND — this is what you use 99% of the time
- Use **net labels** for `GND_SENSE` and `BAT_NEG` — these are only needed for fuel gauge connections

**When protection is ON:** System GND and BAT_NEG are connected through Q1+Q2 (~4mΩ Rds(on) total)

**When protection TRIPS:** System GND disconnects from BAT_NEG — battery is isolated, system powers down

---

## 4. Subsystem Designs

### 4.1 Battery Protection (BQ29700 + Dual MOSFET)

The BQ29700 provides:
- Overcharge protection (4.275V typical)
- Over-discharge protection (2.8V typical)
- Charge overcurrent protection
- Discharge overcurrent protection
- Short-circuit protection

**⚠️ CRITICAL: Variant Selection**

The BQ29700 family has multiple variants with different OCD thresholds:

| Variant  | OCD Threshold | SCD Threshold | Recommended? |
|----------|---------------|---------------|--------------|
| BQ29700  | 6A            | 20A           | ❌ May false-trip |
| BQ29705  | 10A           | 30A           | ✅ **Use this** |

**Use BQ29705DSET** for this design to avoid false trips during normal 5A+ operation.

**⚠️ BQ29705DSET Pin Mapping (DSBGA-6 Ball Designations):**

| Ball | Function | Description |
|------|----------|-------------|
| A1 | VSS | IC ground (connects to BAT-) |
| A2 | DOUT | Discharge FET gate driver output |
| B1 | BAT | Positive supply input (via R1 from BAT+) |
| B2 | COUT | Charge FET gate driver output |
| C1 | V- | OCD/SCD sense input (connects to Q1:Source) |
| C2 | NC | No connection |

**⚠️ KiCad Symbol Note:** Verify your KiCad symbol uses ball designations (A1, B1, etc.) not numeric pins. If the symbol uses numbers 1-6, map them per the datasheet.

The INA180 + comparator provides the *fast* overcurrent protection (µs response).
The BQ29705 provides *last-ditch catastrophic* protection only.

```
    ⚠️ LOW-SIDE PROTECTION TOPOLOGY (MOSFETs in ground path)

    N-channel MOSFETs require Vgs > Vth relative to SOURCE.
    BQ29705 is a low-side driver (no charge pump) - FETs MUST be in ground path.

    BAT+ pad ──┬─────────────────────────────────────► [VBAT_PROT]
               │  (direct connection to charger)        (to charger)
               │
               └── R1: 330Ω ──┬── BAT (U2:B1)  ⚠️ BAT = POSITIVE supply!
                              │
                              └── C_PROT: 100nF ── System GND

    BAT- pad ──┬── VSS (U2:A1)   (IC ground reference)
               │
               └──────────────────────────────────────┐
                                                      │
                    ┌─────────────────────────────────┤
                    │                                 │
                ┌───┴───┐                         ┌───┴───┐
                │ Q2    │                         │ Q1    │
                │ NMOS  │◄── R_G2 ◄── DOUT(A2)    │ NMOS  │◄── R_G1 ◄── COUT(B2)
                │(DIS)  │    (47Ω)                │(CHG)  │    (47Ω)
                └───┬───┘                         └───┬───┘
                    │                                 │
                    │ Source = BAT-                   │ Source = System GND
                    │                                 │
                    └──────────┬──────────────────────┘
                               │
                         Common Drain
                               │
                               └── (internal node)

    ⚠️ V- SENSING (OCD): U2:C1 (V-) ─── R2 (2.2kΩ) ─── Q1:Source (System GND)
       V- must connect to SYSTEM GND side, NOT BAT- side!
       This senses the voltage drop across Q1+Q2 during discharge.
       If V- connected to Q2:Source (= VSS = BAT-), OCD would NEVER trip!


    Current path when ON: BAT(-) → Q2 → Q1 → System GND → Load → VBAT_PROT → BAT(+)

    Fault response: COUT/DOUT → VSS (BAT-), Vgs ≈ 0V, FETs OFF, system isolated

    ⚠️ CRITICAL: BQ29705 BAT pin (pin 5) = POSITIVE power supply. Connect to BAT+!
                 VSS pin (pin 1) = Ground reference. Connect to BAT-.

Recommended MOSFETs: 2x CSD18534Q5A (N-ch, 25V, 43A, 2.2mΩ Rds(on)) in back-to-back config
```

### 4.2 Charging System (BQ25896)

The BQ25896 provides:
- USB input (4.4V to 14V)
- 3A max charge current
- Power path for use while charging
- I2C programmable
- Integrated ADC for monitoring
- NTC thermistor input

```
           USB-C VBUS
               │
    ┌──────────┴──────────┐
    │ Input Protection    │
    │ TVS: SMBJ15A        │
    │ (cathode to VBUS)   │
    │ C: 4.7µF + 100nF    │
    └──────────┬──────────┘
               │
         VBUS  │    ┌───────────────────────────────────────┐
               │    │           BQ25896                     │
               │    │                                       │
               └───►│ VBUS (1)                SYS (15,16) ├─────► VSYS (to boost)
                    │                                       │
    I2C_SDA ───────►│ SDA (6)                   BTST (21) │
    I2C_SCL ───────►│ SCL (5)                              │
                    │                          REGN (22) ├──┐
    ESP32_INT ◄────│ INT (7)                              │ │ 100nF
                    │                                       │ │
    NTC1 ──────────►│ TS (11)               BAT (13,14) ├──┴──► VBAT (protected)
                    │                                       │
    ESP32_OTG ─────►│ OTG_IUSB (8)             PMID (23)│
                    │                                       │
    ESP32_CE ──────►│ CE_N (9)            SW (19,20) ├──┤ L1: 2.2µH
                    │                                       │
                    │          ILIM (10)                   │
                    │            │                          │
                    │    ┌───────┴───────┐                 │
                    │    │ R_ILIM: 800Ω  │                 │
                    │    │ (1.3A limit)  │                 │
                    │    │ I=1040/R_ILIM │                 │
                    │    └───────┬───────┘                 │
                    │            ▼ GND                     │
                    │                                       │
                    │  PGND (17,18)                        │
                    └───────────────────────────────────────┘
```

**⚠️ Input Protection Note:** No series Schottky diode (e.g., SS34) is used on USB VBUS. A series diode would drop 0.2-0.4V, violating USB voltage spec (4.75-5.25V). The USB host provides reverse polarity protection. TVS diode orientation: cathode to VBUS rail, anode to GND.

**⚠️ C_USB1 Voltage Rating:** Use ≥25V rated capacitor for C_USB1. Even though we only request 5V, USB-C chargers can have transients, and a confused PD charger might briefly output higher voltage. A 16V cap risks overvoltage failure.

**I2C Address:** 0x6B

**⚠️ CRITICAL: Charge Current Settings (1C max for safety)**

| Register | Value | Function | Calculation |
|----------|-------|----------|-------------|
| 0x00     | 0x0F  | Input current limit 1.5A | REG00[5:0] × 100mA = 15 × 100mA |
| 0x02     | 0x10  | **Charge current 1.0A (1C rate)** | REG02[6:0] × 64mA = 16 × 64mA = 1024mA |
| 0x04     | 0xB2  | Charge voltage 4.2V | |
| 0x07     | 0x8D  | Enable charging, watchdog disabled | |

**⚠️ REG02 FORMULA:** `ICHG = REG02[6:0] × 64mA`. Common mistake: 0x20 (32) gives 2048mA = 2A!

**DO NOT use 2A charging on a 1000mAh cell — this is 2C and will cause thermal runaway.**

### 4.3 Fuel Gauge (MAX17055)

**⚠️ CRITICAL: MAX17055 Requires LOW-SIDE Current Sensing**

The MAX17055 fuel gauge uses a specific pin architecture:
- **BATT** = Power supply input (connects to Battery+, 2.3V–4.9V operating range)
- **CSP** = IC's internal ground reference (0V reference point)
- **CSN** = Sense resistor negative terminal (Kelvin connection)

The IC is powered by the voltage difference: **V_BATT − V_CSP = 2.3V to 4.9V**

**⚠️ The sense resistor MUST be in the ground return path (LOW-SIDE), NOT the battery+ path.**

If placed in the high-side (battery+ path), V_BATT and V_CSP would both be at battery voltage,
resulting in V_BATT − V_CSP ≈ 0V — the IC would not power up!

**Correct Topology:** Sense resistor in **ground return path**:
```
Battery+ → BQ29705 Protection → VBAT_PROT → BQ25896 BAT (direct connection)

Ground return path (where fuel gauge measures current):
System GND (loads) → Q1:Source → Q1 → Q2 → Q2:Source → R_SENSE_FG → Battery-
                                              ↑                        ↑
                                             CSP                      CSN
                                      (IC ground, GND_SENSE)    (sense negative, BAT_NEG)
```

```
                          VBAT_PROT (from protection) ──────────────► BQ25896 BAT
                                │
                                │ (direct connection, no sense resistor here)
                                │
    ┌───────────────────────────┴───────────────────────────────────────────────┐
    │                              MAX17055                                      │
    │                                                                            │
    │  BATT (B1) ◄───────────────────────────────────────────── VBAT_PROT       │
    │     │        (Power supply: V_BATT - V_CSP = ~3.7V)                        │
    │     │                                                                      │
    │     └── C_FG1 (100nF bypass to CSP)                                       │
    │                │                                                           │
    │                ▼                                                           │
    │  CSP (C3) ◄─────────────────────────────────────────────── GND_SENSE      │
    │     │        (IC ground reference, 0V)                     (Q2:Source)    │
    │     │                                                                      │
    │     └── R_SENSE_FG (5mΩ) ──┐                                              │
    │                            │                                               │
    │  CSN (A3) ◄────────────────┘ ────────────────────────────► BAT_NEG        │
    │              (Sense resistor negative, Kelvin tap)         (Battery-)     │
    │                                                                            │
    │  SCL (A2) ◄──────────────────────────────────────────────── I2C_SCL       │
    │  SDA (C1) ◄──────────────────────────────────────────────── I2C_SDA       │
    │  ALRT (B2) ───────────────────────────────────────────────► FG_ALRT       │
    │  THRM (C2) ─── NC (optional external NTC)                                 │
    │  REG (B3) ─── NC (internal regulator output, can bypass to CSP)           │
    │  AIN (A1) ─── NC (auxiliary input, unused)                                │
    │                                                                            │
    └────────────────────────────────────────────────────────────────────────────┘

Current flow (both directions measured via ground return path):
  DISCHARGE: Battery- ← R_SENSE_FG ← Q2:Source ← Q2 ← Q1 ← Q1:Source (SysGND) ← Load ← VSYS ← BQ25896 ← VBAT_PROT ← Battery+
  CHARGE:    Battery- → R_SENSE_FG → Q2:Source → Q2 → Q1 → Q1:Source (SysGND) → Load → VSYS → BQ25896 → VBAT_PROT → Battery+

During discharge: CSN is slightly BELOW CSP (negative current reading)
During charge:    CSN is slightly ABOVE CSP (positive current reading)
```

**I2C Address:** 0x36

**MAX17055 Calibration Procedure:**

The MAX17055 learns battery characteristics over several charge/discharge cycles. For fastest convergence:

**Initial Configuration (on first power-up):**
```c
#define MAX17055_ADDR       0x36
#define REG_DESIGN_CAP      0x18  // Design capacity (mAh / rsense_factor)
#define REG_I_CHG_TERM      0x1E  // Charge termination current
#define REG_V_EMPTY         0x3A  // Empty voltage target
#define REG_MODEL_CFG       0xDB  // Model configuration

void max17055_init(void) {
    // Wait for chip POR complete
    uint16_t status;
    do {
        i2c_read_word(MAX17055_ADDR, 0x00, &status);
    } while (status & 0x0002);  // Wait for POR bit to clear

    // Check if need to initialize (FSTAT.DNR = 1 means data not ready)
    uint16_t fstat;
    i2c_read_word(MAX17055_ADDR, 0x3D, &fstat);
    if (fstat & 0x0001) {
        // Configure for 1000mAh battery with 5mΩ sense resistor
        // DesignCap = 1000mAh * (5mΩ / 10mΩ) = 500 (0x01F4)
        i2c_write_word(MAX17055_ADDR, REG_DESIGN_CAP, 0x01F4);

        // IChgTerm = 50mA termination (0.05A / 0.15625mA = 320 = 0x0140)
        i2c_write_word(MAX17055_ADDR, REG_I_CHG_TERM, 0x0140);

        // VEmpty = 3.0V (3000mV / 0.078125mV = 38400 = 0x9600 in register format)
        i2c_write_word(MAX17055_ADDR, REG_V_EMPTY, 0x9600);

        // Trigger model load: Refresh=1, R100=0 (NTC disabled), VChg=0
        i2c_write_word(MAX17055_ADDR, REG_MODEL_CFG, 0x8000);

        // Wait for model load complete (Refresh bit clears)
        uint16_t cfg;
        do {
            vTaskDelay(pdMS_TO_TICKS(10));
            i2c_read_word(MAX17055_ADDR, REG_MODEL_CFG, &cfg);
        } while (cfg & 0x8000);
    }

    // Clear POR flag to prevent re-initialization
    i2c_read_word(MAX17055_ADDR, 0x00, &status);
    i2c_write_word(MAX17055_ADDR, 0x00, status & ~0x0002);
}
```

**Learning Cycle (for best accuracy):**
1. Fully charge battery (charge LED off)
2. Use device until it shuts down at 3.0V
3. Fully charge again
4. After 2-3 full cycles, MAX17055 learns battery characteristics

**Saved Learned Parameters:**
After learning, save these registers to NVS for faster boot:
- 0x0D (QRTable00), 0x15 (QRTable10), 0x22 (QRTable20), 0x32 (QRTable30)
- 0x23 (FullCapRep), 0x35 (FullCapNom), 0x45 (Cycles)

**UVLO Recovery Behavior:**

Since the MAX17055 BATT pin connects to VBAT_PROT (battery+) and CSP connects to System GND,
the fuel gauge loses power when the BQ29705 protection opens the ground path:

1. Battery drops below UVLO threshold (BQ29705 opens Q1/Q2)
2. System GND disconnects from Battery- → no current path → MAX17055 loses power
3. Device turns off completely

**Recovery:** When USB is plugged in:
- BQ25896 provides VSYS, BQ29705 may allow charging path
- Once protection clears, ground path restored, MAX17055 powers up
- MAX17055 reads battery OCV (Open Circuit Voltage)
- ModelGauge m5 algorithm estimates SOC from OCV within seconds
- **Result:** Battery percentage recovers almost immediately — no user-visible issue

This behavior is acceptable. The MAX17055's OCV-based estimation is accurate to ±2% for batteries at rest.

### 4.3.1 LDO Regulator (AP2112K-3.3)

The 3.3V LDO provides regulated power for all logic ICs.

**LDO Power Source:**

```
VSYS (from BQ25896) → LDO → VCC_3V3
```

The fuel gauge sense resistor is in the **ground return path** (between System GND and Battery-), measuring all current flow during both charging and discharging. The LDO connects directly to VSYS.

**Schematic Placement:** The LDO is placed on the MCU sheet for cleaner routing.

### 4.4 Boost Converter (TPS61088) with Pass-Through Mode

The TPS61088 provides adjustable output voltage via resistor divider feedback.

**⚠️ CRITICAL: True Load Disconnect Behavior**

The TPS61088 features **True Load Disconnect** during shutdown:
- When `EN = LOW`, the internal high-side MOSFET body diode is **blocked**
- There is **NO connection** between VIN and VOUT when disabled
- **DO NOT set EN=LOW expecting pass-through — the coil will receive 0V!**

| Target Voltage | Battery Voltage | Mode | EN Pin |
|----------------|-----------------|------|--------|
| V_target > V_batt | Any | **Boost Mode** | HIGH |
| V_target ≤ V_batt | Any | **Pass-Through** | **HIGH** (not LOW!) |

**Two-Mode Operation (EN = HIGH for both):**

1. **Boost Mode (V_target > V_batt):**
   - Set MCP4561 for target voltage
   - TPS61088 actively boosts
   - Q4 PWM for fine power control

2. **Pass-Through Mode (V_target ≤ V_batt):**
   - Set MCP4561 to **minimum voltage** (~2.5V, below battery)
   - TPS61088 detects V_IN > V_TARGET
   - Controller stops switching, turns high-side FET **fully ON** (100% duty)
   - Result: V_OUT ≈ V_IN (minus ~15mΩ Rds(on) drop)
   - Q4 PWM duty cycle controls average power to coil

```
    VSYS ────────────────────────────────────────────────────────────────┐
         │                                                                │
         │    ┌──────────────────────────────────────────────────────────┐│
         │    │                   TPS61088                               ││
         │    │                                                          ││
         └───►│ VIN (3,4)                              VOUT (17,18) ├────┼──► V_BOOST
              │                                                          ││   (1.5V-7V)
              │ EN (1) ◄─────────────────────── ESP32_BOOST_EN           ││
              │                                                          ││
              │ MODE (7) ─────────────────────► GND (PFM mode)           ││
              │                                                          ││
              │ SS (6) ───┬─── 22nF to GND (soft start)                  ││
              │           │                                              ││
              │ COMP (9) ─┼─── RC compensation network                   ││
              │           │                                              ││
              │ FB (8) ◄──┴─── Voltage divider network                   ││
              │                                                          ││
              │ SW (13,14,15,16) ──────┬─── L2: 1µH (15A sat)            ││
              │                        │                                  ││
              │ PGND (10,11,12) ───────┴───► GND                         ││
              │                                                          ││
              └──────────────────────────────────────────────────────────┘│
                                                                          │
                                                    ┌─────────────────────────┐
                                                    │   OUTPUT CAPACITORS     │
                                                    │   2× 22µF MLCC          │
                                                    │   + 100µF Electrolytic  │
                                                    └─────────────────────────┘
```

**Adjustable Output Voltage Circuit:**
```
    V_BOOST
       │
       ├───────────────────────────────────────────────┐
       │                                               │
    ┌──┴──┐                                        ┌───┴───┐
    │ R_T │ 100kΩ (top)                            │ C_OUT │
    └──┬──┘                                        └───────┘
       │
       ├───────────────────► FB (TPS61088)
       │
    ┌──┴──┐        ┌─────────────────┐
    │ R_B │ ◄──────┤ MCP4561-104     │◄── I2C from ESP32
    └──┬──┘        │ (Digital Pot)   │
       │           │ **100kΩ**, 257  │
       ├───────────┴─────────────────┘
       │
    ┌──┴──┐
    │R_SAFE│  22kΩ (FAILSAFE - limits V_OUT if pot fails open)
    └──┬──┘
       ▼
      GND
```

**⚠️ CRITICAL: Boost Overvoltage Failsafe (R_SAFE)**

If the MCP4561 wiper fails open (disconnects) or the I2C bus hangs with the pot in high-impedance state, the feedback network sees infinite bottom resistance. This causes V_OUT to rise uncontrollably toward the TPS61088's maximum voltage, potentially damaging the coil or causing thermal runaway.

**R_SAFE (22kΩ) in parallel with MCP4561 provides a ceiling:**
```
If pot fails open: R_B = R_SAFE = 22kΩ
V_OUT_MAX = 1.212V × (1 + 100kΩ / 22kΩ) = 1.212V × 5.55 = 6.7V
```
This limits overvoltage to ~6.7V even in complete pot failure — below the safe 7V maximum.

**⚠️ CRITICAL: Feedback Calculation**

The TPS61088 internal reference voltage is **1.212V** (not 0.6V):

```
V_OUT = V_REF × (1 + R_T / R_B)
V_OUT = 1.212V × (1 + 100kΩ / R_B)
```

| R_B (pot setting) | V_OUT | Mode |
|-------------------|-------|------|
| 100kΩ (max) | 2.42V | Pass-through (below V_BATT) |
| 50kΩ | 3.64V | Low boost |
| 26kΩ | 5.88V | Medium boost |
| 21kΩ | 6.99V | Max boost (~7V) |
| 17kΩ | 8.35V | ❌ Exceeds safe range |

**MCP4561-104 (100kΩ) is REQUIRED.** Do NOT use MCP4561-103 (10kΩ):
- With 10kΩ pot: V_min = 1.212V × 11 = **13.3V** (exceeds TPS61088 12.6V max!)
- With 100kΩ pot: V_min = 1.212V × 2 = **2.42V** ✓

**⚠️ MCP4561 Non-Volatile Memory Wear (Firmware Guidance):**

The MCP4561 has **1 million NVM write cycles**. Frequent saves can exhaust this:
- 100 adjustments/day × 365 = 36,500/year → 27+ years (acceptable)
- 1000 adjustments/day × 365 = 365,000/year → ~3 years (marginal)

**Recommended firmware approach:**
1. Use **volatile wiper register (0x00)** during operation — unlimited writes
2. Save to **NVM register (0x02)** only on:
   - Power-down (detect VSYS falling edge)
   - 30 seconds after last adjustment
   - Explicit user "save" action

### 4.5 555 Timer Watchdog

The 555 timer in monostable mode requires periodic retriggering from the ESP32. If the ESP32 crashes or hangs, the output goes LOW, disabling the coil.

```
                              VCC (3.3V)
                                  │
                    ┌─────────────┼─────────────┐
                    │             │             │
                    │    ┌────────┴────────┐    │
                    │    │ R_RST: 10kΩ     │    │ (pull-up on RST)
                    │    └────────┬────────┘    │
                    │             │             │
                    │    ┌────────┴────────┐    │
                    │    │      NE555      │    │
                    │    │                 │    │
    ESP32_WDT_PING ─┼───►│ TRIG (2) OUT (3)│──┬─┼──► 74LVC1G17 ──► WDT_OK
    (100ms pulse)   │    │                 │  │ │   (Schmitt buffer)
                    │    │ VCC (8) DIS (7)│──┤ │
                    │    │                 │  │ │
                    │    │ RST (4) THR (6)│──┼─┼─┐
                    │    │                 │  │ │ │
                    │    │ GND (1)  CV (5)│  │ │ │
                    │    └───────┬─────────┘  │ │ │
                    │            │            │ │ │
                    │            ▼ GND        │ │ │
                    │                         │ │ │
                    │    ┌───────────────────┐│ │ │
                    │    │ C_T: 10µF         ││◄┘ │
                    │    └───────┬───────────┘│   │
                    │            ▼ GND        │   │
                    │                         │   │
                    │    ┌───────────────────┐│   │
                    └────┤ R_T: 270kΩ        │◄───┘
                         └───────────────────┘

Time constant: T = 1.1 × R × C = 1.1 × 270kΩ × 10µF = **2.97 seconds** (~3s)

**⚠️ 555 Timing Tolerance Stack-up:**
| Parameter | Nominal | Worst Case (Fast) | Worst Case (Slow) |
|-----------|---------|-------------------|-------------------|
| R3 (±10%) | 270kΩ | 243kΩ | 297kΩ |
| C15 (±20%) | 10µF | 8µF | 12µF |
| **Timeout** | **2.97s** | **2.14s** | **3.92s** |

With ESP32 boot time of 1-2s, the worst-case fast timeout (2.14s) leaves only 0.14-1.14s margin for first WDT ping. **Recommendation:** Use 1% tolerance R3 for tighter control, or increase C15 to 15µF if margin is a concern during board bring-up.

ESP32 must pulse WDT_PING every 100-500ms to keep WDT_OK HIGH.
If ESP32 fails to ping for >3s, WDT_OK goes LOW → coil disabled.

**⚠️ BOOT TIME CONSIDERATION:**
ESP32-S3 with Wi-Fi/BT initialization can take 1-2s to boot. The original 1.65s
timeout risked expiring during boot if user held the fire button. The 3s timeout
provides adequate margin for boot + firmware initialization.

**Alternative (tighter timing):** Keep R3=150kΩ (1.65s) but ensure WDT_PING task
starts within 500ms of boot, BEFORE Wi-Fi/BT initialization.

ADDITIONS:
- 74LVC1G17 Schmitt trigger buffer after 555 for clean edges
- 10kΩ pull-up on RST ensures power-up default = LOW output
```

**Timing Requirements:**
- Monostable period: ~3 seconds (2.97s nominal)
- ESP32 ping interval: 100-500ms (with margin)
- Recovery: Automatic when pings resume
- Power-up state: LOW (safe) via RST pull-up timing

### 4.6 Short-Circuit / Overcurrent Detection

Uses a shunt resistor with INA180 current sense amplifier and LM393 comparator for fast analog protection.

**⚠️ CRITICAL FIXES Applied:**
1. **Changed INA180A3 (100V/V) → INA180A2 (50V/V)** for better headroom
2. **Added 10kΩ pull-up on LM393 output** (open-collector)
3. **Added RC filter (100Ω + 10nF)** before comparator for noise immunity
4. **Added hysteresis resistor (1MΩ)** to prevent oscillation
5. **Lowered threshold to 5A (2.5V)** for margin

```
    V_BOOST ──────────────────────────────────────────────────────────────┐
                │                                                          │
                ▼                                                          │
    ┌───────────────────┐                                                  │
    │   Q3: PMOS        │◄──────────────────────────── Q3_GATE (from TC4426)
    │   Si7461DP        │                                                  │
    │   (-60V, -8.6A)   │  ← UPGRADED from Si7309DN                        │
    │   Rds(on)=14.5mΩ  │                                                  │
    └─────────┬─────────┘                                                  │
              │                                                            │
              │ COIL_HIGH                                                  │
              │                                                            │
    ┌─────────┴─────────┐                                                  │
    │  R_SHUNT: 10mΩ    │──────┬──────────────────────────────────────────┤
    │  (2512, 3W)       │      │  ← Changed from 5mΩ for better sensing   │
    └─────────┬─────────┘      │                                          │
              │                │                                          │
              │ COIL_SENSE     │                                          │
              │                │                                          │
    ┌─────────┴─────────┐      │    ┌─────────────────────────────────────┐
    │  510 CONNECTOR    │      │    │        CURRENT SENSE                │
    │  (COIL)           │      │    │                                     │
    └─────────┬─────────┘      │    │    ┌───────────────┐                │
              │                │    │    │   INA180A2    │ ← 50V/V gain   │
              │                │    │    │   (50V/V)     │                │
    ┌─────────┴─────────┐      │    │    │               │                │
    │   Q4: NMOS        │      └───►│───►│ IN+ (1)       │                │
    │   CSD18563Q5A     │           │    │               │                │
    │   (60V, 100A)     │◄── PWM    │    │        OUT (4)│───┬──► I_SENSE │
    │   Rds(on)=2.3mΩ   │           │    │               │   │  (to ADC)  │
    └─────────┬─────────┘           │    │ IN- (2)       │   │            │
              │  ← Optimized FET    └───►│               │   │            │
              ▼                          │ GND (3)       │   │            │
             GND                         │ VCC (5) = 3.3V│   │            │
                                         └───────────────┘   │            │
                                                             │            │
                              ┌───────────────────────────────┘            │
                              │                                            │
                              ▼                                            │
                         ┌─────────┐                                       │
                         │R: 100Ω  │   ← RC filter for noise              │
                         └────┬────┘                                       │
                              │                                            │
                         ┌────┴────┐                                       │
                         │C: 10nF  │                                       │
                         └────┬────┘                                       │
                              │                                            │
                              ▼                                            │
                    ┌───────────────────┐                                  │
                    │      LM393        │                                  │
                    │   (Comparator)    │                                  │
                    │                   │                                  │
        V_THRESH ──►│ IN- (2)           │                                  │
        (2.5V for   │                   │                                  │
         5A limit)  │            OUT (1)│──┬──► !OC_FAULT (active LOW)     │
                    │                   │  │                               │
                    │ IN+ (3)  ◄────────┼──┘   ← 1MΩ hysteresis resistor   │
                    │                   │                                  │
                    │                   │    ┌─────────┐                   │
                    │                   │    │R: 10kΩ  │ ← PULL-UP         │
                    │                   │    └────┬────┘   (REQUIRED!)     │
                    │                   │         │                        │
                    │ GND (4)   VCC (8) │─────────┴───► VCC (3.3V)         │
                    └───────────────────┘                                  │
```

**Current Sense Calculations (INA180A2, 50V/V, 10mΩ shunt):**

| I_COIL | V_SHUNT | V_OUT (INA180) | Status |
|--------|---------|----------------|--------|
| 1A     | 10mV    | 0.5V           | OK     |
| 2A     | 20mV    | 1.0V           | OK     |
| 3A     | 30mV    | 1.5V           | OK     |
| 4A     | 40mV    | 2.0V           | OK     |
| 5A     | 50mV    | 2.5V           | TRIP   |
| 6A     | 60mV    | 3.0V           | TRIP   |

**Overcurrent Threshold:** Fixed at 5A (2.5V) via resistor divider:
```
V_THRESH = VCC × R2 / (R1 + R2)
         = 3.3V × 3.3kΩ / (1kΩ + 3.3kΩ)
         = 2.53V
```

**⚠️ LM393 Response Time Limitation:**

The LM393 has ~1.3µs response time. During a hard short circuit (e.g., liquid bridging coil terminals), current can spike to 20-30A in <1µs before the comparator trips.

**Mitigations:**
1. **INA180 internal clamping:** Output limited to VCC (3.3V)
2. **RC filter (100Ω × 10nF):** Adds ~1µs delay but filters switching noise
3. **Hardware AND gate:** 5-input safety chain — even if LM393 is slow, other paths can trip
4. **TPS61088 current limit:** 10A cycle-by-cycle limit at the source

**Future revision option:** TLV3201 (40ns response) for faster protection. Current design acceptable due to multi-layer approach.

**⚠️ Flyback Diode (D_FLY) - Inductive Kickback Protection:**

When Q3 or Q4 turns OFF rapidly (especially during thermal shutdown), the coil's inductance creates a voltage spike (V = L × dI/dt). D_FLY clamps this by providing a freewheeling path:

```
    COIL_HIGH (from R_SHUNT) ────┬──── 510 COIL+
                                 │
                              D_FLY (SS34)  ← Cathode toward COIL_HIGH
                                 │
    Q4:Drain ────────────────────┴──── 510 COIL-
```

**D_FLY:** SS34 (3A, 40V Schottky) — low Vf (~0.4V) for fast response.

Typical atomizer coils have low inductance (~10-50nH), but rapid shutdown can still generate 10-20V spikes. The Si7461DP body diode provides some protection, but SS34 is faster and reduces thermal stress.

### 4.7 Safety Gate Logic (AND Gate)

All four conditions must be TRUE for coil activation:

```
                                    VCC (3.3V)
                                        │
    ┌───────────────────────────────────┴───────────────────────────────┐
    │                                                                    │
    │                        74LVC08A (Quad 2-input AND)                │
    │                                                                    │
    │    ┌────────────────────────────────────────────────────────────┐ │
    │    │                                                            │ │
    │    │   WDT_OK ────────►│1A          │                           │ │
    │    │   (from Schmitt)  │     1Y ────┼──► STAGE1                 │ │
    │    │                   │1B          │                           │ │
    │    │   !FIRE_BTN ─────►│            │                           │ │
    │    │   (active HIGH    │            │                           │ │
    │    │    when pressed)  │            │                           │ │
    │    │                   │            │                           │ │
    │    │   STAGE1 ────────►│2A          │                           │ │
    │    │                   │     2Y ────┼──► STAGE2                 │ │
    │    │   !OC_FAULT ─────►│2B          │                           │ │
    │    │   (with 10kΩ      │            │                           │ │
    │    │    pull-up)       │            │                           │ │
    │    │                   │            │                           │ │
    │    │   STAGE2 ────────►│3A          │                           │ │
    │    │                   │     3Y ────┼──► SAFETY_GATE            │ │
    │    │   FIRE_EN ───────►│3B          │     (to TC4426 driver)    │ │
    │    │   (with 10kΩ      │            │                           │ │
    │    │    PULL-DOWN!)    │            │                           │ │
    │    │                   │            │                           │ │
    │    └────────────────────────────────────────────────────────────┘ │
    │                                                                    │
    └────────────────────────────────────────────────────────────────────┘
```

**⚠️ CRITICAL: FIRE_EN Pull-Down**

Add 10kΩ pull-down on FIRE_EN to ensure default = LOW (safe) when:
- ESP32 is resetting
- ESP32 GPIO is floating
- ESP32 has crashed

**Gate Driver for Q3 (PMOS) — TC4426 (INVERTING):**

```
    SAFETY_GATE ───────┐
                       │
                       ▼
                  ┌─────────────┐
                  │   TC4426    │  (Dual INVERTING MOSFET Driver)
                  │             │
    V_BOOST ─────►│ VDD         │
                  │             │
                  │ IN_A ───────┼◄── SAFETY_GATE
                  │             │
                  │ OUT_A ──────┼──► R_G3_HS (15Ω) ──┬──► Q3_GATE
                  │             │                    │
                  └─────────────┘                R_GS_Q3 (100kΩ)
                                                     │
                                                  Q3:Source (V_BOOST)

⚠️ R_G3_HS (15Ω): Gate resistor limits current and slows dV/dt to prevent gate ringing.
   Lower than Q4's R_G3 (47Ω) because TC4426 has stronger drive capability.

⚠️ R_GS_Q3 (100kΩ): Gate-to-Source pull-up ensures PMOS stays OFF during power-up.
   During power-up, V_BOOST is present before VCC_3V3 finishes ramping.
   Without R_GS_Q3, Q3:Gate could float low → PMOS ON → uncontrolled firing.
   With R_GS_Q3, Gate is pulled to Source → Vgs = 0 → PMOS guaranteed OFF.

⚠️ CRITICAL: TC4426 is INVERTING (not TC4427!)

When SAFETY_GATE = HIGH → TC4426 OUT = LOW → Q3 Vgs = -V_BOOST → PMOS ON
When SAFETY_GATE = LOW  → TC4426 OUT = HIGH (V_BOOST) → Q3 Vgs = 0 → PMOS OFF

This is FAIL-SAFE: any fault condition (low signal) turns PMOS OFF.
```

---

## 5. ESP32-S3-WROOM-1 Pin Assignments

### 5.1 GPIO Allocation

**⚠️ CRITICAL: Pin Selection Rules**
- Avoided strapping pins (GPIO0, GPIO3, GPIO45, GPIO46) for non-boot functions
- Avoided GPIO35-37 (Octal PSRAM interface on some modules)
- Using GPIO1/2 (UART0) for touch since USB-OTG handles communication

| GPIO | Function | Direction | Notes |
|------|----------|-----------|-------|
| **Display (SPI)** |
| GPIO10 | LCD_CLK | OUT | SPI2 CLK |
| GPIO11 | LCD_MOSI | OUT | SPI2 MOSI |
| GPIO12 | LCD_CS | OUT | Chip select |
| GPIO13 | LCD_DC | OUT | Data/Command |
| GPIO14 | LCD_RST | OUT | Reset |
| GPIO21 | LCD_BL | OUT | PWM backlight |
| **Touch (I2C0)** |
| GPIO8 | TP_SDA | I/O | I2C0 SDA |
| GPIO9 | TP_SCL | OUT | I2C0 SCL |
| GPIO1 | TP_RST | OUT | Touch reset (UART0 TX, safe after boot) |
| GPIO2 | TP_IRQ | IN | Touch interrupt (UART0 RX, safe after boot) |
| **Power Management (I2C1)** |
| GPIO17 | I2C1_SDA | I/O | BQ25896, MAX17055, MCP4561 |
| GPIO18 | I2C1_SCL | OUT | |
| **BQ25896 Charger** |
| GPIO38 | CHG_INT | IN | Interrupt |
| GPIO39 | CHG_CE | OUT | Charge enable |
| GPIO40 | CHG_OTG | OUT | OTG mode |
| GPIO15 | CHG_PG | IN | Power good (optional - for UI status) |
| GPIO45 | CHG_STAT | IN | Charge status (optional - for UI status) |
| **MAX17055 Fuel Gauge** |
| GPIO41 | FG_ALRT | IN | Alert interrupt |
| **Boost Converter** |
| GPIO42 | BOOST_EN | OUT | Enable TPS61088 |
| **Safety System** |
| GPIO48 | WDT_PING | OUT | 555 watchdog trigger |
| GPIO16 | FIRE_EN | OUT | Software fire enable **(10kΩ pull-DOWN!)** |
| GPIO6 | FIRE_BTN | IN | Fire button (ext. 10kΩ pullup to VCC_3V3) |
| GPIO7 | OC_FAULT | IN | Overcurrent fault (active low) |
| **Coil Control** |
| GPIO5 | COIL_PWM | OUT | PWM to Q4 gate (via R_G3 47Ω) |
| GPIO3 | I_SENSE | IN/ADC | Current sense **(ADC1_CH2)** |
| **Boost Voltage** |
| GPIO4 | V_SENSE | IN/ADC | Boost voltage monitor **(ADC1_CH3)** |
| **Misc** |
| GPIO47 | LED_STATUS | OUT | Fire button LED (via R22 → J5) |
| GPIO0 | BOOT | IN | Boot button (strapping - don't use for I/O) |

| GPIO19 | USB_D+ | I/O | USB |
| GPIO20 | USB_D- | I/O | USB |
| GPIO35-37 | - | - | **RESERVED for Octal PSRAM - DO NOT USE** |
| GPIO45 | VDD_SPI | - | **STRAPPING - DO NOT USE** |

| GPIO46 | LOG | - | **STRAPPING - DO NOT USE** |

**⚠️ ADC1 vs ADC2 CRITICAL:**
- I_SENSE and V_SENSE MUST use ADC1 pins (GPIO 1-10)
- ADC2 (GPIO 11-20) conflicts with Wi-Fi/Bluetooth and produces corrupt readings when radio is active
- GPIO3 is a strapping pin but can be used as ADC after boot (JTAG disabled in production)

**⚠️ GPIO3 Strapping Pin Verification (Board Bring-Up):**

GPIO3 selects JTAG mode at boot. The INA180 output connects via RC filter (R6 100Ω + C18 10nF).

**Expected:** At power-on, INA180 output ≈ 0V (no current), RC filter pulls GPIO3 low → normal boot.

**Verification during bring-up:**
1. Measure GPIO3 at power-on (should be <0.3V)
2. Confirm ESP32 enters normal boot (not JTAG mode)
3. If boot issues, use one of these mitigations:
   - **Option A:** Add 10kΩ pull-down on GPIO3
   - **Option B:** Add 1kΩ series resistor (R_ISO) between INA180:OUT and R6 to isolate strapping

### 5.2 I2C Device Addresses

| Device | Address | Bus |
|--------|---------|-----|
| CST816T (Touch) | 0x15 | I2C0 |
| BQ25896 (Charger) | 0x6B | I2C1 |
| MAX17055 (Fuel Gauge) | 0x36 | I2C1 |
| MCP4561 (Digital Pot) | 0x2E | I2C1 |
| MCP9808 (PCB Temp) | 0x18 | I2C1 |

**⚠️ I2C1 Bus Capacitance Note:**

I2C1 has 4 slave devices plus ESP32 (5 total). Each device adds ~10pF input capacitance.

| Parameter | Value |
|-----------|-------|
| Device capacitance (5 × 10pF) | ~50pF |
| PCB trace capacitance | ~10-20pF |
| **Total estimated** | **60-70pF** |
| 400kHz limit with 4.7kΩ pull-ups | 75pF |

**Status:** Marginal but should work. During board bring-up:
1. Verify I2C with oscilloscope (check rise times)
2. If issues occur, either:
   - Reduce to 100kHz Standard Mode, OR
   - Change R13-R16 to 2.2kΩ pull-ups (increases current but handles higher capacitance)

---

## 6. Firmware Safety Logic

### 6.0 GPIO Pin Definitions

All GPIO constants used throughout the firmware code:

```c
// =============================================================================
// GPIO PIN DEFINITIONS - ESP32-S3 (must match §5.1 allocation table!)
// =============================================================================

// Safety & Watchdog
#define GPIO_WDT_PING       48      // Output: 555 timer reset pulse (active LOW)
#define GPIO_FIRE_EN        16      // Output: Master enable to AND gate (10kΩ pull-DOWN!)
#define GPIO_FIRE_BTN        6      // Input: Fire button (ext. 10kΩ pullup to VCC_3V3, debounced)
#define GPIO_OC_FAULT        7      // Input: Overcurrent fault from INA180 (active LOW)

// Coil Driver Control
#define GPIO_COIL_PWM        5      // Output: LEDC PWM to TC4426 for Q4 gate (via R_G3)
#define GPIO_BOOST_EN       42      // Output: TPS61088 enable (active HIGH)

// Analog Inputs (ADC1 - safe to use with WiFi)
#define GPIO_I_SENSE         3      // ADC1_CH2: INA180 current sense output
#define GPIO_V_SENSE         4      // ADC1_CH3: Boost output voltage divider

// I2C0 Bus (Touch Panel)
#define GPIO_TP_SDA          8      // I2C0 data (touch)
#define GPIO_TP_SCL          9      // I2C0 clock (touch)
#define GPIO_TP_RST          1      // Touch reset
#define GPIO_TP_IRQ          2      // Touch interrupt

// I2C1 Bus (Power Management: BQ25896, MAX17055, MCP4561)
#define GPIO_I2C1_SDA       17      // I2C1 data
#define GPIO_I2C1_SCL       18      // I2C1 clock

// BQ25896 Charger
#define GPIO_CHG_INT        38      // Input: Charger interrupt
#define GPIO_CHG_CE         39      // Output: Charge enable
#define GPIO_CHG_OTG        40      // Output: OTG mode enable

// MAX17055 Fuel Gauge
#define GPIO_FG_ALRT        41      // Input: Alert interrupt

// Display (SPI2)
#define GPIO_LCD_CLK        10      // SPI2 CLK
#define GPIO_LCD_MOSI       11      // SPI2 MOSI
#define GPIO_LCD_CS         12      // Chip select
#define GPIO_LCD_DC         13      // Data/Command
#define GPIO_LCD_RST        14      // Reset
#define GPIO_LCD_BL         21      // PWM backlight

// Misc
#define GPIO_LED_STATUS     47      // Fire button LED (via R22 → J5)

// USB (directly to USB-C connector)
#define GPIO_USB_DP         19      // USB D+
#define GPIO_USB_DM         20      // USB D-

// Boot (strapping - don't use for general I/O)
#define GPIO_BOOT            0      // Boot button
```

**⚠️ CRITICAL Pin Notes:**
- ADC1 pins (GPIO1-10) only for analog - ADC2 conflicts with WiFi
- GPIO3 is strapping pin but safe for ADC after boot (JTAG disabled in production)
- GPIO35-37, GPIO45-46 are RESERVED (PSRAM/strapping) - never use
- FIRE_EN has external 10kΩ pull-down for fail-safe boot state

### 6.1 Watchdog Ping Task
```c
// FreeRTOS task - highest priority
void watchdog_ping_task(void *pvParameters) {
    const TickType_t xPeriod = pdMS_TO_TICKS(100);
    TickType_t xLastWakeTime = xTaskGetTickCount();

    while (1) {
        // Generate 10µs pulse on WDT_PING
        gpio_set_level(GPIO_WDT_PING, 0);  // Trigger is active LOW
        esp_rom_delay_us(10);
        gpio_set_level(GPIO_WDT_PING, 1);

        vTaskDelayUntil(&xLastWakeTime, xPeriod);
    }
}
```

### 6.2 Fire Sequence (with Two-Mode Boost Control)

**⚠️ CRITICAL: TPS61088 EN must stay HIGH during firing!**
Setting EN=LOW causes True Load Disconnect (0V to coil).

**⚠️ PWM Frequency: Use ≥20kHz** to avoid audible noise from coil/MOSFET switching.

**⚠️ LEDC Initialization (Required):**
```c
// Call once during system init
void init_coil_pwm(void) {
    ledc_timer_config_t timer_conf = {
        .speed_mode = LEDC_LOW_SPEED_MODE,
        .duty_resolution = LEDC_TIMER_10_BIT,  // 1023 max duty
        .timer_num = LEDC_TIMER_0,
        .freq_hz = 25000,  // 25kHz PWM (above audible range)
        .clk_cfg = LEDC_AUTO_CLK
    };
    ledc_timer_config(&timer_conf);

    ledc_channel_config_t channel_conf = {
        .gpio_num = GPIO_COIL_PWM,
        .speed_mode = LEDC_LOW_SPEED_MODE,
        .channel = LEDC_CHANNEL_0,
        .timer_sel = LEDC_TIMER_0,
        .duty = 0,
        .hpoint = 0
    };
    ledc_channel_config(&channel_conf);
}
```

**⚠️ Soft-Start Timing Coordination:**

The TPS61088 has its own internal soft-start controlled by C17 (22nF):
- Internal SS time ≈ C_SS × V_REF / I_SS ≈ 22nF × 1.2V / 2µA ≈ **13ms**

Firmware soft-start (20ms PWM ramp) must begin **after** TPS61088 output stabilizes.
Add a delay after enabling BOOST_EN before starting PWM ramp:

```c
gpio_set_level(GPIO_BOOST_EN, 1);  // Enable TPS61088
vTaskDelay(pdMS_TO_TICKS(20));     // Wait for internal soft-start + settling
// Then begin firmware soft-start PWM ramp...
```

```c
#define MAX_FIRE_DURATION_MS 10000  // 10 second hard limit
#define MIN_BATTERY_VOLTAGE  3.2f   // Brown-out threshold
#define PASSTHROUGH_VOLTAGE  2.5f   // Below min battery = forces pass-through
#define MAX_BOOST_VOLTAGE    7.0f   // Safety clamp (never exceed)

esp_err_t fire_coil(uint8_t power_watts, uint16_t duration_ms) {
    // 0. Enforce maximum fire duration
    if (duration_ms > MAX_FIRE_DURATION_MS) {
        duration_ms = MAX_FIRE_DURATION_MS;
    }

    // 1. Check battery voltage (brown-out protection)
    float vbat = read_battery_voltage();
    if (vbat < MIN_BATTERY_VOLTAGE) {
        return ESP_ERR_INVALID_STATE;  // Battery too low
    }

    // 2. Verify no faults
    if (gpio_get_level(GPIO_OC_FAULT) == 0) {
        return ESP_ERR_INVALID_STATE;  // Overcurrent fault active
    }

    // 3. Verify button pressed (hardware also checks this)
    if (gpio_get_level(GPIO_FIRE_BTN) == 1) {
        return ESP_ERR_INVALID_STATE;  // Button not pressed
    }

    // 4. Calculate target voltage and current
    float coil_resistance = measure_coil_resistance();
    if (coil_resistance < 0.5f || coil_resistance > 3.5f) {
        return ESP_ERR_INVALID_ARG;  // Invalid coil
    }

    float target_voltage = sqrtf(power_watts * coil_resistance);
    if (target_voltage > MAX_BOOST_VOLTAGE) {
        target_voltage = MAX_BOOST_VOLTAGE;  // Safety clamp
    }
    float expected_current = sqrtf((float)power_watts / coil_resistance);

    // 5. Determine operating mode and configure boost
    // CRITICAL: EN must be HIGH in BOTH modes!
    bool use_boost = (target_voltage > vbat);

    if (use_boost) {
        // Boost mode: Set target voltage via digital pot
        set_boost_voltage(target_voltage);
    } else {
        // Pass-through mode: Set voltage BELOW battery to force 100% duty
        // TPS61088 will turn high-side FET fully ON, passing V_IN to V_OUT
        set_boost_voltage(PASSTHROUGH_VOLTAGE);
    }

    // Enable boost converter (MUST be HIGH for both modes!)
    gpio_set_level(GPIO_BOOST_EN, 1);
    vTaskDelay(pdMS_TO_TICKS(20));  // Wait for TPS61088 internal soft-start (~13ms) + settling

    // Verify boost is working
    float v_boost = read_boost_voltage();
    if (use_boost && fabsf(v_boost - target_voltage) > 0.5f) {
        gpio_set_level(GPIO_BOOST_EN, 0);
        return ESP_ERR_TIMEOUT;  // Boost failed to reach target
    }

    // 6. Calculate PWM duty cycle
    // Use PWM in BOTH modes for fine power control
    float v_out = use_boost ? target_voltage : vbat;
    float p_peak = (v_out * v_out) / coil_resistance;
    float duty_ratio;

    if (use_boost && p_peak <= power_watts) {
        // Boost mode at target voltage: 100% duty
        duty_ratio = 1.0f;
    } else {
        // PWM modulation for power control
        // P_avg ≈ P_peak × duty² (approximate for resistive load)
        duty_ratio = sqrtf((float)power_watts / p_peak);
    }

    uint32_t target_duty = (uint32_t)(duty_ratio * 1023);
    if (target_duty > 1023) target_duty = 1023;
    if (target_duty < 10) target_duty = 10;  // Minimum duty to prevent stalling

    // 7. Enable firing (AND gate input) BEFORE PWM starts
    gpio_set_level(GPIO_FIRE_EN, 1);

    // 8. SOFT-START: Ramp PWM from 0 to target over 20ms
    // Prevents inrush current spike when Q3 turns on (coil looks like short initially)
    #define SOFT_START_MS       20      // Total ramp time
    #define SOFT_START_STEPS    10      // Number of steps
    #define SOFT_START_DELAY_MS (SOFT_START_MS / SOFT_START_STEPS)

    for (int step = 1; step <= SOFT_START_STEPS; step++) {
        uint32_t ramp_duty = (target_duty * step) / SOFT_START_STEPS;
        ledc_set_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0, ramp_duty);
        ledc_update_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0);
        vTaskDelay(pdMS_TO_TICKS(SOFT_START_DELAY_MS));

        // Check for faults during ramp
        if (gpio_get_level(GPIO_OC_FAULT) == 0) {
            gpio_set_level(GPIO_FIRE_EN, 0);
            ledc_set_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0, 0);
            ledc_update_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0);
            return ESP_ERR_INVALID_STATE;  // OC fault during soft-start
        }
    }

    // 9. Wait for duration or fault condition
    TickType_t start = xTaskGetTickCount();
    while ((xTaskGetTickCount() - start) < pdMS_TO_TICKS(duration_ms)) {
        if (gpio_get_level(GPIO_FIRE_BTN) == 1) break;  // Button released
        if (gpio_get_level(GPIO_OC_FAULT) == 0) break;  // Fault detected

        // Check for thermal or brown-out
        if (read_battery_voltage() < MIN_BATTERY_VOLTAGE - 0.2f) break;

        vTaskDelay(pdMS_TO_TICKS(10));
    }

    // 10. Disable firing (order matters!)
    gpio_set_level(GPIO_FIRE_EN, 0);              // Disable safety gate first
    ledc_set_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0, 0);
    ledc_update_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0);
    gpio_set_level(GPIO_BOOST_EN, 0);             // Disable boost last

    return ESP_OK;
}
```

### 6.3 Coil Resistance Measurement

**⚠️ CRITICAL: Accurate coil measurement is essential for power control and safety.**

The coil resistance must be measured before firing to:
1. Calculate correct voltage/current for target power
2. Detect open circuit (no coil) or short circuit (damaged coil)
3. Validate coil is within safe operating range (0.5Ω - 3.5Ω)

```c
#define MEASURE_VOLTAGE     1.0f    // Low test voltage (safe for measurement)
#define MEASURE_DUTY        128     // ~12.5% duty for low power
#define MEASURE_SETTLE_MS   50      // Settling time
#define MIN_VALID_RESISTANCE 0.5f
#define MAX_VALID_RESISTANCE 3.5f

float measure_coil_resistance(void) {
    // 1. Ensure boost is disabled (use battery voltage directly)
    gpio_set_level(GPIO_BOOST_EN, 0);
    vTaskDelay(pdMS_TO_TICKS(10));

    // 2. Read battery voltage for reference
    float v_batt = read_battery_voltage();
    if (v_batt < 3.2f) {
        return -1.0f;  // Battery too low for measurement
    }

    // 3. Enable firing gate (allows current flow)
    gpio_set_level(GPIO_FIRE_EN, 1);

    // 4. Apply low-duty PWM pulse
    ledc_set_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0, MEASURE_DUTY);
    ledc_update_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0);
    vTaskDelay(pdMS_TO_TICKS(MEASURE_SETTLE_MS));

    // 5. Read current via INA180 (multiple samples for accuracy)
    float i_sum = 0;
    for (int i = 0; i < 10; i++) {
        i_sum += read_current_sense();  // Returns amps from ADC
        vTaskDelay(pdMS_TO_TICKS(1));
    }
    float i_avg = i_sum / 10.0f;

    // 6. Stop PWM immediately
    ledc_set_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0, 0);
    ledc_update_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0);
    gpio_set_level(GPIO_FIRE_EN, 0);

    // 7. Calculate resistance
    // V_applied ≈ V_batt × (duty/1023) × duty_factor
    // Note: Actual voltage depends on Q3 Rds(on) and trace resistance
    float duty_factor = (float)MEASURE_DUTY / 1023.0f;
    float v_applied = v_batt * duty_factor;

    if (i_avg < 0.01f) {
        return 999.9f;  // Open circuit (no coil detected)
    }

    float r_coil = v_applied / i_avg;

    // 8. Validate range
    if (r_coil < MIN_VALID_RESISTANCE) {
        return 0.0f;   // Short circuit or damaged coil
    }
    if (r_coil > MAX_VALID_RESISTANCE) {
        return 999.9f; // Open circuit or very high resistance
    }

    return r_coil;
}
```

**Measurement Accuracy Notes:**
- Use ESP32 ADC calibration (`esp_adc_cal`) for I_SENSE readings
- Account for Q3 Rds(on) (~4.6mΩ) and trace resistance (~10mΩ)
- Perform measurement at room temperature (coil resistance varies with temp)
- Store calibration offset in NVS for production units

### 6.4 ESP32 ADC Calibration

**⚠️ ESP-IDF VERSION REQUIREMENT:** This code uses the `adc_cali_create_scheme_curve_fitting()` API from **ESP-IDF v5.0+**. For ESP-IDF v4.x, use `esp_adc_cal_characterize()` instead (see fallback below).

**⚠️ HIGH PRIORITY: ESP32-S3 ADC has ±5% non-linearity without calibration.**

For safety-critical measurements (I_SENSE, V_SENSE), uncalibrated ADC can cause:
- Incorrect power delivery (±0.75W error at 15W)
- OC threshold drift (could miss fault conditions)
- Inaccurate battery percentage

**Required Calibration Steps:**

```c
#include "esp_adc/adc_cali.h"
#include "esp_adc/adc_cali_scheme.h"

static adc_cali_handle_t adc1_cali_handle = NULL;

esp_err_t init_adc_calibration(void) {
    // Use curve fitting calibration (most accurate for ESP32-S3)
    adc_cali_curve_fitting_config_t cali_config = {
        .unit_id = ADC_UNIT_1,
        .atten = ADC_ATTEN_DB_12,    // Full 0-3.3V range
        .bitwidth = ADC_BITWIDTH_12,
    };

    esp_err_t ret = adc_cali_create_scheme_curve_fitting(&cali_config, &adc1_cali_handle);
    if (ret != ESP_OK) {
        ESP_LOGE(TAG, "ADC calibration failed - using uncalibrated readings!");
    }
    return ret;
}

// Convert raw ADC reading to calibrated millivolts
int read_calibrated_mv(adc_channel_t channel) {
    int raw;
    adc_oneshot_read(adc1_handle, channel, &raw);

    int voltage_mv;
    if (adc1_cali_handle) {
        adc_cali_raw_to_voltage(adc1_cali_handle, raw, &voltage_mv);
    } else {
        // Fallback: linear approximation (less accurate)
        voltage_mv = (raw * 3300) / 4095;
    }
    return voltage_mv;
}
```

**ESP-IDF v4.x Fallback:**
```c
#if ESP_IDF_VERSION < ESP_IDF_VERSION_VAL(5, 0, 0)
#include "esp_adc_cal.h"
static esp_adc_cal_characteristics_t adc1_chars;

esp_err_t init_adc_calibration_v4(void) {
    esp_adc_cal_characterize(ADC_UNIT_1, ADC_ATTEN_DB_11, ADC_WIDTH_BIT_12, 1100, &adc1_chars);
    return ESP_OK;
}

int read_calibrated_mv_v4(adc_channel_t channel) {
    int raw = adc1_get_raw(channel);
    return esp_adc_cal_raw_to_voltage(raw, &adc1_chars);
}
#endif
```

**Production Calibration Procedure:**
1. Apply known reference voltage (e.g., 1.000V from precision source)
2. Read ADC and compute offset: `offset = expected_mv - measured_mv`
3. Store offset in NVS: `nvs_set_i32(handle, "adc_offset", offset)`
4. Apply offset in `read_calibrated_mv()` for production accuracy

**Alternative: External ADC**
For production units requiring ±1% accuracy, consider:
- ADS1115 (16-bit, I2C, 4-channel) — LCSC C37593
- MCP3421 (18-bit, I2C, single-channel) — LCSC C67999

These are overkill for prototyping but recommended for mass production safety certification.

### 6.5 PCB Thermal Monitoring and Derating (MCP9808)

The MCP9808 I2C temperature sensor near Q3/Q4 monitors MOSFET junction temperature via PCB thermal coupling.

**Advantages over NTC + ADC:**
- No ADC2/Wi-Fi conflict — pure digital I2C
- ±0.25°C accuracy (vs ±2-5% for NTC + ESP32 ADC)
- Hardware alert pin for immediate over-temp shutdown
- Simpler firmware (no Steinhart-Hart math)

**I2C Address:** 0x18 (on I2C1 bus with BQ25896, MAX17055, MCP4561)

**Temperature Reading:**
```c
#define MCP9808_ADDR        0x18
#define MCP9808_REG_TEMP    0x05  // Ambient temperature register
#define MCP9808_REG_CONFIG  0x01  // Configuration register
#define MCP9808_REG_TUPPER  0x02  // Upper limit register
#define MCP9808_REG_TCRIT   0x04  // Critical limit register

float read_pcb_temperature(void) {
    uint8_t data[2];
    i2c_read_bytes(MCP9808_ADDR, MCP9808_REG_TEMP, data, 2);

    // Convert to temperature (12-bit, 0.0625°C resolution)
    uint16_t raw = (data[0] << 8) | data[1];

    // Clear flag bits (upper 3 bits)
    raw &= 0x1FFF;

    float temp_c;
    if (raw & 0x1000) {
        // Negative temperature
        raw = (~raw & 0x0FFF) + 1;
        temp_c = -((float)raw / 16.0f);
    } else {
        temp_c = (float)raw / 16.0f;
    }

    return temp_c;
}

void mcp9808_init(void) {
    // ⚠️ CRITICAL: Configure for HARDWARE THERMAL INTERLOCK
    // T_ALERT routes to SafetyLogic AND gate - MCU-independent cutoff

    // Set critical temperature limit (85°C) for HARDWARE CUTOFF
    // 85°C = 85 * 16 = 1360 = 0x0550
    uint8_t tcrit[2] = {0x05, 0x50};
    i2c_write_bytes(MCP9808_ADDR, MCP9808_REG_TCRIT, tcrit, 2);

    // Set upper limit (70°C) for SOFTWARE warning only
    // 70°C = 70 * 16 = 1120 = 0x0460
    uint8_t tupper[2] = {0x04, 0x60};
    i2c_write_bytes(MCP9808_ADDR, MCP9808_REG_TUPPER, tupper, 2);

    // Configure ALERT for hardware interlock:
    // Bit 0 = 0: Comparator mode (auto-clear when temp drops)
    // Bit 1 = 0: Active LOW (correct for AND gate - LOW blocks)
    // Bit 2 = 1: CRIT_ONLY (only T_CRIT triggers ALERT, not T_UPPER)
    // Bit 3 = 1: Alert enabled
    // Config = 0x000C
    uint8_t config[2] = {0x00, 0x0C};
    i2c_write_bytes(MCP9808_ADDR, MCP9808_REG_CONFIG, config, 2);
}

// ⚠️ Power-on defaults before MCU init:
// T_CRIT = 95°C, T_UPPER = 80°C - hardware still protected if MCU fails to boot
```

**Thermal Derating (in fire_coil loop):**
```c
#define TEMP_DERATE_START   70.0f   // Begin derating at 70°C (matches T_UPPER)
#define TEMP_DERATE_FULL    85.0f   // Zero power at 85°C (matches T_CRIT)
#define TEMP_HYSTERESIS      5.0f   // 5°C hysteresis

static bool thermal_lockout = false;

float apply_thermal_derate(float requested_power, float pcb_temp) {
    // Hysteresis for lockout
    if (pcb_temp >= TEMP_DERATE_FULL) {
        thermal_lockout = true;
        return 0.0f;  // Full shutdown
    }
    if (thermal_lockout && pcb_temp < (TEMP_DERATE_FULL - TEMP_HYSTERESIS)) {
        thermal_lockout = false;  // Release lockout
    }
    if (thermal_lockout) {
        return 0.0f;
    }

    // Linear derating between start and full
    if (pcb_temp <= TEMP_DERATE_START) {
        return requested_power;  // Full power allowed
    }

    float derate_range = TEMP_DERATE_FULL - TEMP_DERATE_START;
    float derate_factor = 1.0f - ((pcb_temp - TEMP_DERATE_START) / derate_range);
    if (derate_factor < 0.0f) derate_factor = 0.0f;

    return requested_power * derate_factor;
}
```

**Hardware Alert (optional but recommended):**
Connect MCP9808 ALERT pin to GPIO15 for hardware interrupt on over-temp:
```c
void IRAM_ATTR thermal_alert_isr(void *arg) {
    // Critical temperature exceeded - immediate shutdown
    gpio_set_level(GPIO_FIRE_EN, 0);      // Kill software enable
    gpio_set_level(GPIO_BOOST_EN, 0);     // Kill boost
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;
    xEventGroupSetBitsFromISR(event_group, THERMAL_ALERT_BIT, &xHigherPriorityTaskWoken);
}

// In init:
gpio_set_intr_type(GPIO_T_ALERT, GPIO_INTR_NEGEDGE);  // Alert is active-low
gpio_install_isr_service(0);
gpio_isr_handler_add(GPIO_T_ALERT, thermal_alert_isr, NULL);
```

**Integration with fire_coil():**
```c
float pcb_temp = read_pcb_temperature();
float derated_power = apply_thermal_derate(user_power_setting, pcb_temp);
if (derated_power < user_power_setting * 0.5f) {
    // Show warning on display: "DEVICE HOT - Power Limited"
}
// Use derated_power for PWM calculation
```

### 6.6 OC_FAULT State Machine

The hardware overcurrent comparator (LM393 + INA180) provides fast fault detection (<10µs).
Firmware manages fault recovery with hysteresis to prevent oscillation.

**State Machine:**
```
                    ┌──────────────────────────────────────────┐
                    │                                          │
                    ▼                                          │
    ┌──────────┐  OC_FAULT=LOW   ┌──────────────┐             │
    │  NORMAL  │ ───────────────►│ FAULT_ACTIVE │             │
    │  STATE   │                 │   (PWM=0)    │             │
    └──────────┘                 └──────┬───────┘             │
         ▲                              │                      │
         │                              │ OC_FAULT=HIGH        │
         │                              │ for 100ms            │
         │                              ▼                      │
         │                       ┌──────────────┐              │
         │                       │ FAULT_CLEAR  │              │
         │    retry_count < 3    │  (waiting)   │              │
         └───────────────────────└──────┬───────┘              │
                                        │                      │
                                        │ OC_FAULT=LOW again   │
                                        ▼                      │
                                 ┌──────────────┐              │
                                 │   LOCKOUT    │──────────────┘
                                 │ (3 retries)  │   user clears
                                 └──────────────┘   by power cycle
```

**Firmware Implementation:**
```c
typedef enum {
    OC_STATE_NORMAL,
    OC_STATE_FAULT_ACTIVE,
    OC_STATE_FAULT_CLEAR,
    OC_STATE_LOCKOUT
} oc_state_t;

static oc_state_t oc_state = OC_STATE_NORMAL;
static uint8_t oc_retry_count = 0;
static TickType_t oc_fault_time = 0;

#define OC_DEBOUNCE_MS      5      // Fault must persist 5ms
#define OC_CLEAR_DELAY_MS   100    // Wait 100ms after fault clears
#define OC_MAX_RETRIES      3      // Lock out after 3 faults

void oc_fault_handler(void) {
    bool fault_active = (gpio_get_level(GPIO_OC_FAULT) == 0);
    TickType_t now = xTaskGetTickCount();

    switch (oc_state) {
        case OC_STATE_NORMAL:
            if (fault_active) {
                oc_fault_time = now;
                oc_state = OC_STATE_FAULT_ACTIVE;
                // Immediately kill PWM (hardware AND already does this)
                ledc_set_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0, 0);
                ledc_update_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0);
                gpio_set_level(GPIO_FIRE_EN, 0);  // Also kill software enable
            }
            break;

        case OC_STATE_FAULT_ACTIVE:
            if (!fault_active) {
                oc_fault_time = now;
                oc_state = OC_STATE_FAULT_CLEAR;
            }
            // Keep PWM at 0
            break;

        case OC_STATE_FAULT_CLEAR:
            if (fault_active) {
                // Fault returned during clear delay
                oc_retry_count++;
                if (oc_retry_count >= OC_MAX_RETRIES) {
                    oc_state = OC_STATE_LOCKOUT;
                    // Display: "OVERCURRENT LOCKOUT - Remove atomizer"
                } else {
                    oc_state = OC_STATE_FAULT_ACTIVE;
                }
            } else if ((now - oc_fault_time) > pdMS_TO_TICKS(OC_CLEAR_DELAY_MS)) {
                // Fault cleared for sufficient time
                oc_state = OC_STATE_NORMAL;
                // Resume allowed (user must press fire again)
            }
            break;

        case OC_STATE_LOCKOUT:
            // Remain locked until power cycle
            // Could add atomizer removal detection here
            break;
    }
}

// Call this from main loop or ISR every 1-5ms
bool is_oc_fault_active(void) {
    oc_fault_handler();
    return (oc_state != OC_STATE_NORMAL);
}
```

**Hardware Interaction:**
- OC_FAULT is **active LOW** (pulled up by R10)
- Hardware AND gate (U7) blocks SAFETY_GATE when OC_FAULT=LOW
- Firmware monitors GPIO7 for state transitions
- After fault clears, firmware must re-assert FIRE_EN to resume

---

## 7. Bill of Materials (BOM)

### 7.1 JLCPCB-Sourced Components (SMD Assembly)

| Ref | Description | Package | Value | LCSC Part # | Qty |
|-----|-------------|---------|-------|-------------|-----|
| **Power ICs** |
| U1 | Battery Charger | QFN-24 | BQ25896RTWT | C92494 | 1 |
| U2 | Battery Protection | DSBGA-6 (1.5x1.5mm) | **BQ29705DSET** | C191565 | 1 |
| U3 | Fuel Gauge | WLP-9 | MAX17055EWL+T | C191519 | 1 |
| U4 | Boost Converter | QFN-20 | TPS61088RHLR | C130123 | 1 |
| U5 | MCU | Module | ESP32-S3-WROOM-1-**N8R2** (Quad SPI only!) | C2913202 | 1 |
| **Safety Logic ICs** |
| U6 | 555 Timer | SOIC-8 | NE555DR | C7416 | 1 |
| U7 | Quad AND Gate | TSSOP-14 | 74LVC08APW | C10481 | 1 |
| U8 | Current Sense Amp | SOT-23-5 | **INA180A2IDBVR** | C122881 | 1 |
| U9 | Comparator | SOIC-8 | LM393DR | C7950 | 1 |
| U10 | Gate Driver | SOIC-8 | **TC4426ACOA** | C130253 | 1 |
| U11 | Digital Pot (I2C) | SOIC-8 | **MCP4561-104E/MS** (100kΩ!) | C130982 | 1 |
| U12 | Schmitt Buffer | SOT-23-5 | 74LVC1G17GW | C10426 | 1 |
| U13 | Single Inverter (FIRE_BTN) | SOT-23-5 | 74LVC1G04GW | C10421 | 1 |
| U14 | 3.3V LDO Regulator | SOT-23-5 | AP2112K-3.3TRG1 | C51118 | 1 |
| **Power MOSFETs** |
| Q1 | N-ch MOSFET (Charge) | SON 5x6mm | CSD18534Q5A | C151530 | 1 |
| Q2 | N-ch MOSFET (Discharge) | SON 5x6mm | CSD18534Q5A | C151530 | 1 |
| Q3 | P-ch MOSFET (High-side) | PowerPAK SO-8 | **Si7461DP-T1-GE3** | C236078 | 1 |
| Q4 | N-ch MOSFET (Low-side) | SON-8 | **CSD18563Q5A** | C136441 | 1 |
| Q_REV | P-ch MOSFET (Reverse protection) | PowerPAK SO-8 | **Si7461DP-T1-GE3** | C236078 | 1 |
| **Inductors** |
| L1 | Charger Inductor | 7×7mm | 2.2µH, 4A | C408412 | 1 |
| L2 | Boost Inductor | 7×7mm | 1µH, **25A sat** (Coilcraft XAL7070-102MEC) | C6609502 | 1 |
| **Capacitors (Ceramic MLCC)** |
| C_USB1 | VBUS Bulk Cap | 0805 | 4.7µF **25V** X5R | C1835 | 1 |
| C_USB2 | VBUS HF Bypass | 0402 | 100nF 16V | C1525 | 1 |
| C1-C4 | Input/Output Caps | 0805 | 22µF 10V X5R | C45783 | 4 |
| C_SYS | SYS Output Cap | 0805 | 22µF 10V X5R | C45783 | 1 |
| C5-C10 | Bypass Caps | 0402 | 100nF 16V | C1525 | 6 |
| C11-C14 | Bulk Caps | 0805 | 10µF 16V | C15850 | 4 |
| C15 | 555 Timing Cap | 0805 | 10µF 16V | C15850 | 1 |
| C_CV | 555 CV Filter | 0402 | 10nF 16V | C15195 | 1 |
| C16 | Boost Output | 1206 | 100µF 10V | C106076 | 1 |
| C17 | Soft Start | 0402 | 22nF 16V | C1516 | 1 |
| C_COMP | TPS61088 Compensation C | 0402 | 10nF 16V | C15195 | 1 |
| C18 | Comparator Filter | 0402 | 10nF 16V | C15195 | 1 |
| C_PROT | BQ29705 BAT Filter | 0402 | 100nF 16V | C1525 | 1 |
| C_PMID | BQ25896 PMID Decouple | 0402 | 100nF 16V | C1525 | 1 |
| C_SCHMITT | 74LVC1G17 Bypass | 0402 | 100nF 16V | C1525 | 1 |
| C_INV | 74LVC1G04 Bypass | 0402 | 100nF 16V | C1525 | 1 |
| C_DEBOUNCE | FIRE_BTN Debounce | 0402 | 100nF 16V | C1525 | 1 |
| C_LDO_IN | LDO Input Cap | 0805 | 10µF 16V | C15850 | 1 |
| C_LDO_OUT | LDO Output Cap | 0805 | 10µF 16V | C15850 | 1 |
| C_VSENSE | V_SENSE Filter Cap | 0402 | 100nF 16V | C1525 | 1 |
| **Resistors** |
| R1 | BQ29705 BAT | 0402 | 330Ω | C25104 | 1 |
| R2 | BQ29705 V- Sense (OCD) | 0402 | 2.2kΩ | C25879 | 1 |
| R_G1 | Q1 Gate Resistor | 0402 | 47Ω | C25118 | 1 |
| R_G2 | Q2 Gate Resistor | 0402 | 47Ω | C25118 | 1 |
| R_G3 | Q4 Gate Resistor | 0402 | 47Ω | C25118 | 1 |
| R_G3_HS | Q3 Gate Resistor | 0402 | 15Ω | C25092 | 1 |
| R_GS_Q3 | Q3 Gate-Source Pull-up | 0402 | 100kΩ | C25741 | 1 |
| R_REV | Q_REV Gate Pull-down | 0402 | 10kΩ | C25744 | 1 |
| R_QON | BQ25896 QON_N Pull-up | 0402 | 10kΩ | C25744 | 1 |
| R3 | 555 Timing | 0603 | **270kΩ** (3s timeout) | C25795 | 1 |
| R4 | Boost FB Top | 0402 | 100kΩ | C25741 | 1 |
| R_SAFE | Boost FB Failsafe (∥ MCP4561) | 0402 | 22kΩ | C25768 | 1 |
| R_COMP | TPS61088 Compensation R | 0402 | 100kΩ | C25741 | 1 |
| R5 | ILIM | 0402 | **800Ω** (I_INLIM=1040/R=1.3A) | C25779 | 1 |
| R_SHUNT | Current Sense (INA180) | 2512 | **10mΩ 3W** | C211070 | 1 |
| R_SENSE_FG | Fuel Gauge Sense (MAX17055) | 2512 | **5mΩ 1% 1W** | C211069 | 1 |
| R6 | Comparator Filter | 0402 | 100Ω | C25076 | 1 |
| R7 | Comparator Thresh (top) | 0402 | 1kΩ | C25084 | 1 |
| R8 | Comparator Thresh (bot) | 0402 | 3.3kΩ | C25892 | 1 |
| R9 | Comparator Hysteresis | 0603 | 1MΩ | C22935 | 1 |
| R10 | LM393 Pull-up | 0402 | 10kΩ | C25744 | 1 |
| R11 | FIRE_EN Pull-down | 0402 | 10kΩ | C25744 | 1 |
| R12 | 555 RST Pull-up | 0402 | 10kΩ | C25744 | 1 |
| R_DEBOUNCE | FIRE_BTN Debounce | 0402 | 10kΩ | C25744 | 1 |
| R_PU_BTN | FIRE_BTN Pull-up | 0402 | 10kΩ | C25744 | 1 |
| R_ALERT | T_ALERT Pull-up (MCP9808) | 0402 | 10kΩ | C25744 | 1 |
| R13-R16 | I2C Pull-ups | 0402 | 4.7kΩ | C25900 | 4 |
| R19 | TS Divider Top (RT1) | 0402 | 5.23kΩ (5.1kΩ ok) | C25908 | 1 |
| R20 | TS Divider Bottom (RT2) | 0402 | 26.7kΩ (27kΩ ok) | C25959 | 1 |
| R21 | 510 ESD Bleed (optional) | 0603 | 1MΩ | C22935 | 1 |
| R22 | Fire Button LED Resistor | 0402 | 220Ω | C25091 | 1 |
| R_DIV1 | V_SENSE Divider Top | 0402 | 100kΩ 1% | C25741 | 1 | ⚠️ |
| R_DIV2 | V_SENSE Divider Bottom | 0402 | 47kΩ 1% | C25819 | 1 | ⚠️ |

**⚠️ R_DIV1/R_DIV2/C_VSENSE Schematic Placement:** Place on **MCU sheet** near GPIO4 for proper decoupling and short ADC traces. The signal originates from BoostConverter but the divider components should be physically and schematically near the ADC input.
| **Sensors** |
| U15 | PCB Temp Sensor | MSOP-8 | MCP9808T-E/MS | C127401 | 1 |
| C_MCP | MCP9808 Bypass | 0402 | 100nF | C1525 | 1 |
| C_FG1 | MAX17055 VDD Bypass | 0402 | 100nF | C1525 | 1 |
| **Diodes** |
| D2 | TVS Diode (VBUS) | SMB | SMBJ15A | C83594 | 1 |
| D_FLY | Flyback Diode (Coil) | SMA | SS34 (3A 40V Schottky) | C22452 | 1 |
| **Connectors** |
| J2 | USB-C Receptacle | SMD | USB4110-GF-A | C2688619 | 1 |
| J3 | Battery Pads | Solder pads | BAT+/BAT- (14 AWG) | — | 1 |
| J4 | Display FPC | 12-pin 0.5mm | AFC07-S12 | C262270 | 1 |
| J5 | Fire Button LED | 2-pin JST-XH (male) | B2B-XH-A | C158012 | 1 |
| J6 | 510 Connector Pads | 2x Solder Pad (14AWG) | — | — | 1 |
| J7 | NTC Thermistor Pads | Solder pads | TS/GND (22 AWG) | — | 1 |
| **ESD Protection** |
| D3 | USB ESD | SOT-23 | TPD2E001DRLR | C92469 | 1 |
| **USB-C Configuration** |
| R17, R18 | CC1, CC2 Resistors | 0402 | 5.1kΩ | C25905 | 2 |

### 7.2 User-Sourced Components (Hand Solder)

| Ref | Description | Source | Price | Notes |
|-----|-------------|--------|-------|-------|
| BAT1 | Turnigy 1000mAh 1S 20C LiPo | [HobbyKing](https://hobbyking.com/en_us/turnigy-1000mah-1s-20c-lipoly-single-cell.html) | ~£10 | Solder tabs, wire to J3 pads |
| J1 | MM510 V2 Standard SS/16mm×0.5mm | [ModMaker](https://www.modmaker.com/mm510-v2-standard-ss16mm05mm) | £9.50 | 510 connector, crimp male bullet connectors to wires |
| DISP1 | Waveshare 1.69" Touch LCD | [Waveshare](https://www.waveshare.com/1.69inch-touch-lcd-module.htm) | £15 | ST7789V2 + CST816T |
| SW1 | Fire Button (16mm Green LED) | [ModMaker](https://www.modmaker.com/16mm-green-led-stainless-steel-illuminated-switch-with-flat-flush-actuator) | £3.75 | Panel mount, wire LED to JST-XH-F |
| NTC1 | 10K NTC Thermistor (B=3435) | See note below | £3-5 | BQ25896-compatible, wire to J7 |
| - | 3.5mm Bullet Connectors (M+F pairs) | Amazon UK / RC shop | £5 | 2 pairs needed: female on PCB wires, male on 510 wires |
| - | JST-XH 2-pin Female Housing | Amazon UK | £2 | Mates with J5, for LED wiring |
| - | 14 AWG Silicone Wire (Red+Black) | Amazon UK | £8 | Battery + 510 connection |
| - | 22 AWG Hook-up Wire | Amazon UK | £5 | Button/NTC/LED wiring |
| - | Kapton Tape + Thermal Paste | Amazon UK | £5 | NTC mounting |

**⚠️ NTC Thermistor Compatibility (CRITICAL)**

The BQ25896 TS pin requires a specific NTC thermistor configuration for JEITA temperature profiling:

| Parameter | Requirement | Notes |
|-----------|-------------|-------|
| Resistance @ 25°C | **10kΩ ±1%** | Must match internal thresholds |
| Beta (B25/85) | **3435K ±1%** | Critical for accurate temp sensing |
| Reference Part | Semitec 103AT-2 | TI's recommended thermistor |
| Package | Radial leaded or probe | For battery contact |

**Compatible Alternatives:**
- **Murata NCP18XH103F03RB** (0603 SMD, 10K, B=3380) — close enough
- **Vishay NTCLE413E2103F520L** (radial, 10K, B=3435) — exact match
- **Generic MF52-103 10K 3435** — budget option, verify accuracy

**TS Pin Circuit (Required Resistor Divider):**
```
    REGN (3.3V from BQ25896)
         │
    ┌────┴────┐
    │ RT1     │  ← Sets T1/T5 cold thresholds
    │ 5.23kΩ  │     (calculate per datasheet)
    └────┬────┘
         │
         ├────────► TS (BQ25896 pin 9)
         │
    ┌────┴────┐
    │  NTC    │  ← 10K @ 25°C, B=3435
    │  10kΩ   │
    └────┬────┘
         │
    ┌────┴────┐
    │ RT2     │  ← Sets T2/T4 warm thresholds
    │ 26.7kΩ  │     (calculate per datasheet)
    └────┬────┘
         │
         ▼
        GND
```

**JEITA Temperature Thresholds (with 103AT-2):**
| Threshold | Temperature | Action |
|-----------|-------------|--------|
| T1 (COLD) | 0°C | Suspend charging |
| T2 (COOL) | 10°C | Reduce charge current |
| T3 (NORMAL) | 25°C | Full charge current |
| T4 (WARM) | 45°C | Reduce charge voltage |
| T5 (HOT) | 60°C | Suspend charging |

**Installation:**
1. Attach NTC directly to battery cell with Kapton tape
2. Apply thermal paste between NTC and cell for good contact
3. Route wires away from heat sources (boost inductor, MOSFETs)
4. Verify TS voltage reads ~1.5-2.0V at room temperature

**⚠️ NTC Disconnect Detection (Firmware Guidance):**

If the NTC wire breaks or comes loose, the TS pin floats high (pulled by RT1 to REGN). The BQ25896 interprets this as "very cold" battery and may continue full-rate charging.

**Firmware safety check:** Read TS voltage via BQ25896 ADC register (0x12):
- **Normal range:** 0.5V - 2.5V (temperature-dependent)
- **Abnormal HIGH (>2.8V):** NTC disconnected → reduce charge current via I2C
- **Abnormal LOW (<0.3V):** NTC shorted → reduce charge current via I2C

The BQ25896 has internal limits, but firmware check provides defense-in-depth.

**Estimated Total BOM Cost:** ~£70-85 (excluding PCB fabrication)

---

## 8. PCB Design Guidelines

### 8.1 Layer Stack-up (4-Layer, High-Current Optimized)

Given the high currents (15A+ peaks in the boost loop), this specific stack-up is recommended:

```
Layer 1 (Top):     Components, High-Current Traces (Boost/Coil), Safety Logic signals
Layer 2 (Inner 1): SOLID GROUND PLANE — Do NOT break this plane!
Layer 3 (Inner 2): Power Plane (VBAT distribution, VSYS pours)
Layer 4 (Bottom):  Non-critical signals + DUPLICATE high-current traces from Top
```

**Via Stitching Strategy:**
- Use **4mm wide traces with 2oz copper** for high-current paths — this handles 6A+ with acceptable temperature rise
- Stitch the Top and Bottom high-current paths (Battery → Boost → Coil) together with **via rows** (multiple vias in a line along the trace)
- This doubles current capacity and halves trace resistance
- Use 0.3mm vias; place 4-6 vias in a row at each layer transition (a single via only handles ~1A)
- Via row layout: vias spaced ~1mm apart along the trace length, not across the width

**Alternative: Polygon Pours (Copper Zones)**

For maximum current capacity, **polygon pours** can be used instead of or in addition to wide traces:

| Approach | Best For | Via Style |
|----------|----------|-----------|
| 4mm traces | Most high-current routing | Via rows (4-6 in a line) |
| Polygon pours | GND plane, battery return, converging nets | Via farms (grid array) |

**When to prefer pours:**
- GND return path (Layer 2 is typically a solid ground plane)
- Battery negative return area near J3
- Areas where multiple high-current nets converge (e.g., near Q4:Source)
- When board space allows large copper areas

**Via farm for pours:**
```
████████████████████████
██  ○  ○  ○  ○  ○  ○  ██   ← grid pattern within pour
██  ○  ○  ○  ○  ○  ○  ██
████████████████████████
```

Both approaches are valid — choose based on your layout constraints. The key requirement is **2oz copper** and **multiple vias at layer transitions**.

### 8.2 Star Ground at Q4 Source (Critical for OCP Accuracy)

This design uses a **HIGH-SIDE current shunt** (between Q3:Drain and the coil). The shunt **floats** at V_BOOST potential and does NOT connect to ground.

**⚠️ CRITICAL:** The shunt must NOT connect to ground. Connecting it to GND would short V_BOOST to ground!

```
                              HIGH-SIDE SHUNT TOPOLOGY

    V_BOOST ──► Q3:Source ──► Q3:Drain ──► [R_SHUNT] ──► COIL_HIGH ──► 510 Coil
                                               ↑
                                    Shunt FLOATS at V_BOOST potential
                                    INA180 senses differential voltage
                                               │
                                               ▼
                              ┌────────────────────────────────┐
                              │          R_SHUNT (10mΩ)        │
                              │                                │
                   CS_P ──────┤ HIGH SIDE                      ├────── CS_N
                   (to INA180 │   PAD                LOW SIDE  │  (to INA180
                      IN+)    │                        PAD     │     IN-)
                              └────────────────────────────────┘
                                               │
                              ┌────────────────┴────────────────┐
                              │   Kelvin traces (10-15 mil)     │
                              │   Route as differential pair    │
                              │   NAME: CS_P and CS_N           │
                              └────────────────┬────────────────┘
                                               │
                              ┌────────────────┴────────────────┐
                              │          INA180A2               │
                              │   IN+              IN-          │
                              │                                 │
                              │          GND ───────────────────┼──► AGND
                              └─────────────────────────────────┘     (local)
                                                                        │
                                                                    NET TIE
                                                                        │
                                                                        ▼
    ════════════════════════════════════════════════════════════════════════
    ║                         STAR POINT at Q4:SOURCE                      ║
    ║              (Net Tie joins AGND to GND at this point)               ║
    ════════════════════════════════════════════════════════════════════════
                              │                │                │
                         ┌────┴────┐      ┌────┴────┐      ┌────┴────┐
                         │POWER GND│      │ANALOG GND│     │DIGITAL GND│
                         │Q4 Source│      │INA180,   │     │ESP32,     │
                         │Battery- │      │LM393     │     │74LVC08A   │
                         └─────────┘      └──────────┘     └───────────┘
                              │
                              ▼
                       BATTERY NEGATIVE
                    (4mm trace or polygon pour)
```

**Routing Strategy:**
1. **High-Current Path:** Q4 Source → Battery Negative. Use **4mm trace with 2oz copper** (or polygon pour) for 6A return path.
2. **Kelvin Connection:** Two thin (10-15 mil) traces from **inside** the Shunt pads to INA180 inputs. Route as differential pair (CS_P, CS_N). Connect **nothing else** to these traces.
3. **The Star Point:** Net Tie connecting **AGND** (INA180/LM393 ground) to system **GND** at **Q4:Source**.
4. **System Ground:** All digital grounds (ESP32, 555, Boost) connect to ground plane that ties back to Q4:Source.

**Why the Net Tie?**
- GND carries the full 6A coil current back to battery (robust power path)
- AGND carries only milliamps of reference current for INA180/LM393
- Joining them at Q4:Source via Net Tie prevents switching noise from corrupting ADC readings

**⚠️ DO NOT connect the shunt to any ground — it floats at V_BOOST potential!**

### 8.3 TPS61088 Boost Converter Layout

The TPS61088 is a high-frequency (~1MHz) switcher handling high currents.

**The "Hot Loop" (Most Critical):**
```
                    ┌─────────────────────────────────────┐
                    │         TPS61088                    │
                    │                                     │
         L2 ───────►│ SW              VOUT ───────────────┼──┐
                    │                                     │  │
                    │                 PGND ───────────────┼──┼──┐
                    │                                     │  │  │
                    └─────────────────────────────────────┘  │  │
                                                             │  │
    ┌────────────────────────────────────────────────────────┘  │
    │                                                           │
    │    ┌─────────────────────────────────────────────────┐    │
    │    │           OUTPUT CAPACITORS                     │    │
    │    │   C16 (100µF) + C1-C4 (22µF each)               │    │
    │    │                                                 │    │
    │    │   Place IMMEDIATELY adjacent to VOUT and PGND!  │    │
    │    └─────────────────────────────────────────────────┘    │
    │                              │                            │
    └──────────────────────────────┴────────────────────────────┘
                            MINIMIZE THIS LOOP AREA
                         (Acts as antenna for EMI)
```

**SW Node Copper:**
- The SW pin swings from 0V to ~7V in **nanoseconds**
- Keep this copper area **small** to minimize radiated EMI
- But **wide** enough to handle the current
- **DO NOT** pour a massive plane for SW node — just a thick, short trace to inductor pad

**Thermal Vias (Critical):**
- TPS61088 thermal pad requires **9-12 vias** (0.3mm hole)
- Connect to solid ground plane on bottom layer
- This chip **WILL** get hot at 15W output

### 8.4 Current Sense Amplifier (INA180) Isolation

**Differential Pair Routing:**
- Route `IN+` and `IN-` traces as a **differential pair** (parallel, close together, matched length)
- Any noise picked up will be "common mode" — which the amplifier rejects
- Keep traces 10-15 mil wide (current is µA, not amps)
- Avoid vias in sense path if possible

**Ground Moat (EMI Isolation):**
- Keep switching node (TPS61088 SW) **far away** from sense lines
- Place a **Ground Pour "wall"** between the Boost section and Safety Logic section
- This prevents switching noise from coupling into the overcurrent detection

**⚠️ CRITICAL: E-Liquid Isolation (Milling Slots)**

E-juice is conductive and can track across PCB surfaces, especially on humid/dirty boards. If liquid bridges the 510 connector positive to USB/Logic ground, the coil driver will AUTO-FIRE without user input (ground reference for FIRE_BTN).

**Required PCB Layout Features:**
```
┌─────────────────────────────────────────────────────────────────┐
│                         PCB Top View                            │
│                                                                 │
│   ┌───────────┐                                                 │
│   │ 510 AREA  │                                                 │
│   │ (J6 pads) │        ████████████                            │
│   │ HIGH VOLT │        █ MILLING █                            │
│   └───────────┘        █  SLOT   █  ← 2mm wide slot through    │
│                        ████████████    PCB (no copper/FR4)     │
│                                                                 │
│   ┌───────────┐        ┌────────────────────────────────────┐  │
│   │ USB AREA  │        │     LOW VOLTAGE LOGIC AREA         │  │
│   │ (J2)      │        │     (MCU, Safety Logic, etc)       │  │
│   └───────────┘        └────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Isolation Requirements:**
| Feature | Requirement |
|---------|-------------|
| Milling slot | 2mm minimum width, through all layers |
| 510 to USB clearance | 10mm minimum surface distance |
| 510 to Logic GND | 8mm minimum surface distance |
| Conformal coating | **MANDATORY** on all exposed copper |
| Silkscreen warning | "⚠️ KEEP DRY" near 510 pads |

**KiCad Implementation:**
1. Draw slot as **Edge.Cuts** rectangle (2mm wide)
2. Add keepout zone around slot (no copper within 0.5mm of edge)
3. Verify DRC passes with slot

### 8.5 MOSFET & Gate Driver Layout

**Gate Traces (TC4426 to Q3/Q4):**
- These traces carry short, sharp current spikes to charge gate capacitance
- Make traces at least **20-25 mils wide** to minimize inductance
- High inductance causes "Gate Ringing" — can oscillate and destroy the MOSFET
- Keep traces as short as possible

**CSD18563Q5A (SON-8) Stencil Design:**
```
    ┌─────────────────────────────────┐
    │                                 │
    │   ┌─────┐         ┌─────┐      │
    │   │     │         │     │      │
    │   │     │         │     │      │  ← Use WINDOW-PANE design
    │   └─────┘         └─────┘      │    (4 smaller squares)
    │                                 │
    │   ┌─────┐         ┌─────┐      │    NOT one big rectangle!
    │   │     │         │     │      │
    │   │     │         │     │      │    Big rectangle causes chip
    │   └─────┘         └─────┘      │    to "float" during reflow
    │                                 │
    └─────────────────────────────────┘
          THERMAL PAD STENCIL
```
- Window-pane design controls solder volume and prevents voiding
- Add thermal vias (0.3mm) under exposed pad connecting to bottom ground plane

**Q3 (Si7461DP) and Q4 Thermal Management:**
- Both MOSFETs need thermal vias under their exposed pads
- Connect to ground plane on bottom layer
- Use 2oz copper under power MOSFETs

### 8.6 510 Connector (Bullet Connectors) and Fire Button LED (JST-XH)

**510 Connection via Bullet Connectors (J6):**
- **Why not XT30U:** The XT30U-M/F connector is too large to fit through the M12 bolt hole on the MM510 connector
- Use **male bullet connectors** (3.5mm or 4mm) for the 510 connection instead
- Bullet connectors rated for **20-30A** — sufficient for our 6A max requirement

**Assembly Process:**
1. Solder **14 AWG silicone wire** directly to J6 pads on PCB
2. Crimp **female bullet connector** to the other end of each wire
3. At the MM510 connector end: crimp **male bullet connector** to wires from 510
4. Mate female (PCB side) to male (510 side) for quick-disconnect

**Note:** JLCPCB will not source/solder bullet connectors — J6 pads are for hand-soldering wires.

**J6 Layout:**
- Two large solder pads for 14 AWG wire (COIL+ and COIL-)
- Route high-current traces (from boost output) with **4mm wide traces on 2oz copper** to J6 pads
- At layer transitions, use **via rows** (4-6 vias in a line along the trace) for thermal/current handling
- Place near board edge for easy cable routing

**JST-XH (J5) for Fire Button LED:**
- 2-pin JST-XH rated for 3A — sufficient for LED current (~20mA)
- Route LED_STATUS (GPIO47) through current-limiting resistor to J5
- Add **220Ω resistor** in series for typical green LED (adjust for LED Vf)

**Isolation Requirements:**
- J6 pads must be at least **3-4mm away** from board edge or mounting holes
- Prevents cable strain from transferring to solder joints

**Electrical Isolation (Critical — "Live Shell" Hazard):**

The 510 connector shell is connected to Q4 drain (low-side switching topology).
When Q4 is OFF, the shell floats up to V_BOOST (up to 7V).

**⚠️ METAL CHASSIS WARNING:**

If the user installs the atomizer into a **metal enclosure** without proper isolation:
1. Atomizer screws into 510 connector (510 shell contacts chassis)
2. Chassis connects to USB GND (common in metal enclosures)
3. **Result:** Direct bypass around Q4 — coil fires continuously when boost is enabled

This creates an **auto-fire condition** that cannot be stopped by firmware.

**Risk Scenarios:**
| Chassis Type | Risk | Mitigation |
|--------------|------|------------|
| Plastic/3D-printed | Low | None needed |
| Anodized aluminum | Medium | Anodizing provides insulation, but scratches expose metal |
| Bare metal | **HIGH** | MANDATORY isolation required |

**MANDATORY for Metal Chassis:**
- Use **PEEK/Delrin insulating washer** between 510 and chassis
- Or use **plastic 510 mounting insert** (available from ModMaker)
- Or use non-conductive enclosure material
- **NEVER** ground 510 shell to USB GND or chassis

**Assembly Guide Note:** Include bold warning about 510 isolation in user documentation.

**Optional ESD Bleed:** Add 1MΩ resistor from 510 shell to GND to slowly bleed static charge without creating a hazardous short-circuit path.

### 8.7 ESP32-S3 Antenna Placement

**The "Overhang" (Best Practice):**
- Place ESP32 module at the **very edge** of the PCB
- Antenna portion should **hang off the edge** of the FR4 board entirely
- This provides best Bluetooth range

**Keep-Out Zone (If Over PCB):**
```
    ┌─────────────────────────────────────────────────────┐
    │                                                     │
    │    ┌─────────────────────────────────────────┐      │
    │    │          ESP32-S3-WROOM-1               │      │
    │    │                                         │      │
    │    │    ┌─────────┐                          │      │
    │    │    │ ANTENNA │◄── NO COPPER underneath! │      │
    │    │    │  AREA   │    (Top, Bottom, Inner)  │      │
    │    │    └─────────┘                          │      │
    │    │                                         │      │
    │    └─────────────────────────────────────────┘      │
    │                                                     │
    │    ◄────── 15mm ──────►                             │
    │    Keep-out zone on sides                           │
    │                                                     │
    └─────────────────────────────────────────────────────┘
```

- **NO COPPER** (all layers) under antenna area and for **15mm along the sides**
- Copper under antenna will detune it and ruin Bluetooth range
- Ground plane break is acceptable here — it's required

### 8.8 Battery Solder Pads (J3)

**Pad Design:**
- Use **large through-hole pads** (2.5mm hole, 4mm annular ring) for 14 AWG wire
- Through-holes provide mechanical "rivet" strength for strain relief
- Label pads clearly: **BAT+** and **BAT-** with silkscreen

**Layout:**
- Place near board edge for easy battery routing
- Keep **away from heat sources** (TPS61088, Q3, Q4, shunt)
- BAT- pad connects directly to shunt resistor input (high-current path) — use **4mm trace with 2oz copper**
- BAT+ pad connects to BQ29705 protection circuit

**Thermal Considerations:**
- Add thermal relief spokes on BAT- pad (connects to ground plane)
- Without relief, the ground plane acts as heat sink making soldering difficult
- BAT+ can have solid connection (not on large pour)

### 8.9 NTC Thermistor Solder Pads (J7)

**Pad Design:**
- Use **small through-hole pads** (1.0mm hole, 2mm annular ring) for 22 AWG wire
- Label pads: **TS** and **GND** with silkscreen
- TS pad connects to RT1/RT2 divider network → BQ25896 TS pin
- GND pad connects to quiet analog ground

**Layout:**
- Place **near J3** (battery pads) — NTC wire routes alongside battery wires
- Keep close to BQ25896 for short TS signal path
- Route TS signal away from switching nodes (noise sensitive)

**Circuit Connection:**
```
    J7 TS pad ──────► RT1 (5.23kΩ) ──┬──► BQ25896 TS pin
                                     │
                      NTC attached   │
                      to battery     │
                                     │
    J7 GND pad ─────► RT2 (26.7kΩ) ──┴──► GND
```

### 8.10 Additional Layout Rules

1. **Shunt Resistor Thermal Relief:**
   - Use **2oz copper top AND bottom** under shunt
   - ≥6 thermal vias under shunt pads
   - **No solder mask under shunt body** (aids heat dissipation)
   - PWM causes higher RMS heating than DC equivalent

2. **MAX17055 Placement (Noise Sensitive):**
   - Place **near battery connector**, away from switching nodes
   - Keep away from Q4, TPS61088, and boost inductor
   - Dedicated quiet ground return to star point
   - CSP/CSN sense traces routed as differential pair

3. **Signal Integrity:**
   - Keep I2C traces short, 4.7kΩ pull-ups near ESP32
   - SPI to display: Ground guard traces on both sides
   - ADC inputs: RC filter at pin (100Ω + 100nF)

4. **Safety-Critical Routing:**
   - WDT_PING: Direct route, no vias if possible
   - OC_FAULT: Direct route to AND gate, minimal length
   - FIRE_EN: Direct route to AND gate
   - All safety signals: **Away from switching nodes**

---

## 9. Testing Procedure

### 9.1 Pre-Power Smoke Test
1. Visual inspection for solder bridges
2. Continuity test: VBAT to GND should NOT be short
3. Verify all polarized components correctly oriented

### 9.2 Power-On Sequence
1. Connect battery (without USB)
2. Verify VSYS = VBAT (power path)
3. Verify BQ29705 protection active
4. Verify ESP32 boots (UART log)
5. Verify WDT_OK is LOW (555 not yet triggered)

### 9.3 Safety System Verification
1. **555 Watchdog Test:**
   - Stop WDT_PING task
   - Verify WDT_OK goes LOW after ~3s (R3=270kΩ)
   - Verify coil cannot fire

2. **Overcurrent Protection Test:**
   - Connect 0.5Ω dummy load
   - Set PWM duty low
   - Gradually increase, verify OC_FAULT trips at ~5A

3. **Fire Button Interlock Test:**
   - Attempt fire without button pressed
   - Verify no coil current

4. **AND Gate Logic Test:**
   - Test all 16 input combinations
   - Verify truth table

5. **Stuck Button Test (CRITICAL):**
   - Hold fire button while inserting battery
   - Verify system does NOT fire on boot (FIRE_EN pull-down)

6. **PWM Freeze Test:**
   - Manually force PWM high (simulate MCU freeze)
   - Inject overcurrent
   - Verify hardware safety gate cuts power

7. **Boost Failure Under Load Test (CRITICAL):**
   - Fire at 15W on 2Ω coil (drawing ~2.7A)
   - While firing, suddenly short V_BOOST to ~V_BATT (simulate inductor saturation or solder failure)
   - Verify:
     - OC comparator trips (current spike detection)
     - PMOS Q3 shuts off via safety gate
     - System recovers safely (no latch-up, no damage)
     - ESP32 logs the fault condition
   - This simulates real-world component failure modes

### 9.4 Power Output Calibration
1. Connect power meter in-line with coil
2. Calibrate MCP4561 settings for each power level
3. Calibrate PWM duty cycle for low-power mode
4. Store calibration in NVS

---

## 10. Schematic Checklist

- [x] USB-C CC resistors (5.1kΩ to GND)
- [x] ESD protection on USB lines
- [x] ESP32 boot strapping pins avoided (GPIO45/46 not used)
- [ ] Crystal and capacitors for ESP32 (if using bare chip - N/A for module)
- [x] Decoupling capacitors on all IC VCC pins
- [x] Pull-up on I2C lines (4.7kΩ)
- [x] Pull-up on LM393 open-collector output (10kΩ)
- [x] Pull-down on FIRE_EN for safe default (10kΩ)
- [x] Schmitt buffer after 555 for clean edges
- [x] Hysteresis on comparator (1MΩ feedback)
- [x] RC filter on comparator input (100Ω + 10nF)
- [ ] RC snubber on MOSFET gates if ringing observed
- [ ] Test points on critical signals
- [ ] LED indicators for charge status and power
- [x] TC4426 (inverting) driver for PMOS gate

---

## 11. Change Log

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2024-XX-XX | Initial design |
| 2.0 | 2024-XX-XX | Critical fixes per review:<br>• TC4427 → TC4426 (inverting driver)<br>• LM393 pull-up added<br>• Charge current 2A → 1A<br>• Si7309DN → Si7461DP (higher current)<br>• CSD19535KCS → CSD18563Q5A (optimized)<br>• INA180A3 → INA180A2 (better headroom)<br>• GPIO45/46 → GPIO35/36 (strapping fix)<br>• Added FIRE_EN pull-down<br>• Added boost pass-through mode<br>• Added comparator hysteresis/filter<br>• BQ29700 → BQ29705 (higher OCD) |
| 2.1 | 2024-XX-XX | Final polish per review:<br>• **CRITICAL:** Fixed TPS61088 pass-through (EN=HIGH, set V_target < V_batt)<br>• GPIO35/36 → GPIO1/2 (avoid Octal PSRAM conflict)<br>• Specified ESP32-S3-WROOM-1-N8R2 (Quad SPI only)<br>• Added 510 connector isolation requirement<br>• Enabled PWM in boost mode for fine control<br>• Added boost failure under load test<br>• Clarified True Load Disconnect behavior |
| 2.2 | 2024-XX-XX | Feedback network fix:<br>• **CRITICAL:** Fixed V_REF = 1.212V (was incorrectly 0.6V)<br>• **CRITICAL:** MCP4561-103 (10k) → MCP4561-104 (100k)<br>• Without fix: V_min = 13.3V (exceeds 12.6V max!)<br>• With fix: V_min = 2.42V, V_max = 7V ✓<br>• Added NTC thermistor B=3435 requirement + TS divider resistors |
| 2.3 | 2024-12-XX | Final refinements:<br>• PWM frequency ≥20kHz requirement (avoid audible noise)<br>• Firmware voltage clamp (MAX_BOOST_VOLTAGE = 7.0V)<br>• Optional 1MΩ ESD bleed resistor on 510 shell |
| 2.4 | 2024-12-XX | Comprehensive layout guidance:<br>• Star Ground geometry at shunt (critical for OCP accuracy)<br>• TPS61088 "Hot Loop" minimization, SW node copper rules<br>• 9-12 thermal vias for TPS61088 pad<br>• INA180 differential pair routing + ground moat<br>• Gate trace width 20-25 mils (prevent ringing)<br>• SON-8 window-pane stencil design<br>• ESP32 antenna overhang + 15mm keep-out zone<br>• Via farm stitching for high-current paths |
| 2.5 | 2024-12-XX | Connector updates:<br>• J3: Changed to solder pads (BAT+/BAT-) for direct battery wire<br>• J5: JST-XH (male) for fire button LED<br>• J6: XT30U (male) for 510 connector (30A rated)<br>• J7: Solder pads (TS/GND) for NTC thermistor<br>• R22: 220Ω LED current-limiting resistor<br>• Added battery pad layout guidance (§8.8)<br>• Added NTC pad layout guidance (§8.9) |
| 2.6 | 2024-12-XX | USB VBUS protection fix:<br>• **CRITICAL:** Removed D1 (SS34 series Schottky) — caused 0.2-0.4V drop violating USB spec<br>• TVS diode (D2) polarity clarified: cathode to VBUS, anode to GND<br>• C_USB1 changed from 100nF to 4.7µF (USB requires bulk capacitance)<br>• Added C_USB2 (100nF) for HF bypass<br>• USB host provides reverse polarity protection, no series diode needed |
| 2.7 | 2024-12-XX | Battery management & connector fixes:<br>• **BQ29705:** Fixed footprint WCSP-5 → DSBGA-6 (6-bump), pin 5=BAT, pin 6=V-<br>• **BQ29705:** Added V- sense path (R2 to Q2:Source) for OCD detection<br>• **BQ29705:** Renamed C1 → C_PROT to avoid duplicate designator<br>• **BQ25896:** Added missing pins handling (PSEL, PG_N, STAT, QON_N, PMID)<br>• **Protection MOSFETs:** Added R_G1/R_G2 (47Ω) gate resistors for EMI<br>• **Protection MOSFETs:** Changed to 2x discrete CSD18534Q5A<br>• **510 Connector:** XT30U → Bullet connectors (XT30U won't fit through M12 bolt hole on MM510)<br>• **NTC:** Added divider verification formula + TI SLUA917 reference |
| 2.8 | 2025-01-05 | **SHOWSTOPPER FIX** - Final design approval:<br>• **CRITICAL:** Protection MOSFETs moved to LOW-SIDE (ground path)<br>• N-ch FETs on high-side cannot work with BQ29705 (no charge pump)<br>• Q2:Source → BAT-, Q1:Source → System GND, common drain internal<br>• BQ29705 VSS now references raw BAT- (stays powered during fault)<br>• **LDO:** Added AP2112K-3.3 (Step 14) for VCC_3V3 logic rail<br>• **510 Isolation:** Updated for plastic chassis (no isolation needed)<br>• **High-current routing:** Added polygon pour requirement for 6A nets<br>• **STATUS: APPROVED FOR SCHEMATIC CAPTURE** |
| 2.9 | 2025-01-05 | Fuel Gauge routing fix + cleanup:<br>• **CRITICAL:** FuelGauge must be IN-SERIES with load path<br>• Added VSYS_LOAD hierarchical label (output of FuelGauge)<br>• VSYS → FuelGauge only; VSYS_LOAD → BoostConverter, MCU, LDO<br>• Without fix: Fuel gauge reads 0mA, battery % non-functional<br>• Cleaned up Section 8.4 drafting notes |
| 3.0 | 2025-01-05 | **CRITICAL SAFETY FIXES** - Final review corrections:<br>• **BQ29705 BAT wiring:** Fixed R1 connection from BAT- to BAT+ (IC won't power otherwise)<br>• **ADC2/Wi-Fi conflict:** Moved I_SENSE to GPIO3 (ADC1_CH2), V_SENSE to GPIO4 (ADC1_CH3)<br>• Swapped WDT_PING→GPIO15, FIRE_EN→GPIO16, COIL_PWM→GPIO5<br>• **Q4 gate resistor:** Added R_G3 (47Ω) to protect ESP32 from gate capacitance inrush<br>• **STATUS: READY FOR SCHEMATIC CAPTURE** |
| 3.1 | 2025-01-05 | **PCB Layout Preparation:**<br>• **Star Ground:** Added Net Tie (NT1) + GND_PWR/GND separation in CoilDriver<br>• **Differential Pair:** Renamed current sense traces to CS_P/CS_N for KiCad auto-routing<br>• **Liquid Isolation:** Added e-liquid warning for 510/USB bridge risk<br>• **Test Points:** Added TP1-TP7 for smoke test verification<br>• **Mounting Holes:** Added MH1-MH4 with GND connection<br>• **QON Wake Button:** Added optional SW_WAKE for ship mode recovery |
| 3.2 | 2025-01-05 | **CRITICAL FIX - High-Side Shunt Topology:**<br>• **SHOWSTOPPER:** Removed dead short (COIL_HIGH was incorrectly connected to GND_PWR)<br>• **Shunt floats:** High-side shunt at V_BOOST potential must NOT connect to ground<br>• **Star Point relocated:** From shunt to Q4:Source (where 6A returns to battery)<br>• **Net Tie carries milliamps only:** AGND (analog) joins GND (power) at Q4:Source<br>• **Section 8.2 rewritten:** Now correctly describes high-side shunt topology<br>• **STATUS: APPROVED FOR SCHEMATIC CAPTURE** |
| 3.3 | 2025-01-05 | **Final refinements:**<br>• **LDO placement:** Added guidance to power from VSYS<br>• **Recommended:** Place LDO on MCU sheet instead of BatteryManagement<br>• **Liquid ingress warning:** Strengthened 510/USB bridge auto-fire hazard warning<br>• **Conformal coating:** Now listed as mandatory protection<br>• **STATUS: READY FOR IMPLEMENTATION** |
| 3.4 | 2025-01-05 | **V_SENSE voltage divider:**<br>• **CRITICAL:** ESP32 ADC max is 3.3V, V_BOOST can reach 7V+<br>• Added R_DIV1 (100kΩ), R_DIV2 (47kΩ), C_VSENSE (100nF) to BOM<br>• Divider ratio 0.3197: At 7V boost → 2.23V ADC (safe)<br>• At 10V spike → 3.20V ADC (still within tolerance)<br>• **STATUS: READY FOR SCHEMATIC CAPTURE** |
| 3.6 | 2025-01-05 | **Design review fixes (8.5/10 → 9.5/10):**<br>• **Soft-start PWM:** Added 20ms ramp in fire_coil() (§6.2)<br>• **Reverse battery:** Added Q_REV (Si7461DP) + R_REV (10k) to BatteryManagement<br>• **Coil measurement:** Added measure_coil_resistance() (§6.3) with open/short detect<br>• **ADC calibration:** Added ESP32 curve fitting code (§6.4)<br>• **Watchdog timing:** R3 changed 150kΩ→270kΩ (3s timeout vs ESP32 boot)<br>• **MAX17055 calibration:** Added init code + learning procedure<br>• **OC_FAULT state machine:** Added §6.6 with retry logic + lockout<br>• WDT_PING moved GPIO15→GPIO48 |
| 3.7 | 2025-01-05 | **NTC → MCP9808 transition (eliminates ADC2/Wi-Fi conflict):**<br>• **Replaced NTC_PCB + R_NTC** with MCP9808 I2C temperature sensor (U15)<br>• MCP9808 provides ±0.25°C accuracy vs ±2-5% NTC<br>• Removed T_PCB (was GPIO15/ADC2_CH4) — no longer needed<br>• Added MCP9808 to I2C1 bus (address 0x18)<br>• Updated §6.5 firmware for I2C-based temperature reading<br>• Added C_MCP (100nF bypass) to BOM |
| 3.8 | 2025-01-05 | **Hardware thermal interlock (MCU-independent protection):**<br>• **MCP9808 T_ALERT → SafetyLogic AND gate** (5th input)<br>• Over-temperature (≥85°C) cuts coil power in hardware, even if MCU frozen<br>• Added R_ALERT (10kΩ pull-up) for MCP9808 open-drain output<br>• Updated §3 truth table for 5-input safety chain<br>• mcp9808_init() configures T_CRIT=85°C, CRIT_ONLY mode<br>• Power-on defaults (T_CRIT=95°C) provide protection if MCU fails to boot<br>• **STATUS: APPROVED FOR PROTOTYPING (9.5/10)** |
| 3.9 | 2025-01-05 | **Fuel gauge path fix + boost failsafe:**<br>• **CRITICAL: Moved FuelGauge to battery path** (VBAT_PROT → sense → VBAT_SENSE → BQ25896:BAT)<br>• Fuel gauge now measures both charge AND discharge current (fixes SOC sync during charging)<br>• Added VBAT_SENSE net (from MAX17055 CSN to BQ25896 BAT pin)<br>• Removed VSYS_LOAD concept — VSYS goes directly to all system loads<br>• **Added R_SAFE (22kΩ) parallel to MCP4561** — limits V_BOOST to 6.7V if wiper fails open<br>• Added PCB milling slot guidance for 510/USB e-liquid isolation (§8.4)<br>• **STATUS: PRODUCTION-READY (9.5/10)** |
| 3.10 | 2025-01-05 | **Design review response (critical fixes):**<br>• **Added R_G3_HS (15Ω):** Q3 gate resistor between TC4426 and Q3:Gate (prevents ringing/oscillation)<br>• **Added D_FLY (SS34):** Flyback Schottky diode across coil for inductive kickback protection<br>• **LM393 response time:** Documented ~1.3µs limitation and multi-layer mitigations<br>• **GPIO3 strapping:** Added board bring-up verification steps (JTAG selection pin)<br>• **MCP4561 NVM wear:** Added firmware guidance to use volatile register during operation<br>• **NTC disconnect detection:** Added firmware check for abnormal TS voltage (open/short)<br>• **STATUS: READY FOR PROTOTYPE (9/10)** |
| 3.11 | 2025-01-05 | **Final pre-capture cleanup:**<br>• **Added R_GS_Q3 (100kΩ):** Gate-source pull-up ensures Q3 stays OFF during power-up<br>• **L2 footprint verification:** Added checklist for high-current inductor pad geometry<br>• **GPIO3 isolation option:** Added R_ISO alternative for boot strapping issues |
| 3.12 | 2025-01-05 | **🔴 CRITICAL FIXES (Design Review v3.12):**<br>• **BQ29705 V- wiring:** R2 now connects to Q1:Source (System GND), NOT Q2:Source — previous wiring caused OCD/SCD to NEVER trip!<br>• **R5 ILIM:** 2.4kΩ → 800Ω (I_INLIM = 1040/R = 1.3A) — old value gave only 0.43A<br>• **C_USB1 voltage:** 16V → 25V for USB-C transient protection<br>• **C_BOOT size:** 100nF → 10nF to prevent GPIO0 strapping issues<br>• **Added bypass caps:** C_SCHMITT, C_INV for U12/U13 (74LVC1G17/74LVC1G04)<br>• **Added FIRE_BTN debounce:** R_DEBOUNCE (10k) + C_DEBOUNCE (100nF) = 1ms τ<br>• **Added 510 ESD bleed:** R21 (1MΩ) from COIL- to GND (optional)<br>• **Q4 gate drive option:** Documented TC4426 Channel B alternative<br>• **Soft-start timing:** 20ms delay after BOOST_EN before PWM ramp<br>• **LEDC init code:** Added ESP32 PWM initialization example<br>• **STATUS: APPROVED FOR SCHEMATIC CAPTURE** |
| 3.13 | 2025-01-10 | **🔴 CRITICAL: MAX17055 Fuel Gauge Topology Fix:**<br>• **LOW-SIDE sensing required:** MAX17055 CSP pin is IC ground reference (0V), BATT is power supply<br>• **Previous HIGH-SIDE topology was WRONG:** V_BATT − V_CSP ≈ 0V → IC would not power up!<br>• **New topology:** Sense resistor in ground return path (System GND → R_SENSE_FG → Battery-)<br>• **BATT pin** connects to VBAT_PROT (battery+), **CSP** to System GND, **CSN** to Battery- side<br>• **Removed VBAT_SENSE:** BQ25896 BAT now connects directly to VBAT_PROT<br>• **C_FG1 bypass:** 100nF between BATT and CSP (not to system GND)<br>• Updated all signal flow diagrams and schematic guide<br>• **STATUS: REQUIRES SCHEMATIC UPDATE** |
| 3.14 | 2025-01-18 | **High-current routing clarification:**<br>• **4mm traces with 2oz copper are acceptable** for all high-current nets (6A capable)<br>• Polygon pours optional but not required — 4mm traces work fine even for longer runs<br>• **Via rows** (4-6 vias in a line along trace) replace via farms at layer transitions<br>• Single via handles ~1A; always use multiple vias in parallel for 6A paths<br>• Updated HighCurrent net class list with additional nets: VBAT_RAW, GND_SENSE, PROT_MID, OPAMP_IN+<br>• **AGND net added:** INA180/LM393 GND pins connect to AGND → NT1 → GND (star ground topology) |

---

## 12. Design Review Sign-Off

| Review Area | Status | Reviewer Notes |
|-------------|--------|----------------|
| Architecture | ✅ 9.5/10 | Multi-layer hardware safety |
| Electrical Safety | ✅ 9/10 | All critical faults addressed |
| Fault Tolerance | ✅ 9.5/10 | Fail-safe in all tested scenarios |
| Component Selection | ✅ 9/10 | Production-grade parts |
| Manufacturability | ✅ 8.5/10 | JLCPCB-friendly, some hand-solder |

**Verdict:** Approved for prototyping.

---

## References

- [BQ25896 Datasheet](https://www.ti.com/product/BQ25896) - TI Charger IC
- [TPS61088 Datasheet](https://www.ti.com/product/TPS61088) - TI Boost Converter
- [BQ29700 Family Datasheet](https://www.ti.com/product/BQ2970) - TI Battery Protection
- [MAX17055 Datasheet](https://www.analog.com/en/products/max17055.html) - Analog Devices Fuel Gauge
- [INA180 Datasheet](https://www.ti.com/product/INA180) - TI Current Sense Amp
- [TC4426 Datasheet](https://www.microchip.com/en-us/product/tc4426) - Microchip Gate Driver
- [Si7461DP Datasheet](https://www.vishay.com/en/product/72567/) - Vishay PMOS
- [CSD18563Q5A Datasheet](https://www.ti.com/product/CSD18563Q5A) - TI NexFET NMOS
- [ESP32-S3 Datasheet](https://www.espressif.com/en/products/socs/esp32-s3) - Espressif MCU

---

*Document Version: 3.13*
*Date: January 2025*
