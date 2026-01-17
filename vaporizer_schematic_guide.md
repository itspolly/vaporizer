# KiCad Schematic Guide: Safe Vaporizer Design

## Document Overview

This guide provides comprehensive step-by-step instructions for creating the KiCad schematic for the Safe Vaporizer project. The schematic uses a hierarchical structure with a root sheet containing subsheet symbols that represent functional blocks.

**KiCad Version:** 8.0+
**Schematic Structure:** Hierarchical (1 root sheet + 8 subsheets)

---

## 1. Project Structure Overview

### 1.1 Hierarchical Sheet Organization

```
vaporizer.kicad_sch (Root Sheet)
├── USB_Power.kicad_sch
├── BatteryManagement.kicad_sch
├── FuelGauge.kicad_sch
├── BoostConverter.kicad_sch
├── SafetyLogic.kicad_sch
├── CoilDriver.kicad_sch
├── MCU.kicad_sch
└── UI.kicad_sch
```

### 1.2 Signal Flow Between Sheets

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              ROOT SHEET                                          │
│                                                                                  │
│  ┌──────────────┐    ┌─────────────────────┐                                    │
│  │  USB_Power   │───►│  BatteryManagement  │                                    │
│  │              │    │                     │                                    │
│  │ Out: VBUS    │    │ Out: VBAT_PROT  ────┼──────┬──────────────────┐          │
│  │ Out: USB_D+  │    │ Out: VSYS  ─────────┼──┬───┼──────────────────┼───┐      │
│  │ Out: USB_D-  │    │ Out: CHG_INT        │  │   │  (direct to      │   │      │
│  │ Out: CC1     │    │ In:  I2C1_SDA/SCL   │  │   │   BQ25896 BAT)   │   │      │
│  │ Out: CC2     │    │ In:  CHG_CE, CHG_OTG│  │   │                  │   │      │
│  └──────────────┘    │ Out: BAT_NEG ───────┼──┼───┼───┐              │   │      │
│                      └─────────────────────┘  │   │   │              │   │      │
│                                               │   │   ▼              │   │      │
│                                               │   │ ┌────────────┐   │   │      │
│                                               │   │ │ FuelGauge  │   │   │      │
│                                               │   │ │(LOW-SIDE)  │   │   │      │
│                                               │   │ │            │   │   │      │
│                                               │   │ │In:VBAT_PROT├───┘   │      │
│                                               │   │ │In: BAT_NEG │       │      │
│                                               │   │ │In: GND     │       │      │
│                                               │   │ │Out:FG_ALRT │       │      │
│                                               │   │ └────────────┘       │      │
│                                               │               │                 │
│  ┌────────────────┐    ┌────────────────┐    │               │                 │
│  │ BoostConverter │    │  SafetyLogic   │    │    ┌──────────┴───────┐         │
│  │                │    │                │    │    │    CoilDriver    │         │
│  │ In: VSYS  ◄────┼────┼────────────────┼────┘    │                  │         │
│  │ In: BOOST_EN   │    │ In: WDT_PING   │         │ In: V_BOOST      │         │
│  │ In: I2C1_*     │    │ In: FIRE_BTN   │         │ In: SAFETY_GATE  │         │
│  │ Out: V_BOOST ──┼────┼────────────────┼────────►│ In: COIL_PWM     │         │
│  │                │    │ In: OC_FAULT   │         │ Out: I_SENSE     │         │
│  └────────────────┘    │ In: FIRE_EN    │         │ Out: OC_FAULT    │         │
│                        │ In: T_ALERT    │         │ Out: COIL+/COIL- │         │
│                        │ Out:SAFETY_GATE├────────►└──────────────────┘         │
│                        └────────────────┘                                      │
│         ┌──────────────────────────────────────────────────────────────────────┐│
│         │                              MCU (ESP32-S3-WROOM-1)                   ││
│         │  In: VSYS (from BQ25896, via LDO for 3.3V)                           ││
│         │  All GPIO connections, I2C buses, SPI, ADC inputs                    ││
│         └──────────────────────────────────────────────────────────────────────┘│
│         │                                                                        │
│         ▼                                                                        │
│  ┌──────────────┐                                                               │
│  │      UI      │ Display FPC, Fire Button, LED Connector                       │
│  └──────────────┘                                                               │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘

⚠️ CRITICAL: FuelGauge uses LOW-SIDE sensing (sense resistor in ground return path).
   CSP = System GND, CSN = Battery- side, BATT = Battery+ (power supply).
   BQ25896 BAT connects directly to VBAT_PROT (no sense resistor in battery+ path).
```

---

## 2. Creating the Project

### 2.1 Initial Project Setup

1. **Open KiCad** and select **File → New Project**
2. Navigate to your project folder (e.g., `Vape2.0/KiCad/`)
3. Name the project: `vaporizer`
4. Click **Save**

This creates:
- `vaporizer.kicad_pro` (project file)
- `vaporizer.kicad_sch` (root schematic)
- `vaporizer.kicad_pcb` (PCB layout)

### 2.2 Schematic Editor Settings

1. Open the schematic editor (double-click `vaporizer.kicad_sch`)
2. Go to **File → Schematic Setup**
3. Set **Page Size:** A3 (for root sheet)
4. Set **Title Block:**
   - Title: `Safe Vaporizer - Root Sheet`
   - Rev: `3.3`
   - Company: `[Your Name]`

---

## 3. Root Sheet Layout

### 3.1 Root Sheet Organization Diagram

The root sheet contains only hierarchical sheet symbols arranged to show signal flow. No actual components are placed on the root sheet.

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           VAPORIZER ROOT SHEET (A3)                                  │
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐│
│  │                              TITLE BLOCK (bottom right)                         ││
│  │              Safe Vaporizer - Root Sheet | Rev 3.3 | Date                       ││
│  └─────────────────────────────────────────────────────────────────────────────────┘│
│                                                                                      │
│  ╔═══════════════════════════════════════════════════════════════════════════════╗  │
│  ║  LEFT COLUMN (Power Input)        CENTER COLUMN          RIGHT COLUMN (Output) ║  │
│  ║                                                                                ║  │
│  ║  ┌─────────────────┐                                                           ║  │
│  ║  │   USB_Power     │                                                           ║  │
│  ║  │                 │──────────────────────────────────────┐                    ║  │
│  ║  │  VBUS        ───┼─►                                    │                    ║  │
│  ║  │  USB_D+      ───┼─►                                    │                    ║  │
│  ║  │  USB_D-      ───┼─►                                    ▼                    ║  │
│  ║  │  CC1         ───┼─►    ┌──────────────────────┐   ┌────────────────┐        ║  │
│  ║  │  CC2         ───┼─►    │  BatteryManagement   │   │    MCU         │        ║  │
│  ║  └─────────────────┘      │                      │   │                │        ║  │
│  ║                           │  VBUS            ◄───┼───┤  GPIO/I2C/SPI  │        ║  │
│  ║                           │  VSYS            ───┼─┬─┤                 │        ║  │
│  ║                           │  VBAT_PROT       ───┼─┼─►  (to BQ25896)  │        ║  │
│  ║                           │  BAT_NEG         ───┼─┼─►  (GND return)  │        ║  │
│  ║                           │  CHG_INT         ───┼─┼─►  (All ESP32     │        ║  │
│  ║                           │  CHG_CE          ◄───┼─┼──┤   connections) │        ║  │
│  ║                           │  CHG_OTG         ◄───┼─┼──┤                │        ║  │
│  ║                           │  I2C1_SDA        ◄──►┼─┼─►                │        ║  │
│  ║                           │  I2C1_SCL        ◄───┼─┼──┤                │        ║  │
│  ║                           └──────────────────────┘ │  └────────────────┘        ║  │
│  ║                                      ▲             │          ▲                 ║  │
│  ║  ⚠️ LOW-SIDE CURRENT SENSING                       │          │                 ║  │
│  ║  ┌─────────────────┐                 │             │          │                 ║  │
│  ║  │   FuelGauge     │─────────────────┘             │          │                 ║  │
│  ║  │  (GND PATH)     │                               │          │                 ║  │
│  ║  │  BATT        ◄──┼── (from VBAT_PROT = Battery+) │          │                 ║  │
│  ║  │  CSP (0V ref)◄──┼── (GND_SENSE = Q2:Source)     │          │                 ║  │
│  ║  │  CSN         ◄──┼── (BAT_NEG = Battery-)        │          │                 ║  │
│  ║  │  I2C1_SDA    ◄──┼─►                             │          │                 ║  │
│  ║  │  I2C1_SCL    ◄──┼──                             │          │                 ║  │
│  ║  │  FG_ALRT     ───┼─►                             │          │                 ║  │
│  ║  └─────────────────┘                               │          │                 ║  │
│  ║                                                    │          │                 ║  │
│  ║  ┌─────────────────┐      ┌──────────────────────┐ │   ┌──────┴─────────┐       ║  │
│  ║  │  BoostConverter │      │    SafetyLogic       │ │   │   CoilDriver   │       ║  │
│  ║  │                 │      │                      │ │   │                │       ║  │
│  ║  │  VSYS        ◄──┼──────┤  WDT_PING        ◄───┼─┼───┤  V_BOOST    ◄──┼──┐    ║  │
│  ║  │  BOOST_EN    ◄──┼──────┤  FIRE_BTN        ◄───┼─┼───┤  SAFETY_GATE◄──┼──│    ║  │
│  ║  │  I2C1_SDA    ◄──┼─►    │  OC_FAULT        ◄───┼─┼───┤  COIL_PWM   ◄──┼──│    ║  │
│  ║  │  I2C1_SCL    ◄──┼──    │  FIRE_EN         ◄───┼─┼───┤  I_SENSE    ───┼──│    ║  │
│  ║  │  V_BOOST     ───┼─►    │  T_ALERT         ◄───┼─┼───┤  OC_FAULT   ───┼──│    ║  │
│  ║  └─────────────────┘      │  SAFETY_GATE     ───┼─┼───►  COIL+      ───┼──│    ║  │
│  ║                           └──────────────────────┘ │   │  COIL-      ───┼──│    ║  │
│  ║                                                    │   └────────────────┘  │    ║  │
│  ║  ┌─────────────────┐                               │                       │    ║  │
│  ║  │       UI        │◄──────────────────────────────┼───────────────────────┘    ║  │
│  ║  │                 │                               │                            ║  │
│  ║  │  LCD_* (SPI)  ◄─┼──                             │                            ║  │
│  ║  │  TP_* (I2C0)  ◄─┼──                             │                            ║  │
│  ║  │  FIRE_BTN     ──┼─►                             │                            ║  │
│  ║  │  LED_STATUS   ◄─┼──                             │                            ║  │
│  ║  └─────────────────┘                               │                            ║  │
│  ║                                                    │                            ║  │
│  ║   MCU VSYS ◄───────────────────────────────────────┘ (from BQ25896, via LDO)    ║  │
│  ║                                                                                ║  │
│  ╚═══════════════════════════════════════════════════════════════════════════════╝  │
│                                                                                      │
│  ⚠️ CRITICAL: FuelGauge on BATTERY PATH measures charge+discharge current.          │
│     VSYS from BQ25896 goes directly to all system loads.                            │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 Creating Hierarchical Sheet Symbols on Root Sheet

For each subsheet, perform these steps:

1. **Press `S`** or select **Place → Hierarchical Sheet**
2. **Click and drag** to draw the sheet symbol rectangle
3. In the **Sheet Properties** dialog:
   - **Sheet name:** (e.g., `USB_Power`)
   - **Sheet file:** (e.g., `USB_Power.kicad_sch`)
   - Click **OK**

**Recommended Sheet Symbol Sizes:**
| Sheet | Width | Height | Position (approximate) |
|-------|-------|--------|------------------------|
| USB_Power | 40mm | 50mm | Top-left |
| BatteryManagement | 60mm | 80mm | Top-center |
| FuelGauge | 40mm | 40mm | Left-center |
| BoostConverter | 50mm | 50mm | Bottom-left |
| SafetyLogic | 60mm | 60mm | Bottom-center |
| CoilDriver | 50mm | 70mm | Right-center |
| MCU | 80mm | 100mm | Right side |
| UI | 50mm | 60mm | Bottom-right |

---

## 4. Subsheet: USB_Power

### 4.1 Sheet Information

| Property | Value |
|----------|-------|
| Sheet Name | USB_Power |
| File Name | USB_Power.kicad_sch |
| Page Size | A4 |
| Description | USB-C connector, ESD protection, CC resistors |

### 4.2 Components

| Designator | Symbol | Value | Footprint | Description |
|------------|--------|-------|-----------|-------------|
| J2 | USB_C_Receptacle_USB2.0 | USB4110-GF-A | Connector_USB:USB_C_Receptacle_GCT_USB4110-GF-A | USB-C receptacle |
| D2 | SMBJ15A | SMBJ15A | Diode_SMD:D_SMB | TVS diode (surge protection) |
| D3 | TPD2E001 | TPD2E001DRLR | Package_TO_SOT_SMD:SOT-23 | USB ESD protection |
| R17 | R | 5.1k | Resistor_SMD:R_0402_1005Metric | CC1 resistor |
| R18 | R | 5.1k | Resistor_SMD:R_0402_1005Metric | CC2 resistor |
| C_USB1 | C | 4.7µF 25V | Capacitor_SMD:C_0805_2012Metric | VBUS bulk capacitor (≥25V for USB-C transients!) |
| C_USB2 | C | 100nF | Capacitor_SMD:C_0402_1005Metric | VBUS HF bypass |

**Design Note:** No series diode (like SS34) is used on USB VBUS. A series Schottky would drop 0.2-0.4V, violating USB voltage spec (4.75-5.25V required). The host provides reverse polarity protection.

### 4.3 Hierarchical Labels

Create these hierarchical labels (all are **outputs** from this sheet):

| Label Name | Direction | Purpose |
|------------|-----------|---------|
| VBUS | Output | USB power to charger |
| USB_D+ | Bidirectional | USB data+ (to MCU) |
| USB_D- | Bidirectional | USB data- (to MCU) |
| GND | Bidirectional | Ground reference |

### 4.4 Wiring Instructions

#### Step 1: Place the USB-C Connector
1. Press `A` to open symbol chooser
2. Search for `USB_C_Receptacle_USB2.0`
3. Place J2 at the **left side** of the sheet
4. Properties: Reference = `J2`, Value = `USB4110-GF-A`

#### Step 2: Create VBUS Hierarchical Label
1. Press `Ctrl+H` or **Place → Hierarchical Label**
2. Type: `VBUS`
3. Set Direction: **Output** (power flows out of this sheet)
4. Place near J2's VBUS pin (pin A4/B4)

#### Step 3: Wire VBUS Path with Protection
```
J2:VBUS (A4,B4) ────┬──── [VBUS] (Hierarchical Label)
                    │
                    ├─── D2:Cathode (TVS to VBUS rail)
                    │         │
                    │    D2:Anode ──── GND
                    │
                    ├─── C_USB1 (+) (4.7µF bulk)
                    │         │
                    │    C_USB1 (-) ──── GND
                    │
                    └─── C_USB2 (+) (100nF HF bypass)
                              │
                         C_USB2 (-) ──── GND
```

**Wiring steps:**
1. Draw wire from J2 pins A4/A9/B4/B9 (VBUS) to the hierarchical label `VBUS`
2. Connect D2 (SMBJ15A) **cathode** to the VBUS line (cathode stripe toward VBUS)
3. Connect D2 **anode** to GND
4. Add C_USB1 (4.7µF) from VBUS to GND (bulk capacitance)
5. Add C_USB2 (100nF) from VBUS to GND (high-frequency bypass)

**TVS Polarity Note:** Unidirectional TVS diodes clamp positive surges when cathode is connected to the protected rail. The cathode bar/stripe on the component marking points toward the higher potential (VBUS).

**📐 PCB LAYOUT REQUIREMENT:**
D2 (TVS diode) and D3 (ESD protection) must be placed **as close as possible to the USB connector J2**. Long traces between the connector and protection devices reduce effectiveness and allow surge energy to propagate into the circuit. Ideally, place D2 within 5mm of J2's VBUS pins.

#### Step 4: Create and Wire USB Data Lines
1. Create hierarchical label `USB_D+` (Bidirectional)
2. Create hierarchical label `USB_D-` (Bidirectional)

```
J2:D+ (A6,B6) ────┬──── D3:IO1 ──── [USB_D+] (Hierarchical Label)
                  │
J2:D- (A7,B7) ────┼──── D3:IO2 ──── [USB_D-] (Hierarchical Label)
                  │
                  └──── D3:GND ──── GND
```

**Wiring steps:**
1. Connect J2 pins A6/B6 (D+) together, wire to D3 IO1 pin
2. Connect J2 pins A7/B7 (D-) together, wire to D3 IO2 pin
3. Wire D3 outputs to respective hierarchical labels
4. Connect D3 GND to ground

#### Step 5: Wire CC Resistors
```
J2:CC1 (A5) ──── R17 ──── GND
J2:CC2 (B5) ──── R18 ──── GND
```

**Wiring steps:**
1. Place R17 (5.1k) near J2
2. Connect J2 pin A5 (CC1) to R17 terminal 1
3. Connect R17 terminal 2 to GND
4. Repeat for R18 with J2 pin B5 (CC2)

#### Step 6: Add Power Flag and Ground Symbol
1. Press `A`, search for `PWR_FLAG`
2. Place PWR_FLAG connected to VBUS net
3. Add ground symbols (press `P` for power port, select `GND`)

**PWR_FLAG Placement (for ERC):**
KiCad's Electrical Rules Checker (ERC) needs PWR_FLAG on power nets to confirm they have a source. Place PWR_FLAG on:

| Net | Location | Reason |
|-----|----------|--------|
| VBUS | USB_Power sheet | USB connector provides power |
| GND | Root sheet or any sheet | Ground reference |
| VCC_3V3 | BatteryManagement (after LDO) | LDO output is a power source |
| VBAT_PROT | BatteryManagement (after protection FETs) | Battery provides power |

### 4.5 Completed USB_Power Schematic Diagram

```
                                                    To BatteryManagement
                                                           │
