# Vaporizer KiCad Schematic Guide

## Document Purpose

This guide provides comprehensive, step-by-step instructions for creating the vaporizer schematic in KiCad 8.x. It is designed to be self-contained, allowing an agent or user to implement the complete schematic without referencing the original design specification.

---

## 1. Project Setup

### 1.1 Create New Project

1. Open KiCad
2. File → New Project
3. Name: `vaporizer`
4. Location: Choose your working directory
5. KiCad will create:
   - `vaporizer.kicad_pro` (project file)
   - `vaporizer.kicad_sch` (root schematic)
   - `vaporizer.kicad_pcb` (PCB file)

### 1.2 Hierarchical Sheet Structure

The schematic uses a hierarchical design with the following structure:

```
vaporizer.kicad_sch (Root Sheet)
├── power_management.kicad_sch
│   ├── USB-C input and protection
│   ├── BQ25896 battery charger
│   ├── BQ29705 battery protection
│   └── MAX17055 fuel gauge
├── boost_converter.kicad_sch
│   ├── TPS61088 boost converter
│   └── MCP4561 digital potentiometer
├── safety_logic.kicad_sch
│   ├── NE555 watchdog timer
│   ├── 74LVC08A AND gate
│   ├── INA180 current sense
│   └── LM393 comparator
├── coil_driver.kicad_sch
│   ├── TC4426 gate driver
│   ├── Q3 PMOS high-side switch
│   ├── Q4 NMOS low-side switch
│   └── 510 connector interface
└── mcu_peripherals.kicad_sch
    ├── ESP32-S3-WROOM-1
    ├── Display connector
    ├── Touch I2C
    └── Fire button interface
```

### 1.3 Creating Hierarchical Sheets

In the root schematic:

1. Place → Hierarchical Sheet (or press `S`)
2. Draw rectangle for sheet boundary
3. Fill in properties:
   - Sheet name: `Power_Management`
   - File name: `power_management.kicad_sch`
4. Repeat for all five subsheets

---

## 2. Symbol Library Requirements

### 2.1 Required Libraries

Ensure these KiCad libraries are available:

| Library | Components Used |
|---------|-----------------|
| `Device` | R, C, L, LED |
| `Connector` | USB_C_Receptacle, Conn_01x02 |
| `Power` | +3V3, GND, VBUS, VBAT |
| `Transistor_FET` | Generic MOSFET symbols |
| `Timer` | NE555 |
| `Logic_LVC` | 74LVC08, 74LVC1G17 |
| `Comparator` | LM393 |

### 2.2 Custom Symbols Required

Create or download these symbols (not in default libraries):

| Component | Symbol Name | Footprint |
|-----------|-------------|-----------|
| BQ25896 | `BQ25896RTWT` | `QFN-24_4x4mm_P0.5mm` |
| BQ29705 | `BQ29705DSET` | `BGA-5_1.17x0.77mm` |
| MAX17055 | `MAX17055EWL+T` | `WLP-9_1.4x1.4mm` |
| TPS61088 | `TPS61088RHLR` | `QFN-20_4x4mm_P0.5mm` |
| ESP32-S3-WROOM-1 | `ESP32-S3-WROOM-1-N8R2` | `ESP32-S3-WROOM-1` |
| INA180A2 | `INA180A2IDBVR` | `SOT-23-5` |
| TC4426 | `TC4426ACOA` | `SOIC-8` |
| MCP4561-104 | `MCP4561-104E/MS` | `MSOP-8` |
| Si7461DP | `Si7461DP-T1-GE3` | `PowerPAK_SO-8` |
| CSD18563Q5A | `CSD18563Q5A` | `VSON-8_3.3x3.3mm` |
| CSD17571Q2 | `CSD17571Q2` | `VSON-8_3.3x3.3mm` |

---

## 3. Root Sheet Layout

### 3.1 Sheet Placement

Arrange hierarchical sheets in the root schematic:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           ROOT SCHEMATIC                                     │
│                                                                              │
│  ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐        │
│  │                 │     │                 │     │                 │        │
│  │  Power          │     │  Boost          │     │  Coil           │        │
│  │  Management     │────►│  Converter      │────►│  Driver         │        │
│  │                 │     │                 │     │                 │        │
│  └────────┬────────┘     └────────┬────────┘     └────────┬────────┘        │
│           │                       │                       │                  │
│           │              ┌────────▼────────┐              │                  │
│           │              │                 │              │                  │
│           └─────────────►│  Safety         │◄─────────────┘                  │
│                          │  Logic          │                                 │
│                          │                 │                                 │
│                          └────────┬────────┘                                 │
│                                   │                                          │
│                          ┌────────▼────────┐                                 │
│                          │                 │                                 │
│                          │  MCU            │                                 │
│                          │  Peripherals    │                                 │
│                          │                 │                                 │
│                          └─────────────────┘                                 │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 Hierarchical Labels on Root Sheet

Each hierarchical sheet needs pins connecting it to other sheets. Add hierarchical pins by:

1. Double-click the hierarchical sheet
2. Add hierarchical pins matching the labels inside the subsheet

#### Power_Management Sheet Pins:

| Pin Name | Direction | Connected To |
|----------|-----------|--------------|
| `VBUS` | Input | USB-C connector |
| `USB_D+` | Bidirectional | MCU sheet |
| `USB_D-` | Bidirectional | MCU sheet |
| `VSYS` | Output | Boost_Converter, MCU sheet |
| `VBAT` | Bidirectional | Internal (battery connection) |
| `I2C1_SDA` | Bidirectional | MCU sheet |
| `I2C1_SCL` | Input | MCU sheet |
| `CHG_INT` | Output | MCU sheet |
| `CHG_CE` | Input | MCU sheet |
| `CHG_OTG` | Input | MCU sheet |
| `FG_ALRT` | Output | MCU sheet |

#### Boost_Converter Sheet Pins:

| Pin Name | Direction | Connected To |
|----------|-----------|--------------|
| `VSYS` | Input | Power_Management |
| `V_BOOST` | Output | Coil_Driver, Safety_Logic |
| `BOOST_EN` | Input | MCU sheet |
| `I2C1_SDA` | Bidirectional | MCU sheet |
| `I2C1_SCL` | Input | MCU sheet |

#### Safety_Logic Sheet Pins:

| Pin Name | Direction | Connected To |
|----------|-----------|--------------|
| `V_BOOST` | Input | Boost_Converter |
| `WDT_PING` | Input | MCU sheet |
| `FIRE_EN` | Input | MCU sheet |
| `FIRE_BTN` | Input | MCU sheet |
| `OC_FAULT` | Output | MCU sheet |
| `SAFETY_GATE` | Output | Coil_Driver |
| `I_SENSE` | Output | MCU sheet (ADC) |
| `COIL_SENSE` | Input | Coil_Driver |

#### Coil_Driver Sheet Pins:

| Pin Name | Direction | Connected To |
|----------|-----------|--------------|
| `V_BOOST` | Input | Boost_Converter |
| `SAFETY_GATE` | Input | Safety_Logic |
| `COIL_PWM` | Input | MCU sheet |
| `COIL_SENSE` | Output | Safety_Logic |

#### MCU_Peripherals Sheet Pins:

| Pin Name | Direction | Connected To |
|----------|-----------|--------------|
| `VSYS` | Input | Power_Management |
| `USB_D+` | Bidirectional | Power_Management |
| `USB_D-` | Bidirectional | Power_Management |
| `I2C1_SDA` | Bidirectional | All I2C devices |
| `I2C1_SCL` | Output | All I2C devices |
| `WDT_PING` | Output | Safety_Logic |
| `FIRE_EN` | Output | Safety_Logic |
| `FIRE_BTN` | Input | Safety_Logic |
| `OC_FAULT` | Input | Safety_Logic |
| `I_SENSE` | Input | Safety_Logic |
| `COIL_PWM` | Output | Coil_Driver |
| `BOOST_EN` | Output | Boost_Converter |
| `CHG_INT` | Input | Power_Management |
| `CHG_CE` | Output | Power_Management |
| `CHG_OTG` | Output | Power_Management |
| `FG_ALRT` | Input | Power_Management |
| `V_SENSE` | Input | ADC voltage sense |

### 3.3 Root Sheet Wiring

Connect hierarchical pins using wires:

```
Power_Management.VSYS ─────────────────► Boost_Converter.VSYS
                      └────────────────► MCU_Peripherals.VSYS

Boost_Converter.V_BOOST ───────────────► Coil_Driver.V_BOOST
                        └──────────────► Safety_Logic.V_BOOST

Safety_Logic.SAFETY_GATE ──────────────► Coil_Driver.SAFETY_GATE

Coil_Driver.COIL_SENSE ────────────────► Safety_Logic.COIL_SENSE

MCU_Peripherals.WDT_PING ──────────────► Safety_Logic.WDT_PING
MCU_Peripherals.FIRE_EN ───────────────► Safety_Logic.FIRE_EN
MCU_Peripherals.FIRE_BTN ◄─────────────► Safety_Logic.FIRE_BTN
MCU_Peripherals.OC_FAULT ◄─────────────► Safety_Logic.OC_FAULT
MCU_Peripherals.I_SENSE ◄──────────────► Safety_Logic.I_SENSE
MCU_Peripherals.COIL_PWM ──────────────► Coil_Driver.COIL_PWM
MCU_Peripherals.BOOST_EN ──────────────► Boost_Converter.BOOST_EN

(I2C bus connected to all sheets that need it)
MCU_Peripherals.I2C1_SDA ◄────────────► Power_Management.I2C1_SDA
                         ◄────────────► Boost_Converter.I2C1_SDA
MCU_Peripherals.I2C1_SCL ─────────────► Power_Management.I2C1_SCL
                         ─────────────► Boost_Converter.I2C1_SCL

(USB signals)
MCU_Peripherals.USB_D+ ◄──────────────► Power_Management.USB_D+
MCU_Peripherals.USB_D- ◄──────────────► Power_Management.USB_D-

(Charger/Fuel Gauge signals)
MCU_Peripherals.CHG_INT ◄─────────────► Power_Management.CHG_INT
MCU_Peripherals.CHG_CE ───────────────► Power_Management.CHG_CE
MCU_Peripherals.CHG_OTG ──────────────► Power_Management.CHG_OTG
MCU_Peripherals.FG_ALRT ◄─────────────► Power_Management.FG_ALRT
```

