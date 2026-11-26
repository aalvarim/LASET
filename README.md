# Underwater ROV Strobe — Power, Bank Charger & Control Module (`LASET`)

Hardware schematic and engineering changelog for the main power-conversion, capacitor-bank charging, trigger-conditioning, and microcontroller module of an underwater ROV LED strobe (`600 bar` / `6000 m` rated cylinder architecture, see [main-1.pdf](main-1.pdf)).

* **Main KiCad Schematic:** [`buck 23-3.3/buck/buck.kicad_sch`](buck%2023-3.3/buck/buck.kicad_sch)
* **Design Specification:** [`main-1.pdf`](main-1.pdf)
* **Custom Symbol / Footprint Libraries:** [`LIB_LM517701DCPR/`](LIB_LM517701DCPR), [`LIB_NVMFS6H836NLT1G/`](LIB_NVMFS6H836NLT1G), [`LIB_SMCJ60A/`](LIB_SMCJ60A)

---

## 1. System Architecture Summary

| Subsystem | Key Devices | Electrical Specification & Role |
| :--- | :--- | :--- |
| **Input & Protection** | `J1`, `F1`, `Q_REV` (`IRF5210S`), `D_REV` (`12V`), `R_REV`, `TVS1` (`SMCJ60A`), `Cbulk1`, `Cin1`–`Cin3` | `10–60 V DC` ROV tether supply (`12/24/48 V` nominal) via Molex Micro-Fit `1×8`; `15 A / 72 V` fuse, high-side P-MOSFET reverse-polarity protection, `60 V` TVS surge clamping. |
| **Main Buck-Boost Charger** | `IC1` (`LM517701DCPR`), `M1`–`M4` (`NVMFS6H836NLT1G`), `L2` (`10 µH`), `Rcs1` (`4 mΩ`), `RISNS1` (`25 mΩ`) | 4-switch synchronous buck-boost (`410 kHz`). Charges the storage bank in **`2.0 A` Constant-Current (CC)** mode up to **`29.1 V` Constant-Voltage (CV)** (`R_FBT = 158 kΩ`, `R_FBB = 5.62 kΩ`). |
| **Energy Storage Bank** | `CB1`–`CB19` (`19 × 1000 µF / 50 V`) | `19 mF` electrolytic capacitor bank delivering high-current `≤ 1 ms` flash pulses (`~20.2 A` in 4-chain mode, `~40.3 A` in 8-chain mode). |
| **Auxiliary `5 V` Buck** | `U_5V` (`LM5164DDA`), `L_5V` (`47 µH`), `COUT1_5V`/`COUT2_5V` (`22 µF / 16 V`) | Steps `VOUT_SW` (`29.1 V` nominal, `8.5 V` UVLO) down to **`5.00 V`** (`R_FBT_5V = 174 kΩ`, `R_FBB_5V = 54.9 kΩ`) for CAN, RS-485, and trigger transceivers. |
| **Logic `3.3 V` LDO** | `U_3V3` (`TLV75533PDBV`), `CIN_3V3` / `COUT_3V3` (`1 µF`) | Regulates `5.0 V → 3.3 V` for the STM32 MCU, `I/O` rails (`VIO`), Schmitt trigger, and AND gate. |
| **MCU & Telemetry** | `U_MCU` (`STM32G491CCTx`, LQFP-48), `Y_MCU` (`24 MHz`), `J_SWD`, `J_SIG` (`2×8`) | Monitors `ADC_VIN`, `ADC_VBANK`, `nFLT`, `NTC_SENSE`, `FAULT_SENSE`; sets LED current reference `VSET` (`PA4`) and channel enables `EN1`–`EN8`; handles CAN & RS-485 telemetry. |
| **Comms & Trigger Path** | `U_CAN` (`TJA1051T-3`), `U_RS485` (`THVD1450V`), `U_TRIG` (`THVD1450V`), `U_SCHMITT` (`74LVC1G17`), `U_AND` (`74LVC1G08`) | Differential `CAN` and `RS-485` with `120 Ω` jumper-selectable termination (`JP_CAN`, `JP_RS`, `JP_TRG`). Differential optical trigger runs **purely in hardware** through a Schmitt buffer and `~1 ms` RC pulse limiter (`C_PULSE` + `R_PULSE` + `D_PULSE`) directly to `FLASH_FIRE`. |

---