┌─────────────────────────────────────────────────────────┐│
│                      USB_Power                          ││
│                                                         ││
│  ┌─────────────┐                                        ││
│  │             │                                        ││
│  │   USB-C     │    ┌──────┐  ┌──────┐                 ││
│  │   J2        │    │C_USB1│  │C_USB2│                 ▼│
│  │         VBUS├──┬─┤ 4.7µF├──┤100nF ├──[VBUS]────────►│
│  │  A4,A9,B4,B9│  │ └──┬───┘  └──┬───┘                 ││
│  │             │  │    │         │                      ││
│  │             │  │  ┌─┴────┐    │                      ││
│  │             │  │  │  D2  │    │                      ││
│  │             │  └──┤ TVS  │    │   (No series diode!) ││
│  │             │     │SMBJ15│    │                      ││
│  │             │     └──┬───┘    │                      ││
│  │             │        │        │                      ││
│  │             │        ▼        ▼                      ││
│  │             │       GND      GND                     ││
│  │             │                                        ││
│  │             │    ┌───────────┐                       ││
│  │          D+ ├────┤  D3       │                       ││
│  │      A6,B6  │    │TPD2E001   ├────[USB_D+]──────────►│
│  │             │    │           │                       ││
│  │          D- ├────┤           ├────[USB_D-]──────────►│
│  │      A7,B7  │    │           │                       ││
│  │             │    └─────┬─────┘                       ││
│  │             │          │                             ││
│  │         CC1 ├────R17───┴────GND                      ││
│  │          A5 │    5.1k                                ││
│  │             │                                        ││
│  │         CC2 ├────R18────────GND                      ││
│  │          B5 │    5.1k                                ││
│  │             │                                        ││
│  │        GND  ├────────────[GND]──────────────────────►│
│  │     (shell) │                                        ││
│  └─────────────┘                                        ││
│                                                         ││
│  TVS D2: Cathode to VBUS, Anode to GND                  ││
└─────────────────────────────────────────────────────────┘│
```

---

## 5. Subsheet: BatteryManagement

### 5.1 Sheet Information

| Property | Value |
|----------|-------|
| Sheet Name | BatteryManagement |
| File Name | BatteryManagement.kicad_sch |
| Page Size | A3 |
| Description | BQ25896 charger, BQ29705 protection, NTC circuit |

### 5.2 Components

| Designator | Symbol | Value | Footprint | Description |
|------------|--------|-------|-----------|-------------|
| U1 | BQ25896 | BQ25896RTWT | Package_DFN_QFN:QFN-24-1EP_4x4mm_P0.5mm | Battery charger IC |
| U2 | BQ29705 | BQ29705DSET | Package_CSP:DSBGA-6_1.5x1.5mm | Battery protection IC (6-bump WCSP) |
| Q1 | CSD18534Q5A | Transistor_FET:CSD18534Q5A | Package_SON:Texas_VSON-8_5.05x6.05mm | N-ch protection MOSFET (Charge) |
| Q2 | CSD18534Q5A | Transistor_FET:CSD18534Q5A | Package_SON:Texas_VSON-8_5.05x6.05mm | N-ch protection MOSFET (Discharge) |
| L1 | L | 2.2µH | Inductor_SMD:L_7x7mm | Charger inductor |
| R1 | R | 330Ω | Resistor_SMD:R_0402_1005Metric | BQ29705 BAT resistor |
| R2 | R | 2.2k | Resistor_SMD:R_0402_1005Metric | BQ29705 V- sense resistor (OCD threshold) |
| R5 | R | 800Ω | Resistor_SMD:R_0402_1005Metric | ILIM resistor (I_INLIM=1040/R=1.3A) |
| R19 | R | 5.1k | Resistor_SMD:R_0402_1005Metric | TS divider RT1 |
| R20 | R | 27k | Resistor_SMD:R_0402_1005Metric | TS divider RT2 |
| C_PROT | C | 100nF | Capacitor_SMD:C_0402_1005Metric | BQ29705 BAT capacitor |
| C_IN1 | C | 22µF | Capacitor_SMD:C_0805_2012Metric | Input capacitor |
| C_SYS | C | 22µF | Capacitor_SMD:C_0805_2012Metric | SYS output capacitor |
| C_BAT1 | C | 22µF | Capacitor_SMD:C_0805_2012Metric | Battery capacitor |
| C_BTST | C | 100nF | Capacitor_SMD:C_0402_1005Metric | Bootstrap capacitor |
| C_REGN | C | 100nF | Capacitor_SMD:C_0402_1005Metric | REGN bypass |
| C_PMID | C | 100nF | Capacitor_SMD:C_0402_1005Metric | PMID decoupling |
| R_QON | R | 10k | Resistor_SMD:R_0402_1005Metric | QON_N pull-up to REGN |
| R_G1 | R | 47Ω | Resistor_SMD:R_0402_1005Metric | Q1 gate resistor (optional, EMI) |
| R_G2 | R | 47Ω | Resistor_SMD:R_0402_1005Metric | Q2 gate resistor (optional, EMI) |
| J3 | Conn_01x02 | BAT+/BAT- | Custom:Solder_Pads_2x_14AWG | Battery solder pads |
| J7 | Conn_01x02 | TS/GND | Custom:Solder_Pads_2x_22AWG | NTC solder pads |
| Q_REV | Si7461DP | Si7461DP-T1-GE3 | Package_SO:PowerPAK_SO-8_Single | P-ch reverse battery protection |
| R_REV | R | 10k | Resistor_SMD:R_0402_1005Metric | Q_REV gate pull-down resistor |
| R_SENSE_FG | R | 5mΩ | Resistor_SMD:R_2512_6332Metric | Fuel gauge sense resistor (between Q2:Source and BAT-) |

**Note:** The LDO regulator (U14, C_LDO_IN, C_LDO_OUT) has been moved to the MCU sheet — see Section 10.

**⚠️ R_SENSE_FG Placement:** The fuel gauge sense resistor is placed on THIS sheet (BatteryManagement), not on the FuelGauge sheet. It sits between Q2:Source and J3:BAT- in the ground return path.

### 5.3 Hierarchical Labels

| Label Name | Direction | Purpose |
|------------|-----------|---------|
| VBUS | Input | From USB_Power sheet |
| VSYS | Output | System voltage to boost/MCU/LDO |
| VBAT_PROT | Output | Protected battery voltage (to BQ25896 BAT and FuelGauge BATT) |
| GND_SENSE | Output | Q2:Source — high side of R_SENSE_FG (to FuelGauge CSP) |
| BAT_NEG | Output | Battery negative — low side of R_SENSE_FG (to FuelGauge CSN) |
| CHG_INT | Output | Charger interrupt to MCU |
| CHG_CE | Input | Charge enable from MCU |
| CHG_OTG | Input | OTG mode from MCU |
| CHG_PG | Output | Power good status to MCU (optional → GPIO15) |
| CHG_STAT | Output | Charge status to MCU (optional → GPIO45) |
| I2C1_SDA | Bidirectional | I2C data |
| I2C1_SCL | Input | I2C clock |
| GND | Bidirectional | Ground (System GND = Q1:Source, load-side ground reference) |

### 5.3.1 Ground Reference Guide (IMPORTANT)

**⚠️ This design has multiple ground references due to LOW-SIDE battery protection.**

The BQ29705 protection FETs are in the **ground return path**, creating distinct ground nodes:

```
              LOAD SIDE                              BATTERY SIDE
              ─────────                              ────────────
                   │                                      │
              System GND                              Battery-
             (Q1:Source)                            (via R_SENSE_FG)
                   │                                      │
                   └──── Q1 ──── Q2 ──── Q2:Source ──────┘
                        (CHG)   (DIS)        │
                                         GND_SENSE
                                       (FuelGauge CSP)
```

| Term | Where | KiCad Symbol | When to Use |
|------|-------|--------------|-------------|
| **System GND** | Q1:Source | `GND` power symbol | **99% of the time** — all ICs, bypass caps, resistors to ground |
| **GND_SENSE** | Q2:Source | Net label `GND_SENSE` | **Only** for MAX17055 CSP pin connection |
| **BAT_NEG** | Battery- terminal | Net label `BAT_NEG` | **Only** for MAX17055 CSN, BQ29705 VSS, J3 BAT- pad |

**Practical KiCad usage:**
- When a step says "connect to GND" → use the standard **`GND` power symbol** (press `P`, select `GND`)
- When a step says "connect to GND_SENSE" → use a **net label** (press `L`, type `GND_SENSE`)
- When a step says "connect to BAT_NEG" → use a **net label** (press `L`, type `BAT_NEG`)

**⚠️ CRITICAL: LOW-SIDE Fuel Gauge Topology**

The FuelGauge sense resistor is in the ground return path (LOW-SIDE sensing):
```
Battery+ → BQ29705 → VBAT_PROT → BQ25896:BAT (direct connection)

Ground return path (fuel gauge measures current here):
System GND (loads) ← Q1 ← Q2 ← Q2:Source ← R_SENSE_FG ← Battery-
       │                          │                        │
      GND                     GND_SENSE                  BAT_NEG
  (Q1:Source)              (to FuelGauge CSP)        (to FuelGauge CSN)
   KiCad: GND symbol        KiCad: net label         KiCad: net label
```
This allows the fuel gauge to measure **both charge and discharge current**.

### 5.4 Wiring Instructions

#### Step 1: Place BQ25896 Charger IC
1. Press `A`, search for `BQ25896` (you may need to create/import this symbol)
2. Place U1 in the center-left of the sheet
3. Assign: Reference = `U1`, Value = `BQ25896RTWT`

#### Step 2: Create Input Power Hierarchical Label
1. Press `Ctrl+H` for hierarchical label
2. Create `VBUS` with direction **Input**
3. Place at the left edge of the sheet

#### Step 3: Wire VBUS to BQ25896
```
[VBUS] ──┬── C_IN1 (+) ──┬── U1:VBUS (pin 1)
         │               │
         └── C_IN1 (-) ──┴── GND
```

**Wiring steps:**
1. Connect VBUS hierarchical label to U1 pin 1 (VBUS)
2. Place C_IN1 (22µF) close to U1
3. Connect C_IN1 between VBUS and GND

#### Step 4: Create VSYS Output Hierarchical Label
1. Create `VSYS` hierarchical label with direction **Output**
2. Place on the right side of the sheet

#### Step 5: Wire BQ25896 Output Side
```
U1:SYS (pin 15,16) ────┬──── [VSYS] (Hierarchical Label)
                       │
                       └──── C_SYS (+)
                             │
                             C_SYS (-) ──── GND
```

#### Step 6: Wire BTST, SW, and REGN Pins

The BQ25896 contains an internal synchronous buck converter that requires external bootstrap and bypass capacitors for proper operation.

**Understanding BTST (Bootstrap) - Pin 21:**
The BTST pin powers the high-side N-channel MOSFET gate driver. Since the high-side FET source floats at the SW voltage during conduction, the gate must be driven above SW. The bootstrap circuit:
1. When SW is LOW: Internal diode charges C_BTST from REGN (~3.3V)
2. When SW swings HIGH: C_BTST provides gate drive voltage (V_SW + 3.3V) to turn on the high-side FET
3. This "flying capacitor" technique enables N-channel MOSFETs in high-side configurations

**Understanding SW (Switching Node) - Pins 19, 20:**
The SW pins are the internal buck converter's switching output. This node:
- Swings rapidly between GND (low-side FET on) and ~VBUS (high-side FET on)
- Connects to inductor L1, which filters the PWM into smooth DC at the SYS output
- Both pins 19 and 20 must be connected together (paralleled for current handling)

**Understanding REGN (Internal LDO) - Pin 22:**
REGN is a 3.3V internal LDO output that:
- Powers the BQ25896's internal digital logic and analog circuits
- Provides the voltage reference for the TS (thermistor) divider network
- Must have a low-ESR ceramic bypass capacitor for stability

**Detailed Schematic:**
```
                          ┌───────────────────────────────────────┐
                          │              BQ25896                  │
                          │                                       │
    C_BTST (100nF 10V)    │                                       │
         ┌────────────────┤ BTST (pin 21)                         │
         │                │       │                               │
         │                │       │ (internal bootstrap diode     │
         │                │       │  charges C_BTST from REGN)    │
         │                │       ▼                               │
         │                │   ┌───────┐                           │
         │                │   │ Hi-FET│──┐                        │
         │                │   └───┬───┘  │                        │
         │                │       │      │ (internal              │
         └────────────────┼───────┘      │  buck converter)       │
                          │              ▼                        │
    To L1 ◄───────────────┤ SW (pins 19,20)                       │
    (inductor)            │       │      │                        │
                          │   ┌───┴───┐  │                        │
                          │   │ Lo-FET│──┘                        │
                          │   └───┬───┘                           │
                          │       │                               │
                          │      GND                              │
                          │                                       │
    C_REGN (100nF 10V)    │                                       │
         ┌────────────────┤ REGN (pin 22)                         │
         │                │   (3.3V internal LDO output)          │
         │                │                                       │
         └──── GND        │  PGND (pins 17,18) ────── GND         │
                          │                                       │
                          └───────────────────────────────────────┘
```

**Full BTST/SW/REGN Wiring Diagram:**
```
                                      U1:SYS (pins 15,16)
                                            │
                                            │ (to C_SYS and [VSYS])
                                            │
    ┌───────────────────────────────────────┤
    │                                       │
    │   L1 (2.2µH Charger Inductor)         │
    │   ┌───────────────────────────────────┘
    │   │
    │   │  ⚠️ Place L1 as close to U1 as possible!
    │   │     Minimize trace length between SW pins and L1 pad
    │   │
    │   └──────────────────────┐
    │                          │
    └─────────────────────┐    │
                          │    │
U1:BTST (pin 21) ─────────┼────┤
        │                 │    │
        │              C_BTST  │
        │              100nF   │
        │              10V     │
        │                │     │
        └────────────────┼─────┴──── U1:SW (pins 19,20)
                         │                   │
                        (+)                  │
                                            (both pins tied together)

U1:REGN (pin 22) ─────┬───────────────────────────► (to R19 for TS divider)
                      │
                   C_REGN
                   100nF
                   10V
                      │
                     GND
```

**Wiring Steps:**

1. **Place C_BTST (100nF ceramic, ≥10V):**
   - Position directly adjacent to U1, as close to pin 21 as possible
   - Use 0402 or 0603 package for compact placement

2. **Wire BTST capacitor:**
   - Connect C_BTST positive terminal to U1:BTST (pin 21)
   - Connect C_BTST negative terminal to U1:SW (pins 19 AND 20)
   - **Critical:** The negative side connects to SW, NOT to GND!

3. **Place L1 (2.2µH inductor):**
   - Position between U1 and the SYS output area
   - Use 7×7mm shielded inductor rated ≥4A saturation current

4. **Wire SW to inductor:**
   - Connect U1:SW pin 19 to L1 terminal 1
   - Connect U1:SW pin 20 to L1 terminal 1 (same node)
   - Connect L1 terminal 2 to U1:SYS (pins 15,16)
   - **⚠️ Layout Critical:** Keep SW traces short and wide — this node switches at high frequency and carries significant current

5. **Place C_REGN (100nF ceramic, ≥10V):**
   - Position close to U1 pin 22
   - Use 0402 package

6. **Wire REGN bypass:**
   - Connect C_REGN positive terminal to U1:REGN (pin 22)
   - Connect C_REGN negative terminal to GND
   - This node also feeds the TS divider (wired in Step 10)

**Component Values:**
| Ref | Value | Package | Notes |
|-----|-------|---------|-------|
| C_BTST | 100nF | 0402 | X5R/X7R, ≥10V rating |
| C_REGN | 100nF | 0402 | X5R/X7R, ≥10V rating |
| L1 | 2.2µH | 7×7mm | ≥4A saturation, shielded, low DCR (<50mΩ) |

**⚠️ Common Mistakes to Avoid:**
1. **C_BTST to GND:** The bootstrap capacitor negative terminal MUST connect to SW, not GND
2. **Long SW traces:** Keep switching node traces as short as possible to minimize EMI and ringing
3. **Single SW pin:** Both pins 19 and 20 must be connected — they share the switching current
4. **Missing REGN capacitor:** The internal LDO requires bypassing for stable operation

#### Step 7: Wire ILIM Resistor
```
U1:ILIM (pin 10) ──── R5 (800Ω) ──── GND      (I_INLIM = 1040V / 800Ω = 1.3A)
```

#### Step 8: Create and Wire I2C Labels
1. Create `I2C1_SDA` hierarchical label (Bidirectional)
2. Create `I2C1_SCL` hierarchical label (Input)

```
U1:SDA (pin 6) ──── [I2C1_SDA]
U1:SCL (pin 5) ──── [I2C1_SCL]
```

#### Step 9: Create and Wire Control Signals
1. Create `CHG_INT` hierarchical label (Output)
2. Create `CHG_CE` hierarchical label (Input)
3. Create `CHG_OTG` hierarchical label (Input)

```
U1:INT (pin 7) ──── [CHG_INT]
U1:CE_N (pin 9) ──── [CHG_CE]
U1:OTG_IUSB (pin 8) ──── [CHG_OTG]
```

#### Step 10: Wire NTC/TS Circuit
1. Place J7 (NTC pads) at the left edge
2. Wire the TS divider (REGN already has C_REGN to GND from Step 6):

```
U1:REGN (pin 22) ──┬── C_REGN ──── GND  (from Step 6)
                   │
                   └── R19 (5.1k) ──┬──── U1:TS (pin 11)
                                    │
                                    ├──── J7:TS (pad 1)
                                    │
                                    └──── R20 (27k) ──┬── GND
                                                      │
                                                      └── J7:GND (pad 2)
```

**Note:** The actual NTC thermistor connects externally between J7 pads (TS to GND).

**NTC Divider Verification:**
The R19/R20 values assume a **10kΩ NTC at 25°C**. Verify your NTC matches, or recalculate:
```
V_TS = V_REGN × (R_NTC ∥ R20) / (R19 + (R_NTC ∥ R20))
```
At 25°C with 10kΩ NTC: V_TS ≈ 3.3V × (10k∥27k)/(5.1k + 10k∥27k) ≈ **1.53V**

This should fall within BQ25896 WARM/COOL thresholds. Consult TI's SLUA917 app note for other NTC values.

**⚠️ NTC Disconnect Detection (Firmware Guidance):**

If the NTC wire breaks or comes loose, the BQ25896 loses temperature feedback. The TS pin floats high (pulled by R19 to REGN), which BQ25896 interprets as "very cold" battery — it may continue charging at full rate.

**Firmware safety check:** Read TS voltage via BQ25896 register 0x12 (ADC):
- **Normal range:** 0.5V - 2.5V (depending on temperature)
- **Abnormal HIGH (>2.8V):** NTC disconnected or failed — reduce charge current via I2C
- **Abnormal LOW (<0.3V):** NTC shorted to GND — reduce charge current

```c
// Example firmware check
float ts_voltage = read_bq25896_ts_adc();
if (ts_voltage > 2.8f || ts_voltage < 0.3f) {
    set_charge_current(100);  // Reduce to 100mA safety mode
    flag_ntc_fault();
}
```

The BQ25896 has internal safety limits, but this firmware layer provides defense-in-depth.

#### Step 11: Wire BQ25896 BAT Pins (connects to protection circuit in Step 12)

The BQ25896 BAT pins connect to the protected battery rail created by the BQ29705 circuit (wired in Step 12). **Do not create a VBAT_PROT input label** — the connection is made directly on this sheet to the protection circuit output.

```
U1:BAT (pins 13,14) ────┬──── C_BAT1 (22µF) ──── GND
                        │
                        └──── (wire to VBAT_PROT node after Step 12)
```

**Wiring Steps:**
1. Connect U1:BAT pins 13 and 14 together
2. Add C_BAT1 (22µF) from the BAT node to GND for decoupling
3. **Leave a wire stub or net label** — connect to the VBAT_PROT node after completing Step 12

**Complete topology (after Step 12 is wired):**
```
J3:BAT+ ── Q_REV ──┬── [VBAT_PROT] (OUTPUT to other sheets)
                   │
                   ├── U1:BAT (pins 13,14) ── C_BAT1 ── GND  ◄── YOU ARE HERE
                   │
                   └── R1 (330Ω) ── U2:BAT (BQ29705 power)
```

VBAT_PROT is an **OUTPUT** hierarchical label from this sheet — it provides the protected battery voltage to BoostConverter and FuelGauge sheets.

**⚠️ NOTE:** The fuel gauge uses LOW-SIDE sensing in the ground return path, NOT in the battery+ path.
See Section 6 (FuelGauge) for the correct current sensing topology.

#### Step 11b: Wire Additional BQ25896 Pins
These pins need explicit handling:

```
U1:PSEL (pin 2) ──── GND                    (USB input mode; tie to VBUS for adapter mode)
U1:PG_N (pin 3) ──── [CHG_PG]               (Optional: Power Good output to MCU)
U1:STAT (pin 4) ──── [CHG_STAT]             (Optional: Charge status to MCU or LED)
U1:QON_N (pin 12) ──── R_QON (10k) ──── U1:REGN  (Tie high via pull-up to disable ship mode)
U1:PMID (pin 23) ──── C_PMID (100nF) ──── GND    (Decouple if using PMID output)
```

**Notes:**
- PSEL: LOW = USB/OTG input, HIGH = Adapter input (higher current)
- PG_N: Active-low power good, useful for MCU power sequencing
- STAT: LOW = Charging, HIGH-Z = Complete or no battery, HIGH = Fault
- QON_N: LOW triggers ship mode (ultra-low power). Tie high unless ship mode button needed
- PMID: Internal node, decoupling recommended if used as intermediate voltage rail

#### Step 12: Place and Wire BQ29705 Protection IC
1. Place U2 (BQ29705DSET) below U1
2. Place Q1 and Q2 (CSD18534Q5A) next to U2 in back-to-back **LOW-SIDE** configuration

**⚠️ CRITICAL: N-channel MOSFETs MUST be in the ground path (LOW side)**

The BQ29705 is a low-side gate driver. Its COUT/DOUT pins drive to VSS (battery negative) when OFF,
and pull toward BAT when ON. N-channel MOSFETs require Vgs > Vth relative to their SOURCE pin.
Placing them on the positive rail would require a charge pump (which BQ29705 lacks).

```
                    ┌─────────────────────────────────────────────────────────────┐
                    │         REVERSE BATTERY PROTECTION (Q_REV)                   │
                    │                                                              │
J3:BAT+ (pad 1) ────┤► Q_REV:Source (Si7461DP P-channel)                          │
                    │                                                              │
                    │  Q_REV:Drain ──┬── [VBAT_PROT] ──► To BQ25896:BAT (direct) │
                    │                │                     & FuelGauge:BATT       │
                    │                └── R1 (330Ω) ──┬── U2:BAT (pin 5)           │
                    │                                │                             │
                    │                                └── C_PROT (100nF) ── Sys GND │
                    │                                                              │
                    │  Q_REV:Gate ──── R_REV (10k) ──── J3:BAT- (pad 2)           │
                    └─────────────────────────────────────────────────────────────┘

**⚠️ LOW-SIDE SENSING:** FuelGauge measures current in GND return path (CSP=System GND, CSN→BAT_NEG)

J3:BAT- (pad 2) ──── U2:VSS (pin 1)      (IC references raw battery negative)

U2:COUT (pin 3) ──── R_G1 (47Ω) ──── Q1:Gate (Charge FET)
U2:DOUT (pin 2) ──── R_G2 (47Ω) ──── Q2:Gate (Discharge FET)
U2:V- (pin 6) ──── R2 (2.2k) ──── Q1:Source  (System GND - senses FET voltage drop)
```

**⚠️ REVERSE BATTERY PROTECTION (Q_REV) — How It Works:**

| Condition | Q_REV Gate | Q_REV Source | Vgs | Result |
|-----------|------------|--------------|-----|--------|
| **Correct polarity** | 0V (via R_REV to BAT-) | +4.2V (BAT+) | -4.2V | ON (conducts) |
| **Reversed polarity** | ~0V (R_REV to positive terminal) | 0V (at negative) | ~0V | OFF (blocks) |

When the battery is reversed:
- Q_REV's body diode is reverse-biased (cathode at Source, anode at Drain)
- Gate is pulled toward the "new positive" through R_REV but cannot exceed Source
- MOSFET blocks current, protecting BQ25896 and BQ29705 from damage

**Note:** The Si7461DP body diode conducts briefly during insertion until Vgs reaches threshold.
This is acceptable; the ~4.6mΩ Rds(on) drop is negligible during normal operation.

**⚠️ CRITICAL: BQ29705 Pin Connections (DSBGA-6 Ball Designations)**
- **BAT (B1)** = Positive power supply input. Connect to **BAT+** via R1.
- **VSS (A1)** = Ground reference. Connect to **BAT-** (raw battery negative).
- If BAT is connected to BAT-, the IC will not power up and FETs will remain OFF.

Ball mapping: A1=VSS, A2=DOUT, B1=BAT, B2=COUT, C1=V-, C2=NC (see Design Spec §5 for full table)

**⚠️ CRITICAL: V- Pin Wiring for Overcurrent Detection:**

The V- pin detects overcurrent by measuring the voltage drop across the protection FETs (Q1+Q2).
- **VSS (A1)** connects to BAT- (raw battery negative)
- **V- (C1)** connects to Q1:Source (System GND) via R2

During discharge, current flows: BAT- → Q2 → Q1 → System GND
The FET Rds(on) creates a voltage drop: V_drop = I × (Rds_Q1 + Rds_Q2)
System GND becomes **positive** relative to BAT-, and V- senses this.

**If V- were connected to Q2:Source (same as VSS/BAT-):**
- V- and VSS would be at the same potential
- BQ29705 would see 0V differential regardless of current
- **OCD and SCD would NEVER trip** — catastrophic safety failure!

R2 (2.2kΩ) sets the OCD filtering. The BQ29705 has internal 100mV threshold.

#### Step 13: Wire Protection MOSFETs and Fuel Gauge Sense Resistor (LOW-SIDE Configuration)

**⚠️ CRITICAL: R_SENSE_FG goes on THIS sheet, between Q2:Source and Battery-**

```
Q1:Drain ────┬──── Q2:Drain        (common drain - tied together internally)