---

## 4. Power Management Subsheet

### 4.1 Sheet Setup

1. Open `power_management.kicad_sch`
2. Add title block: "Power Management - USB, Charger, Protection, Fuel Gauge"
3. Add hierarchical labels matching root sheet pins

### 4.2 Hierarchical Labels Placement

Place these labels at the sheet edges:

**Left edge (Inputs):**
```
VBUS (Input) ─────────────────────────────────────────────────────►
USB_D+ (Bidirectional) ◄──────────────────────────────────────────►
USB_D- (Bidirectional) ◄──────────────────────────────────────────►
I2C1_SDA (Bidirectional) ◄────────────────────────────────────────►
I2C1_SCL (Input) ─────────────────────────────────────────────────►
CHG_CE (Input) ───────────────────────────────────────────────────►
CHG_OTG (Input) ──────────────────────────────────────────────────►
```

**Right edge (Outputs):**
```
◄─────────────────────────────────────────────────────── VSYS (Output)
◄─────────────────────────────────────────────────── CHG_INT (Output)
◄─────────────────────────────────────────────────── FG_ALRT (Output)
```

### 4.3 USB-C Connector and Protection

#### Components:

| Ref | Symbol | Value/Part | Footprint | Description |
|-----|--------|------------|-----------|-------------|
| J2 | USB_C_Receptacle_USB2.0 | USB4110-GF-A | USB_C_GCT_USB4110 | USB-C connector |
| D1 | D_Schottky | SS34 | SMA | Input protection diode |
| D2 | D_TVS | SMBJ15A | SMB | Transient voltage suppressor |
| D3 | TPD2E001 | TPD2E001DRLR | SOT-23 | USB ESD protection |
| R17 | R | 5.1k | 0402 | CC1 pull-down |
| R18 | R | 5.1k | 0402 | CC2 pull-down |

#### Wiring Diagram:

```
                                    VBUS hierarchical label
                                           ▲
                                           │
                    ┌──────────────────────┤
                    │                      │
              ┌─────┴─────┐          ┌─────┴─────┐
              │    D2     │          │    D1     │
              │  SMBJ15A  │          │   SS34    │
              │   (TVS)   │          │(Schottky) │
              └─────┬─────┘          └─────┬─────┘
                    │                      │
                    ▼ GND                  │
                                           │
    ┌─────────────────────────────────────────────────────────┐
    │                    USB-C CONNECTOR (J2)                  │
    │                                                          │
    │   VBUS (A4,A9,B4,B9) ────────────────────────────────────┤
    │                                                          │
    │   GND (A1,A12,B1,B12) ──────────────────────► GND        │
    │                                                          │
    │   CC1 (A5) ──────────┬───────────────────────────────────┤
    │                      │                                   │
    │                ┌─────┴─────┐                             │
    │                │   R17     │                             │
    │                │   5.1k    │                             │
    │                └─────┬─────┘                             │
    │                      ▼ GND                               │
    │                                                          │
    │   CC2 (B5) ──────────┬───────────────────────────────────┤
    │                      │                                   │
    │                ┌─────┴─────┐                             │
    │                │   R18     │                             │
    │                │   5.1k    │                             │
    │                └─────┬─────┘                             │
    │                      ▼ GND                               │
    │                                                          │
    │   D+ (A6,B6) ────────┬───────────────────────────────────┤
    │                      │      ┌─────────────┐              │
    │                      ├──────┤    D3       │              │
    │                      │      │ TPD2E001    ├──► USB_D+ label
    │   D- (A7,B7) ────────┼──────┤             │              │
    │                      │      │             ├──► USB_D- label
    │                      │      │     GND     │              │
    │                      │      └──────┬──────┘              │
    │                      │             ▼ GND                 │
    │                      │                                   │
    └──────────────────────────────────────────────────────────┘
```

### 4.4 BQ25896 Battery Charger

#### Components:

| Ref | Symbol | Value/Part | Footprint | Description |
|-----|--------|------------|-----------|-------------|
| U1 | BQ25896RTWT | BQ25896RTWT | QFN-24_4x4mm | Battery charger IC |
| L1 | L | 2.2uH/4A | 7x7mm | Charger inductor |
| C20 | C | 22uF/10V | 0805 | Input capacitor |
| C21 | C | 22uF/10V | 0805 | Output capacitor |
| C22 | C | 100nF | 0402 | BTST capacitor |
| C23 | C | 100nF | 0402 | REGN bypass |
| C24 | C | 1uF | 0402 | SYS decoupling |
| R5 | R | 2.4k | 0402 | ILIM resistor (sets 1.3A input limit) |
| R19 | R | 5.23k (use 5.1k) | 0402 | TS divider top (RT1) |
| R20 | R | 26.7k (use 27k) | 0402 | TS divider bottom (RT2) |

#### Wiring Diagram:

```
    VBUS (from USB protection)
         │
         │
    ┌────┴────────────────────────────────────────────────────────────────────┐
    │                                                                          │
    │    ┌──────┐                                                              │
    │    │ C20  │  22µF                                                        │
    │    └──┬───┘                                                              │
    │       │                                                                  │
    │       ▼ GND                                                              │
    │                                                                          │
    │                        ┌──────────────────────────────────────────┐      │
    │                        │              BQ25896 (U1)                │      │
    │                        │                                          │      │
    ├───────────────────────►│ VBUS (1) ──────────────────────────────  │      │
    │                        │                                          │      │
    │                        │ PMID (20) ───────────────► VSYS label    │      │
    │                        │                     │                    │      │
    │                        │                ┌────┴────┐               │      │
    │                        │                │  C24    │ 1µF           │      │
    │                        │                └────┬────┘               │      │
    │                        │                     ▼ GND                │      │
    │                        │                                          │      │
    │                        │ SYS (21) ──────────────────► (internal)  │      │
    │                        │                                          │      │
    │                        │ SW (14,15,16) ─────┬───────────────────  │      │
    │                        │                    │                     │      │
    │                        │               ┌────┴────┐                │      │
    │                        │               │   L1    │ 2.2µH          │      │
    │                        │               └────┬────┘                │      │
    │                        │                    │                     │      │
    │                        │ BAT (17) ◄─────────┴──────────────────►  │ ─────┼──► To BQ29705
    │                        │                    │                     │      │
    │                        │               ┌────┴────┐                │      │
    │                        │               │  C21    │ 22µF           │      │
    │                        │               └────┬────┘                │      │
    │                        │                    ▼ GND                 │      │
    │                        │                                          │      │
    │                        │ BTST (18) ─────────┬──────────────────   │      │
    │                        │               ┌────┴────┐                │      │
    │                        │               │  C22    │ 100nF          │      │
    │                        │               └────┬────┘                │      │
    │                        │                    │                     │      │
    │                        │                    └──────► SW node      │      │
    │                        │                                          │      │
    │                        │ REGN (19) ─────────┬──────────────────   │      │
    │                        │               ┌────┴────┐                │      │
    │                        │               │  C23    │ 100nF          │      │
    │                        │               └────┬────┘                │      │
    │                        │                    ▼ GND                 │      │
    │                        │                                          │      │
    │                        │ TS (9) ◄───────────┬──────────────────   │      │
    │                        │                    │                     │      │
    │                        │               ┌────┴────┐                │      │
    │                        │               │  R19    │ 5.1k (RT1)     │      │
    │                        │               └────┬────┘                │      │
    │                        │                    │                     │      │
    │                        │                    ├──────────────────►  │ ─────┼──► To J7 (NTC pads)
    │                        │                    │                     │      │
    │                        │               ┌────┴────┐                │      │
    │                        │               │  R20    │ 27k (RT2)      │      │
    │                        │               └────┬────┘                │      │
    │                        │                    ▼ GND                 │      │
    │                        │                                          │      │
    │                        │ ILIM (4) ──────────┬──────────────────   │      │
    │                        │               ┌────┴────┐                │      │
    │                        │               │   R5    │ 2.4k           │      │
    │                        │               └────┬────┘                │      │
    │                        │                    ▼ GND                 │      │
    │                        │                                          │      │
    │ I2C1_SDA label ◄──────►│ SDA (8)                                  │      │
    │                        │                                          │      │
    │ I2C1_SCL label ───────►│ SCL (7)                                  │      │
    │                        │                                          │      │
    │ CHG_INT label ◄────────│ INT (6)                                  │      │
    │                        │                                          │      │
    │ CHG_CE label ─────────►│ CE (5)                                   │      │
    │                        │                                          │      │
    │ CHG_OTG label ────────►│ OTG (3)                                  │      │
    │                        │                                          │      │
    │                        │ D+/D- (if used) ────────► (not used)     │      │
    │                        │                                          │      │
    │                        │ PGND (10,11,12,13) ─────► GND            │      │
    │                        │                                          │      │
    │                        └──────────────────────────────────────────┘      │
    │                                                                          │
    └──────────────────────────────────────────────────────────────────────────┘
```