## 2. Detailed Schematic Changelog & Design Rationale

This section documents every schematic modification made from the initial baseline (`2026-11-24`, git commit `04432f28`) through the electrical-review iterations (`2026-11-26`), explaining **what changed** and **why each change was required**.

---

### 2.1 Input Protection & Reverse-Polarity Stage

* **What changed:**
  * Added a series `15 A / 72 V` fuse specification on `F1` (previously an unrated `Fuse`).
  * Added high-side **P-MOSFET reverse-polarity protection** (`Q_REV` / `Q2` = `IRF5210S`, `-100 V / -40 A`) in series between `F1` and `Vin`, biased by a `12 V` Zener clamp (`D_REV` / `D1`, cathode on `Vin`, anode on gate) and a `100 kΩ` pull-down resistor (`R_REV` / `R1`, gate to `GND`).
  * Kept `TVS1` (`SMCJ60A`) downstream of `Q_REV` on the `Vin` rail.
* **Why:**
  * The initial schematic had **no reverse-polarity protection** — reversing the `10–60 V` ROV battery leads at `J1` would have destroyed `IC1` (`LM51770`), `Cbulk1`, and `U_5V` (`LM5164`).
  * A P-MOSFET conducts cleanly through its low `R_DS(on)` channel under normal polarity (unlike a high-loss Schottky diode at `6–15 A`) and blocks negative voltage up to `100 V` when polarity is inverted.
  * Because `Vin` can reach `60 V` while MOSFET gates are typically rated to `±20 V`, `D_REV` (`12 V`) clamps `|V_GS|` safely at `12 V`.
  * Placing `TVS1` downstream of `Q_REV` prevents the unidirectional TVS diode from forward-conducting and blowing the fuse/burning traces under reverse-battery faults, while still clamping positive hot-plug inductive transients on `Vin`.

---

### 2.2 Main `LM51770` Buck-Boost Converter & Power Stage

#### 1. Fixed Polymer Output Capacitors `Cout1_1` / `Cout1_2` (`68 µF / 50 V`) Placement
* **What changed:** Moved the positive terminals of `Cout1_1` and `Cout1_2` (`2 × 68 µF / 50 V` polymer bulk capacitors) from `BANK+` (downstream of `RISNS1`) back to **`VOUT_SW`** (upstream of `RISNS1`), alongside the ceramic output capacitors `Cout2_1`–`Cout2_3` (`3 × 15 µF / 100 V`).
* **Why:** As specified in `main-1.pdf` (p. 3), both the ceramic HF output capacitors and the polymer damping capacitors must sit directly at the boost switch stage **before** the `25 mΩ` shunt `RISNS1`. Placing bulk polymer capacitance after `RISNS1` shifts ripple/damping away from `M3`/`M4` and disturbs the `ISNSP`/`ISNSN` average-current control loop.

#### 2. Eliminated Duplicate / Conflicting `FB` Voltage Divider (`RFBT1`/`RFBB1` vs. `R_FBT`/`R_FBB`)
* **What changed:**
  * Initially, the schematic contained **two** parallel feedback dividers attached to `IC1.19` (`FB`): `RFBT1` (`158 kΩ`) / `RFBB1` (`5.62 kΩ`) referenced to `BANK-`, **plus** `R_FBT` (`158 kΩ`) / `R_FBB` (`5.62 kΩ`) referenced to `AGND`.
  * After removing a conflicting vertical bridge (which briefly left `FB` floating), the duplicate `RFBT1`/`RFBB1` pair was deleted completely, leaving **only** `R_FBT` (`158 kΩ`, from `BANK+`) and `R_FBB` (`5.62 kΩ`, to `AGND`) connected cleanly to `IC1.19` (`FB`).
* **Why:**
  * Two dividers in parallel halved the effective divider impedance (`79 kΩ / 2.81 kΩ`), increased quiescent drain on the bank, and **shorted `AGND` directly to `BANK-`/`PGND` through the `FB` node**, defeating ground separation.
  * Keeping only `R_FBT`/`R_FBB` referenced to clean `AGND` sets the factory `CV` regulation target accurately:
    $$V_{\text{OUT}} = 1.00\,\text{V} \cdot \left(1 + \frac{158\,\text{k}\Omega}{5.62\,\text{k}\Omega}\right) = 29.11\,\text{V}$$