Q1:Source ──── GND symbol          (System GND - protected ground for all circuits)

Q2:Source ──┬── [GND_SENSE] ──── R_SENSE_FG (5mΩ) ──── [BAT_NEG] ──┬── J3:BAT- (pad 2)
            │                                                       │
     (to FuelGauge CSP)                                    (to FuelGauge CSN)
```

**Wiring Steps:**
1. Connect Q1:Drain to Q2:Drain (common drain node)
2. Connect Q1:Source to a `GND` power symbol (this defines System GND)
3. Place R_SENSE_FG (5mΩ, 2512 package) near Q2
4. Connect Q2:Source to R_SENSE_FG terminal 1
5. Connect R_SENSE_FG terminal 2 to J3:BAT- (pad 2)
6. Create hierarchical label `GND_SENSE` (Output) at Q2:Source (same node as R_SENSE_FG terminal 1)
7. Create hierarchical label `BAT_NEG` (Output) at R_SENSE_FG terminal 2 / J3:BAT- junction

**Hierarchical Labels for FuelGauge sheet:**
| Label | Location | Direction | Purpose |
|-------|----------|-----------|---------|
| GND_SENSE | Q2:Source / R_SENSE_FG high side | Output | To MAX17055 CSP pin (IC ground reference) |
| BAT_NEG | R_SENSE_FG low side / J3:BAT- | Output | To MAX17055 CSN pin (sense negative) |

**Current flow when ON:** Battery(-) → R_SENSE_FG → Q2 → Q1 → System GND → Load → VBAT_PROT → Battery(+)

**Fuel gauge current sensing:** The MAX17055 measures voltage across R_SENSE_FG:
- CSP connects to GND_SENSE (Q2:Source, high side of resistor)
- CSN connects to BAT_NEG (Battery-, low side of resistor)
- During discharge: CSN is below CSP (negative current)
- During charge: CSN is above CSP (positive current)

When BQ29705 detects fault:
- COUT/DOUT pull to VSS (battery negative) → Vgs ≈ 0V → FETs turn OFF
- System GND disconnects from battery negative, isolating the battery

#### Step 14: 3.3V LDO Regulator — See MCU Sheet (Section 10)

**⚠️ The LDO has been moved to the MCU Sheet (Section 10.4, Step 17).**

The LDO is powered from `VSYS` (system voltage from BQ25896). Since the fuel gauge is now on the **battery path** (measuring charge+discharge current), all system loads including the LDO connect directly to VSYS.

**Do NOT place the LDO on this sheet.** Proceed to Section 10.4, Step 17 for LDO implementation.

### 5.5 Completed BatteryManagement Schematic Diagram

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                           BatteryManagement                                       │
│                                                                                   │
│ From USB_Power                                                    To Other Sheets │
│       │                                                                  │        │
│       ▼                                                                  ▼        │
│ [VBUS]──┬──C_IN1──┬───────────────────────────────────────────────────────────   │
│         │  22µF   │                                                               │
│         └─────────┼─►┌──────────────────────────┐                                │
│                   │  │         U1               │                                │
│                   │  │       BQ25896            │                                │
│                   │  │                          │                                │
│                   └─►│ VBUS(1)                  │                                │
│                      │                SYS(15,16)├──┬──[VSYS]────────────────────►│
│  [CHG_OTG]──────────►│ OTG_IUSB(8)              │  │                             │
│                      │                          │  └──C_SYS──GND                 │
│  [CHG_CE]───────────►│ CE_N(9)                  │       22µF                     │
│                      │                          │                                │
│  [CHG_INT]◄──────────│ INT(7)                   │                                │
│                      │                          │                                │
│  [I2C1_SCL]─────────►│ SCL(5)                   │                                │
│                      │                   BTST(21)├──C_BTST──┐                    │
│  [I2C1_SDA]◄────────►│ SDA(6)                   │  100nF    │                    │
│                      │                          │           │                    │
│  J7:TS──┬──R19──┬───►│ TS(11)          SW(19,20)├──────────┴──L1──┘             │
│  NTC    │ 5.1k  │    │                          │          2.2µH                 │
│  Pads   │       │    │                  REGN(22)├──C_REGN──GND                   │
│         │       │    │                          │  100nF                         │
│  J7:GND─┴─R20───┴─GND│                BAT(13,14)├──C_BAT1──┬──[VBAT_PROT]───┐   │
│            27k       │                          │  22µF    │                │   │
│                      │              ILIM(10)    │          │                │   │
│                      │                 │        │          │                │   │
│                      │                 R5       │          │                ▼   │
│                      │                800Ω      │          │    ┌───────────────┐│
│                      │                 │        │          │    │    U2         ││
│                      │                GND       │          │    │  BQ29705      ││
│                      │                          │          │    │               ││
│                      │            PGND(17,18)   │          │    │  BAT(4)◄─R1───┘│
│                      │                 │        │          │    │       330Ω     │
│                      └─────────────────┼────────┘          │    │               ││
│                                        │                   │    │ COUT(3)──►Q1:G ││
│                                        ▼                   │    │               ││
│                                       GND                  │    │ DOUT(2)──►Q2:G ││
│                                                            │    │               ││
│                                                            │    │ VSS(1)────────┘│
│                              ┌─────────────────────────────┘    │               ││
│                              │                                  └───────────────┘│
│                              │                                        │          │
│                              │                                        │ (to BAT-)│
│                              │                                        ▼          │
│  J3:BAT+ ────────────────────┴──► [VBAT_PROT]                                    │
│  (Battery +)                      (direct - no FETs)                             │
│                                                                                  │
│               ┌──────────────────────────────────────────────────────────────┐   │
│               │     LOW-SIDE PROTECTION (Ground Path)                        │   │
│               │                                                              │   │
│               │  J3:BAT- ──┬── U2:VSS ──┬── R1 ── U2:BAT                    │   │
│               │  (Bat -)   │            │   330Ω    │                        │   │
│               │            │            │           └── C_PROT ── Sys GND    │   │
│               │            │            │                                    │   │
│               │            └────────────┼────────────────────────────────┐   │   │
│               │                         │                                │   │   │
│               │            ┌────────────┼────────────┐ ┌────────────┐    │   │   │
│               │            │ Q2 (Dschg) │            │ │ Q1 (Chg)   │    │   │   │
│               │            │ CSD18534Q5A│            │ │ CSD18534Q5A│    │   │   │
│               │            │            │            │ │            │    │   │   │
│               │            │ S ─────────┘ (to BAT-)  │ │ S ─────────┼────┼───┼──►│
│               │            │                         │ │            │    │   │   │
│               │            │ D ──────────────────────┼─┼─ D         │    │   │   │
│               │            │            (common)     │ │            │    │   │   │
│               │            └─────────────────────────┘ └────────────┘    │   │   │
│               │                                              │           │   │   │
│               │    U2:V- ── R2 ──────────────────────────────┘           │   │   │
│               │            2.2k   (sense at Q1 source = System GND)      │   │   │
│               │                                                          │   │   │
│               └──────────────────────────────────────────────────────────┘   │   │
│                                                                              │   │
│                                                               System GND ◄───┘   │
│                                                                                   │
└──────────────────────────────────────────────────────────────────────────────────┘
```

---

## 6. Subsheet: FuelGauge

### 6.1 Sheet Information

| Property | Value |
|----------|-------|
| Sheet Name | FuelGauge |
| File Name | FuelGauge.kicad_sch |
| Page Size | A4 |
| Description | MAX17055 fuel gauge IC — **LOW-SIDE current sensing** |

### 6.2 Components

| Designator | Symbol | Value | Footprint | Description |
|------------|--------|-------|-----------|-------------|
| U3 | MAX17055 | MAX17055EWL+T | Package_CSP:WLP-9_1.53x1.53mm_P0.4mm | Fuel gauge IC |
| C_FG1 | C | 100nF | Capacitor_SMD:C_0402_1005Metric | BATT-CSP bypass |

**⚠️ NOTE: R_SENSE_FG is NOT on this sheet!** The sense resistor is placed on the BatteryManagement sheet (Section 5, Step 13), between Q2:Source and J3:BAT-. This sheet only receives the GND_SENSE and BAT_NEG labels from BatteryManagement.

### 6.3 Hierarchical Labels

| Label Name | Direction | Purpose |
|------------|-----------|---------|
| VBAT_PROT | Input | Protected battery+ (from BatteryManagement) → to MAX17055 BATT pin for power |
| GND_SENSE | Input | Q2:Source (from BatteryManagement) → to MAX17055 CSP pin (IC ground reference) |
| BAT_NEG | Input | Battery- / R_SENSE_FG low side (from BatteryManagement) → to MAX17055 CSN pin |
| I2C1_SDA | Bidirectional | I2C data |
| I2C1_SCL | Input | I2C clock |
| FG_ALRT | Output | Alert interrupt to MCU |

**Where do these labels come from?**
- `VBAT_PROT` — Created on BatteryManagement at Q_REV:Drain (protected battery+)
- `GND_SENSE` — Created on BatteryManagement at Q2:Source (high side of R_SENSE_FG)
- `BAT_NEG` — Created on BatteryManagement at J3:BAT- / R_SENSE_FG low side

**⚠️ CRITICAL: MAX17055 Requires LOW-SIDE Current Sensing**

The MAX17055 pin architecture:
- **BATT (B1)** = Power supply input (connects to Battery+, provides 2.3V–4.9V to IC)
- **CSP (C3)** = IC's internal ground reference (0V reference point)
- **CSN (A3)** = Sense resistor negative terminal (Kelvin connection to Battery-)

The IC is powered by: **V_BATT − V_CSP = ~3.7V** (battery voltage)

**⚠️ The sense resistor MUST be in the ground return path (LOW-SIDE).**

If placed in the high-side (battery+ path), V_BATT and V_CSP would both be at battery voltage,
resulting in V_BATT − V_CSP ≈ 0V — **the IC would not power up!**

**Correct Topology:** Sense resistor in **ground return path**:
```
Battery+ → BQ29705 Protection → VBAT_PROT → BQ25896 BAT (direct, no sense resistor)

Ground return (fuel gauge measures current here):
System GND (loads) ← Q1 ← Q2 ← R_SENSE_FG ← Battery-
     ↑                            ↑
    CSP                          CSN
 (IC ground)                (sense negative)
```

This ensures the fuel gauge measures **both charge and discharge current**.

**UVLO Recovery Behavior:**

Since MAX17055 BATT connects to battery+ and CSP to System GND, the fuel gauge loses power
when BQ29705 opens the ground path (Q1/Q2 OFF):
- Ground path opens → no current flow → MAX17055 loses power
- Device turns off completely

**Recovery:** When USB plugged in and protection clears, ModelGauge m5 estimates SOC from
Open Circuit Voltage (OCV) within seconds. Battery percentage recovers immediately.

### 6.4 Wiring Instructions

#### Step 1: Place MAX17055
1. Press `A`, search for `MAX17055`
2. Place U3 in the center of the sheet
3. Assign: Reference = `U3`, Value = `MAX17055EWL+T`

#### Step 2: Create Hierarchical Labels
1. Create `VBAT_PROT` (Input) - left edge (battery+ for BATT pin power)
2. Create `GND_SENSE` (Input) - left edge (Q2:Source, high side of sense resistor, for CSP)
3. Create `BAT_NEG` (Bidirectional) - left edge (Battery-, low side of sense resistor, for CSN)
4. Create `I2C1_SDA` (Bidirectional) - left edge
5. Create `I2C1_SCL` (Input) - left edge
6. Create `FG_ALRT` (Output) - right edge

#### Step 3: Wire Power Supply (BATT from battery+, CSP to GND_SENSE)
```
[VBAT_PROT] ──┬── U3:BATT (pin B1)
              │
              └── C_FG1 (+)
                      │
                      └── C_FG1 (-) ──┬── U3:CSP (pin C3)
                                      │
                                      └── [GND_SENSE] (Q2:Source)
```