### 4.5 BQ29705 Battery Protection

#### Components:

| Ref | Symbol | Value/Part | Footprint | Description |
|-----|--------|------------|-----------|-------------|
| U2 | BQ29705DSET | BQ29705DSET | BGA-5 | Battery protection IC |
| Q1,Q2 | CSD17571Q2 | CSD17571Q2 | VSON-8 | Dual N-ch protection MOSFETs |
| R1 | R | 330 | 0402 | BAT pin resistor |
| C1 | C | 100nF | 0402 | BAT pin filter |
| R2 | R | 2.2k | 0402 | VSS sense resistor |

#### Wiring Diagram:

```
    VBAT+ (from BQ25896 BAT pin)
         │
         ├──────────────────────────────────────────────────────────────────┐
         │                                                                   │
    ┌────┴────┐                                                              │
    │   R1    │ 330Ω                                                         │
    └────┬────┘                                                              │
         │                                                                   │
         ├───────────────────────────────────────────────┐                   │
         │                                               │                   │
    ┌────┴────┐                                          │                   │
    │   C1    │ 100nF                                    │                   │
    └────┬────┘                                          │                   │
         ▼ GND                                           │                   │
                                                         │                   │
                        ┌────────────────────────────────┤                   │
                        │        BQ29705 (U2)            │                   │
                        │                                │                   │
                        │ BAT (4) ◄───────────────────────┘                   │
                        │                                                    │
                        │ COUT (3) ──────────────────────┬──────────────────┤
                        │                                │                   │
                        │                           ┌────┴────┐              │
                        │                           │  Q1     │ CSD17571Q2   │
                        │                           │ (CHG)   │ (charge FET) │
                        │                           └────┬────┘              │
                        │                                │                   │
                        │                           ┌────┴────┐              │
                        │                           │  Q2     │ CSD17571Q2   │
                        │                           │ (DIS)   │ (discharge)  │
                        │                           └────┬────┘              │
                        │                                │                   │
                        │ DOUT (2) ──────────────────────┘                   │
                        │                                │                   │
                        │ VSS (1) ───────────────────────┼──────────────────┤
                        │               │                │                   │
                        │          ┌────┴────┐           │                   │
                        │          │   R2    │ 2.2k      │                   │
                        │          └────┬────┘           │                   │
                        │               │                │                   │
                        │               ▼ GND            │                   │
                        │                                │                   │
                        │ VC (5) ────────► VCC (3.3V)    │                   │
                        │                                │                   │
                        └────────────────────────────────┘                   │
                                                                             │
                                                                             │
    To Battery Pads (J3) ◄───────────────────────────────────────────────────┘
         │
         │  BAT+ pad ────► VBAT+ (protected, to BQ25896)
         │  BAT- pad ────► GND (through protection MOSFETs)
         │
         └── Use 14 AWG wire, solder directly to pads
```

### 4.6 MAX17055 Fuel Gauge

#### Components:

| Ref | Symbol | Value/Part | Footprint | Description |
|-----|--------|------------|-----------|-------------|
| U3 | MAX17055EWL+T | MAX17055EWL+T | WLP-9 | Fuel gauge IC |
| R_SENSE_FG | R | 5m | 2512 | System current sense (separate from coil shunt) |
| C25 | C | 100nF | 0402 | VDD bypass |

#### Wiring Diagram:

```
    VSYS (from BQ25896 PMID)
         │
         ├───────────────────────────────────────────────────────────────────┐
         │                                                                    │
    ┌────┴────┐                                                               │
    │  C25    │ 100nF                                                         │
    └────┬────┘                                                               │
         ▼ GND                                                                │
                                                                              │
                        ┌────────────────────────────────────────────────┐    │
                        │              MAX17055 (U3)                     │    │
                        │                                                │    │
         ───────────────┤ VDD (1) ◄──────────────────────────────────────┼────┘
                        │                                                │
    I2C1_SDA label ◄───►│ SDA (3)                                        │
                        │                                                │
    I2C1_SCL label ────►│ SCL (2)                                        │
                        │                                                │
    FG_ALRT label ◄─────│ ALRT (8)                                       │
                        │                                                │
                        │ CSP (5) ──────────────────┬────────────────────┤
                        │                           │                    │
                        │                      ┌────┴────┐               │
                        │                      │R_SENSE  │ 5mΩ           │
                        │                      │  (FG)   │ (2512, 1W)    │
                        │                      └────┬────┘               │
                        │                           │                    │
                        │ CSN (6) ──────────────────┤                    │
                        │                           │                    │
                        │                           └─────────► Load path│
                        │                                                │
                        │ THRM (4) ─────────────────► (optional 2nd NTC) │
                        │                                                │
                        │ GND (7) ──────────────────► GND                │
                        │                                                │
                        └────────────────────────────────────────────────┘
```

### 4.7 NTC Thermistor Connection (J7)

#### Components:

| Ref | Symbol | Value/Part | Footprint | Description |
|-----|--------|------------|-----------|-------------|
| J7 | Conn_01x02 | Solder_Pads | Custom 2-pad | NTC thermistor pads |

#### Wiring Diagram:

```
    REGN (3.3V from BQ25896)
         │
    ┌────┴────┐
    │  R19    │ 5.1k (RT1)
    └────┬────┘
         │
         ├───────────────────────────────────────────► BQ25896 TS pin (9)
         │
    ┌────┴────┐
    │  J7     │ TS pad ◄──── NTC connects here (10K, B=3435)
    │         │
    │         │ GND pad ◄── NTC other terminal
    └────┬────┘
         │
    ┌────┴────┐
    │  R20    │ 27k (RT2)
    └────┬────┘
         ▼ GND

    Note: NTC thermistor (10K @ 25°C, B=3435) is soldered
    to J7 pads and attached to battery with Kapton tape.
```

### 4.8 Battery Connection Pads (J3)

#### Components:

| Ref | Symbol | Value/Part | Footprint | Description |
|-----|--------|------------|-----------|-------------|
| J3 | Conn_01x02 | Solder_Pads | Custom 2-pad (2.5mm hole) | Battery solder pads |

#### Wiring:

```
    ┌─────────────────────────────────────────────────────────┐
    │                     J3 - BATTERY PADS                    │
    │                                                          │
    │   BAT+ pad ────────────────────────► VBAT+ (to BQ29705)  │
    │   (Label: "BAT+")                                        │
    │                                                          │
    │   BAT- pad ────────────────────────► GND (through Q1/Q2) │
    │   (Label: "BAT-")                                        │
    │                                                          │
    │   Use 14 AWG silicone wire to battery                    │
    │   Turnigy 1000mAh 1S 20C LiPo                           │
    └─────────────────────────────────────────────────────────┘
```

---

## 5. Boost Converter Subsheet

### 5.1 Hierarchical Labels Placement

**Left edge (Inputs):**
```
VSYS (Input) ─────────────────────────────────────────────────────►
BOOST_EN (Input) ─────────────────────────────────────────────────►
I2C1_SDA (Bidirectional) ◄────────────────────────────────────────►
I2C1_SCL (Input) ─────────────────────────────────────────────────►
```

**Right edge (Outputs):**
```
◄───────────────────────────────────────────────────── V_BOOST (Output)
```

### 5.2 TPS61088 Boost Converter

#### Components:

| Ref | Symbol | Value/Part | Footprint | Description |
|-----|--------|------------|-----------|-------------|
| U4 | TPS61088RHLR | TPS61088RHLR | QFN-20_4x4mm | Boost converter IC |
| L2 | L | 1uH/15A | 10x10mm | Boost inductor (15A saturation!) |
| C30 | C | 22uF/10V | 0805 | Input capacitor |
| C31 | C | 22uF/10V | 0805 | Input capacitor |
| C32 | C | 22uF/10V | 0805 | Output capacitor |
| C33 | C | 22uF/10V | 0805 | Output capacitor |
| C16 | C | 100uF/10V | 1206 | Bulk output capacitor |
| C17 | C | 22nF | 0402 | Soft-start capacitor |
| C34 | C | 100nF | 0402 | VIN bypass |
| R4 | R | 100k | 0402 | FB top resistor |
| R_COMP | R | 10k | 0402 | Compensation resistor |
| C_COMP | C | 100pF | 0402 | Compensation capacitor |

#### Wiring Diagram:

```
    VSYS (from Power Management)
         │
         ├──────────────────────────────────────────────────────────────────┐
         │                                                                   │
    ┌────┴────┐  ┌────────┐                                                  │
    │  C30    │  │  C31   │  22µF each                                       │
    │  22µF   │  │  22µF  │                                                  │
    └────┬────┘  └────┬───┘                                                  │
         │            │                                                      │
         ▼ GND        ▼ GND                                                  │
                                                                             │
    ┌────────┐                                                               │
    │  C34   │ 100nF (close to VIN pins)                                     │
    └────┬───┘                                                               │
         ▼ GND                                                               │
                                                                             │
                        ┌────────────────────────────────────────────────┐   │
                        │              TPS61088 (U4)                     │   │
                        │                                                │   │
         ───────────────┤ VIN (3,4) ◄────────────────────────────────────┼───┘
                        │                                                │
    BOOST_EN label ────►│ EN (1)                                         │
                        │                                                │
                        │ MODE (7) ────────────────────────► GND         │
                        │              (PFM mode for light load)         │
                        │                                                │
                        │ SS (6) ──────────────┬─────────────────────    │
                        │                 ┌────┴────┐                    │
                        │                 │  C17    │ 22nF               │
                        │                 └────┬────┘                    │
                        │                      ▼ GND                     │
                        │                                                │
                        │ SW (13,14,15,16) ────┬─────────────────────    │
                        │                      │                         │
                        │                 ┌────┴────┐                    │
                        │                 │   L2    │ 1µH, 15A sat       │
                        │                 └────┬────┘                    │
                        │                      │                         │
                        │ VOUT (17,18) ◄───────┴─────────────────────────┼──► V_BOOST label
                        │                      │                         │
                        │                 ┌────┴────┐  ┌────────┐        │
                        │                 │  C32    │  │  C33   │ 22µF   │
                        │                 └────┬────┘  └────┬───┘        │
                        │                      ▼ GND        ▼ GND        │
                        │                                                │
                        │                 ┌────────┐                     │
                        │                 │  C16   │ 100µF bulk          │
                        │                 └────┬───┘                     │
                        │                      ▼ GND                     │
                        │                                                │
                        │ FB (8) ◄─────────────┬─────────────────────    │
                        │                      │                         │
                        │                      │ (see feedback network)  │
                        │                      │                         │
                        │ COMP (9) ────────────┼─────────────────────    │
                        │           │          │                         │
                        │      ┌────┴────┐     │                         │
                        │      │ R_COMP  │ 10k │                         │
                        │      └────┬────┘     │                         │
                        │           │          │                         │
                        │      ┌────┴────┐     │                         │
                        │      │ C_COMP  │ 100pF                         │
                        │      └────┬────┘     │                         │
                        │           ▼ GND      │                         │
                        │                      │                         │
                        │ PGND (10,11,12) ─────┴───────► GND             │
                        │                                                │
                        │ EP (thermal pad) ────────────► GND (via farm)  │
                        │                                                │
                        └────────────────────────────────────────────────┘
```

### 5.3 Feedback Network with MCP4561 Digital Potentiometer

#### Components:

| Ref | Symbol | Value/Part | Footprint | Description |
|-----|--------|------------|-----------|-------------|
| U11 | MCP4561-104E/MS | MCP4561-104E/MS | MSOP-8 | 100k digital pot |
| R4 | R | 100k | 0402 | FB top resistor (fixed) |
| C35 | C | 100nF | 0402 | MCP4561 bypass |

**CRITICAL: Use MCP4561-104 (100k), NOT MCP4561-103 (10k)!**

#### Wiring Diagram:

```
    V_BOOST
         │
    ┌────┴────┐
    │   R4    │ 100kΩ (fixed top resistor)
    └────┬────┘
         │
         ├───────────────────────────────────────────────► TPS61088 FB pin
         │
         │
         │      ┌────────────────────────────────────────┐
         │      │           MCP4561 (U11)                │
         │      │            100kΩ                       │
         │      │                                        │
         └─────►│ P0A (5) ◄────────────────────────────  │ (wiper high side)
                │                                        │
                │ P0W (6) ────────────────────► FB node  │ (wiper output)
                │                                        │
                │ P0B (7) ────────────────────► GND      │ (wiper low side)
                │                                        │
    I2C1_SDA ◄─►│ SDA (3)                                │
                │                                        │
    I2C1_SCL ──►│ SCL (2)                                │
                │                                        │
                │ A0 (1) ─────────────────────► GND      │ (address = 0x2E)
                │                                        │
                │ VDD (8) ────────────────────► +3V3     │
                │         │                              │
                │    ┌────┴────┐                         │
                │    │  C35    │ 100nF                   │
                │    └────┬────┘                         │
                │         ▼ GND                          │
                │                                        │
                │ VSS (4) ────────────────────► GND      │
                │                                        │
                └────────────────────────────────────────┘

    Voltage calculation:
    V_OUT = 1.212V × (1 + R4 / R_POT)

    At R_POT = 100kΩ (max): V_OUT = 2.42V (pass-through mode)
    At R_POT = 21kΩ:        V_OUT = 6.99V (max boost)
```

---

## 6. Safety Logic Subsheet

### 6.1 Hierarchical Labels Placement

**Left edge (Inputs):**
```
V_BOOST (Input) ──────────────────────────────────────────────────►
WDT_PING (Input) ─────────────────────────────────────────────────►
FIRE_EN (Input) ──────────────────────────────────────────────────►
FIRE_BTN (Input) ─────────────────────────────────────────────────►
COIL_SENSE (Input) ───────────────────────────────────────────────►
```

**Right edge (Outputs):**
```
◄─────────────────────────────────────────────────── SAFETY_GATE (Output)
◄─────────────────────────────────────────────────── OC_FAULT (Output)
◄─────────────────────────────────────────────────── I_SENSE (Output)
```

### 6.2 555 Timer Watchdog

#### Components:

| Ref | Symbol | Value/Part | Footprint | Description |
|-----|--------|------------|-----------|-------------|
| U6 | NE555DR | NE555DR | SOIC-8 | 555 timer IC |
| U12 | 74LVC1G17GW | 74LVC1G17GW | SOT-23-5 | Schmitt trigger buffer |
| R3 | R | 150k | 0603 | Timing resistor |
| R12 | R | 10k | 0402 | RST pull-up |
| C15 | C | 10uF | 0805 | Timing capacitor |
| C40 | C | 100nF | 0402 | Control voltage bypass |
| C41 | C | 100nF | 0402 | VCC bypass |

**Timing: T = 1.1 × R × C = 1.1 × 150k × 10µF = 1.65 seconds**

#### Wiring Diagram:

```
                                              +3V3
                                               │
                           ┌───────────────────┼───────────────────────────────┐
                           │                   │                                │
                           │              ┌────┴────┐                           │
                           │              │  C41    │ 100nF (VCC bypass)        │
                           │              └────┬────┘                           │
                           │                   ▼ GND                            │
                           │                                                    │
                           │              ┌────┴────┐                           │
                           │              │  R12    │ 10kΩ (RST pull-up)        │
                           │              └────┬────┘                           │
                           │                   │                                │
                           │                   │                                │
    ┌──────────────────────┼───────────────────┼────────────────────────────────┤
    │                      │                   │                                │
    │                 ┌────┴────────────────────────────────┐                   │
    │                 │              NE555 (U6)             │                   │
    │                 │                                     │                   │
    │                 │ VCC (8) ◄───────────────────────────┼───────── +3V3     │
    │                 │                                     │                   │
    │                 │ RST (4) ◄───────────────────────────┤                   │
    │                 │                                     │                   │
    │                 │ DIS (7) ────────────┬───────────────┤                   │
    │                 │                     │               │                   │
    │                 │                ┌────┴────┐          │                   │
    │                 │                │   R3    │ 150kΩ    │                   │
    │                 │                └────┬────┘          │                   │
    │                 │                     │               │                   │
    │                 │ THR (6) ◄───────────┤               │                   │
    │                 │                     │               │                   │
    │                 │                ┌────┴────┐          │                   │
    │                 │                │  C15    │ 10µF     │                   │
    │                 │                └────┬────┘          │                   │
    │                 │                     ▼ GND           │                   │
    │                 │                                     │                   │
    │ WDT_PING ──────►│ TRIG (2)                            │                   │
    │ (from ESP32)    │    (active LOW trigger)             │                   │
    │                 │                                     │                   │
    │                 │ CV (5) ─────────────┬───────────────│                   │
    │                 │                ┌────┴────┐          │                   │
    │                 │                │  C40    │ 100nF    │                   │
    │                 │                └────┬────┘          │                   │
    │                 │                     ▼ GND           │                   │
    │                 │                                     │                   │
    │                 │ OUT (3) ────────────────────────────┼───► To Schmitt    │
    │                 │                                     │                   │
    │                 │ GND (1) ────────────────────────────┼───► GND           │
    │                 │                                     │                   │
    │                 └─────────────────────────────────────┘                   │
    │                                                                           │
    │                                                                           │
    │                 ┌─────────────────────────────────────┐                   │
    │                 │         74LVC1G17 (U12)             │                   │
    │                 │        (Schmitt Buffer)             │                   │
    │                 │                                     │                   │
    │   555 OUT ─────►│ A (2)                               │                   │
    │                 │                                     │                   │
    │                 │ Y (4) ──────────────────────────────┼───► WDT_OK        │
    │                 │                                     │     (to AND gate) │
    │                 │ VCC (5) ────────────────────────────┼───► +3V3          │
    │                 │                                     │                   │
    │                 │ GND (3) ────────────────────────────┼───► GND           │
    │                 │                                     │                   │
    │                 └─────────────────────────────────────┘                   │
    │                                                                           │
    └───────────────────────────────────────────────────────────────────────────┘
```

### 6.3 Overcurrent Detection (INA180 + LM393)

#### Components:

| Ref | Symbol | Value/Part | Footprint | Description |
|-----|--------|------------|-----------|-------------|
| U8 | INA180A2IDBVR | INA180A2IDBVR | SOT-23-5 | Current sense amp (50V/V) |
| U9 | LM393DR | LM393DR | SOIC-8 | Comparator |
| R6 | R | 100 | 0402 | Filter resistor |
| R7 | R | 1k | 0402 | Threshold divider top |
| R8 | R | 3.3k | 0402 | Threshold divider bottom |
| R9 | R | 1M | 0603 | Hysteresis resistor |
| R10 | R | 10k | 0402 | LM393 pull-up (REQUIRED!) |
| C18 | C | 10nF | 0402 | Filter capacitor |
| C42 | C | 100nF | 0402 | INA180 bypass |

**Threshold calculation:**
- V_THRESH = 3.3V × 3.3k / (1k + 3.3k) = 2.53V
- At 50V/V gain with 10mΩ shunt: trips at 5A

#### Wiring Diagram:

