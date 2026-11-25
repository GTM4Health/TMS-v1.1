# Calling Unit C3 — Hardware Design & Datasheet Reference Guide

This document provides complete, verified technical references, page numbers, schematic citations, and engineering rationales for the circuit design of the **Calling Unit C3** PCB.

---

## 1. Primary Component Datasheets Index

| Component | Part Number / Specification | Document File Name | Source / Standard |
| :--- | :--- | :--- | :--- |
| **Microcontroller / RF Module** | ESP32-C3-MINI-1-N4X | `ESP32-C3-MINI-1-N4X.pdf` (Rev 1.4) | Espressif Systems |
| **Voltage Regulator (LDO)** | AMS1117-3.3 | `AMS1117-3.3V.pdf` | Advanced Monolithic Systems |
| **USB ESD Protection** | USBLC6-2SC6 | `USBLC6-2SC6.pdf` (Rev 4) | STMicroelectronics |
| **USB Type-C Connector** | USB Type-C Receptacle (16-Pin) | `USB_Type-C_Specification_Release_1.3.pdf` | USB-IF Specification Release 1.3 |

---

## 2. Power Supply & LDO Circuit Verification

### 2.1 Voltage Regulator (AMS1117-3.3)
- **Document:** `AMS1117-3.3V.pdf`
- **Datasheet Page References:**
  - **Page 1 (Pinout & Features):** SOT-223 package pin assignment:
    - **Pin 1:** Ground (`GND`)
    - **Pin 2 & Tab:** Output Voltage (`VOUT` = 3.3V fixed)
    - **Pin 3:** Input Voltage (`VIN` = 5V from USB `VBUS`)
  - **Page 2 (Electrical Characteristics):**
    - Dropout voltage: typical $1.1\text{ V}$, maximum $1.3\text{ V}$ at $1\text{ A}$ load. For $5\text{ V} \rightarrow 3.3\text{ V}$, headroom is $5.0 - 3.3 = 1.7\text{ V}$, which safely exceeds the $1.3\text{ V}$ dropout requirement.
  - **Page 2 & 5 (Application Hints — Stability):**
    - **Output Capacitor Requirement:** Requires an output capacitor to maintain loop stability. Recommends a minimum of $22\text{ }\mu\text{F}$ solid tantalum or low-ESR electrolytic capacitor. On our board, **`C2` ($22\text{ }\mu\text{F}$ ceramic/tantalum)** is placed directly across the 3.3V output rail and ground.