**⚠️ C_FG1 bypasses BATT to CSP (the IC's power rails). CSP is NOT system ground — it's Q2:Source.**

#### Step 4: Wire I2C
```
[I2C1_SDA] ──── U3:SDA (pin C1)
[I2C1_SCL] ──── U3:SCL (pin A2)
```

#### Step 5: Wire Alert
```
U3:ALRT (pin B2) ──── [FG_ALRT]
```

#### Step 6: Wire Current Sense Inputs (from BatteryManagement sheet)

**⚠️ R_SENSE_FG is NOT on this sheet!** It's on the BatteryManagement sheet. This sheet only connects the hierarchical labels to the MAX17055 pins.

```
[GND_SENSE] ──── U3:CSP (pin C3)     ← IC ground reference (receives Q2:Source from BatteryManagement)

[BAT_NEG] ──── U3:CSN (pin A3)       ← Sense negative (receives Battery- side from BatteryManagement)
```

**Wiring Steps:**
1. Create hierarchical label `GND_SENSE` (Input) on the left side of the sheet
2. Connect `GND_SENSE` directly to U3:CSP (pin C3)
3. Create hierarchical label `BAT_NEG` (Input) on the left side of the sheet
4. Connect `BAT_NEG` directly to U3:CSN (pin A3)

**What's happening on BatteryManagement (for reference):**
```
Q2:Source ──[GND_SENSE]── R_SENSE_FG (5mΩ) ──[BAT_NEG]── J3:BAT-
                │                                  │
          (to CSP here)                      (to CSN here)
```

The MAX17055 measures voltage across R_SENSE_FG via the hierarchical labels:
- CSP sees GND_SENSE (high side of sense resistor = Q2:Source)
- CSN sees BAT_NEG (low side of sense resistor = Battery-)
- During discharge: CSN is below CSP (negative current reading)
- During charge: CSN is above CSP (positive current reading)

#### Step 7: Wire Unused Pins
```
U3:THRM (pin C2) ──── NC (or optional external NTC)
U3:REG (pin B3) ──── NC (internal regulator, can bypass to CSP if needed)
U3:AIN (pin A1) ──── NC (auxiliary input, unused)
```

### 6.5 Completed FuelGauge Schematic Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              FuelGauge                                       │
│                           (LOW-SIDE Sensing)                                 │
│                                                                              │
│  From BatteryManagement                                                      │
│  (protected battery+)                                                        │
│         │                                                                    │
│         ▼                                                                    │
│  [VBAT_PROT] ──────────────────────────────────────────────────┐            │
│         │                                                       │            │
│         │  (Power supply to IC: V_BATT - V_CSP = ~3.7V)        │            │
│         │                                                       │            │
│         ├──────────────────┐                                    │            │
│         │                  │                                    │            │
│         │         C_FG1    │    ┌──────────────────────────┐    │            │
│         │         100nF    │    │          U3              │    │            │
│         │           │      │    │       MAX17055           │    │            │
│         │           │      │    │       (WLP-9)            │    │            │
│         │           │      └───►│ BATT (B1) ◄──────────────┘    │            │
│         │           │           │   (Power input)               │            │
│         │           │           │                               │            │
│  [I2C1_SCL]─────────┼──────────►│ SCL (A2)                     │            │
│         │           │           │                               │            │
│  [I2C1_SDA]◄────────┼──────────►│ SDA (C1)                     │            │
│         │           │           │                               │            │
│         │           │           │ THRM (C2) ─── NC              │            │
│         │           │           │                               │            │
│         │           │           │ ALRT (B2) ────────────────────┼─[FG_ALRT] │
│         │           │           │                               │            │
│         │           │           │ REG (B3) ─── NC               │            │
│         │           │           │                               │            │
│         │           │           │ AIN (A1) ─── NC               │            │
│         │           │           │                               │            │
│         │           └──────────►│ CSP (C3) ◄────────────────────┼─[GND_SENSE]│
│         │                       │   (IC ground reference)       │  (INPUT)   │
│         │                       │                               │            │
│         │                       │ CSN (A3) ◄────────────────────┼─[BAT_NEG] │
│         │                       │   (Sense resistor negative)   │  (INPUT)   │
│         │                       │                               │            │
│         │                       └───────────────────────────────┘            │
│         │                                                                    │
│  ⚠️ NOTE: R_SENSE_FG is on BatteryManagement sheet, NOT here!              │
│                                                                              │
│  On BatteryManagement sheet:                                                 │
│  Q2:Source ──[GND_SENSE]── R_SENSE_FG (5mΩ) ──[BAT_NEG]── J3:BAT-          │
│                                                                              │
│  This sheet just receives the labels and connects them to MAX17055:         │
│  • GND_SENSE → CSP (high side of sense resistor = Q2:Source)               │
│  • BAT_NEG → CSN (low side of sense resistor = Battery-)                   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Subsheet: BoostConverter

### 7.1 Sheet Information

| Property | Value |
|----------|-------|
| Sheet Name | BoostConverter |
| File Name | BoostConverter.kicad_sch |
| Page Size | A4 |
| Description | TPS61088 boost converter with MCP4561 adjustable feedback |

### 7.2 Components

| Designator | Symbol | Value | Footprint | Description |
|------------|--------|-------|-----------|-------------|
| U4 | TPS61088 | TPS61088RHLR | Package_DFN_QFN:QFN-20-1EP_4x4mm_P0.5mm | Boost converter IC |
| U11 | MCP4561 | MCP4561-104E/MS | Package_SO:SOIC-8_3.9x4.9mm_P1.27mm | 100kΩ digital potentiometer |
| L2 | L | 1µH | Inductor_SMD:L_Coilcraft_XAL7070 | Boost inductor (25A sat, Coilcraft XAL7070-102MEC, LCSC C6609502) |
| R4 | R | 100k | Resistor_SMD:R_0402_1005Metric | FB top resistor |
| C_BOUT1 | C | 22µF | Capacitor_SMD:C_0805_2012Metric | Output cap 1 |
| C_BOUT2 | C | 22µF | Capacitor_SMD:C_0805_2012Metric | Output cap 2 |
| C16 | C | 100µF | Capacitor_SMD:C_1206_3216Metric | Bulk output cap |
| C17 | C | 22nF | Capacitor_SMD:C_0402_1005Metric | Soft start cap |
| C_COMP | C | 10nF | Capacitor_SMD:C_0402_1005Metric | Compensation cap |
| R_COMP | R | 100k | Resistor_SMD:R_0402_1005Metric | Compensation resistor |
| C_BIN1 | C | 22µF | Capacitor_SMD:C_0805_2012Metric | Input bypass |
| R_SAFE | R | 22k | Resistor_SMD:R_0402_1005Metric | MCP4561 failsafe (overvoltage ceiling) |
| C_BOOT | C | 100nF | Capacitor_SMD:C_0402_1005Metric | Bootstrap capacitor (BOOT to SW) |
| C_VCC | C | 1µF | Capacitor_SMD:C_0402_1005Metric | VCC LDO bypass |
| R_ILIM | R | 5.36k | Resistor_SMD:R_0402_1005Metric | Input current limit (~10A) |

**⚠️ L2 Inductor: Coilcraft XAL7070-102MEC (LCSC C6609502)**

Specifications:
- Inductance: 1µH ±20%
- Saturation Current: 25A (exceeds 15A requirement)
- Rated Current: 34.8A
- DCR: 2.84mΩ typ
- Package: 7.0mm × 7.0mm × 4.0mm (XAL7070)

**Footprint:** Use `Inductor_SMD:L_Coilcraft_XAL7070` or create custom footprint from Coilcraft datasheet. The XAL7070 has bottom terminations with 3.4mm pad width.

**Note:** This inductor exceeds the original 15A requirement with significant margin, allowing for transient peaks during boost operation.

### 7.3 Hierarchical Labels

| Label Name | Direction | Purpose |
|------------|-----------|---------|
| VSYS | Input | System voltage from BQ25896 (3.5V-4.2V) |
| BOOST_EN | Input | Enable signal from MCU |
| I2C1_SDA | Bidirectional | I2C data (for MCP4561) |
| I2C1_SCL | Input | I2C clock |
| V_BOOST | Output | Boosted voltage output (1.5V-7V) |
| GND | Bidirectional | Ground |

### 7.4 Wiring Instructions

#### Step 1: Place TPS61088
1. Press `A`, search for `TPS61088`
2. Place U4 in the center-left of the sheet
3. Assign: Reference = `U4`, Value = `TPS61088RHLR`

#### Step 2: Create Hierarchical Labels
1. Create `VSYS` (Input) - left edge (from BQ25896)
2. Create `BOOST_EN` (Input) - left edge
3. Create `I2C1_SDA` (Bidirectional) - left edge
4. Create `I2C1_SCL` (Input) - left edge
5. Create `V_BOOST` (Output) - right edge

#### Step 3: Wire Input Power
1. Connect VSYS label to U4:VIN (pin 9)
2. Place C_BIN1 (22µF) with (+) on VSYS/VIN net
3. Connect C_BIN1 (-) to GND

#### Step 4: Wire Enable
```
[BOOST_EN] ──── U4:EN (pin 2)
```

#### Step 5: Wire MODE Pin
```
U4:MODE (pin 13) ──── GND   (forces PFM mode at light loads)
```

#### Step 6: Wire Inductor
```
U4:SW (pins 4,5,6,7) ──── L2 (1µH) ──── U4:VOUT (pins 14,15,16)
```

#### Step 7: Wire Output Capacitors and Create V_BOOST
1. Connect U4:VOUT (pins 14,15,16) together
2. Place C_BOUT1 (22µF) from VOUT to GND
3. Place C_BOUT2 (22µF) from VOUT to GND
4. Place C16 (100µF) from VOUT to GND
5. Create V_BOOST label on the VOUT net

#### Step 8: Wire Soft Start
```
U4:SS (pin 10) ──── C17 (22nF) ──── GND
```

#### Step 9: Wire Compensation Network
```
U4:COMP (pin 18) ──── R_COMP (100k) ──── C_COMP (10nF) ──── GND
```
Note: R_COMP and C_COMP are in series from COMP to GND.

#### Step 9a: Wire Bootstrap Capacitor
```
U4:BOOT (pin 8) ──── C_BOOT (100nF) ──── U4:SW (pins 4,5,6,7)
```
The bootstrap capacitor connects between BOOT and the SW node.

#### Step 9b: Wire VCC Bypass
```
U4:VCC (pin 1) ──── C_VCC (1µF) ──── GND
```
VCC is the internal 5V LDO output — bypass to GND.

#### Step 9c: Wire Current Limit
```
U4:ILIM (pin 19) ──── R_ILIM (5.36k) ──── GND
```
Sets input current limit: I_ILIM = 53550 / R_ILIM ≈ 10A

#### Step 9d: Wire FSW (Frequency Select)
```
U4:FSW (pin 3) ──── U4:VIN (pin 9)
```
Tie FSW to VIN for maximum switching frequency (500kHz).

#### Step 9e: Wire Ground Pins
```
U4:AGND (pin 20) ──── GND
U4:PGND (pin 21) ──── GND
```

#### Step 9f: NC Pins
Pins 11 and 12 are NC (no connection) — leave unconnected.

#### Step 10: Place and Wire MCP4561 Digital Potentiometer
1. Place U11 (MCP4561) near U4
2. Wire I2C connections:

```
[I2C1_SDA] ──── U11:SDA (pin 3)
[I2C1_SCL] ──── U11:SCL (pin 2)
```

3. Wire address pin:
```
U11:HVC/A0 (pin 1) ──── GND   (I2C address = 0x2C)
```

#### Step 11: Wire Adjustable Feedback Network with R_SAFE Failsafe

**Linear step-by-step:**

1. Place R4 (100k) near U4
2. Connect R4 terminal 1 to V_BOOST net (the output rail)
3. Connect R4 terminal 2 to U4:FB (pin 17) — this junction is the "FB node"
4. Place R_SAFE (22k) near R4
5. Connect R_SAFE terminal 1 to FB node (R4:2 / U4:FB junction)
6. Connect R_SAFE terminal 2 to GND
7. Connect U11:P0W (pin 6, wiper) to FB node
8. Connect U11:P0A (pin 5) to GND
9. Connect U11:P0B (pin 7) to GND

**Resulting circuit:**
```
[V_BOOST] ──── R4 (100k) ──┬── U4:FB (pin 17)
                           │
                           ├── U11:P0W (pin 6, wiper)
                           │
                           └── R_SAFE (22k) ──── GND

U11:P0A (pin 5) ──── GND
U11:P0B (pin 7) ──── GND
```

**Explanation:** The digital pot (0-100kΩ) sets the bottom resistor (R_B) in the feedback divider. R_SAFE provides **overvoltage protection** if MCP4561 wiper fails open:

| Condition | R_B | V_BOOST |
|-----------|-----|---------|
| Pot = 0Ω | 0Ω ∥ 22kΩ ≈ 0Ω | 1.21V (minimum) |
| Pot = 100kΩ | 100kΩ ∥ 22kΩ = 18kΩ | 7.95V (normal max) |
| **Pot wiper open** | **22kΩ only** | **6.7V (failsafe ceiling)** |

Without R_SAFE, wiper failure → infinite R_B → uncontrolled voltage rise → catastrophic failure.

#### Step 12: Wire MCP4561 Power
```
[VSYS] ──── U11:VDD (pin 8)
            U11:VSS (pin 4) ──── GND
```

**⚠️ MCP4561 Non-Volatile Memory Wear (Firmware Guidance):**

The MCP4561 has **1 million write cycles** to NVM (non-volatile memory). If voltage is saved on every adjustment:
- Heavy user adjusting power 100×/day = 36,500 writes/year
- NVM exhaustion in ~27 years (acceptable)

**However**, if firmware writes on every button press:
- 1000 adjustments/day possible = 365,000 writes/year
- NVM exhaustion in ~3 years (marginal)

**Recommended firmware approach:**
1. Use **volatile wiper register (0x00)** during operation — unlimited writes
2. Only save to **NVM register (0x02)** on:
   - Power-down (detect VSYS falling edge via ADC)
   - 30 seconds after last adjustment (debounce)
   - User explicit "save" action

This extends NVM life to effectively unlimited for typical use.

#### Step 13: Wire Ground Planes
```
U4:PGND (pins 10,11,12) ──── GND
U4:EP (exposed pad) ──── GND
```

#### Step 14: Add V_SENSE Voltage Divider for MCU ADC

**⚠️ CRITICAL: The ESP32 ADC cannot tolerate voltages above 3.3V. V_BOOST can reach 7V or higher. You MUST step it down.**

1. Place components:
   - `R_DIV1`: 100kΩ, 0402, 1% (top resistor)
   - `R_DIV2`: 47kΩ, 0402, 1% (bottom resistor)
   - `C_VSENSE`: 100nF, 0402, 16V (filter capacitor)

2. Wire the voltage divider:
```
[V_BOOST] ──── R_DIV1 (100k) ──┬── [V_SENSE] ──► To MCU (GPIO4)
                               │
                               ├── R_DIV2 (47k) ──── GND
                               │
                               └── C_VSENSE (100nF) ──── GND
```

**Calculation:**
- At 7.0V boost: V_SENSE = 7.0V × 47/(100+47) ≈ **2.23V** ✓ Safe
- At 10V spike: V_SENSE = 10V × 47/(100+47) ≈ **3.20V** ✓ Still safe
- Divider ratio: 0.3197 (multiply ADC reading by 3.128 in firmware)

**Placement Note:** Place R_DIV1, R_DIV2, and C_VSENSE near the MCU, not near the boost converter. This keeps the high-impedance V_SENSE trace short, reducing noise pickup.

### 7.5 Completed BoostConverter Schematic Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              BoostConverter                                      │
│                                                                                  │
│  From BatteryManagement                                        To CoilDriver    │
│         │                                                            │          │
│         ▼                                                            ▼          │
│  [VSYS]─┬──C_BIN1──┬────────────────────────────────────────────────────────   │
│         │  22µF    │                                                            │
│         │          │           ┌──────────────────────────┐                     │
│         │          │           │          U4              │                     │
│         │          │           │       TPS61088           │                     │
│         │          └──────────►│ VIN(3,4)                 │                     │
│         │                      │                          │                     │
│  [BOOST_EN]───────────────────►│ EN(1)                    │                     │
│                                │                          │                     │
│                                │ MODE(7)──GND             │                     │
│                                │                          │                     │
│                                │ SS(6)──C17──GND          │                     │
│                                │        22nF              │                     │
│                                │                          │                     │
│                                │ SW(13-16)────────────────┼──L2────┐            │
│                                │                          │  1µH   │            │
│                                │                          │        │            │
│                                │ VOUT(17,18)◄─────────────┼────────┘            │
│                                │      │                   │                     │
│                                │      ├──C_BOUT1──┬──C_BOUT2──┬──C16──┬──[V_BOOST]►
│                                │      │   22µF    │   22µF    │ 100µF │         │
│                                │      │           │           │       │         │
│                                │      │           GND        GND     GND        │
│                                │      │                                         │
│                                │ FB(8)├──────────────────────────────┐          │
│                                │      │                              │          │
│                                │ COMP(9)──R_COMP──C_COMP──GND        │          │
│                                │          100k    10nF               │          │
│                                │                                     │          │
│                                │ PGND(10-12)──GND                    │          │
│                                │                                     │          │
│                                └──────────────────────────────────────┼──────   │
│                                                                      │          │
│                                         ┌────────────────────────────┘          │
│                                         │                                       │
│                      R4 (100k)          │                                       │
│  [V_BOOST] ──────────/\/\/\─────────────┤                                       │
│                                         │                                       │
│                                         │                                       │
│                         ┌───────────────┴───────────────┐                       │
│                         │           U11                 │                       │
│                         │        MCP4561-104            │                       │
│                         │         (100kΩ)               │                       │
│                         │                               │                       │
│  [I2C1_SDA]◄───────────►│ SDA(1)                        │                       │
│                         │                      VDD(8)◄──┼── [VSYS]              │
│  [I2C1_SCL]────────────►│ SCL(2)                        │                       │
│                         │                               │                       │
│                         │ A0(3)──GND (I2C addr = 0x2E)  │                       │
│                         │                               │                       │
│                         │ VSS(4)──GND                   │                       │
│                         │                               │                       │
│                         │ A(5)──┬──GND (tied to B)      │                       │
│                         │       │                       │                       │
│                         │ W(6)──┼──► To FB junction     │                       │
│                         │       │                       │                       │
│                         │ B(7)──┴──GND                  │                       │
│                         │                               │                       │
│                         └───────────────────────────────┘                       │
│                                         │                                       │
│                                   R_SAFE (22k)                                  │
│                                         │                                       │
│                                        GND                                      │
│                                                                                 │
│            ⚠️ R_SAFE limits V_BOOST to 6.7V if MCP4561 wiper fails open       │
│                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────────┐│
│  │  V_SENSE Divider (place near MCU, not here):                                ││
│  │                                                                             ││
│  │  [V_BOOST] ──── R_DIV1 (100k) ──┬── [V_SENSE] ──► MCU GPIO4                 ││
│  │                                 ├── R_DIV2 (47k) ──── GND                   ││
│  │                                 └── C_VSENSE (100nF) ── GND                 ││
│  └─────────────────────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 8. Subsheet: SafetyLogic

### 8.1 Sheet Information

| Property | Value |
|----------|-------|
| Sheet Name | SafetyLogic |
| File Name | SafetyLogic.kicad_sch |
| Page Size | A4 |
| Description | 555 watchdog, AND gate safety logic, Schmitt buffer |

### 8.2 Components

| Designator | Symbol | Value | Footprint | Description |
|------------|--------|-------|-----------|-------------|
| U6 | NE555 | NE555DR | Package_SO:SOIC-8_3.9x4.9mm_P1.27mm | 555 timer |
| U7 | 74LVC08A | 74LVC08APW | Package_SO:TSSOP-14_4.4x5mm_P0.65mm | Quad AND gate |
| U12 | 74LVC1G17 | 74LVC1G17GW | Package_TO_SOT_SMD:SOT-353_SC-70-5 | Schmitt trigger buffer |
| U13 | 74LVC1G04 | 74LVC1G04GW | Package_TO_SOT_SMD:SOT-353_SC-70-5 | Single inverter (FIRE_BTN) |
| R3 | R | 270k | Resistor_SMD:R_0603_1608Metric | 555 timing resistor (3s timeout) |
| R11 | R | 10k | Resistor_SMD:R_0402_1005Metric | FIRE_EN pull-down |
| R12 | R | 10k | Resistor_SMD:R_0402_1005Metric | 555 RST pull-up |
| R_ALERT | R | 10k | Resistor_SMD:R_0402_1005Metric | T_ALERT pull-up (MCP9808 open-drain) |
| C15 | C | 10µF | Capacitor_SMD:C_0805_2012Metric | 555 timing capacitor |
| C_555 | C | 100nF | Capacitor_SMD:C_0402_1005Metric | 555 VCC bypass |
| C_AND | C | 100nF | Capacitor_SMD:C_0402_1005Metric | AND gate VCC bypass |
| C_SCHMITT | C | 100nF | Capacitor_SMD:C_0402_1005Metric | U12 (74LVC1G17) VCC bypass |
| C_INV | C | 100nF | Capacitor_SMD:C_0402_1005Metric | U13 (74LVC1G04) VCC bypass |
| C_CV | C | 10nF | Capacitor_SMD:C_0402_1005Metric | 555 CV pin filter (optional) |
| C_DEBOUNCE | C | 100nF | Capacitor_SMD:C_0402_1005Metric | FIRE_BTN hardware debounce |
| R_DEBOUNCE | R | 10k | Resistor_SMD:R_0402_1005Metric | FIRE_BTN debounce resistor |

### 8.3 Hierarchical Labels

| Label Name | Direction | Purpose |
|------------|-----------|---------|
| VCC_3V3 | Input | 3.3V power supply |
| WDT_PING | Input | Watchdog trigger from MCU (GPIO4) |
| FIRE_BTN | Input | Fire button input (active LOW) |
| OC_FAULT | Input | Overcurrent fault (active LOW from CoilDriver) |
| FIRE_EN | Input | Software fire enable from MCU (GPIO5) |
| T_ALERT | Input | MCP9808 thermal alert (active LOW, open-drain) |
| SAFETY_GATE | Output | Combined safety signal to CoilDriver |
| GND | Bidirectional | Ground |

### 8.4 Wiring Instructions

#### Step 1: Place 555 Timer
1. Press `A`, search for `NE555` or `555`
2. Place U6 in the upper-left area
3. Assign: Reference = `U6`, Value = `NE555DR`

#### Step 2: Create Input Hierarchical Labels
1. Create `VCC_3V3` (Input) - top-left
2. Create `WDT_PING` (Input) - left side
3. Create `FIRE_BTN` (Input) - left side
4. Create `OC_FAULT` (Input) - left side
5. Create `FIRE_EN` (Input) - left side

#### Step 3: Create Output Hierarchical Label
1. Create `SAFETY_GATE` (Output) - right side

#### Step 4: Wire 555 Timer in Monostable Configuration
```
[VCC_3V3] ──┬── R12 (10k) ──┬── U6:VCC (pin 8)
            │               │
            │               └── U6:RST (pin 4)
            │
            └── C_555 (100nF) ──── GND

[WDT_PING] ──── U6:TRIG (pin 2)   (active LOW trigger)

U6:THR (pin 6) ──┬── U6:DIS (pin 7)
                 │
                 ├── R3 (270k) ──── [VCC_3V3]
                 │
                 └── C15 (10µF) ──── GND

U6:CV (pin 5) ──── C_CV (10nF) ──── GND   (optional noise filter)

U6:GND (pin 1) ──── GND
```

#### Step 5: Place and Wire Schmitt Buffer
1. Place U12 (74LVC1G17) next to U6
2. Wire:

```
U6:OUT (pin 3) ──── U12:A (pin 2, input)
U12:Y (pin 4, output) ──── WDT_OK (internal net) #TODO

[VCC_3V3] ──┬── C_SCHMITT (100nF) ──┬── U12:VCC (pin 5)
            │                       │
           GND ────────────────────U12:GND (pin 3)
```

#### Step 6: Place Quad AND Gate
1. Place U7 (74LVC08A) in the center-right
2. Wire power pins:

```
[VCC_3V3] ──┬── C_AND (100nF) ──┬── U7:VCC (pin 14)
            │                   │
            └───────────────────┴── GND ──── U7:GND (pin 7)
```

#### Step 7: Wire FIRE_EN Pull-Down
```
[FIRE_EN] ──┬── R11 (10k) ──── GND
            │
            └── (to AND gate input)
```

**Critical:** This ensures FIRE_EN is LOW (safe) when ESP32 GPIO is floating.

#### Step 8: Wire First AND Gate (WDT_OK AND !FIRE_BTN)

**Fire Button Logic:**
The fire button is active LOW (has external pull-up, goes LOW when pressed).
The AND gate requires HIGH = "allow fire". To convert button-pressed (LOW) to HIGH,
we add an inverter (U13: 74LVC1G04).

**⚠️ U13 Placement:** Place U13 on the **SafetyLogic sheet** near U7 (the AND gate), NOT on the UI sheet.

Add component:
| Designator | Symbol | Value | Footprint | Description |
|------------|--------|-------|-----------|-------------|
| U13 | 74LVC1G04 | 74LVC1G04GW | Package_TO_SOT_SMD:SOT-353_SC-70-5 | Single inverter |

```
                    ┌── C_INV (100nF) ──┐
                    │                   │
[VCC_3V3] ─────────┬┴── U13:VCC (pin 5) │
                   │                    │
                  GND ── U13:GND (pin 2)┘

[FIRE_BTN] ──── R_DEBOUNCE (10k) ──┬── U13:A (pin 1)
                                   │
                        C_DEBOUNCE (100nF)
                                   │
                                  GND

U13:Y (pin 4) ──── FIRE_BTN_INV (internal net) ──► U7:1B (pin 2) # TODO
```

**⚠️ FIRE_BTN Debounce:** The RC network (10kΩ + 100nF) provides ~1ms hardware debounce, preventing contact bounce from reaching the safety logic. Time constant τ = R × C = 10kΩ × 100nF = 1ms.

Alternatively, use an N-ch MOSFET as inverter:
```
[VCC_3V3] ──── R_INV (10k) ──┬── FIRE_BTN_INV ──► U7:1B
                             │
[FIRE_BTN] ──── Q_INV (NMOS) ─┘
                    │
                   GND
```

For this guide, we'll proceed with the 74LVC1G04 approach.

#### Step 9: Wire Second AND Gate (STAGE1 AND !OC_FAULT)
```
STAGE1 ──────── U7:2A (pin 4)
                        ├──► U7:2Y (pin 6) ──── STAGE2 (internal net)
[OC_FAULT] ──── U7:2B (pin 5)

(Note: OC_FAULT is already active LOW with pull-up, so when NO fault,
 OC_FAULT = HIGH. When fault occurs, OC_FAULT = LOW.
 This is correct for the AND gate: HIGH AND HIGH = proceed,
 LOW from fault = output LOW = safe.)
```

#### Step 10: Wire Third AND Gate (STAGE2 AND FIRE_EN)
```
STAGE2 ─────────── U7:3A (pin 9)
                           ├──► U7:3Y (pin 8) ──── STAGE3 (internal net)
[FIRE_EN] ──┬── R11 ──┬── U7:3B (pin 10)
            │         │
            │        GND (pull-down)
            │
            └── (from MCU GPIO5)
```

#### Step 11: Wire Fourth AND Gate (STAGE3 AND T_ALERT) - Hardware Thermal Interlock
```
STAGE3 ─────────── U7:4A (pin 12)
                           ├──► U7:4Y (pin 11) ──── [SAFETY_GATE]
[T_ALERT] ──┬── R_ALERT ──┬── U7:4B (pin 13)
            │    10k      │
            │             │
            [VCC_3V3] ────┘ (pull-up for open-drain)
```

**⚠️ Hardware Thermal Shutdown:** The MCP9808 ALERT pin is open-drain, active LOW:
- **T_ALERT HIGH** (normal): Temperature below T_CRIT → Gate passes signal
- **T_ALERT LOW** (over-temp): Temperature ≥ T_CRIT → SAFETY_GATE forced LOW immediately

This provides MCU-independent thermal protection. Even if the MCU crashes, overheating will cut power to the coil.

### 8.5 Completed SafetyLogic Schematic Diagram

```
┌────────────────────────────────────────────────────────────────────────────────────┐
│                                SafetyLogic                                          │
│                                                                                     │
│  From MCU / UI                                                  To CoilDriver       │
│       │                                                              │              │
│       ▼                                                              ▼              │
│  [VCC_3V3]──┬──────────────────────────────────────────────────────────────────    │
│             │                                                                       │
│             ├── R12 ──┐                                                             │
│             │   10k   │                                                             │
│             │         │                                                             │
│             │    ┌────┴────┐                                                        │
│             │    │   U6    │                                                        │
│             │    │  NE555  │                                                        │
│             │    │         │                                                        │
│             └───►│VCC(8)   │                                                        │
│                  │    RST(4)│◄──┘ (tied to VCC via R12)                             │
│                  │         │                                                        │
│  [WDT_PING]─────►│TRIG(2)  │                                                        │
│                  │         │                                                        │
│                  │   THR(6)├──┬── R3 ──── [VCC_3V3]                                 │
│                  │         │  │   270k                                              │
│                  │   DIS(7)├──┤                                                     │
│                  │         │  │                                                     │
│                  │         │  └── C15 ──── GND                                      │
│                  │         │      10µF                                              │
│                  │    CV(5)├── 10nF ── GND                                          │
│                  │         │                                                        │
│                  │   OUT(3)├────────────────────────────┐                           │
│                  │         │                            │                           │
│                  │  GND(1) │                            ▼                           │
│                  └────┬────┘               ┌───────────────────┐                    │
│                       │                    │       U12         │                    │
│                       ▼                    │    74LVC1G17      │                    │
│                      GND                   │                   │                    │
│                                            │ A ──► Y           │                    │
│                                            │       │           │                    │
│                                            └───────┼───────────┘                    │
│                                                    │                                │
│                                                    │ WDT_OK                         │
│                                                    ▼                                │
│                                         ┌──────────────────────────────┐            │
│                                         │          U7                  │            │
│                                         │       74LVC08A               │            │
│                                         │    (Quad 2-input AND)        │            │
│                                         │                              │            │
│                            WDT_OK ─────►│ 1A(1)                        │            │
│                                         │          1Y(3)───► STAGE1    │            │
│  [FIRE_BTN]──►U13(INV)──►FIRE_BTN_INV──►│ 1B(2)                        │            │
│                                         │                              │            │
│                            STAGE1 ─────►│ 2A(4)                        │            │
│                                         │          2Y(6)───► STAGE2    │            │
│  [OC_FAULT]────────────────────────────►│ 2B(5)                        │            │
│   (active LOW with pull-up)             │                              │            │
│                                         │                              │            │
│                            STAGE2 ─────►│ 3A(9)                        │            │
│                                         │          3Y(8)───► STAGE3    │            │
│  [FIRE_EN]──┬── R11 ───────────────────►│ 3B(10)                       │            │
│             │   10k                     │                              │            │
│             │    │                      │                              │            │
│             │   GND (pull-down)         │                              │            │
│             │                           │                              │            │
│                            STAGE3 ─────►│ 4A(12)                       │            │
│                                         │          4Y(11)──►[SAFETY_GATE]─────────►│
│  [T_ALERT]──┬── R_ALERT ───────────────►│ 4B(13)                       │            │
│             │   10k                     │                              │            │
│             │    │                      │                              │            │
│          [VCC_3V3] (pull-up)            │                              │            │
│             │                           │                              │            │
│             │                           │ VCC(14)◄── [VCC_3V3]         │            │
│             │                           │ GND(7) ──► GND               │            │
│             │                           └──────────────────────────────┘            │
│             │                                                                       │
│             └── (to GPIO5 on MCU)                                                   │
│                                                                                     │
│                                                                                     │
│  ┌───────────────────┐                                                              │
│  │       U13         │                                                              │
│  │    74LVC1G04      │                                                              │
│  │    (Inverter)     │                                                              │
│  │                   │                                                              │
│  │ A ◄── [FIRE_BTN]  │                                                              │
│  │                   │                                                              │
│  │ Y ──► FIRE_BTN_INV│                                                              │
│  │                   │                                                              │
│  │ VCC◄── [VCC_3V3]  │                                                              │
│  │ GND──► GND        │                                                              │
│  └───────────────────┘                                                              │
│                                                                                     │
└────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 9. Subsheet: CoilDriver

### 9.1 Sheet Information

| Property | Value |
|----------|-------|
| Sheet Name | CoilDriver |
| File Name | CoilDriver.kicad_sch |
| Page Size | A4 |
| Description | PMOS/NMOS coil driver, gate driver, current sense, OC comparator |

### 9.2 Components

| Designator | Symbol | Value | Footprint | Description |
|------------|--------|-------|-----------|-------------|
| Q3 | Si7461DP | Si7461DP-T1-GE3 | Package_SO:PowerPAK_SO-8_Single | P-ch high-side MOSFET |
| Q4 | CSD18563Q5A | CSD18563Q5A | Package_SON:Texas_DQK_SON-8_3.3x3.3mm | N-ch low-side MOSFET |
| U8 | INA180A2 | INA180A2IDBVR | Package_TO_SOT_SMD:SOT-23-5 | Current sense amp (50V/V) |
| U9 | LM393 | LM393DR | Package_SO:SOIC-8_3.9x4.9mm_P1.27mm | Comparator |
| U10 | TC4426 | TC4426ACOA | Package_SO:SOIC-8_3.9x4.9mm_P1.27mm | Inverting gate driver |
| R_G3_HS | R | 15Ω | Resistor_SMD:R_0402_1005Metric | Q3 gate resistor (prevents ringing) |
| R_GS_Q3 | R | 100kΩ | Resistor_SMD:R_0402_1005Metric | Q3 gate-source pull-up (power-up safety) |
| D_FLY | D_Schottky | SS34 | Diode_SMD:D_SMA | Flyback diode (inductive kickback) |
| R_SHUNT | R | 10mΩ | Resistor_SMD:R_2512_6332Metric_Pad1.40x3.35mm_HandSolder | Current shunt (3W) |
| R6 | R | 100Ω | Resistor_SMD:R_0402_1005Metric | Comparator filter R |
| R7 | R | 1k | Resistor_SMD:R_0402_1005Metric | Threshold divider top |
| R8 | R | 3.3k | Resistor_SMD:R_0402_1005Metric | Threshold divider bottom |
| R9 | R | 1M | Resistor_SMD:R_0603_1608Metric | Hysteresis resistor |
| R10 | R | 10k | Resistor_SMD:R_0402_1005Metric | LM393 pull-up |
| R21 | R | 1M | Resistor_SMD:R_0603_1608Metric | 510 connector ESD bleed (optional) |
| C18 | C | 10nF | Capacitor_SMD:C_0402_1005Metric | Comparator filter C |
| C_INA | C | 100nF | Capacitor_SMD:C_0402_1005Metric | INA180 bypass |
| C_TC | C | 100nF | Capacitor_SMD:C_0402_1005Metric | TC4426 bypass |
| J6 | Conn_01x02 | COIL+/COIL- | Custom:Solder_Pads_2x_14AWG | 510 connector pads (wire to bullet connectors) |
| NT1 | Net_Tie_2 | Net_Tie | NetTie:NetTie-2_SMD_Pad0.5mm | Star ground tie point at shunt |
| U15 | MCP9808 | MCP9808T-E/MS | Package_SO:MSOP-8_3x3mm_P0.65mm | I2C temperature sensor (near Q3/Q4) |
| C_MCP | C | 100nF | Capacitor_SMD:C_0402_1005Metric | MCP9808 bypass capacitor |

**⚠️ CRITICAL FOOTPRINT: R_SHUNT (Current Sense Resistor)**
At 6A continuous, R_SHUNT dissipates 0.36W (P = I²R). With safety margin, use a **3W-rated 2512 package**. A smaller package (1206/0805) WILL overheat, potentially desoldering itself and creating an open circuit (loss of current sensing = no overcurrent protection).

### 9.3 Hierarchical Labels

| Label Name | Direction | Purpose |
|------------|-----------|---------|
| V_BOOST | Input | Boosted voltage from BoostConverter |
| SAFETY_GATE | Input | Safety gate signal from SafetyLogic |
| COIL_PWM | Input | PWM signal from MCU (GPIO5) |
| VCC_3V3 | Input | 3.3V for logic ICs |
| I_SENSE | Output | Current sense output to MCU ADC |
| OC_FAULT | Output | Overcurrent fault (active LOW) |
| I2C1_SDA | Bidirectional | I2C data (for MCP9808) |
| I2C1_SCL | Input | I2C clock |
| T_ALERT | Output | MCP9808 over-temperature alert (active LOW, to SafetyLogic AND gate) |
| COIL+ | Output | Positive coil connection |
| COIL- | Output | Negative coil connection (ground via Q4) |
| GND | Bidirectional | System/Power ground (Q4:Source, battery return) |

**⚠️ CRITICAL: High-Side Shunt Topology**

This design uses a **HIGH-SIDE current shunt** (between Q3:Drain and the coil).
The shunt **FLOATS** at V_BOOST potential — it does NOT connect to ground.

```
V_BOOST → Q3:Source → Q3:Drain → [R_SHUNT] → COIL_HIGH → 510 Coil → Q4:Drain → Q4:Source → GND
                                     ↑
                          Shunt floats at high voltage
                          INA180 measures differential voltage across it
```

**⚠️ DO NOT connect the shunt to GND — this would short V_BOOST to ground!**

**Star Ground Implementation:**

For accurate current sensing, use a local `AGND` (Analog Ground) label for the INA180 and comparator.
Connect `AGND` to system `GND` via **NT1 (Net Tie)** placed near **Q4:Source**.

| Net Name | Purpose | Connects To |
|----------|---------|-------------|
| `GND` | System/Power ground | Q4:Source, J6:COIL- (via Q4), battery return, high current path |
| `AGND` | Analog ground (local) | INA180:GND, LM393:GND, connects to GND via NT1 |

**NT1 (Net Tie)** joins `AGND` to `GND` at exactly ONE point — near Q4:Source.
This ensures the high-current return path is robust (6A through GND), while NT1 only carries milliamps of reference current.

**Differential Pair Naming:**
Use `CS_P` and `CS_N` for current sense traces (INA180 inputs) to enable KiCad differential pair routing.

### 9.4 Wiring Instructions

#### Step 1: Place MOSFETs
1. Place Q3 (Si7461DP) - P-channel high-side
2. Place Q4 (CSD18563Q5A) - N-channel low-side
3. Position Q3 above Q4 in the power path

#### Step 2: Create Hierarchical Labels
1. Create `V_BOOST` (Input) - top-left
2. Create `SAFETY_GATE` (Input) - left side
3. Create `COIL_PWM` (Input) - left side
4. Create `VCC_3V3` (Input) - left side
5. Create `I_SENSE` (Output) - right side
6. Create `OC_FAULT` (Output) - right side

#### Step 3: Wire High-Side Power Path
```
[V_BOOST] ──── Q3:Source ──── Q3:Drain ──── R_SHUNT ──┬── COIL_HIGH (internal)
                                   │                  │
                                   │              (to INA180)
                                   │
                              (P-ch: Source at high voltage,
                               Drain goes to load)
```

**Note on PMOS:** For Si7461DP:
- Source connects to V_BOOST (high side)
- Drain connects to load (through shunt)
- Gate: LOW = ON, HIGH = OFF

#### Step 4: Place and Wire TC4426 Gate Driver
```
[V_BOOST] ──┬── C_TC (100nF) ──┬── U10:VDD (pin 6)
            │                  │
            └──────────────────┴── GND ── U10:GND (pin 3)

[SAFETY_GATE] ──── U10:IN_A (pin 2)
                          │
U10:OUT_A (pin 7) ──── R_G3_HS (15Ω) ──┬── Q3:Gate
                                       │
                                    R_GS_Q3 (100kΩ)
                                       │
                                    Q3:Source ──── [V_BOOST]
```

**R_G3_HS Purpose:** The 15Ω gate resistor limits gate current and slows dV/dt transitions to prevent parasitic oscillation and gate ringing. Lower value than R_G3 (47Ω) because TC4426 has strong drive capability.

**⚠️ R_GS_Q3 Purpose (Power-Up Safety):**

During power-up, V_BOOST (from battery) is present before VCC_3V3 finishes ramping. The TC4426 output may float or be undefined during this window.

**Without R_GS_Q3:** Q3:Gate could float low → PMOS turns ON → uncontrolled current flow before safety logic initializes.

**With R_GS_Q3 (100kΩ):** Gate is pulled to Source (V_BOOST) → Vgs = 0 → PMOS guaranteed OFF until TC4426 actively drives the gate low.

This is a **fail-safe default**: the coil cannot fire during power-up, even if the driver IC is unpowered.

**Critical:** TC4426 is **INVERTING**:
- SAFETY_GATE HIGH → OUT_A LOW → Q3 Vgs negative → PMOS ON
- SAFETY_GATE LOW → OUT_A HIGH (V_BOOST) → Q3 Vgs = 0 → PMOS OFF

#### Step 5: Wire Current Shunt and Sense Amplifier

**Current Path (from design spec):**
```
V_BOOST → Q3:Source → Q3:Drain → R_SHUNT → COIL_HIGH → 510 Coil → Q4:Drain → Q4:Source → GND
```

**Shunt and INA180 Wiring (High-Side Differential Sensing):**
```
Q3:Drain ──┬── CS_P ──────────► U8:IN+ (pin 1)  ← High side of shunt
           │
        R_SHUNT (10mΩ)    ⚠️ SHUNT FLOATS - NO GND CONNECTION!
           │
           ├── CS_N ──────────► U8:IN- (pin 2)  ← Low side of shunt
           │
           └── COIL_HIGH (internal net to 510 connector)
```

**⚠️ Differential Pair:** Name the shunt sense traces `CS_P` and `CS_N` for KiCad auto-routing.

**⚠️ CRITICAL:** The shunt is on the HIGH SIDE and floats at V_BOOST potential.
**DO NOT** connect CS_N or COIL_HIGH to any ground — this would short V_BOOST to GND!

The INA180 senses voltage drop across R_SHUNT:
- IN+ = CS_P (higher voltage, before shunt)
- IN- = CS_N (lower voltage, after shunt)
- V_sense = I_coil × R_SHUNT × Gain = I × 10mΩ × 50 = 0.5V/A

#### Step 6: Wire INA180 Power and Output
1. Place U8 (INA180A2)
2. Wire the differential inputs (floating at high voltage):

```
Q3:Drain ────────────────► CS_P ──► U8:IN+ (pin 1)
                    │
                 R_SHUNT (10mΩ)
                    │
COIL_HIGH ◄─────────┴────► CS_N ──► U8:IN- (pin 2)

                    ⚠️ NO GROUND CONNECTION HERE - SHUNT FLOATS!

[VCC_3V3] ──┬── C_INA (100nF) ──┬── U8:VCC (pin 5)
            │                   │
            └───────────────────┴── AGND ── U8:GND (pin 3)

U8:OUT (pin 4) ──┬── [I_SENSE] (to MCU ADC)
                 │
                 └── R6 (100Ω) ──── C18 (10nF) ──┬── U9:IN+ (comparator)
                                                │
                                                AGND
```

**Note:** INA180:GND connects to `AGND` (Analog Ground), which joins system `GND` via NT1 near Q4:Source.

#### Step 7: Wire LM393 Comparator for Overcurrent Detection
1. Place U9 (LM393)
2. Wire threshold voltage divider:

```
[VCC_3V3] ──── R7 (1k) ──┬── U9:IN- (pin 2)
                         │
                         R8 (3.3k)
                         │
                        AGND

V_THRESH = 3.3V × 3.3k / (1k + 3.3k) = 2.53V → trips at ~5A
```

3. Wire comparator input and output:

```
(from RC filter after INA180) ──── U9:IN+ (pin 3)

U9:OUT (pin 1) ──┬── R10 (10k) ──── [VCC_3V3]  (PULL-UP - required!)
                 │
                 ├── R9 (1M) ──────► U9:IN+ (pin 3)  (Hysteresis)
                 │
                 └── [OC_FAULT]  (active LOW when overcurrent)
```

4. Wire comparator power:

```
[VCC_3V3] ──── U9:VCC (pin 8)
               U9:GND (pin 4) ──── AGND   (Analog ground via NT1)
```

**Note:** Only one comparator in LM393 is used. Tie unused inputs:
```
U9:IN+_B (pin 5) ──── AGND
U9:IN-_B (pin 6) ──── [VCC_3V3]
U9:OUT_B (pin 7) ──── NC
```

**⚠️ LM393 Response Time Limitation:**

The LM393 has ~1.3µs response time. During a hard short circuit (e.g., liquid bridging coil terminals), current can spike to 20-30A in <1µs before the comparator trips.

**Mitigations in this design:**
1. **INA180 output clamping:** INA180 output is limited to VCC (3.3V) internally
2. **RC filter (R6/C18):** 100Ω × 10nF = 1µs time constant adds minor delay but filters noise
3. **Hardware AND gate:** Even if LM393 is slow, the 5-input safety chain requires ALL conditions to fire
4. **Boost converter current limit:** TPS61088 has 10A cycle-by-cycle limit at the source

**For future revisions:** Consider TLV3201 (40ns response) for faster protection, though LM393 is adequate for this design with the multi-layer safety approach.

#### Step 8: Wire Low-Side MOSFET (Q4)

**⚠️ Add Gate Resistor to Protect ESP32 GPIO**

The CSD18563Q5A has high gate capacitance (~2000pF). Direct drive from ESP32 3.3V GPIO
causes large current spikes during PWM transitions that can damage the MCU over time.

Add component:
| Designator | Symbol | Value | Footprint | Description |
|------------|--------|-------|-----------|-------------|
| R_G3 | R | 47Ω | Resistor_SMD:R_0402_1005Metric | Q4 gate resistor (limits inrush, slows edges) |

```
[COIL_PWM] ──── R_G3 (47Ω) ──── Q4:Gate

J6:COIL- (pin 2) ──── Q4:Drain

Q4:Source ──── GND   (System ground - 6A return path to battery)
```

**Note:** 47Ω limits gate current to ~70mA peak (safe for ESP32) while maintaining fast enough switching for 20kHz+ PWM.

**⚠️ OPTIONAL IMPROVEMENT: Use TC4426 Channel B for Q4 Gate Drive**

The CSD18563Q5A has Qg ≈ 28nC and is optimized for 4.5V+ gate drive. Direct 3.3V GPIO drive works but causes slower switching transitions and higher losses.

Since TC4426 (U10) has an unused Channel B, you can use it for Q4:

```
[COIL_PWM] ──── U10:IN_B (pin 4)
U10:OUT_B (pin 5) ──── R_G4_LS (15Ω) ──── Q4:Gate
```

**Important:** TC4426 is **inverting**. Two options:
1. **Firmware inversion:** Set PWM duty=0 for full power, duty=MAX for off
2. **Use TC4427 instead:** Non-inverting variant (requires different IC)

**Benefit:** 1.5A gate drive current → faster switching → lower losses → cooler Q4.
**Trade-off:** Firmware complexity or component change.

**If implementing this option:**
- Add R_G4_LS (15Ω 0402) to BOM
- Remove R_G3 connection to Q4 (R_G3 only goes to Q3)
- Update firmware: `ledc_set_duty()` logic is inverted (0% duty = full power)

If using direct GPIO drive (simpler), ensure good thermal copper under Q4 and keep PWM at 20kHz (lowest non-audible).

#### Step 8b: Place Net Tie for Star Ground

**Finding/Creating Net Tie in KiCad 8.0:**
1. **Symbol:** Place → Add Symbol → Search "Net_Tie" → Select `Device:Net_Tie_2`
2. **Footprint:** Assign footprint `NetTie:NetTie-2_SMD_Pad0.5mm` (or similar small pad)
3. **Alternative (custom):** In Symbol Editor, create 2-pin symbol with both pins as "Passive" type

**Wiring Steps:**
1. Place NT1 (Net_Tie_2) **physically near Q4:Source** on the schematic
2. Wire:

```
Q4:Source ──── GND ──┬── NT1:1 (Power ground side)
                     │
            AGND ────┴── NT1:2 (Analog ground side)
```

**⚠️ CRITICAL: Net Labeling Rule**
- The wire entering **NT1:Pin 1** MUST be explicitly labeled `GND`
- The wire entering **NT1:Pin 2** MUST be explicitly labeled `AGND`
- **Do NOT** just draw a wire across the pins without the component — KiCad DRC needs the Net Tie symbol to legally join two different net names

**⚠️ Star Ground Purpose:**
- `GND` carries the full 6A coil current back to battery (robust power path)
- `AGND` is the quiet reference for INA180/LM393 (milliamps only)
- NT1 joins them at ONE point near Q4:Source, preventing switching noise from corrupting ADC readings

**📐 PCB LAYOUT REQUIREMENT:**
In the PCB editor, the `Net_Tie_2` footprint must be placed **physically touching the Q4:Source copper pour**. If placed elsewhere, the star ground benefit is lost and switching noise will corrupt ADC readings.

#### Step 9: Wire 510 Connector Pads
```
COIL_HIGH ──── J6:COIL+ (pin 1)

Q4:Drain ──── J6:COIL- (pin 2)
```

Optional ESD bleed resistor:
```
J6:COIL+ ──── R21 (1M) ──── GND
```

**Flyback Diode (D_FLY) - Inductive Kickback Protection:**
```
J6:COIL+ (COIL_HIGH) ────┬──── To Atomizer (+)
                         │
                      D_FLY (SS34)  ← Cathode (stripe) toward COIL+
                         │
J6:COIL- (Q4:Drain) ─────┴──── To Atomizer (-)
```

**Purpose:** When Q3 or Q4 turns OFF rapidly (especially during thermal shutdown), the coil's inductance creates a voltage spike (V = L × dI/dt). D_FLY clamps this spike by providing a freewheeling path. The SS34 (3A, 40V Schottky) has low forward voltage (~0.4V) for fast response.

**Note:** Typical atomizer coils have low inductance (~10-50nH), but the spike can still reach 10-20V during rapid shutdown. The Si7461DP body diode provides some protection, but SS34 responds faster and reduces thermal stress on Q3.

**⚠️ 510 Connector Shell Voltage Note**

The 510 connector shell is electrically connected to **COIL-** (Q4 Drain), **NOT Ground**.

In this low-side switching topology:
- When Q4 is OFF, COIL- floats up to V_BOOST voltage
- The 510 shell is therefore at V_BOOST potential when idle

**Plastic Chassis (This Design):**
✅ **Reduced isolation concern** — plastic is non-conductive, so the 510 shell cannot short to chassis.
The 510 connector can mount directly to the plastic enclosure without special insulation.

**🔴 CRITICAL: LIQUID INGRESS AUTO-FIRE HAZARD**

**A leak that bridges the 510 positive pin (COIL_HIGH) to the USB port or chassis ground will cause an auto-fire event that the MCU CANNOT stop.**

E-liquid is conductive (0.5Ω - 50kΩ depending on saturation). If juice bridges:
- 510 COIL+ pin → USB shell (GND) → Current bypasses Q4 entirely
- Path: Battery → Boost → Coil → Juice → GND → **UNCONTROLLABLE FIRING**

**MANDATORY physical protections:**
- **Conformal coating** on PCB areas near 510 and USB (highly recommended)
- **Silicone gaskets** around 510 and USB connector openings
- **Physical separation** of 510 and USB on opposite sides of enclosure
- **Drain holes** in enclosure bottom to prevent juice pooling

**⚠️ METAL CHASSIS WARNING:**

If the user installs the atomizer into a **metal enclosure** without proper isolation:
1. Atomizer screws into 510 connector (510 shell contacts chassis)
2. Chassis connects to USB GND (common in metal enclosures)
3. **Result:** Direct bypass around Q4 — coil fires continuously when boost is enabled

This creates an **auto-fire condition that cannot be stopped by firmware**.

| Chassis Type | Risk | Mitigation |
|--------------|------|------------|
| Plastic/3D-printed | Low | None needed |
| Anodized aluminum | Medium | Anodizing provides insulation, but scratches expose metal |
| Bare metal | **HIGH** | MANDATORY isolation required |

**Metal Chassis Mitigation:**
- Use PEEK/plastic insulating washer between 510 and metal chassis
- Use non-conductive 510 connector adapter
- Ensure chassis is NOT connected to USB GND (floating chassis)

**ALWAYS:**
- Ensure 510 connector shell connects **ONLY** to COIL- pad
- **DO NOT** connect 510 shell to PCB GND or chassis ground

**Optional: 510 ESD Bleed Resistor (R21)**

Add a 1MΩ resistor from COIL- to GND to prevent the 510 shell from floating to undefined potentials via leakage current:

```
J6:COIL- (pin 2) ──── R21 (1MΩ) ──── GND
```

| Designator | Symbol | Value | Footprint | Description |
|------------|--------|-------|-----------|-------------|
| R21 | R | 1MΩ | Resistor_SMD:R_0402_1005Metric | 510 shell ESD bleed |

**Note:** This does NOT change the live shell hazard during firing (shell still reaches V_BOOST when Q4 ON). It only bleeds static charge when idle.

#### Step 10: Wire PCB Temperature Sensor (MCP9808)

Place U15 (MCP9808) **physically close to Q3 and Q4** on the schematic (and PCB).

```
[VCC_3V3] ──┬── C_MCP (100nF) ──┬── U15:VDD (pin 1)
            │                   │
            │                  GND ── U15:GND (pin 5, 8)
            │
[I2C1_SDA] ─────────────────────────── U15:SDA (pin 6)
[I2C1_SCL] ─────────────────────────── U15:SCL (pin 7)

U15:ALERT (pin 3) ──── [T_ALERT] (to SafetyLogic AND gate - HARDWARE THERMAL INTERLOCK)

U15:A0 (pin 4) ──── GND  }
U15:A1 (pin 2) ──── GND  } I2C address = 0x18
```

**I2C Address:** 0x18 (with A0=A1=A2=GND)

**MCP9808 Features:**
- ±0.25°C accuracy (typ), ±0.5°C max (0°C to +85°C)
- 12-bit resolution (0.0625°C)
- Alert output for hardware over-temperature interrupt
- No ADC conflicts — pure digital I2C

**⚠️ PCB LAYOUT REQUIREMENT:**
Place U15 footprint within 5mm of Q3/Q4 thermal pad area. The sensor monitors MOSFET junction temperature via PCB thermal coupling.

**Alert Configuration (optional):**
The ALERT pin can trigger a hardware interrupt when temperature exceeds a programmable threshold (e.g., 80°C). Connect to a free GPIO for immediate thermal shutdown without polling.

### 9.5 Completed CoilDriver Schematic Diagram

```
┌───────────────────────────────────────────────────────────────────────────────────────┐
│                                    CoilDriver                                          │
│                                                                                        │
│  From BoostConverter                                                 To 510 Atomizer  │
│         │                                                                   │          │
│         ▼                                                                   ▼          │
│  [V_BOOST]──┬─────────────────────────────────────────────────────────────────────    │
│             │                                                                          │
│             │    ┌────────────────────┐                                               │
│             │    │        U10         │                                               │
│             └───►│      TC4426        │                                               │
│                  │   (Inverting)      │                                               │
│                  │                    │                                               │
│ [SAFETY_GATE]──►│ IN_A(2)     VDD(6)◄┼── [V_BOOST]                                   │
│                  │                    │      │                                        │
│                  │ OUT_A(7)───────────┼──┐   └── C_TC ── GND                         │
│                  │                    │  │       100nF                               │
│                  │         GND(3)     │  │                                           │
│                  └──────────┬─────────┘  │                                           │
│                             │            │                                           │
│                             ▼            │                                           │
│                            GND           │                                           │
│                                          │                                           │
│  [V_BOOST]───────────────────────────────┼────────────────────────────┐              │
│                                          │                            │              │
│                                          ▼                            │              │
│                              ┌───────────────────────┐                │              │
│                              │         Q3            │                │              │
│                              │      Si7461DP         │                │              │
│                              │       (P-ch)          │                │              │
│                              │                       │                │              │
│                              │ Gate ◄────────────────┘                │              │
│                              │                       │                │              │
│                              │ Source ◄──────────────┘ (V_BOOST)      │              │
│                              │                       │                               │
│                              │ Drain ────────────────┤                               │
│                              └───────────────────────┘                               │
│                                          │                                           │
│                                          │                                           │
│                                          ├────────► U8:IN+ (INA180)                  │
│                                          │                                           │
│                                     R_SHUNT                                          │
│                                      10mΩ                                            │
│                                       3W                                             │
│                                          │                                           │
│                                          ├────────► U8:IN- (INA180)                  │
│                                          │                                           │
│                                          │ COIL_HIGH                                 │
│                                          │                                           │
│  ┌───────────────────────────────────────┼───────────────────────────────────────┐  │
│  │                                       │                                        │  │
│  │  [VCC_3V3]──┬── C_INA ──┬─────────────┼───────────────────────────────────┐   │  │
│  │             │  100nF    │             │                                   │   │  │
│  │             │           │             │                                   │   │  │
│  │             │    ┌──────┴──────┐      │                                   │   │  │
│  │             │    │     U8      │      │                                   │   │  │
│  │             │    │  INA180A2   │      │                                   │   │  │
│  │             │    │   (50V/V)   │      │                                   │   │  │
│  │             │    │             │      │                                   │   │  │
│  │             │    │ IN+(1) ◄────┼──────┘ (before shunt)                    │   │  │
│  │             │    │             │                                          │   │  │
│  │             │    │ IN-(2) ◄────┼──────── (after shunt = COIL_HIGH)        │   │  │
│  │             │    │             │                                          │   │  │
│  │             │    │ VCC(5)◄─────┘                                          │   │  │
│  │             │    │             │                                          │   │  │
│  │             │    │ GND(3)──────┼──► GND                                   │   │  │
│  │             │    │             │                                          │   │  │
│  │             │    │ OUT(4)──────┼──┬──► [I_SENSE] (to MCU ADC)            │   │  │
│  │             │    │             │  │                                       │   │  │
│  │             │    └─────────────┘  │                                       │   │  │
│  │             │                     │                                       │   │  │
│  │             │                     └── R6 ── C18 ──┬──► U9:IN+            │   │  │
│  │             │                        100Ω  10nF   │                       │   │  │
│  │             │                                    GND                      │   │  │
│  │             │                                                             │   │  │
│  │             │         ┌───────────────────────────────────────────────┐   │   │  │
│  │             │         │                    U9                         │   │   │  │
│  │             │         │                  LM393                        │   │   │  │
│  │             │         │               (Comparator)                    │   │   │  │
│  │             │         │                                               │   │   │  │
│  │             ├─────────┼── R7 ──┬──► IN-(2)                           │   │   │  │
│  │             │         │   1k   │                                      │   │   │  │
│  │             │         │        R8                                     │   │   │  │
│  │             │         │       3.3k                                    │   │   │  │
│  │             │         │        │                                      │   │   │  │
│  │             │         │       GND  (V_THRESH = 2.53V)                 │   │   │  │
│  │             │         │                                               │   │   │  │
│  │             │         │ (from RC filter) ──► IN+(3)                   │   │   │  │
│  │             │         │                         │                     │   │   │  │
│  │             │         │                    R9 (1M)                    │   │   │  │
│  │             │         │                    (Hysteresis)               │   │   │  │
│  │             │         │                         │                     │   │   │  │
│  │             ├─── R10 ─┼─────────────────────────┼──► OUT(1) ──► [OC_FAULT]    │  │
│  │             │   10k   │  (PULL-UP required!)    │                     │   │   │  │
│  │             │         │                         │                     │   │   │  │
│  │             │         │                   VCC(8)◄─────────────────────┘   │   │  │
│  │             │         │                   GND(4)──► GND                   │   │  │
│  │             │         │                                                   │   │  │
│  │             │         │  Unused comparator:                               │   │  │
│  │             │         │    IN+_B(5)──GND                                  │   │  │
│  │             │         │    IN-_B(6)──VCC                                  │   │  │
│  │             │         │    OUT_B(7)──NC                                   │   │  │
│  │             │         └───────────────────────────────────────────────────┘   │  │
│  │             │                                                                 │  │
│  └─────────────┼─────────────────────────────────────────────────────────────────┘  │
│                │                                                                     │
│                │                                                                     │
│  COIL_HIGH ────┼────────────────────────────┬────────────────────────────────────   │
│                │                            │                                        │
│                │                            │                                        │
│                │                            ├── R21 (1M, optional) ── GND           │
│                │                            │   (ESD bleed)                          │
│                │                            │                                        │
│                │                            │    ┌────────────────┐                  │
│                │                            └───►│ J6: 510 Pads   │                  │
│                │                                 │                │                  │
│                │                                 │ COIL+ (pin 1)  │◄── To 510        │
│                │                                 │                │    Atomizer      │
│                │                                 │ COIL- (pin 2)  │◄──┐              │
│                │                                 │                │   │              │
│                │                                 └────────────────┘   │              │
│                │                                                      │              │
│                │                                                      │              │
│                │                   ┌─────────────────────────────────┘              │
│                │                   │                                                 │
│                │                   │                                                 │
│                │    ┌──────────────┴──────────────┐                                 │
│                │    │            Q4               │                                 │
│                │    │       CSD18563Q5A           │                                 │
│                │    │          (N-ch)             │                                 │
│                │    │                             │                                 │
│  [COIL_PWM]───►│───►│ Gate                        │                                 │
│   (from MCU)   │    │                             │                                 │
│                │    │ Drain ◄─────────────────────┘ (from J6 COIL-)                 │
│                │    │                             │                                 │
│                │    │ Source ─────────────────────┼──► GND                          │
│                │    │                             │                                 │
│                │    └─────────────────────────────┘                                 │
│                │                                                                     │
│                │                                                                     │
└────────────────┴─────────────────────────────────────────────────────────────────────┘
```

---

## 10. Subsheet: MCU

### 10.1 Sheet Information

| Property | Value |
|----------|-------|
| Sheet Name | MCU |
| File Name | MCU.kicad_sch |
| Page Size | A3 |
| Description | ESP32-S3-WROOM-1-N8R2 module and connections |

### 10.2 Components

| Designator | Symbol | Value | Footprint | Description |
|------------|--------|-------|-----------|-------------|
| U5 | ESP32-S3-WROOM-1 | ESP32-S3-WROOM-1-N8R2 | RF_Module:ESP32-S3-WROOM-1 | MCU module |
| R13 | R | 4.7k | Resistor_SMD:R_0402_1005Metric | I2C0 SDA pull-up |
| R14 | R | 4.7k | Resistor_SMD:R_0402_1005Metric | I2C0 SCL pull-up |
| R15 | R | 4.7k | Resistor_SMD:R_0402_1005Metric | I2C1 SDA pull-up |
| R16 | R | 4.7k | Resistor_SMD:R_0402_1005Metric | I2C1 SCL pull-up |
| C_3V3_1 | C | 100nF | Capacitor_SMD:C_0402_1005Metric | 3V3 bypass 1 |
| C_3V3_2 | C | 10µF | Capacitor_SMD:C_0805_2012Metric | 3V3 bulk |
| SW_BOOT | SW_Push | BOOT | Button_Switch_SMD:SW_SPST_TL3342 | Boot button |
| SW_RST | SW_Push | RESET | Button_Switch_SMD:SW_SPST_TL3342 | Reset button |
| C_BOOT | C | 10nF | Capacitor_SMD:C_0402_1005Metric | Boot button debounce (small value to avoid strapping issues) |
| C_RST | C | 100nF | Capacitor_SMD:C_0402_1005Metric | Reset button debounce |
| U14 | AP2112K-3.3 | AP2112K-3.3TRG1 | Package_TO_SOT_SMD:SOT-23-5 | 3.3V LDO regulator |
| C_LDO_IN | C | 10µF | Capacitor_SMD:C_0805_2012Metric | LDO input capacitor |
| C_LDO_OUT | C | 10µF | Capacitor_SMD:C_0805_2012Metric | LDO output capacitor |

### 10.3 Hierarchical Labels

| Label Name | Direction | Purpose |
|------------|-----------|---------|
| VCC_3V3 | Output | 3.3V power (generated by LDO from VSYS) |
| VSYS | Input | System voltage from BQ25896 (for LDO input) |
| GND | Bidirectional | Ground |
| **I2C Bus 0 (Touch)** |
| I2C0_SDA | Bidirectional | GPIO8 - Touch SDA |
| I2C0_SCL | Output | GPIO9 - Touch SCL |
| TP_RST | Output | GPIO1 - Touch reset |
| TP_IRQ | Input | GPIO2 - Touch interrupt |
| **I2C Bus 1 (Power Management)** |
| I2C1_SDA | Bidirectional | GPIO17 - Power I2C SDA |
| I2C1_SCL | Output | GPIO18 - Power I2C SCL |
| **Charger Control** |
| CHG_INT | Input | GPIO38 - Charger interrupt |
| CHG_CE | Output | GPIO39 - Charge enable |
| CHG_OTG | Output | GPIO40 - OTG mode |
| CHG_PG | Input | GPIO15 - Power good (optional) |
| CHG_STAT | Input | GPIO45 - Charge status (optional) |
| **Fuel Gauge** |
| FG_ALRT | Input | GPIO41 - Fuel gauge alert |
| **Boost Control** |
| BOOST_EN | Output | GPIO42 - Boost enable |
| V_SENSE | Input | GPIO4 - Boost voltage ADC **(ADC1_CH3)** |
| **Safety System** |
| WDT_PING | Output | GPIO48 - Watchdog trigger |
| FIRE_EN | Output | GPIO16 - Software fire enable |
| FIRE_BTN | Input | GPIO6 - Fire button input |
| OC_FAULT | Input | GPIO7 - Overcurrent fault |
| **Coil Control** |
| COIL_PWM | Output | GPIO5 - PWM to Q4 |
| I_SENSE | Input | GPIO3 - Current sense ADC **(ADC1_CH2)** |
| **Thermal Monitoring** |
| ~~T_ALERT~~ | — | **NOT on MCU sheet** — routes directly from CoilDriver (§9) to SafetyLogic (§8). See §8.4 Step 11. |

**⚠️ T_ALERT Routing:** The T_ALERT signal from MCP9808 goes directly to SafetyLogic AND gate input, bypassing the MCU entirely. This ensures hardware thermal protection works even if the MCU crashes. Do NOT create a T_ALERT hierarchical label on the MCU sheet.

**⚠️ ADC1 vs ADC2 CRITICAL:**
- `I_SENSE` and `V_SENSE` MUST use **ADC1** pins (GPIO 1-10)
- ADC2 (GPIO 11-20) conflicts with Wi-Fi/Bluetooth and will produce corrupt readings when radio is active
- PCB temperature uses MCP9808 on I2C1 — no ADC conflict
| **Display (SPI)** |
| LCD_CLK | Output | GPIO10 - SPI CLK |
| LCD_MOSI | Output | GPIO11 - SPI MOSI |
| LCD_CS | Output | GPIO12 - Chip select |
| LCD_DC | Output | GPIO13 - Data/Command |
| LCD_RST | Output | GPIO14 - Reset |
| LCD_BL | Output | GPIO21 - Backlight PWM |
| **USB** |
| USB_D+ | Bidirectional | GPIO19 - USB D+ |
| USB_D- | Bidirectional | GPIO20 - USB D- |
| **Misc** |
| LED_STATUS | Output | GPIO47 - Fire button LED |

### 10.4 Wiring Instructions

#### Step 1: Place ESP32-S3-WROOM-1 Module
1. Press `A`, search for `ESP32-S3-WROOM-1`
2. Place U5 in the center of the sheet
3. Assign: Reference = `U5`, Value = `ESP32-S3-WROOM-1-N8R2`

**⚠️ Pin Number Verification:** The GPIO-to-module-pin mappings below are from the ESP32-S3-WROOM-1 datasheet v1.6. Verify against current datasheet before wiring.

**ESP32-S3-WROOM-1 GPIO-to-Pin Reference (Used in this design):**

| GPIO | Module Pin | Function | Notes |
|------|-----------|----------|-------|
| GPIO0 | 27 | BOOT | Strapping pin (pull-up) |
| GPIO1 | 39 | TP_RST | Touch reset |
| GPIO2 | 38 | TP_IRQ | Touch interrupt |
| GPIO3 | 15 | I_SENSE | ADC1_CH2, strapping (JTAG) |
| GPIO4 | 4 | V_SENSE | ADC1_CH3 |
| GPIO5 | 5 | COIL_PWM | PWM output |
| GPIO6 | 6 | FIRE_BTN | Button input |
| GPIO7 | 7 | OC_FAULT | Overcurrent input |
| GPIO8 | 12 | I2C0_SDA | Touch I2C |
| GPIO9 | 17 | I2C0_SCL | Touch I2C |
| GPIO10 | 18 | LCD_CLK | SPI CLK |
| GPIO11 | 19 | LCD_MOSI | SPI MOSI |
| GPIO12 | 20 | LCD_CS | SPI CS |
| GPIO13 | 21 | LCD_DC | Data/Command |
| GPIO14 | 22 | LCD_RST | LCD Reset |
| GPIO15 | 8 | CHG_PG | Power good (optional) |
| GPIO16 | 9 | FIRE_EN | Fire enable |
| GPIO17 | 10 | I2C1_SDA | Power I2C |
| GPIO18 | 11 | I2C1_SCL | Power I2C |
| GPIO19 | 13 | USB_D- | USB Data- |
| GPIO20 | 14 | USB_D+ | USB Data+ |
| GPIO21 | 23 | LCD_BL | Backlight PWM |
| GPIO38 | 31 | CHG_INT | Charger interrupt |
| GPIO39 | 32 | CHG_CE | Charge enable (JTAG) |
| GPIO40 | 33 | CHG_OTG | OTG mode (JTAG) |
| GPIO41 | 34 | FG_ALRT | Fuel gauge alert (JTAG) |
| GPIO42 | 35 | BOOST_EN | Boost enable (JTAG) |
| GPIO45 | 26 | CHG_STAT | Charge status (optional), strapping |
| GPIO47 | 24 | LED_STATUS | Status LED |
| GPIO48 | 25 | WDT_PING | Watchdog trigger |

**Source:** [ESP32-S3-WROOM-1 Datasheet](https://www.espressif.com/sites/default/files/documentation/esp32-s3-wroom-1_wroom-1u_datasheet_en.pdf), Table 3-1

#### Step 2: Create All Hierarchical Labels
Create all labels listed in Section 10.3 above with their respective directions.

#### Step 3: Wire Power Supply
```
[VCC_3V3] ──┬── C_3V3_2 (10µF) ──┬── C_3V3_1 (100nF) ──┬── U5:3V3 (pin 2)
            │                    │                     │
            └────────────────────┴─────────────────────┴── GND