```
    COIL_SENSE (from shunt resistor high side)
         │
         │      ┌────────────────────────────────────────────┐
         │      │           INA180A2 (U8)                    │
         │      │            50V/V gain                      │
         │      │                                            │
         └─────►│ IN+ (1) ◄────────────────────────────────  │
                │                                            │
    GND ───────►│ IN- (2)                                    │
    (shunt low) │                                            │
                │ OUT (4) ─────────────────────────────────  │──► I_SENSE label
                │                    │                       │    (to ESP32 ADC)
                │               ┌────┴────┐                  │
                │               │   R6    │ 100Ω (filter)    │
                │               └────┬────┘                  │
                │                    │                       │
                │               ┌────┴────┐                  │
                │               │  C18    │ 10nF (filter)    │
                │               └────┬────┘                  │
                │                    │                       │
                │                    ▼ to comparator         │
                │                                            │
                │ VCC (5) ─────────────────────► +3V3        │
                │         │                                  │
                │    ┌────┴────┐                             │
                │    │  C42    │ 100nF                       │
                │    └────┬────┘                             │
                │         ▼ GND                              │
                │                                            │
                │ GND (3) ─────────────────────► GND         │
                │                                            │
                └────────────────────────────────────────────┘
                                     │
                                     │ (filtered I_SENSE)
                                     ▼
                ┌────────────────────────────────────────────┐
                │              LM393 (U9)                    │
                │            (Comparator)                    │
                │                                            │
                │                                            │
                │ IN+ (3) ◄───────────────────────────────   │ ◄── filtered I_SENSE
                │           │                                │
                │           │                                │
                │      ┌────┴────┐                           │
                │      │   R9    │ 1MΩ (hysteresis)          │
                │      └────┬────┘                           │
                │           │                                │
                │           └───────────────────┬────────────│
                │                               │            │
                │ OUT (1) ──────────────────────┴────────────│──► !OC_FAULT
                │           │                                │    (active LOW)
                │      ┌────┴────┐                           │
                │      │  R10    │ 10kΩ PULL-UP (REQUIRED!)  │
                │      └────┬────┘                           │
                │           │                                │
                │           └────────────────────► +3V3      │
                │                                            │
                │ IN- (2) ◄──────────────────────────────────│ ◄── V_THRESH
                │                                            │
                │                                            │
                │ VCC (8) ─────────────────────► +3V3        │
                │                                            │
                │ GND (4) ─────────────────────► GND         │
                │                                            │
                └────────────────────────────────────────────┘

    Threshold Voltage Divider:

         +3V3
          │
     ┌────┴────┐
     │   R7    │ 1kΩ
     └────┬────┘
          │
          ├────────────────────────────────────► LM393 IN- (2)
          │
     ┌────┴────┐
     │   R8    │ 3.3kΩ
     └────┬────┘
          ▼ GND

    V_THRESH = 3.3V × 3.3k / (1k + 3.3k) = 2.53V
    Trip current = 2.53V / 50 / 0.010Ω = 5.06A
```

### 6.4 Safety AND Gate (74LVC08A)

#### Components:

| Ref | Symbol | Value/Part | Footprint | Description |
|-----|--------|------------|-----------|-------------|
| U7 | 74LVC08APW | 74LVC08APW | TSSOP-14 | Quad 2-input AND gate |
| R11 | R | 10k | 0402 | FIRE_EN pull-down (CRITICAL!) |
| C43 | C | 100nF | 0402 | VCC bypass |

#### Wiring Diagram:

```
                                              +3V3
                                               │
                           ┌───────────────────┼──────────────────────────────┐
                           │                   │                               │
                           │              ┌────┴────┐                          │
                           │              │  C43    │ 100nF                    │
                           │              └────┬────┘                          │
                           │                   ▼ GND                           │
                           │                                                   │
    ┌──────────────────────┼───────────────────────────────────────────────────┤
    │                      │                                                   │
    │                 ┌────┴────────────────────────────────────────┐          │
    │                 │              74LVC08A (U7)                  │          │
    │                 │          (Quad 2-Input AND)                 │          │
    │                 │                                             │          │
    │                 │ VCC (14) ◄──────────────────────────────────┼── +3V3   │
    │                 │                                             │          │
    │                 │ GND (7) ────────────────────────────────────┼── GND    │
    │                 │                                             │          │
    │                 │                                             │          │
    │                 │        GATE 1 (WDT × FIRE_BTN)              │          │
    │                 │                                             │          │
    │ WDT_OK ────────►│ 1A (1)                                      │          │
    │ (from 74LVC1G17)│                                             │          │
    │                 │ 1Y (3) ────────────────────────► STAGE1     │          │
    │                 │                                             │          │
    │ !FIRE_BTN ─────►│ 1B (2)                                      │          │
    │ (active HIGH    │     (button pressed = HIGH)                 │          │
    │  when pressed)  │                                             │          │
    │                 │                                             │          │
    │                 │        GATE 2 (STAGE1 × !OC_FAULT)          │          │
    │                 │                                             │          │
    │ STAGE1 ────────►│ 2A (4)                                      │          │
    │                 │                                             │          │
    │                 │ 2Y (6) ────────────────────────► STAGE2     │          │
    │                 │                                             │          │
    │ !OC_FAULT ─────►│ 2B (5)                                      │          │
    │ (from LM393,    │     (no fault = HIGH)                       │          │
    │  with pull-up)  │                                             │          │
    │                 │                                             │          │
    │                 │        GATE 3 (STAGE2 × FIRE_EN)            │          │
    │                 │                                             │          │
    │ STAGE2 ────────►│ 3A (9)                                      │          │
    │                 │                                             │          │
    │                 │ 3Y (8) ────────────────────────► SAFETY_GATE│──► label │
    │                 │                                             │          │
    │ FIRE_EN ───────►│ 3B (10)                                     │          │
    │ (from ESP32)    │     │                                       │          │
    │                 │     │                                       │          │
    │                 │ ┌───┴───┐                                   │          │
    │                 │ │  R11  │ 10kΩ PULL-DOWN (CRITICAL!)        │          │
    │                 │ └───┬───┘                                   │          │
    │                 │     ▼ GND                                   │          │
    │                 │                                             │          │
    │                 │        GATE 4 (unused - tie inputs)         │          │
    │                 │                                             │          │
    │                 │ 4A (12) ────────────────────────► GND       │          │
    │                 │ 4B (13) ────────────────────────► GND       │          │
    │                 │ 4Y (11) ────────────────────────► NC        │          │
    │                 │                                             │          │
    │                 └─────────────────────────────────────────────┘          │
    │                                                                          │
    └──────────────────────────────────────────────────────────────────────────┘

    TRUTH TABLE:
    ┌──────────┬────────────┬──────────┬──────────┬─────────────┐
    │ WDT_OK   │ !FIRE_BTN  │ !OC_FAULT│ FIRE_EN  │ SAFETY_GATE │
    ├──────────┼────────────┼──────────┼──────────┼─────────────┤
    │    0     │     X      │    X     │    X     │      0      │
    │    1     │     0      │    X     │    X     │      0      │
    │    1     │     1      │    0     │    X     │      0      │
    │    1     │     1      │    1     │    0     │      0      │
    │    1     │     1      │    1     │    1     │      1      │ ◄── FIRES
    └──────────┴────────────┴──────────┴──────────┴─────────────┘
```

---

## 7. Coil Driver Subsheet

### 7.1 Hierarchical Labels Placement

**Left edge (Inputs):**
```
V_BOOST (Input) ──────────────────────────────────────────────────►
SAFETY_GATE (Input) ──────────────────────────────────────────────►
COIL_PWM (Input) ─────────────────────────────────────────────────►
```

**Right edge (Outputs):**
```
◄───────────────────────────────────────────────────── COIL_SENSE (Output)
```

### 7.2 TC4426 Gate Driver

#### Components:

| Ref | Symbol | Value/Part | Footprint | Description |
|-----|--------|------------|-----------|-------------|
| U10 | TC4426ACOA | TC4426ACOA | SOIC-8 | Dual INVERTING MOSFET driver |
| C50 | C | 100nF | 0402 | VDD bypass (close to IC) |
| C51 | C | 1uF | 0603 | VDD bulk |

**CRITICAL: TC4426 is INVERTING! When SAFETY_GATE=HIGH, output=LOW, PMOS turns ON.**

#### Wiring Diagram:

```
    V_BOOST (typically 3-7V)
         │
         ├───────────────────────────────────────────────────────────────────┐
         │                                                                    │
    ┌────┴────┐  ┌────────┐                                                   │
    │  C50    │  │  C51   │                                                   │
    │  100nF  │  │  1µF   │ (both close to VDD pin)                           │
    └────┬────┘  └────┬───┘                                                   │
         │            │                                                       │
         ▼ GND        ▼ GND                                                   │
                                                                              │
                        ┌────────────────────────────────────────────────┐    │
                        │              TC4426 (U10)                      │    │
                        │        (Dual INVERTING Driver)                 │    │
                        │                                                │    │
         ───────────────┤ VDD (6) ◄──────────────────────────────────────┼────┘
                        │                                                │
    SAFETY_GATE ───────►│ IN_A (2)                                       │
    (from AND gate)     │                                                │
                        │ OUT_A (7) ─────────────────────────────────────┼──► Q3_GATE
                        │                                                │    (to PMOS gate)
                        │                                                │
    COIL_PWM ──────────►│ IN_B (4)                                       │
    (from ESP32)        │                                                │
                        │ OUT_B (5) ─────────────────────────────────────┼──► Q4_GATE
                        │                                                │    (to NMOS gate)
                        │                                                │
                        │ GND (3) ───────────────────────────────────────┼──► GND
                        │                                                │
                        └────────────────────────────────────────────────┘

    INVERTING BEHAVIOR:
    ┌─────────────┬───────────┬────────────┐
    │ SAFETY_GATE │ OUT_A     │ Q3 (PMOS)  │
    ├─────────────┼───────────┼────────────┤
    │     LOW     │ HIGH(Vboost)│   OFF    │ ← Safe state
    │     HIGH    │ LOW (0V)  │   ON       │ ← Fires coil
    └─────────────┴───────────┴────────────┘
```