- **Input Capacitor Requirement & Technical Article Citation:**
  - **Citation:** [UltraLibrarian: AMS1117 Datasheet Explained — Voltage Regulator Basics](https://www.ultralibrarian.com/2026/07/10/ams1117-datasheet-explained-pinout-features-and-voltage-regulator-basics-ulc)
  - **Engineering Rationale for 10µF Input Capacitor (`C1`):**
    - USB cables have parasitic inductance ($L \approx 1\text{ }\mu\text{H/m}$). When a device is hot-plugged, input inrush current creates an LC resonant tank with input capacitance that can cause high-frequency voltage spikes exceeding the regulator's $15\text{ V}$ maximum rating.
    - A **$10\text{ }\mu\text{F}$ input capacitor (`C1`)** damps the input line inductance, absorbs input ripple, and provides a low-impedance reservoir for the internal bipolar pass transistor during rapid transient load steps (such as ESP32 WiFi RF bursts).

---

## 3. USB-C Power Delivery & CC Resistor Termination

### 3.1 Upstream Facing Port (UFP / Device) Configuration
- **Document:** `USB_Type-C_Specification_Release_1.3.pdf`
- **Specification Page & Section References:**
  - **Section 4.5.1.3.1 (Page 137):** *Source Configuration Channel Functional Model*:
    - A USB Type-C host/charger (Source) monitors the CC1 and CC2 pins for an $R_d$ pull-down termination to ground to detect when a downstream device (Sink) has been attached.
    - If neither CC pin has an $R_d$ termination, the host interprets the port as unattached and **maintains $V_{BUS}$ at 0V (no power output)**.
  - **Section 4.5.3.2.1 & Table 4-21 (Page 197):** *Sink CC Termination ($R_d$) Requirements*:
    - Nominal $R_d$ value: **$5.1\text{ k}\Omega \pm 10\%$ (or $\pm 20\%$) pull-down resistor to GND**.
  - **Section 4.5.1.2.1 (Pages 70–72, 133):**
    - Both CC1 (Pin A5) and CC2 (Pin B5) in a receptacle must be terminated to GND with independent $5.1\text{ k}\Omega$ resistors (`R1` and `R2`).
    - **Design Implementation:** `R1` ($5.1\text{ k}\Omega$) connects CC1 to GND; `R2` ($5.1\text{ k}\Omega$) connects CC2 to GND. This guarantees 5V $V_{BUS}$ power negotiation with all USB-C chargers, smart adapters, and modern Type-C to Type-C cables.

---

## 4. ESP32-C3 Native USB-Serial/JTAG Communication

### 4.1 USB Pins GPIO18 and GPIO19
- **Document:** `ESP32-C3-MINI-1-N4X.pdf`
- **Datasheet Page References:**
  - **Table 2-2 (Page 11) & Schematics (Page 32):** *Pin Definitions*:
    - **Pin 26 (`IO18`):** Function is `GPIO18`, hardware assigned to **`USB_D-`**.
    - **Pin 27 (`IO19`):** Function is `GPIO19`, hardware assigned to **`USB_D+`**.
  - **Section 9 (Page 34, Figure 9-1):** *Peripheral Schematics*:
    - Demonstrates direct differential routing of `USB_D-` and `USB_D+` from the USB connector to Pins 26 and 27 of the ESP32-C3-MINI-1 module.
- **Espressif Documentation Citation:**
  - **Citation:** [Espressif ESP-IDF Documentation: Establish Serial Connection on ESP32-C3](https://docs.espressif.com/projects/esp-idf/en/stable/esp32c3/get-started/establish-serial-connection.html)
  - **Engineering Rationale:**
    - The ESP32-C3 chip contains an integrated hardware **USB-Serial/JTAG Controller**.
    - Unlike older ESP32 or ESP8266 boards that require external USB-to-UART conversion chips (such as CP2102, FT232, or CH340), the ESP32-C3 connects directly to the USB differential lines `D+` and `D-`.
    - It supports standard USB Communication Device Class (CDC), allowing direct code flashing, serial terminal debugging, and hardware JTAG debugging through a single USB-C cable.

---

## 5. Boot Mode Selection & 10kΩ Strapping Resistors

### 5.1 Boot Strapping Pins: GPIO8 & GPIO9
- **Document:** `ESP32-C3-MINI-1-N4X.pdf`
- **Datasheet Page References:**
  - **Chapter 4 "Boot Configurations", Table 4-1 (Page 12):** *Default Configuration of Strapping Pins*:
    - `GPIO8`: Default configuration is **Floating** (`–`).
    - `GPIO9`: Default configuration is **Weak pull-up** (`1`).
  - **Chapter 4.1 "Chip Boot Mode Control", Table 4-3 (Page 13):** *Chip Boot Mode Control Table*:

    | Boot Mode | GPIO2 | GPIO8 | GPIO9 | Description |
    | :--- | :---: | :---: | :---: | :--- |
    | **SPI Boot (Normal Run)** | Any value | Any value | **1 (HIGH)** | Boots user firmware directly from internal SPI flash. |
    | **Joint Download Boot (Flashing)** | 1 | **1 (HIGH)** | **0 (LOW)** | ROM bootloader activates USB-Serial/JTAG & UART0 download mode. |

  - **Table 4-2 & Figure 4-1 (Page 13):** *Strapping Pin Timing Parameters*:
    - Strapping pin logic levels are latched on the rising edge of `CHIP_EN` (reset release).
  - **Chapter 9 "Peripheral Schematics", Figure 9-1 (Page 34):**
    - Recommends an external pull-up resistor $R = 10\text{ k}\Omega$ (`R9` in datasheet, `R7` on our PCB) to `VDD33` on `GPIO9`.
- **Espressif Boot Mode Documentation Citation:**
  - **Citation:** [Espressif esptool Documentation: ESP32-C3 Boot Mode Selection](https://docs.espressif.com/projects/esptool/en/latest/esp32c3/advanced-topics/boot-mode-selection.html)
  - **Why 10kΩ Pull-Ups on GPIO8 (`R10`) and GPIO9 (`R7`) are Required:**
    1. **GPIO8:** Table 4-3 dictates that for code upload (Joint Download Boot), **`GPIO8` must be HIGH (1)**. Because the internal state of GPIO8 is floating (uncommitted), leaving it floating can cause stray capacitive charge to float low during reset, causing flashing to fail. An external **$10\text{ k}\Omega$ pull-up (`R10`) to 3.3V** firmly holds GPIO8 at logic 1.
    2. **GPIO9:** During normal power-up, **`GPIO9` must be HIGH (1)** to boot from SPI flash. Although GPIO9 has an internal weak pull-up ($R_{PU} \approx 45\text{ k}\Omega$, Table 4-9, Page 21), Espressif hardware design guidelines mandate an external **$10\text{ k}\Omega$ pull-up (`R7`) to 3.3V** to guard against electromagnetic noise from motor switching, relays, or WiFi transmission pulling the pin below the $V_{IL}$ threshold ($0.25 \times V_{DD}$).
    3. **Boot Switch (`SW5`):** Connects `GPIO9` to GND. When pressed during reset, it pulls `GPIO9` to 0V (LOW) through a low-resistance path, overcoming `R7` and forcing the chip into download boot mode.

---

## 6. Reset & Chip Enable (EN) RC Delay Timing

### 6.1 Power-On Reset Circuit
- **Document:** `ESP32-C3-MINI-1-N4X.pdf`
- **Datasheet Page References:**
  - **Section 4.3 (Pages 14–15):** *Chip Power-up and Reset*:
    - "Once power is supplied to the chip, its power rails need a short time to stabilize. After that, `CHIP_EN` is pulled high to activate the chip."
    - Figure 4-2 shows that the 3.3V power rail must reach the reset release voltage before `CHIP_EN` rises above $V_{IH\_nRST}$ ($0.75 \times V_{DD}$).
  - **Section 9 (Page 34, Figure 9-1 & Notes):**
    - Specifies the exact values for the RC delay circuit:
      $$\text{Recommended: } R = 10\text{ k}\Omega, \quad C = 1\text{ }\mu\text{F} \quad (\text{or } 0.1\text{ }\mu\text{F})$$
    - Time constant calculation:
      $$\tau = R \times C = 10\text{ k}\Omega \times 1\text{ }\mu\text{F} = 10\text{ ms}$$
    - This provides a $10\text{ ms}$ delay upon USB insertion, guaranteeing that the AMS1117 3.3V rail and decoupling caps are fully charged and stabilized before the ESP32-C3 begins executing code.
  - **Design Implementation:** `R3` ($10\text{ k}\Omega$) pulls `CHIP_EN` to 3.3V, `C4` ($1\text{ }\mu\text{F}$) connects `CHIP_EN` to GND, and `SW4` (EN button) pulls `CHIP_EN` to GND when pressed.

---

## 7. USB ESD Protection (USBLC6-2SC6)

### 7.1 Rail-to-Rail Low-Capacitance Clamping
- **Document:** `USBLC6-2SC6.pdf`
- **Datasheet Page References:**
  - **Page 1 (Description & Pinout):** SOT-23-6L pinout:
    - **Pin 1:** `I/O1` (connected to USB-C `D+` input)
    - **Pin 2:** `GND` (connected to PCB ground plane)
    - **Pin 3:** `I/O2` (connected to USB-C `D-` input)
    - **Pin 4:** `I/O2` pass-through (connected to ESP32-C3 `GPIO18`)
    - **Pin 5:** `VBUS` (connected to 5V rail for rail-to-rail clamping)
    - **Pin 6:** `I/O1` pass-through (connected to ESP32-C3 `GPIO19`)
  - **Page 1 & 8:** Ultra-low line capacitance: $C_{I/O-GND} = 0.6\text{ pF}$ typical, matched to $\Delta C = 0.015\text{ pF}$. This preserves high-speed differential signal integrity and maintains USB 2.0 full-speed ($12\text{ Mbps}$) eye-diagram compliance.
  - **Page 9, Section 2.5 (Figure 13):** Typical USB application circuit matching the exact schematic wiring used on this board.

---

## 8. Summary Traceability Matrix

| Subsystem | Circuit Component(s) | Value / Net | Primary Reference Document | Page / Section |
| :--- | :--- | :--- | :--- | :--- |
| **5V Input Regulation** | `U2` (AMS1117-3.3) | $5\text{V} \rightarrow 3.3\text{V}$, $1\text{A}$ | `AMS1117-3.3V.pdf` | Pages 1, 2, 5 |
| **LDO Input Damping** | `C1` | $10\text{ }\mu\text{F}$ on 5V | UltraLibrarian Engineering Guide | Section "Input Capacitor" |
| **LDO Loop Stability** | `C2` | $22\text{ }\mu\text{F}$ on 3.3V | `AMS1117-3.3V.pdf` | Page 2, "Stability" |
| **USB-C 5V Negotiation**| `R1`, `R2` | $5.1\text{ k}\Omega$ to GND | `USB_Type-C_Specification_Release_1.3.pdf` | Page 197, Table 4-21 |
| **Native USB Data** | ESP32-C3 Pins 26, 27 | `GPIO18` (D-), `GPIO19` (D+) | `ESP32-C3-MINI-1-N4X.pdf` & ESP-IDF Docs | Page 11, Table 2-2 |
| **USB ESD Protection** | `D1` (USBLC6-2SC6) | SOT-23-6 in-line | `USBLC6-2SC6.pdf` | Pages 1, 4, 9 |
| **SPI Boot Lock** | `R7` | $10\text{ k}\Omega$ pull-up on GPIO9 | `ESP32-C3-MINI-1-N4X.pdf` & esptool Docs | Page 13, Table 4-3 |
| **Download Boot Lock** | `R10` | $10\text{ k}\Omega$ pull-up on GPIO8 | `ESP32-C3-MINI-1-N4X.pdf` & esptool Docs | Page 13, Table 4-3 |
| **Power-On Reset RC** | `R3`, `C4` | $10\text{ k}\Omega + 1\text{ }\mu\text{F}$ | `ESP32-C3-MINI-1-N4X.pdf` | Page 14 & Page 34 |
| **Buttons (Next/Again/Reset)**| `R4`, `R5`, `R6` | $10\text{ k}\Omega$ pull-ups to 3.3V | Microcontroller Standard Practice | Active-Low Input |
| **Power Indicator LED**| `D2`, `R8` | Red LED + $470\Omega$ | Standard 3.3V Indicator | $I_F \approx 2.8\text{ mA}$ |
| **WiFi Status LED** | `D3`, `R9` | Green LED + $470\Omega$ | GPIO0 Status Pin | $I_F \approx 2.3\text{ mA}$ |