```

#### Step 4: Wire Reset Circuit
```
[VCC_3V3] ──┬── U5:EN (pin 3)
            │
            └── SW_RST ──┬── GND
                         │
                         └── C_RST (100nF) ──── GND
```

#### Step 5: Wire Boot Button
```
U5:GPIO0 (pin 27) ──┬── SW_BOOT ──── GND
                    │
                    └── C_BOOT (10nF) ──── GND
```

**⚠️ C_BOOT Size:** Use 10nF (not 100nF) for GPIO0 debounce. GPIO0 is a strapping pin sampled during boot. A large capacitor could cause the pin to be sampled as LOW (internal pull-up can't charge it fast enough), putting the chip into download mode unintentionally.

#### Step 6: Wire I2C Bus 0 (Touch) with Pull-ups
```
U5:GPIO8 (pin 12) ──┬── R13 (4.7k) ──── [VCC_3V3]
                    │
                    └── [I2C0_SDA]

U5:GPIO9 (pin 17) ──┬── R14 (4.7k) ──── [VCC_3V3]
                    │
                    └── [I2C0_SCL]

U5:GPIO1 (pin 39) ──── [TP_RST]
U5:GPIO2 (pin 38) ──── [TP_IRQ]
```

#### Step 7: Wire I2C Bus 1 (Power Management) with Pull-ups
```
U5:GPIO17 (pin 10) ──┬── R15 (4.7k) ──── [VCC_3V3]
                     │
                     └── [I2C1_SDA]