### 7.3 High-Side PMOS Switch (Q3)

#### Components:

| Ref | Symbol | Value/Part | Footprint | Description |
|-----|--------|------------|-----------|-------------|
| Q3 | Si7461DP-T1-GE3 | Si7461DP-T1-GE3 | PowerPAK_SO-8 | P-ch MOSFET (-60V/-8.6A) |
| R_GATE3 | R | 10 | 0402 | Gate resistor (optional, prevents ringing) |

#### Wiring Diagram:

```
    V_BOOST
         │
         │
    ┌────┴────┐
    │   Q3    │ Si7461DP (P-channel MOSFET)
    │  PMOS   │
    │         │
    │    S ◄──┼─────────────────────────────────── V_BOOST (Source)
    │         │
    │    G ◄──┼──────┬──────────────────────────── Q3_GATE (from TC4426)
    │         │      │
    │         │ ┌────┴────┐
    │         │ │R_GATE3  │ 10Ω (optional, damps ringing)
    │         │ └────┬────┘
    │         │      │
    │         │      └─────► TC4426 OUT_A
    │         │
    │    D ───┼─────────────────────────────────── COIL_HIGH (to shunt)
    │         │
    └─────────┘

    Operation:
    - When Q3_GATE = V_BOOST: Vgs = 0, PMOS OFF (safe)
    - When Q3_GATE = 0V: Vgs = -V_BOOST, PMOS ON (current flows)
```

### 7.4 Current Sense Shunt

#### Components:

| Ref | Symbol | Value/Part | Footprint | Description |
|-----|--------|------------|-----------|-------------|
| R_SHUNT | R | 10m | 2512_3W | Current sense resistor (10mΩ, 3W) |

#### Wiring Diagram:

```
    Q3 Drain (COIL_HIGH)
         │
         │
    ┌────┴────┐
    │ R_SHUNT │ 10mΩ, 3W, 2512 package
    │  10mΩ   │
    └────┬────┘
         │
         ├──────────────────────────────────────► COIL_SENSE label
         │                                        (to INA180 IN+)
         │
    ┌────┴────┐
    │  510    │ 510 Connector interface (to XT30U J6)
    │ CENTER  │
    └────┬────┘
         │
         ▼ to Q4 Drain (low-side switch)
```

### 7.5 Low-Side NMOS Switch (Q4)

#### Components:

| Ref | Symbol | Value/Part | Footprint | Description |
|-----|--------|------------|-----------|-------------|
| Q4 | CSD18563Q5A | CSD18563Q5A | VSON-8 | N-ch MOSFET (60V/100A) |
| R_GATE4 | R | 10 | 0402 | Gate resistor (optional) |

#### Wiring Diagram:

```
    From 510 Connector (J6)
         │
         │
    ┌────┴────┐
    │   Q4    │ CSD18563Q5A (N-channel MOSFET)
    │  NMOS   │
    │         │
    │    D ◄──┼─────────────────────────────────── From 510 connector
    │         │
    │    G ◄──┼──────┬──────────────────────────── Q4_GATE (from TC4426)
    │         │      │
    │         │ ┌────┴────┐
    │         │ │R_GATE4  │ 10Ω (optional)
    │         │ └────┬────┘
    │         │      │
    │         │      └─────► TC4426 OUT_B
    │         │
    │    S ───┼─────────────────────────────────── GND (star ground!)
    │         │
    └─────────┘

    Operation:
    - PWM on Q4_GATE controls average power to coil
    - Q3 must be ON (SAFETY_GATE=HIGH) for current to flow
```

### 7.6 510 Connector Interface (XT30U)

#### Components:

| Ref | Symbol | Value/Part | Footprint | Description |
|-----|--------|------------|-----------|-------------|
| J6 | XT30U-M | XT30U-M | XT30U_Male | 510 connector interface (30A rated) |
| R21 | R | 1M | 0603 | ESD bleed resistor (optional) |

#### Wiring Diagram:

```
    ┌─────────────────────────────────────────────────────────────────────────┐
    │                           XT30U-M (J6)                                   │
    │                      (Male connector on PCB)                             │
    │                                                                          │
    │   Pin 1 (+) ◄────────────────────────────── From R_SHUNT low side        │
    │              │                               (COIL output)               │
    │              │                                                           │
    │         ┌────┴────┐                                                      │
    │         │  R21    │ 1MΩ (optional ESD bleed)                             │
    │         └────┬────┘                                                      │
    │              │                                                           │
    │              ▼ GND                                                       │
    │                                                                          │
    │   Pin 2 (-) ◄────────────────────────────── From Q4 Drain                │
    │                                              (510 return)                │
    │                                                                          │
    └─────────────────────────────────────────────────────────────────────────┘

    External Wiring:
    - XT30U-F (female) connector with 14 AWG wire
    - Connect to MM510 V2 connector:
      - XT30U (+) → 510 center pin
      - XT30U (-) → 510 threads/shell

    ISOLATION WARNING:
    - 510 shell floats to V_BOOST when Q4 OFF
    - Use PEEK insulator between 510 and chassis!
```

### 7.7 Complete Coil Driver Circuit

```
    V_BOOST ─────────────────────────────────────────────────────────────────┐
         │                                                                    │
         │                                                                    │
         │    ┌─────────────────────────────────────────────────────────┐    │
         │    │                                                         │    │
         │    │      ┌───────────────┐                                  │    │
         │    │      │   Q3 (PMOS)   │ Si7461DP                         │    │
         │    │      │               │                                  │    │
         │    └─────►│ S          D  │───────┬──────────────────────────│    │
         │           │               │       │                          │    │
         │           │       G       │       │  COIL_HIGH               │    │
         │           └───────┬───────┘       │                          │    │
         │                   │               │                          │    │
         │                   │          ┌────┴────┐                     │    │
         │       ┌───────────┴──────┐   │ R_SHUNT │ 10mΩ                │    │
         │       │                  │   └────┬────┘                     │    │
         │       │   TC4426 OUT_A   │        │                          │    │
         │       │                  │        ├──────► COIL_SENSE label  │    │
         │       │                  │        │                          │    │
         │       └──────────────────┘   ┌────┴────┐                     │    │
         │                              │  J6     │ XT30U (+)           │    │
         │                              │ (XT30U) │                     │    │
         │                              └────┬────┘                     │    │
         │                                   │                          │    │
         │                                   │  To 510 via wire         │    │
         │                                   │                          │    │
         │                              ┌────┴────┐                     │    │
         │                              │  J6     │ XT30U (-)           │    │
         │                              │ (XT30U) │                     │    │
         │                              └────┬────┘                     │    │
         │                                   │                          │    │
         │                              ┌────┴────┐                     │    │
         │                              │   Q4    │ CSD18563Q5A         │    │
         │                              │  NMOS   │                     │    │
         │                              │    D    │                     │    │
         │       ┌───────────────┐      │         │                     │    │
         │       │               │      │    G ◄──┼──── TC4426 OUT_B    │    │
         │       │ TC4426 (U10)  │      │         │     (PWM)           │    │
         │       │               │      │    S ───┼──── GND (star)      │    │
         │       │  VDD ◄────────┼──────┘         │                     │    │
         │       │               │      └─────────┘                     │    │
         │       │ IN_A ◄────────┼──────────────────── SAFETY_GATE label│    │
         │       │               │                                      │    │
         │       │ OUT_A ────────┼──────────────────── Q3 Gate          │    │
         │       │               │                                      │    │
         │       │ IN_B ◄────────┼──────────────────── COIL_PWM label   │    │
         │       │               │                                      │    │
         │       │ OUT_B ────────┼──────────────────── Q4 Gate          │    │
         │       │               │                                      │    │
         │       │ GND ──────────┼──────────────────── GND              │    │
         │       │               │                                      │    │
         │       └───────────────┘                                      │    │
         │                                                              │    │
         └──────────────────────────────────────────────────────────────┘────┘
```

---

## 8. MCU Peripherals Subsheet

### 8.1 Hierarchical Labels Placement

**Left edge (Power/Control Inputs):**
```
VSYS (Input) ──────────────────────────────────────────────────────►
USB_D+ (Bidirectional) ◄──────────────────────────────────────────►
USB_D- (Bidirectional) ◄──────────────────────────────────────────►
OC_FAULT (Input) ─────────────────────────────────────────────────►
I_SENSE (Input) ──────────────────────────────────────────────────►
FIRE_BTN (Input) ─────────────────────────────────────────────────►
CHG_INT (Input) ──────────────────────────────────────────────────►
FG_ALRT (Input) ──────────────────────────────────────────────────►
```

**Right edge (Control Outputs):**
```
◄─────────────────────────────────────────────────────── WDT_PING (Output)
◄─────────────────────────────────────────────────────── FIRE_EN (Output)
◄─────────────────────────────────────────────────────── COIL_PWM (Output)
◄─────────────────────────────────────────────────────── BOOST_EN (Output)
◄─────────────────────────────────────────────────────── CHG_CE (Output)
◄─────────────────────────────────────────────────────── CHG_OTG (Output)
◄─────────────────────────────────────────────────────── I2C1_SDA (Bidirectional)
◄─────────────────────────────────────────────────────── I2C1_SCL (Output)
```

