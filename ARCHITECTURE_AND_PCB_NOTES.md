# PCB Architecture Decisions, Improvements & Single-PCB Tradeoff Analysis (`LASET`)

This note summarizes the recommended PCB engineering adjustments, alignment notes between the initial requirements report ([`Modelo_LaSET-4-1.pdf`](Modelo_LaSET-4-1.pdf)) and the detailed power design ([`main-1.pdf`](main-1.pdf)), and the engineering answer to **"Why aren't the Capacitor Bank, Converter, MCU, and LED Drivers all on a single PCB?"**

---

## 1. Schematic Status & Next PCB Steps

Your converter/control schematic ([`buck 23-3.3/buck/buck.kicad_sch`](buck%2023-3.3/buck/buck.kicad_sch)) is electrically verified (`0` real ERC errors) and **does not need to be redesigned**.

* **In this KiCad project (`Board 1: Converter + Control`):**
  1. Assign/correct component footprints (notably fixing pre-existing mismatches on `U_TRIG`, `U_AND`, `F1`, `D_PULSE`, `R_PULSE`, `TP_SW1`, and `U_5V`/`U_3V3`/`U_MCU` capacitors).
  2. Keep `CB1`–`CB19` (`19 × 1000 µF / 50 V`) in the schematic for full-system ERC if desired, but mark them **Exclude from board** when generating `Board 1` PCB (or move them into a dedicated Bank KiCad project).
* **Additional KiCad PCB Projects (under the `main-1.pdf` `4`-board modular architecture):**
  1. **Bank PCB:** `19 × 1000 µF / 50 V` (`EEU-FR1H102`) + heavy `BANK+` / `GND` power interconnect pads.
  2. **4-Channel Driver PCB:** `4 ×` linear current sinks (`Op-Amp + Q3 NVMFS6H836NL + 50 mΩ RS`) + quad analog switch + quad AND + `NTC` + `EN1–4` / `EN5–8` solder jumper.
  3. **Passive LED Matrix (`MCPCB`):** `4×4S` or `8×4S` `XFLC20` LEDs + `NTC`.

---

## 2. Recommended Engineering Improvements & `Modelo_LaSET` Alignment

| Area | Original Plan (`Modelo_LaSET` / `main-1.pdf`) | Recommended Adjustment | Engineering Rationale |
| :--- | :--- | :--- | :--- |
| **1. Inter-board Wiring** | Rigid vertical pin headers stacked between `Board 1 ↔ 2 ↔ 3` (`30°` / `150°`) | Use **short flexible wire harnesses** (cabled `J_SIG` + `12–14 AWG` silicone leads / XT30 for `BANK+`/`GND`) | Rigid blind-mating pins fight the tight mechanical Z-tolerance needed to compress thermal pads between `Board 3` (`Q3`s) and the `Ø99.9 mm` aluminum cooling disc. |
| **2. `CB1`–`CB19` Placement** | Drawn inside [`buck.kicad_sch`](buck%2023-3.3/buck/buck.kicad_sch) | Set `CB1`–`CB19` to **Exclude from board** (or split into a separate Bank project) | `19 × Ø16 mm` radial capacitors physically fill an entire `Ø96 mm` disc (`main-1.pdf` Fig. 7a). |
| **3. Rev-1 Assembly Order** | Populate all `153` parts at once | Solder and power-test **`LM51770` + `5 V` + `3.3 V` first**, then populate `STM32` and comms | Avoids debugging high-voltage switching power and `0.5 mm`-pitch digital ICs simultaneously on first bring-up. |
| **4. `V_OUT` Calibration (`R_FBT`)** | Fixed `158 kΩ` (`29.1 V` worst-case) | Bring up at `158 kΩ`, then swap `R_FBT` (`0805`) per `main-1.pdf` Sec. 6 after measuring LED `V_F` | Drops `Q3` linear MOSFET dissipation by **3–4×** (`~2.8 W → ~0.7 W` per FET). |
| **5. Report Consistency (`Modelo_LaSET-4-1.pdf`)** | Early report mentions: **Buck-only** (`RNF1`), **sapphire window + internal MCPCB** (`RNF2`/`RNF3`), **ambient-light brightness** (`RF2`), **SEACON MCBH** (`RNF4`), **no TRIG in Fig. 4.1** | Keep current hardware and document 3 evolutions in the final report: **(a)** upgraded to **4-switch Buck-Boost** (`LM51770`) so `12 V`/`24 V` ROVs can boost to `29.1 V`; **(b)** clarify **remote oil LED head** (`main-1.pdf`) vs. **sapphire window** (`Modelo_LaSET`); **(c)** note `RF2` ambient light comes via **CAN/RS-485** (no onboard light sensor on `U_MCU`) and `J1` wires to the **SEACON MCBH-8** bulkhead | Eliminates apparent contradictions between the requirements document and final hardware schematic. |

---

## 3. Why Not Put Everything on ONE Single PCB? (And When You Actually Can)

### Direct Answer
* **Will putting everything (`Converter + MCU + Bank + LED Drivers`) on ONE PCB work electrically?**
  **Yes — electrically, a single PCB works *better*!** You eliminate inter-board connectors/harnesses, shrink `BANK+`/`GND` loop inductance during `20–40 A` `1 ms` flashes, and share one continuous solid ground plane.