U5:GPIO18 (pin 11) ──┬── R16 (4.7k) ──── [VCC_3V3]
                     │
                     └── [I2C1_SCL]
```

#### Step 8: Wire Charger Control Signals
```
U5:GPIO38 (pin 31) ──── [CHG_INT]
U5:GPIO39 (pin 32) ──── [CHG_CE]
U5:GPIO40 (pin 33) ──── [CHG_OTG]
```

**Optional:** Wire charger status signals for UI feedback:
```
U5:GPIO15 (pin 8) ──── [CHG_PG]       (Power good - LOW when valid input power)
U5:GPIO45 (pin 26) ──── [CHG_STAT]    (Charge status - see BQ25896 datasheet Table 11)
```
These signals allow the firmware to display charging status (charging, full, fault) without I2C polling. GPIO45 is a strapping pin but safe for input after boot.

#### Step 9: Wire Fuel Gauge Alert
```
U5:GPIO41 (pin 34) ──── [FG_ALRT]
```

#### Step 10: Wire Boost Control
```
U5:GPIO42 (pin 35) ──── [BOOST_EN]
U5:GPIO4 (pin 4) ──── [V_SENSE]   (ADC1_CH3 - safe with Wi-Fi)
```

#### Step 11: Wire Safety System Signals
```
U5:GPIO48 (pin 25) ──── [WDT_PING]
U5:GPIO16 (pin 9) ──── [FIRE_EN]
U5:GPIO6 (pin 6) ──── [FIRE_BTN]
U5:GPIO7 (pin 7) ──── [OC_FAULT]
```

#### Step 12: Wire Coil Control
```
U5:GPIO5 (pin 5) ──── [COIL_PWM]
U5:GPIO3 (pin 15) ──── [I_SENSE]  (ADC1_CH2 - safe with Wi-Fi)
```

**Note:** I_SENSE and V_SENSE use ADC1 pins to avoid ADC2/Wi-Fi conflicts.

**⚠️ GPIO3 Strapping Pin Verification (Board Bring-Up):**

GPIO3 is a strapping pin (JTAG selection) sampled at boot. The INA180 output connects here via R6 (100Ω) and C18 (10nF) RC filter (~1µs time constant).

**Expected behavior:** During power-on, INA180 output is near 0V (no current flowing), so GPIO3 is pulled low by the RC filter. This should not affect normal boot.

**Verification during board bring-up:**
1. Measure GPIO3 voltage at power-on (should be <0.3V)
2. Verify ESP32 enters normal boot mode (not JTAG mode)
3. If boot issues occur, use one of these mitigations:

**Mitigation Option A (Preferred):** Add 10kΩ pull-down on GPIO3 to override INA180 during boot.

**Mitigation Option B (If Option A fails):** Add 1kΩ series resistor (R_ISO) between INA180 output and the RC filter to isolate the strapping function:
```
U8:OUT (pin 4) ──── R_ISO (1kΩ) ──── R6 (100Ω) ──── C18 (10nF) ──┬── U5:GPIO3
                                                                  │
                                                                 GND