### 8.2 ESP32-S3-WROOM-1 Module

#### Components:

| Ref | Symbol | Value/Part | Footprint | Description |
|-----|--------|------------|-----------|-------------|
| U5 | ESP32-S3-WROOM-1-N8R2 | ESP32-S3-WROOM-1-N8R2 | ESP32-S3-WROOM-1 | MCU module (8MB flash, 2MB PSRAM) |
| C60 | C | 100nF | 0402 | 3V3 bypass |
| C61 | C | 10uF | 0805 | 3V3 bulk |
| R13 | R | 4.7k | 0402 | I2C0 SDA pull-up |
| R14 | R | 4.7k | 0402 | I2C0 SCL pull-up |
| R15 | R | 4.7k | 0402 | I2C1 SDA pull-up |
| R16 | R | 4.7k | 0402 | I2C1 SCL pull-up |

**CRITICAL: Use N8R2 variant (Quad SPI), NOT N8R8 (Octal SPI)!**

#### Pin Mapping:

| GPIO | Function | Direction | Connection |
|------|----------|-----------|------------|
| GPIO10 | LCD_CLK | OUT | Display SPI CLK |
| GPIO11 | LCD_MOSI | OUT | Display SPI MOSI |
| GPIO12 | LCD_CS | OUT | Display chip select |
| GPIO13 | LCD_DC | OUT | Display data/command |
| GPIO14 | LCD_RST | OUT | Display reset |
| GPIO21 | LCD_BL | OUT | Backlight PWM |
| GPIO8 | TP_SDA | I/O | Touch I2C0 SDA |
| GPIO9 | TP_SCL | OUT | Touch I2C0 SCL |
| GPIO1 | TP_RST | OUT | Touch reset |
| GPIO2 | TP_IRQ | IN | Touch interrupt |
| GPIO17 | I2C1_SDA | I/O | Power mgmt I2C |
| GPIO18 | I2C1_SCL | OUT | Power mgmt I2C |
| GPIO38 | CHG_INT | IN | Charger interrupt |
| GPIO39 | CHG_CE | OUT | Charge enable |
| GPIO40 | CHG_OTG | OUT | OTG mode |
| GPIO41 | FG_ALRT | IN | Fuel gauge alert |
| GPIO42 | BOOST_EN | OUT | Enable TPS61088 |
| GPIO4 | WDT_PING | OUT | 555 watchdog trigger |
| GPIO5 | FIRE_EN | OUT | Software fire enable |
| GPIO6 | FIRE_BTN | IN | Fire button input |
| GPIO7 | OC_FAULT | IN | Overcurrent fault |
| GPIO15 | COIL_PWM | OUT | PWM to Q4 gate |
| GPIO16 | I_SENSE | IN/ADC | Current sense ADC |
| GPIO47 | LED_STATUS | OUT | Fire button LED |
| GPIO48 | V_SENSE | IN/ADC | Boost voltage monitor |
| GPIO19 | USB_D+ | I/O | USB data+ |
| GPIO20 | USB_D- | I/O | USB data- |

#### Wiring Diagram (simplified):

```
    VSYS (3.0-4.2V from power management)
         │
         │      ┌────────┐
         ├──────┤ 3.3V   │ (from internal LDO or external regulator)
         │      │ REG    │
         │      └────┬───┘
         │           │
    ┌────┴────┐ ┌────┴────┐
    │  C60    │ │  C61    │
    │  100nF  │ │  10µF   │
    └────┬────┘ └────┬────┘
         ▼ GND       ▼ GND


    ┌─────────────────────────────────────────────────────────────────────────┐
    │                     ESP32-S3-WROOM-1-N8R2 (U5)                          │
    │                                                                          │
    │                                                                          │
    │   POWER:                                                                 │
    │   3V3 (pin 2) ◄──────────────────────────────── +3V3                     │
    │   GND (pins 1,40,41) ────────────────────────── GND                      │
    │   EN (pin 3) ────────────────────────────────── +3V3 (via 10k)           │
    │                                                                          │
    │   USB:                                                                   │
    │   GPIO19 (pin 13) ◄─────────────────────────── USB_D+ label              │
    │   GPIO20 (pin 14) ◄─────────────────────────── USB_D- label              │
    │                                                                          │
    │   DISPLAY (SPI):                                                         │
    │   GPIO10 (pin 15) ──────────────────────────── LCD_CLK (to J4)           │
    │   GPIO11 (pin 16) ──────────────────────────── LCD_MOSI (to J4)          │
    │   GPIO12 (pin 17) ──────────────────────────── LCD_CS (to J4)            │
    │   GPIO13 (pin 18) ──────────────────────────── LCD_DC (to J4)            │
    │   GPIO14 (pin 19) ──────────────────────────── LCD_RST (to J4)           │
    │   GPIO21 (pin 25) ──────────────────────────── LCD_BL (PWM, to J4)       │
    │                                                                          │
    │   TOUCH (I2C0):                                                          │
    │   GPIO8 (pin 12) ◄─────────────────────────── TP_SDA (to J4)             │
    │             │                                                            │
    │        ┌────┴────┐                                                       │
    │        │  R13    │ 4.7k pull-up                                          │
    │        └────┬────┘                                                       │
    │             └──────────────────────────────── +3V3                       │
    │                                                                          │
    │   GPIO9 (pin 11) ──────────────────────────── TP_SCL (to J4)             │
    │             │                                                            │
    │        ┌────┴────┐                                                       │
    │        │  R14    │ 4.7k pull-up                                          │
    │        └────┬────┘                                                       │
    │             └──────────────────────────────── +3V3                       │
    │                                                                          │
    │   GPIO1 (pin 27) ──────────────────────────── TP_RST (to J4)             │
    │   GPIO2 (pin 26) ◄─────────────────────────── TP_IRQ (from J4)           │
    │                                                                          │
    │   POWER MANAGEMENT (I2C1):                                               │
    │   GPIO17 (pin 22) ◄────────────────────────── I2C1_SDA label             │
    │             │                                                            │
    │        ┌────┴────┐                                                       │
    │        │  R15    │ 4.7k pull-up                                          │
    │        └────┬────┘                                                       │
    │             └──────────────────────────────── +3V3                       │
    │                                                                          │
    │   GPIO18 (pin 23) ─────────────────────────── I2C1_SCL label             │
    │             │                                                            │
    │        ┌────┴────┐                                                       │
    │        │  R16    │ 4.7k pull-up                                          │
    │        └────┬────┘                                                       │
    │             └──────────────────────────────── +3V3                       │
    │                                                                          │
    │   CHARGER CONTROL:                                                       │
    │   GPIO38 (pin 34) ◄────────────────────────── CHG_INT label              │
    │   GPIO39 (pin 35) ─────────────────────────── CHG_CE label               │
    │   GPIO40 (pin 36) ─────────────────────────── CHG_OTG label              │
    │                                                                          │
    │   FUEL GAUGE:                                                            │
    │   GPIO41 (pin 37) ◄────────────────────────── FG_ALRT label              │
    │                                                                          │
    │   BOOST CONTROL:                                                         │
    │   GPIO42 (pin 38) ─────────────────────────── BOOST_EN label             │
    │                                                                          │
    │   SAFETY SYSTEM:                                                         │
    │   GPIO4 (pin 8) ───────────────────────────── WDT_PING label             │
    │   GPIO5 (pin 9) ───────────────────────────── FIRE_EN label              │
    │   GPIO6 (pin 10) ◄─────────────────────────── FIRE_BTN label             │
    │   GPIO7 (pin 11) ◄─────────────────────────── OC_FAULT label             │
    │                                                                          │
    │   COIL CONTROL:                                                          │
    │   GPIO15 (pin 20) ─────────────────────────── COIL_PWM label             │
    │   GPIO16 (pin 21) ◄────────────────────────── I_SENSE label (ADC)        │
    │                                                                          │
    │   MISC:                                                                  │
    │   GPIO47 (pin 32) ─────────────────────────── LED_STATUS (to J5)         │
    │   GPIO48 (pin 33) ◄────────────────────────── V_SENSE (boost ADC)        │
    │                                                                          │
    │   BOOT (strapping - don't use for I/O):                                  │
    │   GPIO0 (pin 7) ───────────────────────────── Boot button (optional)     │
    │   GPIO3 (pin 28) ──────────────────────────── JTAG (reserved)            │
    │   GPIO45 (pin 31) ─────────────────────────── STRAPPING - DO NOT USE     │
    │   GPIO46 (pin 30) ─────────────────────────── STRAPPING - DO NOT USE     │
    │                                                                          │
    └─────────────────────────────────────────────────────────────────────────┘
```

### 8.3 Display Connector (J4)

#### Components:

| Ref | Symbol | Value/Part | Footprint | Description |
|-----|--------|------------|-----------|-------------|
| J4 | Conn_01x12_FPC | AFC07-S12 | FPC_12pin_0.5mm | Display FPC connector |

#### Pinout for Waveshare 1.69" Touch LCD:

| Pin | Signal | Direction | ESP32 GPIO |
|-----|--------|-----------|------------|
| 1 | VCC | Power | +3V3 |
| 2 | GND | Power | GND |
| 3 | CLK | OUT | GPIO10 |
| 4 | MOSI | OUT | GPIO11 |
| 5 | CS | OUT | GPIO12 |
| 6 | DC | OUT | GPIO13 |
| 7 | RST | OUT | GPIO14 |
| 8 | BL | OUT | GPIO21 (PWM) |
| 9 | TP_SDA | I/O | GPIO8 |
| 10 | TP_SCL | OUT | GPIO9 |
| 11 | TP_RST | OUT | GPIO1 |
| 12 | TP_IRQ | IN | GPIO2 |