#### 3. Fixed `ISNSP` / `ISNSN` Differential Current-Sense RC Filter (`R_100_P`, `R_100_N`, `C_ISNS1`)
* **What changed:** Rewired `C_ISNS1` (`1 µF`, originally `C_ISNS`) from sitting directly across the power buses (`ISNSP` ↔ `BANK+`) to sitting **across the filtered IC sense pins** (`IC1.23 ISNSP` ↔ `IC1.22 ISNSN`), downstream of the two `100 Ω` series resistors (`R_100_P` and `R_100_N`).
* **Why:** Putting `1 µF` directly across a `25 mΩ` shunt gives negligible filtering (`τ = R_shunt × C ≈ 25 ns`) and bypasses the purpose of `R_100_P`/`R_100_N`. Placing `C_ISNS1` after `2 × 100 Ω` creates a proper low-pass differential filter (`f_c ≈ 796 Hz`), suppressing high-frequency switching spikes on the small `50 mV` (`2.0 A × 25 mΩ`) current-limit signal.

#### 4. Added 4-Wire Kelvin `NetTie_2` Bridges (`NT1`–`NT4`) & Analog Ground Star Tie (`NT_GND`)
* **What changed:**
  * Inserted dedicated `NetTie_2` components to physically separate high-current power pours from Kelvin sense traces in PCB layout:
    * **`NT1`**: `VOUT_SW` (power) ↔ `ISNSP` (Kelvin trace to `R_100_P.1` & `TP8`)
    * **`NT2`**: `BANK+` (power) ↔ `ISNSN` (Kelvin trace to `R_100_N.1`)
    * **`NT3`**: `SW1` (power) ↔ `CSA` (Kelvin trace to `R_10CSA.2` & `Cdiff.1`)
    * **`NT4`**: `CSB_PWR` (`Rcs1.1` ↔ `L2.1` power node) ↔ `CSB` (Kelvin trace to `R_10CSB.2` & `Cdiff.2`)
    * **`NT_GND`**: `AGND` ↔ `GND`
  * Fixed intermediate net-label collisions along the way (removed duplicate `ISNSP`, `ISNSN`, `CSA`, `CSB`, and `VBANK` labels from the power buses, corrected `NT2` pin 2 from `ISNSP` to `ISNSN` so it wouldn't short across `RISNS1`, corrected `NT4` pin 1 from `SW2` to `CSB_PWR` so it wouldn't short across inductor `L2`, and unified `BANK-` into `GND`).
* **Why:**
  * Both `Rcs1` (`4 mΩ`) and `RISNS1` (`25 mΩ`) drop only tens of millivolts (`~50 mV` at limit). Even `1–2 mΩ` of shared PCB copper resistance in series with sense traces would shift current limits significantly and couple `di/dt` noise into the `LM51770` sense amplifiers.
  * `NetTie_2` guarantees KiCad treats power (`VOUT_SW`, `BANK+`, `SW1`, `CSB_PWR`) and sense (`ISNSP`, `ISNSN`, `CSA`, `CSB`) as distinct nets routed separately right up to the resistor pads, while eliminating ERC short errors.
  * `NT_GND` anchors `IC1`'s quiet analog ground (`AGND` — pin 39 `EP`, pin 17 `AGND`, `DTRK`, `MODE`, `R_FBB`, `Rcomp`/`Ccomp`, `Rimon`/`Cimon`, `Css`, `R_Rt`, `Rcfg`, `Rslope`, `R_UV2`/`C_UV`) at one star point to `GND`/`PGND` (pin 30).

#### 5. Added `IC1.VIN` HF Decoupling, Reserve DNP Snubbers, & Test Points
* **What changed:**
  * Added **`C_VIN_HF` (`C16`, `100 nF / 100 V`)** directly at `IC1` pin 3 (`VIN`) to `GND`.
  * Added reserve **RC snubbers** (`R_SNB1` `3 Ω` + `C_SNB1` `680 pF` on `SW1`; `R_SNB2` `3 Ω` + `C_SNB2` `680 pF` on `SW2`), flagged with KiCad's `(dnp yes)` property.
  * Added test points **`TP1`–`TP10`** (`TP_SW1`, `TP_SW2`, `TP3` `COMP`, `TP4` `IMONOUT`, `TP5` `SS/ATRK`, `TP6` `FB`, `TP7` `BANK+`, `TP8` `ISNSP`, `TP9` `Vin`, `TP10` `GND`).
  * Standardized `Rimon` formatting from `43,2k` to `43.2k`.