```
The R_ISO creates a voltage divider with R6, reducing INA180's influence during the boot strapping window while maintaining ADC accuracy during normal operation.

#### Step 13: Wire Display SPI
```
U5:GPIO10 (pin 18) ──── [LCD_CLK]
U5:GPIO11 (pin 19) ──── [LCD_MOSI]
U5:GPIO12 (pin 20) ──── [LCD_CS]
U5:GPIO13 (pin 21) ──── [LCD_DC]
U5:GPIO14 (pin 22) ──── [LCD_RST]
U5:GPIO21 (pin 23) ──── [LCD_BL]
```

#### Step 14: Wire USB
```
U5:GPIO19 (pin 13) ──── [USB_D+]
U5:GPIO20 (pin 14) ──── [USB_D-]
```

#### Step 15: Wire LED Status
```
U5:GPIO47 (pin 24) ──── [LED_STATUS]
```

#### Step 16: Connect Ground Pins
```
U5:GND (pins 40, 41) ──── GND
U5:GND (EP) ──── GND  (Exposed pad)
```

#### Step 17: Wire 3.3V LDO Regulator

**LDO Input Source**
The LDO input (**U14:VIN**) connects to **`VSYS`** (system voltage from BQ25896).

Since the fuel gauge is on the **battery path** (not system path), VSYS goes directly to all system loads.

1. Press `A`, search for `AP2112K-3.3`
2. Place U14 near the ESP32 module
3. Assign: Reference = `U14`, Value = `AP2112K-3.3TRG1`

**Wiring:**
```
[VSYS] ──┬── C_LDO_IN (10µF) ──── GND
         │
         └── U14:VIN (pin 1)
                 │
             U14:EN (pin 3) ──── U14:VIN (tie high for always-on)
                 │
             U14:VOUT (pin 5) ──┬── C_LDO_OUT (10µF) ──── GND
                                │
                                └── [VCC_3V3]

U14:GND (pin 2) ──── GND
```

**AP2112K-3.3 Pinout:**
| Pin | Name | Connection |
|-----|------|------------|
| 1 | VIN | VSYS |
| 2 | GND | GND |
| 3 | EN | VIN (always enabled) |
| 4 | NC | No connection |
| 5 | VOUT | VCC_3V3 |

### 10.5 MCU Schematic Overview

Due to the large number of connections, here's a simplified representation:

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                      MCU                                             │
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                        ESP32-S3-WROOM-1-N8R2                                │    │
│  │                              U5                                              │    │
│  │                                                                              │    │
│  │  [VCC_3V3]──►3V3(2)                                              GND(40,41)├──►GND
│  │                                                                              │    │
│  │  [I2C0_SDA]◄──►GPIO8(12)  ──┐                                               │    │
│  │  [I2C0_SCL]◄────GPIO9(13)  ──┼── I2C Bus 0 (Touch)                          │    │
│  │  [TP_RST]◄──────GPIO1(39)  ──┤                                               │    │
│  │  [TP_IRQ]──────►GPIO2(38)  ──┘                                               │    │
│  │                                                                              │    │
│  │  [I2C1_SDA]◄──►GPIO17(18) ──┐                                               │    │
│  │  [I2C1_SCL]◄────GPIO18(19) ──┼── I2C Bus 1 (Power)                          │    │
│  │                              │                                               │    │
│  │  [CHG_INT]─────►GPIO38(5)  ──┤                                               │    │
│  │  [CHG_CE]◄──────GPIO39(4)  ──┼── Charger                                     │    │
│  │  [CHG_OTG]◄─────GPIO40(3)  ──┘                                               │    │
│  │                                                                              │    │
│  │  [FG_ALRT]─────►GPIO41(2)  ──── Fuel Gauge                                  │    │
│  │                                                                              │    │
│  │  [BOOST_EN]◄────GPIO42(1)  ──── Boost Enable                                │    │
│  │                                                                              │    │
│  │  [V_SENSE]─────►GPIO4(26)  ──── Boost Voltage (ADC1_CH3)                    │    │
│  │                                                                              │    │
│  │  [WDT_PING]◄────GPIO48(33) ──┐                                               │    │
│  │  [FIRE_EN]◄─────GPIO16(17) ──┼── Safety System                              │    │
│  │  [FIRE_BTN]────►GPIO6(24)  ──┤                                               │    │
│  │  [OC_FAULT]────►GPIO7(23)  ──┘                                               │    │
│  │                                                                              │    │
│  │  [COIL_PWM]◄────GPIO5(25)  ──── Coil PWM                                    │    │
│  │  [I_SENSE]─────►GPIO3(27)  ──── Coil Current (ADC1_CH2)                     │    │
│  │                                                                              │    │
│  │  [LCD_CLK]◄─────GPIO10(14) ──┐                                               │    │
│  │  [LCD_MOSI]◄────GPIO11(15) ──┤                                               │    │
│  │  [LCD_CS]◄──────GPIO12(8)  ──┼── Display SPI                                │    │
│  │  [LCD_DC]◄──────GPIO13(9)  ──┤                                               │    │
│  │  [LCD_RST]◄─────GPIO14(10) ──┤                                               │    │
│  │  [LCD_BL]◄──────GPIO21(20) ──┘                                               │    │
│  │                                                                              │    │
│  │  [USB_D+]◄─────►GPIO19(21) ──┐                                               │    │
│  │  [USB_D-]◄─────►GPIO20(22) ──┼── USB                                        │    │
│  │                              │                                               │    │
│  │  [LED_STATUS]◄──GPIO47(34) ──── Status LED                                  │    │
│  │                                                                              │    │
│  │                  GPIO0(27) ──── SW_BOOT ──── GND                            │    │
│  │                    EN(3)   ──── SW_RST  ──── GND                            │    │
│  │                                                                              │    │
│  └──────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                      │
│  I2C Pull-ups:                                                                       │
│    R13, R14 (4.7k) on I2C0_SDA, I2C0_SCL                                            │
│    R15, R16 (4.7k) on I2C1_SDA, I2C1_SCL                                            │
│                                                                                      │
│  ⚠️ ADC pins on ADC1 (GPIO3, GPIO4) are safe with Wi-Fi/Bluetooth active           │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 11. Subsheet: UI

### 11.1 Sheet Information

| Property | Value |
|----------|-------|
| Sheet Name | UI |
| File Name | UI.kicad_sch |
| Page Size | A4 |
| Description | Display FPC, fire button, LED connector |

### 11.2 Components

| Designator | Symbol | Value | Footprint | Description |
|------------|--------|-------|-----------|-------------|
| J4 | Conn_01x12 | SM12B-GHS-TB | Connector_JST:JST_GH_SM12B-GHS-TB_1x12-1MP_P1.25mm_Horizontal | Display connector (JST-GH 12-pin) |
| J5 | Conn_01x04 | B4B-XH-A | Connector_JST:JST_XH_B4B-XH-A_1x04_P2.50mm_Vertical | Fire button connector (JST-XH 4-pin) |
| R22 | R | 220Ω | Resistor_SMD:R_0402_1005Metric | LED current limiting |
| C_DISP | C | 100nF | Capacitor_SMD:C_0402_1005Metric | Display VCC bypass |

### 11.3 Hierarchical Labels

| Label Name | Direction | Purpose |
|------------|-----------|---------|
| VCC_3V3 | Input | 3.3V power for display |
| GND | Bidirectional | Ground |
| **Display SPI** |
| LCD_CLK | Input | SPI clock |
| LCD_MOSI | Input | SPI data |
| LCD_CS | Input | Chip select |
| LCD_DC | Input | Data/Command |
| LCD_RST | Input | Reset |
| LCD_BL | Input | Backlight PWM |
| **Touch I2C** |
| I2C0_SDA | Bidirectional | I2C data |
| I2C0_SCL | Input | I2C clock |
| TP_RST | Input | Touch reset |
| TP_IRQ | Output | Touch interrupt |
| **Fire Button** |
| FIRE_BTN | Output | Button signal (active LOW) |
| LED_STATUS | Input | LED control |

### 11.4 Wiring Instructions

#### Step 1: Place Display Connector
1. Press `A`, search for `Conn_01x12`
2. Place J4 in the upper portion of the sheet
3. Assign: Reference = `J4`, Value = `SM12B-GHS-TB`
4. Assign footprint: `Connector_JST:JST_GH_SM12B-GHS-TB_1x12-1MP_P1.25mm_Horizontal`

**⚠️ Pinout Verification:** The pinout below is based on the 1.69" Round IPS LCD with Touch Panel (240×280, ST7789V2 + CST816T, JST-GH 12-pin connector). **Before ordering PCBs**, verify the connector pinout against your actual module. Pin 1 is typically marked on the module PCB.

#### Step 2: Create Hierarchical Labels
Create all labels from Section 11.3.

#### Step 3: Wire Display Connector (J4)

**Display Module:** 1.69" Round IPS LCD with Touch (ST7789V2 + CST816T)
**Connector:** JST-GH 12-pin (SM12B-GHS-TB)

```
J4 Pin | Signal      | Hierarchical Label
-------|-------------|-------------------
  1    | VCC (3.3V)  | [VCC_3V3]
  2    | GND         | GND
  3    | LCD_DIN     | [LCD_MOSI]
  4    | LCD_CLK     | [LCD_CLK]
  5    | LCD_CS      | [LCD_CS]
  6    | LCD_DC      | [LCD_DC]
  7    | LCD_RST     | [LCD_RST]
  8    | LCD_BL      | [LCD_BL]
  9    | TP_SDA      | [I2C0_SDA]
 10    | TP_SCL      | [I2C0_SCL]
 11    | TP_RST      | [TP_RST]
 12    | TP_IRQ      | [TP_IRQ]
```

Wire each pin to its corresponding hierarchical label.

Add bypass capacitor:
```
[VCC_3V3] ──── C_DISP (100nF) ──── GND
```

#### Step 4: Wire Fire Button Connector (J5)

J5 is a 4-pin JST-XH connector for the illuminated fire button (LED + switch).

1. Place J5 (JST-XH 4-pin)
2. Wire according to this pinout:

**Pinout (notches facing you, left to right):**

| Pin | Signal | Description |
|-----|--------|-------------|
| 1 | LED- | LED cathode (GND) |
| 2 | FIRE_BTN | Switch signal |
| 3 | GND | Switch common |
| 4 | LED+ | LED anode (from R22) |

```
                                J5 (Fire Button - JST-XH 4-pin)
                                ┌─────────────────────────────┐
                                │                             │
           GND ────────────────►│ Pin 1 (LED-)                │
                                │                             │
      [FIRE_BTN] ◄──────────────│ Pin 2 (Switch Signal)       │
                                │                             │
           GND ────────────────►│ Pin 3 (Switch GND)          │
                                │                             │
[LED_STATUS] ─── R22 (220Ω) ───►│ Pin 4 (LED+)                │
                                │                             │
                                └─────────────────────────────┘
```

**Wiring steps:**
1. Wire J5:Pin 1 to GND
2. Wire J5:Pin 2 to FIRE_BTN hierarchical label
3. Wire J5:Pin 3 to GND
4. Wire LED_STATUS hierarchical label to R22 pin 1
5. Wire R22 pin 2 to J5:Pin 4

**⚠️ Fire Button Pull-up:** The FIRE_BTN signal may need a pull-up resistor (10kΩ to VCC_3V3) if not provided by the button assembly. When pressed, the button pulls the line LOW.

### 11.5 Completed UI Schematic Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                    UI                                            │
│                                                                                  │
│  From MCU                                              To External Components    │
│     │                                                           │                │
│     ▼                                                           ▼                │
│                                                                                  │
│  ┌───────────────────────────────────────────────────────────────────────┐      │
│  │              Display Connector (J4) - JST-GH 12-pin                   │      │
│  │                 1.69" Round IPS LCD with Touch                        │      │
│  │                                                                       │      │
│  │  Pin 1  ──── [VCC_3V3] ──┬── C_DISP ── GND                           │      │
│  │  Pin 2  ──── GND         │   100nF                                   │      │
│  │  Pin 3  ◄─── [LCD_MOSI]  │                                           │      │
│  │  Pin 4  ◄─── [LCD_CLK]   │                                           │      │
│  │  Pin 5  ◄─── [LCD_CS]    │                                           │      │
│  │  Pin 6  ◄─── [LCD_DC]    │                                           │      │
│  │  Pin 7  ◄─── [LCD_RST]   │                                           │      │
│  │  Pin 8  ◄─── [LCD_BL]    │                                           │      │
│  │  Pin 9  ◄──► [I2C0_SDA]  │                                           │      │
│  │  Pin 10 ◄─── [I2C0_SCL]  │                                           │      │
│  │  Pin 11 ◄─── [TP_RST]    │                                           │      │
│  │  Pin 12 ───► [TP_IRQ]    │                                           │      │
│  │                          │                                           │      │
│  └──────────────────────────┼───────────────────────────────────────────┘      │
│                             │                                                   │
│                             │     ┌─────────────────────────────────────┐       │
│                             │     │   Fire Button Connector (J5)        │       │
│                             │     │        JST-XH 4-pin (B4B-XH-A)      │       │
│                             │     │                                     │       │
│                    GND ─────┼────►│  Pin 1 (LED-)                       │       │
│                             │     │                                     │       │
│  [FIRE_BTN] ◄───────────────┼─────│  Pin 2 (Switch Signal)              │──►To  │
│                             │     │                                     │ Fire  │
│                    GND ─────┼────►│  Pin 3 (Switch GND)                 │Button │
│                             │     │                                     │       │
│  [LED_STATUS] ──── R22 ─────┼────►│  Pin 4 (LED+)                       │       │
│                    220Ω     │     │                                     │       │
│                             │     └─────────────────────────────────────┘       │
│                             │                                                   │
│  (Optional: Add 10kΩ pull-up on FIRE_BTN if not in button assembly)            │
│                             │                                                   │
└─────────────────────────────┴───────────────────────────────────────────────────┘
```

---

## 12. Synchronizing Hierarchical Labels to Root Sheet

After completing all subsheets, you must synchronize the hierarchical labels to the root sheet. This creates the hierarchical pins on the sheet symbols in the root sheet.

### 12.1 Synchronization Process

For **each subsheet**, perform these steps:

#### Step 1: Open the Root Sheet
1. Open `vaporizer.kicad_sch` (root sheet)

#### Step 2: Select the Subsheet Symbol
1. Click on the sheet symbol (e.g., `USB_Power`)

#### Step 3: Open Sheet Symbol Properties
1. Right-click → **Sheet Properties** OR press `E`
2. Click on the **Hierarchical Pins** tab (or similar in your KiCad version)

#### Step 4: Import Hierarchical Labels
1. Click **Import Sheet Pins** or **Synchronize Sheet Pins**
2. KiCad will scan the subsheet for all hierarchical labels
3. Hierarchical pins will be created on the sheet symbol

#### Step 5: Position the Hierarchical Pins
After importing, the pins appear on the sheet symbol. Arrange them logically:

**For USB_Power:**
```
┌─────────────────────┐
│     USB_Power       │
│                     │
│                VBUS ├──►
│              USB_D+ ├──►
│              USB_D- ├──►
│                 GND ├──►
│                     │
└─────────────────────┘
```

Drag pins to edges:
- **Output pins** → Right edge (arrow pointing right)
- **Input pins** → Left edge (arrow pointing left)
- **Bidirectional pins** → Either edge (diamond symbol)

#### Step 6: Repeat for All Subsheets

Perform synchronization for:
1. USB_Power
2. BatteryManagement
3. FuelGauge
4. BoostConverter
5. SafetyLogic
6. CoilDriver
7. MCU
8. UI

### 12.2 Connecting Sheet Pins on Root Sheet

After synchronization, connect the hierarchical pins between sheets using wires and net labels.

#### Step 1: Wire Power Nets
Use **global labels** or **power symbols** for common power nets:

```
1. VBUS:     USB_Power:VBUS ────────────► BatteryManagement:VBUS
2. VSYS:     BatteryManagement:VSYS ──┬─► FuelGauge:VSYS
                                      ├─► BoostConverter:VSYS
                                      └─► MCU:VSYS (for monitoring)
3. V_BOOST:  BoostConverter:V_BOOST ──┬─► CoilDriver:V_BOOST
                                      └─► MCU:V_SENSE (via divider)
4. VCC_3V3:  (Create 3.3V regulator or derive from VSYS)
             Connect to: SafetyLogic, CoilDriver, MCU, UI
5. GND:      Use ground symbols throughout (all sheets connect to same GND)
```

#### Step 2: Wire I2C Buses
```
I2C Bus 1 (Power Management):
  BatteryManagement:I2C1_SDA ◄──►──┬──► BoostConverter:I2C1_SDA
                                   └──► FuelGauge:I2C1_SDA
                                   └──► MCU:I2C1_SDA

  BatteryManagement:I2C1_SCL ◄────┬──► BoostConverter:I2C1_SCL
                                  └──► FuelGauge:I2C1_SCL
                                  └──► MCU:I2C1_SCL

I2C Bus 0 (Touch):
  UI:I2C0_SDA ◄──► MCU:I2C0_SDA
  UI:I2C0_SCL ◄─── MCU:I2C0_SCL
  UI:TP_RST   ◄─── MCU:TP_RST
  UI:TP_IRQ   ───► MCU:TP_IRQ
```

#### Step 3: Wire Safety Signals
```
Safety Chain:
  MCU:WDT_PING   ────────► SafetyLogic:WDT_PING
  MCU:FIRE_EN    ────────► SafetyLogic:FIRE_EN
  UI:FIRE_BTN    ────────► SafetyLogic:FIRE_BTN ────► MCU:FIRE_BTN
  CoilDriver:OC_FAULT ───► SafetyLogic:OC_FAULT ───► MCU:OC_FAULT
  CoilDriver:T_ALERT ────► SafetyLogic:T_ALERT       (⚠️ HARDWARE THERMAL INTERLOCK)
  SafetyLogic:SAFETY_GATE ────────────────────────► CoilDriver:SAFETY_GATE
```

#### Step 4: Wire Coil Control
```
MCU:COIL_PWM ────────► CoilDriver:COIL_PWM
CoilDriver:I_SENSE ──► MCU:I_SENSE
```

#### Step 5: Wire Display
```
MCU:LCD_CLK  ────► UI:LCD_CLK
MCU:LCD_MOSI ────► UI:LCD_MOSI
MCU:LCD_CS   ────► UI:LCD_CS
MCU:LCD_DC   ────► UI:LCD_DC
MCU:LCD_RST  ────► UI:LCD_RST
MCU:LCD_BL   ────► UI:LCD_BL
```

#### Step 6: Wire USB
```
USB_Power:USB_D+ ◄──► MCU:USB_D+
USB_Power:USB_D- ◄──► MCU:USB_D-
```

#### Step 7: Wire LED Status
```
MCU:LED_STATUS ────► UI:LED_STATUS
```