### 8.4 Fire Button LED (J5)

#### Components:

| Ref | Symbol | Value/Part | Footprint | Description |
|-----|--------|------------|-----------|-------------|
| J5 | Conn_01x02 | B2B-XH-A | JST-XH_2pin | Fire button LED connector |
| R22 | R | 220 | 0402 | LED current-limiting resistor |

#### Wiring Diagram:

```
    GPIO47 (LED_STATUS)
         │
    ┌────┴────┐
    │  R22    │ 220Ω (adjust for LED Vf)
    └────┬────┘
         │
    ┌────┴────┐
    │   J5    │ JST-XH 2-pin male
    │  pin 1  │ ────────────────────► LED+ (to button LED anode)
    │         │
    │  pin 2  │ ────────────────────► LED- (to button LED cathode)
    └────┬────┘
         │
         ▼ GND (through LED)

    External wiring:
    - JST-XH 2-pin female housing
    - Connect to fire button integrated LED
    - Green LED (16mm switch from ModMaker)
```

### 8.5 Voltage Sense Divider

#### Components:

| Ref | Symbol | Value/Part | Footprint | Description |
|-----|--------|------------|-----------|-------------|
| R23 | R | 100k | 0402 | V_SENSE divider top |
| R24 | R | 47k | 0402 | V_SENSE divider bottom |
| C62 | C | 100nF | 0402 | ADC filter |

#### Wiring Diagram:

```
    V_BOOST (up to 7V)
         │
    ┌────┴────┐
    │  R23    │ 100kΩ
    └────┬────┘
         │
         ├───────────────────────────────────────► GPIO48 (V_SENSE)
         │
    ┌────┴────┐
    │  R24    │ 47kΩ
    └────┬────┘
         │
    ┌────┴────┐
    │  C62    │ 100nF (filter)
    └────┬────┘
         ▼ GND

    Calculation:
    V_SENSE = V_BOOST × 47k / (100k + 47k)
    V_SENSE = V_BOOST × 0.32

    At V_BOOST = 7V: V_SENSE = 2.24V (safe for 3.3V ADC)
```

---

## 9. Power Symbols and Net Classes

### 9.1 Power Symbols

Place these power symbols as needed:

| Symbol | Net Name | Description |
|--------|----------|-------------|
| `+3V3` | +3V3 | 3.3V regulated supply |
| `GND` | GND | Ground reference |
| `VBUS` | VBUS | USB 5V input |
| `VBAT` | VBAT | Battery voltage (protected) |
| `VSYS` | VSYS | System voltage (from PMID) |
| `V_BOOST` | V_BOOST | Boost output (1.5-7V) |

### 9.2 Net Classes for PCB

Define these net classes in schematic for PCB design:

| Net Class | Nets | Track Width | Clearance |
|-----------|------|-------------|-----------|
| Power_High | VBAT, V_BOOST, COIL_HIGH | 2.0mm | 0.3mm |
| Power_Medium | VSYS, VBUS | 1.0mm | 0.25mm |
| Signal | All others | 0.25mm | 0.2mm |
| Differential | USB_D+, USB_D- | 0.2mm | 0.2mm |

---

## 10. Annotation and BOM

### 10.1 Reference Designator Prefixes

| Prefix | Component Type |
|--------|----------------|
| U | Integrated circuits |
| Q | Transistors/MOSFETs |
| R | Resistors |
| C | Capacitors |
| L | Inductors |
| D | Diodes |
| J | Connectors |
| SW | Switches |

### 10.2 Component Summary

| Ref Range | Count | Description |
|-----------|-------|-------------|
| U1-U12 | 12 | ICs |
| Q1-Q4 | 4 | MOSFETs (Q1/Q2 are dual) |
| R1-R24, R_SHUNT | 25 | Resistors |
| C1-C62 | ~40 | Capacitors |
| L1-L2 | 2 | Inductors |
| D1-D3 | 3 | Diodes |
| J2-J7 | 6 | Connectors |

### 10.3 Annotate Schematic

1. Tools → Annotate Schematic
2. Select "Annotate entire schematic"
3. Use "First free number" option
4. Click "Annotate"
5. Verify all components have unique designators

---

## 11. Design Rule Checks

### 11.1 Run ERC

Before proceeding to PCB:

1. Inspect → Electrical Rules Checker
2. Check for:
   - Unconnected pins (should be intentional only)
   - Power pin conflicts
   - Missing power flags

### 11.2 Common ERC Fixes

| Warning | Fix |
|---------|-----|
| "Pin not driven" on power input | Add `PWR_FLAG` symbol |
| Unconnected pin | Add "No Connect" flag or wire |
| Input pin driven by input | Review connection, may be bidirectional |

---

## 12. Hierarchical Net Summary

### 12.1 Net Direction Reference

| Net Name | Type | From Sheet | To Sheet |
|----------|------|------------|----------|
| VBUS | Power | Power_Management | USB connector |
| VSYS | Power | Power_Management | Boost, MCU |
| V_BOOST | Power | Boost_Converter | Coil_Driver, Safety |
| USB_D+ | Bidirectional | Power_Management | MCU |
| USB_D- | Bidirectional | Power_Management | MCU |
| I2C1_SDA | Bidirectional | MCU | Power_Mgmt, Boost |
| I2C1_SCL | Output | MCU | Power_Mgmt, Boost |
| WDT_PING | Output | MCU | Safety_Logic |
| FIRE_EN | Output | MCU | Safety_Logic |
| FIRE_BTN | Input | Safety_Logic | MCU |
| OC_FAULT | Output | Safety_Logic | MCU |
| I_SENSE | Output | Safety_Logic | MCU |
| SAFETY_GATE | Output | Safety_Logic | Coil_Driver |
| COIL_PWM | Output | MCU | Coil_Driver |
| COIL_SENSE | Output | Coil_Driver | Safety_Logic |
| BOOST_EN | Output | MCU | Boost_Converter |
| CHG_INT | Output | Power_Management | MCU |
| CHG_CE | Output | MCU | Power_Management |
| CHG_OTG | Output | MCU | Power_Management |
| FG_ALRT | Output | Power_Management | MCU |

---

## 13. Final Checklist

### 13.1 Before Generating Netlist

- [ ] All hierarchical sheets created and populated
- [ ] All hierarchical pins connected on root sheet
- [ ] Power symbols placed in all sheets
- [ ] All components have values assigned
- [ ] All components have footprints assigned
- [ ] ERC passes with no errors
- [ ] Schematic annotated
- [ ] Net classes defined

### 13.2 Critical Design Verification

- [ ] TC4426 (inverting) used, NOT TC4427
- [ ] MCP4561-104 (100k) used, NOT MCP4561-103 (10k)
- [ ] BQ29705 used, NOT BQ29700 (higher OCD threshold)
- [ ] INA180A2 (50V/V) used, NOT INA180A3 (100V/V)
- [ ] ESP32-S3-WROOM-1-N8R2 (Quad SPI) used
- [ ] 10k pull-down on FIRE_EN
- [ ] 10k pull-up on LM393 output
- [ ] 10k pull-up on 555 RST
- [ ] All I2C buses have 4.7k pull-ups
- [ ] USB CC pins have 5.1k to GND

---

## Appendix A: I2C Device Addresses

| Device | Address (7-bit) | Bus |
|--------|-----------------|-----|
| CST816T (Touch) | 0x15 | I2C0 |
| BQ25896 (Charger) | 0x6B | I2C1 |
| MAX17055 (Fuel Gauge) | 0x36 | I2C1 |
| MCP4561 (Digital Pot) | 0x2E | I2C1 |

---

## Appendix B: Symbol/Footprint Quick Reference

| Component | KiCad Symbol | Footprint |
|-----------|--------------|-----------|
| BQ25896 | Custom/BQ25896RTWT | QFN-24_4x4mm_P0.5mm |
| BQ29705 | Custom/BQ29705DSET | BGA-5_1.17x0.77mm |
| MAX17055 | Custom/MAX17055EWL+T | WLP-9_1.4x1.4mm |
| TPS61088 | Custom/TPS61088RHLR | QFN-20_4x4mm_P0.5mm |
| ESP32-S3-WROOM-1 | RF_Module:ESP32-S3-WROOM-1 | ESP32-S3-WROOM-1 |
| NE555 | Timer:NE555 | SOIC-8 |
| 74LVC08A | Logic_LVC:74LVC08A | TSSOP-14 |
| 74LVC1G17 | Logic_LVC:74LVC1G17 | SOT-23-5 |
| INA180A2 | Custom/INA180A2IDBVR | SOT-23-5 |
| LM393 | Comparator:LM393 | SOIC-8 |
| TC4426 | Custom/TC4426ACOA | SOIC-8 |
| MCP4561-104 | Custom/MCP4561-104E/MS | MSOP-8 |
| Si7461DP | Custom/Si7461DP-T1-GE3 | PowerPAK_SO-8 |
| CSD18563Q5A | Custom/CSD18563Q5A | VSON-8_3.3x3.3mm |
| CSD17571Q2 | Custom/CSD17571Q2 | VSON-8_3.3x3.3mm |
| USB-C | Connector:USB_C_Receptacle_USB2.0 | USB_C_GCT_USB4110 |
| XT30U-M | Custom/XT30U-M | XT30U_Male |
| JST-XH 2-pin | Connector:Conn_01x02 | JST_XH_B2B-XH-A |

---

*Document Version: 1.0*
*Based on Vaporizer Design Specification v2.5*
*Created: December 2024*