* **Why:**
  * `C_VIN_HF` provides an ultra-low-inductance bypass path right at `LM51770` `VIN` (matching `main-1.pdf` p. 3).
  * Marking the snubbers `DNP` ensures PCB pads exist if bench testing reveals `SW1`/`SW2` ringing above MOSFET ratings, without costing BOM parts or idle losses by default.
  * `TP1`–`TP10` cover every critical probe point recommended in `main-1.pdf` (p. 14).

---

### 2.3 Auxiliary `5 V` Buck (`U_5V`) & `3.3 V` LDO (`U_3V3`)

* **What changed:**
  1. Moved `U_5V` (`LM5164DDA`, pin 2 `VIN` + `CIN1_5V`/`CIN2_5V` + `RUV1_5V`) from **`BANK+`** (after `RISNS1`) to **`VOUT_SW`** (before `RISNS1`).
  2. Verified and explicitly labeled input caps `CIN1_5V`/`CIN2_5V` as **`2.2u/50V`** and output caps `COUT1_5V`/`COUT2_5V` as **`22u/16V`** (moving the loose `"16V"` text note inside the capacitor values).
  3. **Fixed broken `U_3V3` (`TLV75533PDBV`) output wiring:** Previously, `COUT_3V3` (`1 µF`, originally `C9`) was wired **in series** between `U_3V3` pin 5 (`OUT`) and the `+3.3V` net. Rewired pin 5 directly to `+3.3V` and connected `COUT_3V3` in parallel from `+3.3V` to `GND`.
  4. Added a KiCad `No-Connect (X)` flag on unused `U_5V.6` (`PGOOD`).
* **Why:**
  * Sourcing `U_5V` from `BANK+` would have caused transceiver and MCU supply current to count against the `2.0 A` constant-current recharge limit across `RISNS1` and exposed the `5 V` converter to bank discharge droop. Sourcing from `VOUT_SW` keeps logic power decoupled from bank charging.
  * A series capacitor (`C9`) on the `3.3 V` LDO output completely blocked DC voltage (`+3.3V` would remain unpowered at `0 V`!). Shunting `COUT_3V3` to `GND` delivers regulated `3.3 V` while meeting `TLV75533` output-stability ESR/capacitance requirements.

---

### 2.4 CAN, RS-485 & Optical Flash Trigger (`TRIGGER`)

* **What changed:**
  1. **Replaced over-volted `3.3 V` transceivers (`TCAN330` & `MAX3485`):**
     * `U4` (`TCAN330`, max `3.6 V` supply, wired to `+5V` with pins 5/8 grounded) → replaced with **`U_CAN` (`TJA1051T-3`)** (`VCC = +5V`, `VIO = +3.3V`, `S = GND`).
     * `U5` (`MAX3485`, `3.3 V` transceiver wired to `+5V`) → replaced with **`U_RS485` (`THVD1450V`)** (`VCC = +5V`, `VIO = +3.3V`).
     * `U6` (`MAX3485`, `3.3 V` transceiver wired to `+5V` with mismatched `RO`/`RE`/`DE` routing) → replaced with **`U_TRIG` (`THVD1450V`)** (`VCC = +5V`, `VIO = +3.3V`, `~RE = GND`, `DE = TRIG_DIR` via `PA9` with `10 kΩ` pull-down `R_PP`, `DI = PA8`, `10 kΩ` pull-up `R_TRIG_RO` on `RO`).
  2. **Added jumper-selectable `120 Ω` bus terminations:** Placed `JP_CAN`, `JP_RS`, and `JP_TRG` jumpers in series with `R_CAN`, `R_RS485`, and `R_TRIG` (`120 Ω`) instead of hard-wiring them across `CAN_H`/`CAN_L`, `RS485_A`/`B`, and `TRIG_A`/`B`.
  3. **Upgraded `TRIGGER` logic & hardware pulse limiter:**
     * Replaced quad TTL `7408` (`U8`, `5 V` family with unused gates floating) with a single-gate `3.3 V` CMOS **`U_AND` (`74LVC1G08`)**.
     * Upgraded `D_PULSE` from `1N4148` to a **`BAT54` Schottky diode** across `R_PULSE` (`15 kΩ`) on `U_AND` pin 2.
     * Removed the stray global label `LIMIT_EN` on the RC pulse-limiter node (`U_AND.2`).
  4. **Added local `100 nF` supply decoupling capacitors** across every comms/trigger supply pin (`C_CAN_5V`, `C_CAN_3V3`, `C_RS_5V`, `C_RS_3V3`, `C_TRG_5V`, `C_TRG_3V3`, `C_SCH`, `C_AND`).