### 12.3 Root Sheet Final Layout Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                    VAPORIZER ROOT SHEET                                          │
│                                                                                                  │
│  ┌─────────────────┐                           ┌────────────────────────────┐                   │
│  │   USB_Power     │                           │     BatteryManagement      │                   │
│  │                 │                           │                            │                   │
│  │            VBUS ├───────────────────────────► VBUS                       │                   │
│  │          USB_D+ ├─────────────────────┐     │                       VSYS ├───────────────┐   │
│  │          USB_D- ├─────────────────────┼─┐   │                  VBAT_PROT ├──────┐        │   │
│  │             GND ├────(Power Symbol)   │ │   │                    CHG_INT ├────┐ │        │   │
│  │                 │                     │ │   │    CHG_CE ◄────────────────┤    │ │        │   │
│  └─────────────────┘                     │ │   │    CHG_OTG ◄───────────────┤    │ │        │   │
│                                          │ │   │   I2C1_SDA ◄───────────────┤    │ │        │   │
│                                          │ │   │   I2C1_SCL ◄───────────────┤    │ │        │   │
│                                          │ │   │                        GND ├────┼─┼─Power  │   │
│                                          │ │   └────────────────────────────┘    │ │        │   │
│                                          │ │                                     │ │        │   │
│                                          │ │   ┌────────────────────────────┐    │ │        │   │
│                                          │ │   │        FuelGauge           │    │ │        │   │
│                                          │ │   │                            │    │ │        │   │
│                                          │ │   │      VSYS ◄────────────────┼────┼─┼────────┤   │
│                                          │ │   │  I2C1_SDA ◄────────────────┼──┐ │ │        │   │
│                                          │ │   │  I2C1_SCL ◄────────────────┼──┼─┤ │        │   │
│                                          │ │   │    FG_ALRT ────────────────┼──┼─┼─┼──┐     │   │
│                                          │ │   │                        GND ├──┼─┼─┼──┼Power│   │
│                                          │ │   └────────────────────────────┘  │ │ │  │     │   │
│                                          │ │                                   │ │ │  │     │   │
│  ┌─────────────────┐                     │ │   ┌────────────────────────────┐  │ │ │  │     │   │
│  │  BoostConverter │                     │ │   │        SafetyLogic         │  │ │ │  │     │   │
│  │                 │                     │ │   │                            │  │ │ │  │     │   │
│  │      VSYS ◄─────┼─────────────────────┼─┼───┤───────────────────────────│   │ │ │  │     │   │
│  │  BOOST_EN ◄─────┼───────────────────┐ │ │   │     VCC_3V3 ◄─────────────┤   │ │ │  │     │   │
│  │  I2C1_SDA ◄─────┼───────────────────┼─┼─┼───┤────────────────────────────│   │ │ │  │     │   │
│  │  I2C1_SCL ◄─────┼───────────────────┼─┼─┼───┤────────────────────────────│   │ │ │  │     │   │
│  │   V_BOOST ──────┼───────────────────┼─┼─┼───┼────────────────────────────┼───┼─┼─┼──┼─┐   │   │
│  │       GND ──────┼─Power             │ │ │   │     WDT_PING ◄─────────────┤   │ │ │  │ │   │   │
│  └─────────────────┘                   │ │ │   │      FIRE_BTN ◄────────────┤   │ │ │  │ │   │   │
│                                        │ │ │   │     OC_FAULT ◄─────────────┤   │ │ │  │ │   │   │
│                                        │ │ │   │      FIRE_EN ◄─────────────┤   │ │ │  │ │   │   │
│                                        │ │ │   │   SAFETY_GATE ─────────────┼───┼─┼─┼──┼─┼─┐ │   │
│                                        │ │ │   │                        GND ├───┼─┼─┼──┼─┼─┼Power
│                                        │ │ │   └────────────────────────────┘   │ │ │  │ │ │ │   │
│                                        │ │ │                                    │ │ │  │ │ │ │   │
│  ┌─────────────────┐                   │ │ │                                    │ │ │  │ │ │ │   │
│  │   CoilDriver    │                   │ │ │                                    │ │ │  │ │ │ │   │
│  │                 │                   │ │ │                                    │ │ │  │ │ │ │   │
│  │   V_BOOST ◄─────┼───────────────────┼─┼─┼────────────────────────────────────┼─┼─┼──┼─┘ │ │   │
│  │ SAFETY_GATE ◄───┼───────────────────┼─┼─┼────────────────────────────────────┼─┼─┼──────┘ │   │
│  │   COIL_PWM ◄────┼───────────────────┼─┼─┼───┐                                │ │ │        │   │
│  │    VCC_3V3 ◄────┼─Power             │ │ │   │                                │ │ │        │   │
│  │    I_SENSE ─────┼───────────────────┼─┼─┼───┼─┐                              │ │ │        │   │
│  │   OC_FAULT ─────┼───────────────────┼─┼─┼───┼─┼─► (to SafetyLogic)           │ │ │        │   │
│  │      COIL+ ─────┼─► (to J6)         │ │ │   │ │                              │ │ │        │   │
│  │      COIL- ─────┼─► (to J6)         │ │ │   │ │                              │ │ │        │   │
│  │        GND ─────┼─Power             │ │ │   │ │                              │ │ │        │   │
│  └─────────────────┘                   │ │ │   │ │                              │ │ │        │   │
│                                        │ │ │   │ │                              │ │ │        │   │
│  ┌─────────────────────────────────────┼─┼─┼───┼─┼──────────────────────────────┼─┼─┼────────┴─┐ │
│  │                 MCU                 │ │ │   │ │                              │ │ │          │ │
│  │        ESP32-S3-WROOM-1             │ │ │   │ │                              │ │ │          │ │
│  │                                     │ │ │   │ │                              │ │ │          │ │
│  │  VCC_3V3 ◄──────────────────────────┼─┼─┼───┼─┼──────────────────────────────┼─┼─┼─Power    │ │
│  │      VSYS ◄─────────────────────────┼─┼─┼───┼─┼──────────────────────────────┘ │ │          │ │
│  │      GND ◄──────────────────────────┼─┼─┼───┼─┼─Power                          │ │          │ │
│  │                                     │ │ │   │ │                                │ │          │ │
│  │ I2C1_SDA ◄──────────────────────────┼─┼─┼───┼─┼────────────────────────────────┘ │          │ │
│  │ I2C1_SCL ◄──────────────────────────┼─┼─┼───┼─┼──────────────────────────────────┘          │ │
│  │  CHG_INT ◄──────────────────────────┼─┼─┼───┼─┼─(from BatteryMgmt)                          │ │
│  │   CHG_CE ───────────────────────────┼─┼─┼───┼─┼─► (to BatteryMgmt)                          │ │
│  │  CHG_OTG ───────────────────────────┼─┼─┼───┼─┼─► (to BatteryMgmt)                          │ │
│  │  FG_ALRT ◄──────────────────────────┼─┼─┼───┼─┼─(from FuelGauge)                            │ │
│  │ BOOST_EN ───────────────────────────┼─┼─┼───┼─┼─► (to BoostConverter)                       │ │
│  │  V_SENSE ◄──────────────────────────┼─┼─┼───┼─┼─(from voltage divider on V_BOOST)           │ │
│  │                                     │ │ │   │ │                                             │ │
│  │ WDT_PING ───────────────────────────┼─┼─┼───┼─┼─► (to SafetyLogic)                          │ │
│  │  FIRE_EN ───────────────────────────┼─┼─┼───┼─┼─► (to SafetyLogic)                          │ │
│  │ FIRE_BTN ◄──────────────────────────┼─┼─┼───┼─┼─(from SafetyLogic/UI)                       │ │
│  │ OC_FAULT ◄──────────────────────────┼─┼─┼───┼─┼─(from CoilDriver)                           │ │
│  │                                     │ │ │   │ │                                             │ │
│  │ COIL_PWM ───────────────────────────┼─┼─┼───┘ └─► (to CoilDriver)                           │ │
│  │  I_SENSE ◄──────────────────────────┼─┼─┼──────(from CoilDriver)                            │ │
│  │                                     │ │ │                                                   │ │
│  │  LCD_CLK ───────────────────────────┼─┼─┼─► (to UI)                                         │ │
│  │ LCD_MOSI ───────────────────────────┼─┼─┼─► (to UI)                                         │ │
│  │   LCD_CS ───────────────────────────┼─┼─┼─► (to UI)                                         │ │
│  │   LCD_DC ───────────────────────────┼─┼─┼─► (to UI)                                         │ │
│  │  LCD_RST ───────────────────────────┼─┼─┼─► (to UI)                                         │ │
│  │   LCD_BL ───────────────────────────┼─┼─┼─► (to UI)                                         │ │
│  │                                     │ │ │                                                   │ │
│  │ I2C0_SDA ◄─────────────────────────►│ │ │─► (to UI)                                         │ │
│  │ I2C0_SCL ───────────────────────────┼─┼─┼─► (to UI)                                         │ │
│  │   TP_RST ───────────────────────────┼─┼─┼─► (to UI)                                         │ │
│  │   TP_IRQ ◄──────────────────────────┼─┼─┼──(from UI)                                        │ │
│  │                                     │ │ │                                                   │ │
│  │   USB_D+ ◄──────────────────────────┘ │ │                                                   │ │
│  │   USB_D- ◄────────────────────────────┘ │                                                   │ │
│  │                                         │                                                   │ │
│  │ LED_STATUS ─────────────────────────────┼─► (to UI)                                         │ │
│  │                                         │                                                   │ │
│  └─────────────────────────────────────────┴───────────────────────────────────────────────────┘ │
│                                                                                                  │
│  ┌─────────────────┐                                                                            │
│  │       UI        │                                                                            │
│  │                 │                                                                            │
│  │    VCC_3V3 ◄────┼─Power                                                                      │
│  │        GND ◄────┼─Power                                                                      │
│  │                 │                                                                            │
│  │    LCD_CLK ◄────┼─(from MCU)                                                                 │
│  │   LCD_MOSI ◄────┼─(from MCU)                                                                 │
│  │     LCD_CS ◄────┼─(from MCU)                                                                 │
│  │     LCD_DC ◄────┼─(from MCU)                                                                 │
│  │    LCD_RST ◄────┼─(from MCU)                                                                 │
│  │     LCD_BL ◄────┼─(from MCU)                                                                 │
│  │                 │                                                                            │
│  │   I2C0_SDA ◄───►┼─(to MCU)                                                                   │
│  │   I2C0_SCL ◄────┼─(from MCU)                                                                 │
│  │     TP_RST ◄────┼─(from MCU)                                                                 │
│  │     TP_IRQ ─────┼─► (to MCU)                                                                 │
│  │                 │                                                                            │
│  │   FIRE_BTN ─────┼─► (to SafetyLogic & MCU)                                                   │
│  │ LED_STATUS ◄────┼─(from MCU)                                                                 │
│  │                 │                                                                            │
│  └─────────────────┘                                                                            │
│                                                                                                  │
│  Power Distribution (not shown as sheet, but as power symbols throughout):                      │
│  - VCC_3V3: Derived from VSYS via 3.3V LDO (add to BatteryManagement or separate sheet)        │
│  - GND: Common ground throughout                                                                │
│                                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 13. Additional Notes

### 13.1 3.3V Regulator

**✓ Now included in Section 10.4, Step 17** — The AP2112K-3.3 LDO regulator is placed on the MCU sheet, powered from VSYS.

### 13.2 Net Classes and High-Current Routing

Configure net classes in **File → Schematic Setup → Net Classes**:

| Net Class | Wire Width | Description |
|-----------|------------|-------------|
| Power | 0.5mm | Medium-current power nets (VBUS, VCC_3V3) |
| Signal | 0.25mm | Standard signal nets |
| HighCurrent | — | See critical list below (do NOT use traces!) |

**⚠️ CRITICAL: High-Current Path Routing**

**Assign these nets to the `HighCurrent` net class:**

| Net Name | Max Current | Path Description |
|----------|-------------|------------------|
| VBAT_PROT | 6A | Protected battery voltage (to BQ25896:BAT and FuelGauge:BATT) |
| BAT_NEG | 6A | Battery negative return (from R_SENSE_FG to Battery-) |
| VSYS | 6A | System rail from charger to boost/MCU/LDO |
| V_BOOST | 6A | Boost converter output (7V) |
| COIL+ | 6A | Positive coil connection to 510 connector |
| COIL- | 6A | Negative coil connection (switched by Q4) |
| GND | 6A+ | Power ground return path |

These nets can carry up to **6A continuous**.

**A 1.0mm trace on 1oz copper can only handle ~3A with 10°C rise.** At 6A, this will overheat.

**Required approach for high-current nets:**
1. Use **polygon pours (copper zones)** instead of traces
2. If traces are necessary, use **minimum 3-4mm width** or route on multiple layers with via stitching
3. Use **2oz copper** on power layers (specify when ordering PCB)
4. Add **via farms** (multiple vias in parallel) at layer transitions
5. Match trace capacity to 14AWG wire equivalent (~2.08mm²)

See Design Spec Section 8.2 for detailed layout guidance.

### 13.3 Design Rule Check (DRC)

Before proceeding to PCB layout, run **Inspect → Electrical Rules Checker**:

1. Fix all errors (missing connections, conflicting net names)
2. Review warnings (unconnected pins should be intentional)
3. Ensure all power pins have proper power flags

### 13.4 Annotation

After completing all sheets, annotate the schematic:

1. Go to **Tools → Annotate Schematic**
2. Select **Use the entire schematic**
3. Choose annotation order (by sheet, by X/Y position)
4. Click **Annotate**

### 13.5 Generate Bill of Materials

Generate a BOM for procurement:

1. Go to **Tools → Generate Bill of Materials**
2. Configure output format (CSV recommended)
3. Include: Reference, Value, Footprint, LCSC Part #

### 13.6 Mounting Holes

Add mounting holes on the **Root Sheet** for mechanical stability and EMI shielding:

| Designator | Symbol | Footprint | Notes |
|------------|--------|-----------|-------|
| MH1-MH4 | MountingHole_Pad | MountingHole:MountingHole_3.2mm_M3_Pad | Connect to GND |

```
MH1:Pad ──── GND
MH2:Pad ──── GND
MH3:Pad ──── GND
MH4:Pad ──── GND
```

**Placement:** Position at board corners, minimum 3mm from edge.
**Purpose:** Provides EMI shielding and mechanical grounding.

**📐 PCB LAYOUT TIP:**
When routing, ensure mounting holes connect to the **System Ground Plane** (typically Layer 2), NOT the high-current switching path of the Coil/Boost converter. Connecting to the noisy power path will radiate switching noise into the chassis.

### 13.7 Test Points

Add test points on critical nets for the "Pre-Power Smoke Test" (Design Spec Section 9.1):

| Designator | Net | Location (Sheet) | Purpose |
|------------|-----|------------------|---------|
| TP1 | VBAT_PROT | BatteryManagement | Battery voltage before charger |
| TP2 | VSYS | BatteryManagement | System voltage output |
| TP3 | V_BOOST | BoostConverter | Boost converter output |
| TP4 | VCC_3V3 | BatteryManagement | 3.3V logic rail |
| TP5 | WDT_OK | SafetyLogic | Watchdog output (between Schmitt buffer and AND) |
| TP6 | OC_FAULT | CoilDriver | Overcurrent fault signal |
| TP7 | SAFETY_GATE | SafetyLogic | Final AND gate output |

**Symbol:** Use `TestPoint` from Device library
**Footprint:** `TestPoint:TestPoint_Pad_D1.5mm` or similar

**Tip:** Place TP5 (WDT_OK) as an internal net test point — this is invaluable for debugging the safety chain without firing the coil.

### 13.8 Optional: System Wake Button (QON)

If battery protection trips (UVLO), the BQ25896 enters shipping mode. Add pads to wake without USB:

| Designator | Symbol | Value | Footprint | Description |
|------------|--------|-------|-----------|-------------|
| SW_WAKE | SW_Push or TestPoint | QON_WAKE | Button_Switch_SMD or TestPoint pads | Momentarily pulls QON_N to GND |

```
BQ25896:QON_N ──┬── R_QON (10k) ──── REGN
                │
                └── SW_WAKE ──── GND  (optional debug button)
```

**Note:** This is optional but helpful for debugging if protection latch-up occurs.

---

## 14. Summary

This guide covered:

1. **Project Structure** - Hierarchical organization with 8 functional subsheets
2. **USB_Power** - USB-C connector, ESD protection, CC resistors
3. **BatteryManagement** - BQ25896 charger, BQ29705 protection, NTC circuit
4. **FuelGauge** - MAX17055 battery fuel gauge
5. **BoostConverter** - TPS61088 with MCP4561 adjustable output
6. **SafetyLogic** - 555 watchdog, AND gate interlock system
7. **CoilDriver** - PMOS/NMOS power stage, current sensing, OC protection
8. **MCU** - ESP32-S3-WROOM-1 with all GPIO connections
9. **UI** - Display FPC, fire button LED connector
10. **Synchronization** - Importing hierarchical labels to root sheet and wiring

**Key Safety Features Implemented:**
- Hardware watchdog (555 timer)
- 4-input AND gate for fire authorization
- Overcurrent protection (INA180 + LM393)
- FIRE_EN pull-down for safe boot
- Inverting gate driver for fail-safe PMOS control

**Next Steps:**
1. Complete schematic entry in KiCad
2. Run ERC and fix any issues
3. Assign footprints to all components
4. Proceed to PCB layout following the guidelines in the design specification

---

*Document Version: 3.13*
*Based on: Safe Vaporizer Design Specification v3.13*
*Last Updated: January 2025*

**v3.13 Changes (Critical Topology Fix):**
- **🔴 CRITICAL: MAX17055 Fuel Gauge LOW-SIDE Topology Correction**
  - The MAX17055 MUST use LOW-SIDE current sensing (sense resistor in GND return path)
  - Previous v3.9 HIGH-SIDE placement was INCORRECT — IC would not power up!
  - MAX17055 is powered by V_BATT − V_CSP; high-side placement results in 0V differential
  - **Correct topology:** BATT→Battery+, CSP→Q2:Source (GND_SENSE), CSN→Battery- (BAT_NEG)
  - R_SENSE_FG between Q2:Source and Battery- (NOT in battery+ path)
- **New hierarchical labels:**
  - Added `GND_SENSE` — Q2:Source, high side of R_SENSE_FG (for FuelGauge CSP)
  - Added `BAT_NEG` — Battery-, low side of R_SENSE_FG (for FuelGauge CSN)
  - Removed `VBAT_SENSE` (no longer used)
- Updated Section 6 (FuelGauge) with complete LOW-SIDE wiring
- Updated BQ25896:BAT to connect directly to VBAT_PROT (not via fuel gauge)
- Updated all signal flow diagrams and net current tables

**v3.12 Changes (Critical Fixes):**
- **🔴 CRITICAL: BQ29705 V- wiring fix** — R2 now connects to Q1:Source (System GND), NOT Q2:Source
  - Previous wiring would result in 0V differential → OCD/SCD would NEVER trip!
- **🔴 CRITICAL: R5 ILIM value fix** — Changed 2.4kΩ → 800Ω for correct 1.3A input limit
  - Formula: I_INLIM = 1040V / R_ILIM; 2.4kΩ gave only 0.43A (too low)
- **R3 timing resistor fix** — Updated schematic guide to match BOM (270kΩ for 3s timeout)
- **C_USB1 voltage rating** — Changed 16V → 25V for USB-C transient protection
- **C_BOOT size reduction** — Changed 100nF → 10nF to prevent GPIO0 strapping issues
- **Added bypass caps** — C_SCHMITT, C_INV for U12/U13 (74LVC1G17/74LVC1G04)
- **Added FIRE_BTN debounce** — R_DEBOUNCE (10k) + C_DEBOUNCE (100nF) RC filter
- **Added 510 ESD bleed** — R21 (1MΩ) from COIL- to GND (optional)
- **Q4 gate drive option** — Documented TC4426 Channel B alternative for better switching
- **Soft-start timing** — Added TPS61088 internal SS delay (20ms) before PWM ramp
- **LEDC init code** — Added ESP32 PWM initialization example

**v3.10/v3.11 Changes:**
- **Added R_G3_HS (15Ω):** Q3 gate resistor between TC4426:OUT_A and Q3:Gate (prevents ringing)
- **Added D_FLY (SS34):** Flyback diode across coil for inductive kickback protection
- **LM393 speed limitation:** Documented ~1.3µs response time and mitigations
- **GPIO3 strapping verification:** Added board bring-up verification steps
- **MCP4561 NVM wear guidance:** Firmware recommendations to extend NVM life
- **NTC disconnect detection:** Added firmware safety check for broken NTC wire

**v3.9 Changes:** *(⚠️ HIGH-SIDE topology was INCORRECT — see v3.13 for fix)*
- ~~**CRITICAL: Moved FuelGauge to battery path** (VBAT_PROT → sense → VBAT_SENSE → BQ25896:BAT)~~
  - **CORRECTED in v3.13:** MAX17055 requires LOW-SIDE sensing (GND return path)
- Fuel gauge goal: measure both charge AND discharge current (fixes SOC tracking during charging)
- Removed VSYS_LOAD concept — VSYS now goes directly to all system loads
- ~~Added VBAT_SENSE hierarchical label~~ → Replaced with BAT_NEG in v3.13
- Added **R_SAFE (22kΩ)** parallel to MCP4561 in BoostConverter (overvoltage failsafe)
- Updated all signal flow diagrams for new battery-path topology
- Updated LDO wiring: now powered from VSYS (not VSYS_LOAD)

**v3.8 Changes:**
- Added **hardware thermal interlock**: MCP9808 T_ALERT → SafetyLogic AND gate (Gate 4)
- T_ALERT now routes to SafetyLogic instead of MCU GPIO (MCU-independent thermal protection)
- Added R_ALERT (10kΩ pull-up) to SafetyLogic for MCP9808 open-drain output
- Updated SafetyLogic to use all 4 AND gates (5-input safety chain)
- Updated signal routing: CoilDriver:T_ALERT → SafetyLogic:T_ALERT

**v3.7 Changes:**
- **Replaced NTC thermistor with MCP9808 I2C temperature sensor** (eliminates ADC2/Wi-Fi conflict)
- Removed NTC_PCB, R_NTC from CoilDriver — replaced with U15 (MCP9808) + C_MCP
- Removed T_PCB hierarchical label — replaced with T_ALERT, I2C1_SDA, I2C1_SCL
- Added MCP9808 to I2C address table (0x18 on I2C1)
- Updated Section 6.5 firmware for I2C-based temperature reading

**v3.6 Changes:**
- Added Q_REV (Si7461DP) + R_REV (10k) reverse battery protection to BatteryManagement
- Moved WDT_PING from GPIO15 to GPIO48
- Updated watchdog timing resistor R3: 150kΩ → 270kΩ (3s timeout for ESP32 boot)

**v3.5 Changes:**
- Moved LDO (U14) from BatteryManagement to MCU sheet
- Added TVS diode (D2) PCB placement note for USB protection
- Added explicit HighCurrent net class member list with current ratings