* **Then why did [`main-1.pdf`](main-1.pdf) split it into 3–4 mini PCBs?**
  Purely due to **two physical/mechanical constraints** of the **`Ø100 mm` stacked circular disc** layout — **not** because a single board won't work electrically:

1. **Cross-Section Area (`Ø96 mm` circle limit):**
   * A `Ø96 mm` PCB has `~6,000 mm²` usable area per face (after edge keepouts and `3× M3` mounting holes at `r = 43 mm`).
   * The `19 × EEU-FR1H102` capacitors (`Ø16 mm × 25 mm` tall, `7.5 mm` THT radial pitch) require a `~Ø84 mm` footprint (`~4,800–5,500 mm²`) across **both sides** of the PCB (because THT pins pass through to bottom copper).
   * Your `Converter + MCU + Comms` circuit needs `~2,700 mm²` (`main-1.pdf` p. 10), plus `~1,000 mm²` for the 4 linear channels.
   * **`4,800–5,500 + 2,700 + 1,000 ≈ 8,500–9,200 mm² > 6,000 mm²`** — it simply **cannot physically fit** on one `Ø96 mm` circular board.

2. **Thermal Clamping vs. `25 mm`-Tall Capacitors & `A4`/`A8` Modularity:**
   * The `4 × Q3` linear MOSFETs (`NVMFS6H836NL`, dissipating up to `~11 W` worst-case) must bolt flat against a `Ø99.9 mm` aluminum cooling disc pressed to the cylinder wall (`main-1.pdf` p. 6, 9–10). Tall `25 mm` radial caps (`CB1`–`CB19`) and `10 mm` inductors (`L2`) restrict where a flat clamping plate can sit.
   * Splitting `Bank` and `4-Ch Driver` into separate identical boards also allows scaling between **4 chains (`A4`)** and **8 chains (`A8`)** inside the same tube without redesigning any PCB.

---

### Practical Architectural Options Compared

If you want to reduce the number of "mini PCBs" in the project while respecting physics and cooling, here are the three realistic options:

| Option | Boards Inside Cylinder | External / Front Board | Pros | Cons / Tradeoffs |
| :--- | :--- | :--- | :--- | :--- |
| **Option A — Current `main-1.pdf` Stack (`Ø96 mm` discs)** | **3 designs:**<br>1. `Converter + MCU` (`buck.kicad_sch`)<br>2. `Bank` (`19 × 1000 µF`) *(1× for A4, 2× for A8)*<br>3. `4-Ch Driver` *(1× for A4, 2× for A8)* | **1:** Passive `LED Matrix` (`MCPCB`) | Fits shortest `Ø100 mm` cylinder (`~140 mm`), clean `A4 ↔ A8` scaling, `buck.kicad_sch` is already done | More separate PCB files (`4` designs total) and inter-board cables |
| **Option B — Consolidate to 2 Round PCBs (`Ø96 mm` discs, A4)** | **2 designs:**<br>1. `Converter + MCU + 4-Ch Driver` (`~3,700 mm²` fits on one `Ø96 mm` disc!)<br>2. `Bank` (`19 × 1000 µF`) | **1:** Passive `LED Matrix` (`MCPCB`) | Eliminates `J_SIG` cable & `Board 3` entirely; leaves only `1` high-current `BANK+/GND` pair to `Bank` PCB | Requires adding the `4` channels onto [`buck.kicad_sch`](buck%2023-3.3/buck/buck.kicad_sch) and routing `Q3` heat through thermal vias to a flat disc on the bottom face; fixed to 4 channels unless adding `Board 3` later |
| **Option C — Single Longitudinal Rectangular PCB (inside `Ø100 mm` tube)** | **1 single PCB (`~75 × 170 mm`):**<br>All electronics (`Converter + MCU + Comms + 19 Caps + 4-Ch Driver`) along the tube axis | **1:** Passive `LED Matrix` (`MCPCB`) | **Best electrical performance** (zero inter-board harnesses, shortest `20–40 A` loop, single solid `GND` plane, `1` main board to fabricate!) | Needs custom aluminum side/bottom heatsink bracket for `Q3`s against the tube wall, slightly longer tube (`~170–180 mm` vs `140 mm` for `A4`), and requires laying out one combined rectangular PCB |

> **Geometry check for Option C (`Ø100 mm` ID tube):** With a `70 mm`-wide rectangular PCB offset `8–10 mm` below the cylinder centerline, the vertical chord clearance at the board edges (`x = ±35 mm`) is still `√(50² - 35²) ≈ 35.7 mm` from center — leaving **`> 43 mm` vertical headroom above the PCB** (easily clearing `25 mm`-tall `EEU-FR1H102` caps) and **`~26–28 mm` below**.

---

### Summary Recommendation
* If you want to **finish cleanly with zero schematic rework** and preserve `A4`/`A8` scalability: keep **Option A** (your current schematic + Bank + Channels + LED Matrix) and connect the boards with **short wire harnesses** instead of rigid vertical pins.
* If your professor strongly prefers **fewer PCBs** and you only need 4 channels (`A4`): **Option C** (one longitudinal rectangular main PCB + passive LED `MCPCB`) gives you **only 2 PCBs total** for the entire strobe and is actually **electrically superior**.