* **Why:**
  * Powering `3.3 V`-only transceivers (`TCAN330` / `MAX3485`) from `+5V` exceeded absolute maximum ratings and risked immediate silicon failure. `TJA1051T-3` and `THVD1450V` use separate **`+5V` bus transceivers (`VCC`)** and **`+3.3V` MCU logic interfaces (`VIO`)**, giving full `5 V` differential noise margin over long wet-mate ROV cables without needing external level shifters.
  * Series termination jumpers (`JP_*`) allow nodes on a multi-drop ROV bus to enable `120 Ω` termination only at physical cable endpoints.
  * The `74LVC1G17` Schmitt trigger + `C_PULSE` (`100 nF`) / `R_PULSE` (`15 kΩ`) differentiator + `74LVC1G08` AND gate caps maximum `FLASH_FIRE` duration at `≤ ~1 ms` **strictly in hardware**, protecting LED chains and linear driver MOSFETs (`Q3`) even if firmware hangs or the camera trigger stays stuck high. Replacing `1N4148` with `BAT54` clamps negative falling-edge undershoot at `U_AND` pin 2 within the `74LVC` `-0.5 V` input spec.
  * `R_PP` (`10 kΩ` pull-down on `DE`) guarantees the trigger transceiver defaults to **listen/receive** during MCU reset; `PA8`/`PA9` allow firmware-driven master mode when configured over CAN (`main-1.pdf` p. 14).
  * Local `100 nF` bypass caps prevent ground/rail bounce during fast RS-485/CAN transitions and `74LVC1G` switching edges.

---

### 2.5 STM32G491 Microcontroller, Telemetry & `J_SIG` Connector

* **What changed:**
  1. **Fixed `SWD` and `CAN` pinout errors:**
     * Moved `SWDIO`/`SWCLK` from incorrect pins `34`/`37` (`PA12`/`PA13`) to true hardware SWD pins **`37` (`PA13`) and `38` (`PA14`)**, connected to `J_SWD`.
     * Moved `CAN_RX`/`CAN_TX` from `PB8`/`PB9` (pins `45`/`46`) to `FDCAN1` pins **`33` (`PA11`) and `34` (`PA12`)**.
     * Wired `RS485_TX` (`PB10`, pin 22), `RS485_RX` (`PB11`, pin 25), and `RS485_DE` (`PB12`, pin 26).
  2. **Connected `VREF+`, HSE Crystal, `NRST` Filter, & `BOOT0` Pull-Down:**
     * Tied `VREF+` (pin 20) to `+3.3V` and decoupled with `C_VREF` (`100 nF`).
     * Added **`24 MHz` HSE crystal (`Y_MCU` / `Crystal_ext`)** with `18 pF` load capacitors (`C_XTAL1`, `C_XTAL2`) on `PF0`/`PF1`.
     * Added **`100 nF` reset capacitor (`C_NRST`)** on `PG10` (`NRST`).
     * Added **`10 kΩ` pull-down resistor (`R_boot0`)** on `PB8` (`BOOT0`, pin 45) — also resolving an initial `R_pp` vs. `R_PP` case-collision.
  3. **Added & fixed `Vin` and `BANK+` ADC Sensing Dividers:**
     * Added `R_VIN_T` (`97.6 kΩ`) / `R_VIN_B` (`5.1 kΩ`) + `C_ADC_VIN` (`100 nF`) on `PA0` (`ADC_VIN`).
     * Added `R_VBK_T` (`63.4 kΩ`) / `R_VBK_B` (`5.1 kΩ`) + `C_ADC_BANK` (`100 nF`) on `PA1` (`ADC_VBANK`).
     * Fixed an intermediate wiring bug where `C_ADC_VIN` and `C_ADC_BANK` were connected **in series** on the ADC trace rather than shunted in parallel to `GND` (also corrected `R4` `VBANK` → `BANK+`).
  4. **Expanded Inter-Board Signal Header (`J_SIG` / `J2`: `1×4` → `2×8`):**
     * Replaced the initial 4-pin header (`VSET`, `FLASH_FIRE`, 2 floating/GND pins) with a `16-pin` (`Conn_02x08_Counter_Clockwise`) inter-board signal header carrying:
       * Power: `+5V` (pin 1), `+3.3V` (pin 2), `GND` (pin 3)
       * Analog / Trigger / Status: `VSET` (`PA4`, pin 4), `FLASH_FIRE` (pin 5), `nFLT` (`PA5` with `10 kΩ` `Rnflt` pull-up, pin 6), `NTC_SENSE` (`PA2`, pin 7), `FAULT_SENSE` (`PA3`, pin 8)
       * Per-channel enables: `EN8`–`EN1` (`PB15`, `PB14`, `PB13`, `PB9`, `PB7`, `PB6`, `PB5`, `PB4`, pins 9–16).
* **Why:**
  * With `SWD` shifted onto `PA12`/`PA13`, programming and debugging the STM32 would have been impossible (`PA12` is `FDCAN1_TX`, not `SWDIO`), and `CAN` on `PB8`/`PB9` conflicted with `BOOT0` (`PB8`).
  * STM32G4 enables hardware `BOOT0` (`PB8`) by default (`nSWBOOT0 = 1`) without an active internal pull-down before option bytes load; leaving `PB8` floating risks random boots into the ROM bootloader instead of application Flash.
  * The `24 MHz` HSE crystal provides strict clock tolerance for `CAN` (`FDCAN`) and `RS-485` baud generation.
  * Scaling `Vin` (`60 V max → 2.98 V`) and `BANK+` (`29.1 V nom → 2.17 V`, `44.3 V max < 3.3 V`) with `100 nF` parallel RC filters protects STM32 ADC pins and stabilizes SAR sampling.
  * Expanding `J_SIG` to `2×8` supports both the 4-chain (`A4`) and 8-chain (`A8`) stacked channel-board configurations, per-channel firmware thermal/fault throttling (`NTC_SENSE`, `FAULT_SENSE`, `EN1`–`EN8`), and `BANK_OK` (`nFLT`) interlocks (`main-1.pdf` p. 14).

---

### 2.6 Schematic ERC Cleanup & Naming Standardization

* **What changed:**
  * Placed `3` **`PWR_FLAG`** symbols on `+5V`, `VOUT_SW`, and `GND`.
  * Added **`No-Connect (X)`** markers on all intentionally unused pins (`U_MCU` spare GPIOs, `U_5V.6 PGOOD`, `IC1` `NC_1..6`, `HO1_LL`, `HO2_LL`).
  * Deleted an off-sheet stray dangling `EN4` label.
  * Updated `COUT1_5V`/`COUT2_5V` value strings (`22u → 22u/16V`) and `Rimon` (`43,2k → 43.2k`).
  * Replaced generic `R#`/`C#`/`U#` designators across all newly added blocks with descriptive names (`Q_REV`, `D_REV`, `R_REV`, `Y_MCU`, `C_XTAL1`/`2`, `C_NRST`, `C_ADC_*`, `R_VIN_*`, `R_VBK_*`, `R_SNB1`/`2`, `C_SNB1`/`2`, `J_SIG`, `TP_SW1`/`2`, `C_CAN_*`, `C_RS_*`, `C_TRIG_*`, `C_SCH`, `C_AND`, `C_VIN_HF`).
* **Why:**
  * Clearing all electrical-rule (`ERC`) errors (`34+ → 0`) and single-pin nets verifies that no power rails, feedback loops, net-ties, or MCU signals are silently broken before moving on to footprint assignment and PCB layout.

---

## 3. Current Status & Final Pre-Footprint Checklist (`2026-11-26 13:29`)


1. **Next milestone (PCB / Layout Phase):**
   * Assign & verify footprints for every symbol (including high-current inter-board `BANK+` (`CANAIS`) and `GND` connectors between stacked `Ø96 mm` boards).
   * Respect the `96 mm` circular board layout constraints in `main-1.pdf` (p. 10–11): keep switching nodes (`SW1`, `SW2`) compact, route `NT1`–`NT4` Kelvin pairs differentially up to the `RISNS1`/`Rcs1` pads, isolate `AGND` under `IC1` to `NT_GND`, and place `C6`–`C16` right at their respective IC pins.
