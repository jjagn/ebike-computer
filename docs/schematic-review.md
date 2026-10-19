# Schematic review and fixes

Review of the e-bike computer schematic (STM32F405, 10S Li-ion input), and the changes made to resolve it.

- **Pre-fix snapshot:** `d89a280` on `main`
- **Fixes:** `176e56e` on branch `schematic-review-fixes`
- **Battery:** 10S Li-ion, 36–42.0 V on net `VCC` (J8/J9)

## How the review was done

- Netlist, BOM and ERC were exported with `kicad-cli` (KiCad 10.0.3). Every finding was checked against the netlist, not the drawings.
- Findings were checked against the primary datasheets:
  - TI LM65625/35/45 (SNVSCH3) and its -Q1 version (SNVSCH2A)
  - TI TCAN33x (SLLSEQ7F), TCAN1044A-Q1 (SLLSFJ3D), TPS2116 (SLVSFG1A)
  - ST STM32F405 datasheet (Rev 4), AN4488, AN2867
  - USB 2.0 §7.2.4.1 and USB Type-C R2.0 §4.5.2.2.3.1
  - Bourns SMBJ datasheet
  - LCSC, JLCPCB and DigiKey part pages
- Each finding was confirmed by at least two independent sources. A separate verification pass re-checked every claim that rested on a single source.
- Three initial suspicions were **refuted**:
  - **LM65625 FB strapping is correct.** FB→GND gives 3.3 V and FB→VCC gives 5 V (Table 5-1).
  - **R3–R5 are fitted.** Only R6/R7 are DNP. The original BOM export merged them into one "R3–R7 DNP" line.
  - **The CAN TX/RX straps are the right way round.** R3/R4 are the straight mapping: PB8 = CAN1_RX, PB9 = CAN1_TX (DS Table 9).

**Result after the fixes:**
- ERC: 6 messages, down from 55. All six are pre-existing and intentional; see [Remaining ERC items](#remaining-erc-items).
- About 120 connectivity assertions pass against KiCad's exported netlist, covering every fix below.

## Findings and fixes

### Critical: the board would not have worked

| # | Finding | Evidence | Fix |
|---|---|---|---|
| 1 | Buck input caps C3–C6 and C20–C23 were in series between `VCC` and VIN. There was no DC path to the regulators. | Netlist: `VCC` held only cap pin 1s and J8/J9. VIN1/VIN2/EN sat on the other cap pins. | Caps now go from VIN to GND. U4 VIN is fed from `VIN_PROT` and U3 VIN from `+5V`. |
| 2 | The LM656x5 exposed pad (DAP) was unconnected. | The symbol GND pin had an empty number, so footprint pad 21 and vias 22–27 had no net. Datasheet Table 5-1 and §8.5.1.1 require the DAP on GND. | Project symbol numbers the pin `21`. The new footprint numbers every thermal-pad copper and via as `21`. |
| 3 | The HSE crystal Y2 had one terminal shorted to GND. | The 3-pin `Crystal_GND2_Small` symbol sat on a 2-pad footprint (C115962 is 2-pad 5032), so pad 2 went to GND. | `Device:Crystal_Small` (2-pin). Nets renamed `OSC32_*` → `HSE_*`, because PH0/PH1 is the 8 MHz oscillator, not the 32 kHz one. |
| 4 | GNDA (VSSA, C27, C28) was floating. | The net held only those three pins. STM32 DS §3: "VSSA must be connected to VSS". | Tied to GND. |
| 5 | USB D+/D− were not connected to the MCU. | Neither sheet symbol had a `USB{USB}` sheet pin (ERC `hier_label_mismatch`). | Added sheet pins on both sheet symbols and a bus between them. |
| 6 | Output caps were 94 µF and 55 µF in 0402, which can't be bought. | The values came from the LM6xx35 column of TI Table 8-1. They are *derated* totals. | 4× 47 µF 10 V 1206 per rail, giving ≥75 µF (3V3) / ≥50 µF (5V) effective per the LM6xx25 column. |
| 7 | Input caps were 4.7 µF and 100 nF in 0402 on a 42 V input. | The TI design example uses 100 V parts. No 0402 4.7 µF part is rated anywhere near 42 V. | U4 (42 V input): 4.7 µF 100 V 1210 and 100 nF 100 V 0603. U3 (now 5 V input): 4.7 µF 25 V 0805 and 100 nF 0402. |
| 8 | `LM65625SRZTR` is not an orderable part. | It is absent from the datasheet's orderable addendum, and the TI part page returns 404. | `LM65625SRZTRQ1`, which has an identical pinout (SNVSCH2A Table 5-1). |

### Major

| # | Finding | Fix |
|---|---|---|
| 9 | USB VBUS was tied straight to the +5V buck output. That back-drives the host, which Type-C forbids for a sink, and puts about 50–150 µF on VBUS against the USB 2.0 limit of 10 µF. | VBUS is its own net. A TPS2116 power mux (U5) feeds `+5V` from the buck when it is up, and from VBUS otherwise; it blocks backfeed into the host. USBLC6-2SC6 ESD (U6), 4.7 µF on VBUS (C30), and a VBUS-sense divider 22k/33k into PA9. |
| 10 | No battery input protection. The regulator's limits are 72 V absolute, 65 V recommended DC, and 70 V for under 100 ms. | F1 60 V PTC, D4 SS210 100 V reverse-polarity Schottky, D5 SMBJ43A TVS (Vc 69.4 V, below 72 V), C37 47 µF/100 V bulk to damp cable ringing (TI §8.4). This creates the protected rail `VIN_PROT`. |
| 11 | BOOT0 floated, because SW1/SW2 had no footprints and SW2 was the only path. | R16 10k pull-down, R2 changed to 1k so the switch still pulls BOOT0 high, SW2 → latching SPDT. Footprints are XKB TS-1187A and Shouhan MSK12C02. |
| 12 | Crystal load caps were 15 pF for a crystal needing a 20 pF load, and the assigned part C1644 is 0603 on an 0402 footprint. | 33 pF C0G 0402 (C1562), from AN2867: 2 × (20 − 3…5 pF). |
| 13 | The inductors were also taken from the LM6xx35 column. | L2 → 10 µH (SRG0630-100-M). L1 stays at 6.8 µH: with U3 now running from 5 V, that gives about 16% ripple, and the larger L helps stability at 66% duty. |
| 14 | TCAN330 had no decoupling capacitor. | C29 100 nF at pin 3. |
| 15 | The custom `Jackson:RZT0020A-MFG` footprint did not match TI's land pattern. Its copper polygons had no net, and the library name's case did not match. | New project footprint `ebike-controller:TI_RZT0020A`, built from TI drawing 4228518/C: L-shaped corner pads, notched EP, 2-aperture paste (85%), and 6 tented Ø0.2 mm vias. |

### Minor

| Finding | Fix |
|---|---|
| Blue LED (1k) and true-green LED (Vf 3.1 V) would be dim or vary widely. | D2 → yellow-green KT-0603YG (Vf about 2.1 V). R10 → 68 Ω for the blue LED. |
| J2 exported raw GPIOs. PA4 is TTa, so not 5 V tolerant. | R19–R23 100 Ω series, U7 SP0505BAHT 5-channel ESD, and a sheet note about PA4. |
| Exported +5V/+3V3 rails had no protection. | F2 0.5 A and F3 0.35 A PTCs feed `+5V_EXT` / `+3V3_EXT`. |
| EN was tied to VIN, so there was no battery undervoltage cutoff. | R24 232k / R25 10k on U4 EN: on at 30.3 V, off at 24.2 V. |
| PG outputs were unused. | U3 PG drives `NRST` as open-drain, with R1 pulling up to +3V3 (NRST is not 5 V tolerant). |
| TCAN330 S/SHDN pins floated. TI says not to rely on the internal pull-downs. | Both tied to GND. |
| CAN termination was hard-fitted. | JP1 solder jumper (bridged by default) in series with R8. Cut it for a mid-bus node. |
| 27 unused MCU pins had no no-connect flags. | Flags added. PA9 is now used for VBUS sense. |
| CAN/SPI netclasses were undefined, and the display bus pulled +3V3/GND into the SPI class. | Classes defined, and the power nets removed from SPI. |
| C1 was 10 µF in 0402 and keeps only about 30–40% of that at 3.3 V. | 0805 25 V part (C15850). |
| The stock symbol text said "MAX INPUT 75V". | Changed to "VIN MAX 65V". |

## Architecture after the fixes

```
J8/J9 VCC (10S, 36-42V)
  └─ F1 PTC ─ D4 Schottky ─ VIN_PROT ─┬─ D5 TVS, C37 bulk
                                      └─ U4 LM65625 (UVLO 30.3/24.2V) ─ 5V_BUCK ─┐
USB-C VBUS ─ U6 ESD ─────────────────────────────────────────────────────────────┤
                                                       U5 TPS2116 (VIN1 priority) ┘
                                                         └─ +5V ─┬─ J2/J10 via F2
                                                                 └─ U3 LM65625 ─ +3V3 ─ MCU, CAN, display
```

- U3 is cascaded from +5V, so the MCU runs from either the battery or USB.
- The TPS2116 selects VIN1 (the buck) when PR1 is above about 4.3 V. It uses a 33k/10k divider, VREF 0.92–1.08 V.
- The TPS2116 blocks reverse current into VBUS: IREV is 0.15 µA max at 105 °C.

## Project library changes

KiCad's stock `Converter_DCDC:LM656x5` symbol has several errors: a blank exposed-pad number, wrong pin types on PG/RT/MODE/VCC, and wrong "75V" text. Editing only the copy embedded in the schematic would be undone by "Update symbols from library". So the fixes live in project libraries:

- **`ebike-controller.kicad_sym`:**
  - `LM656x5` with the EP numbered `21`.
  - PG changed to open-collector, RT and MODE/SYNC to input, VCC to power-out, and SW2 to passive (it is the same switch node as SW1).
  - `TPS2116`, drawn from SLVSFG1A Table 5-1 because KiCad has no stock symbol.
- **`ebike-controller.pretty/TI_RZT0020A.kicad_mod`:** the regulator footprint.
- **`sym-lib-table` and `fp-lib-table`:** project library tables.

The stock symbol bug is worth reporting to the KiCad library maintainers.

## BOM and sourcing

- Every BOM line has `Manufacturer`, `Manufacturer Part Number` and `LCSC Part Number` fields.
- Values are normalised (no more `100nf` / `100nF` / `0.1uF` splits). The empty `Field5`, `Digikey Part Number` and `Mouser Part Number` fields were removed.
- JP1 and J3 are PCB-only and excluded from the BOM.
- When exporting the BOM, group by `${DNP}` as well as Value/Footprint, so DNP parts get their own line.

**Supply notes** (stock checked 2026-10-02):

| Part | Note |
|---|---|
| U3/U4 LM65625SRZTRQ1 | 0 at JLC/LCSC (C49305977), about 4,000 at DigiKey. The pin-compatible LM65645SRZTRQ1 (C49256228) is at JLC, but it needs different L/COUT values (Table 8-1). |
| U2 TCAN330DR | Only 18 at LCSC (C2652876), about 3,400 at DigiKey. |
| C7, C26, C31–C36, C38 (47 µF 1206, C96123) | 0 at LCSC, about 300k at JLC (basic part). |
| J4 USB4105-GF-A (C3020560) | 0 at LCSC, 1,224 at JLC. |
| Molex Micro-Fit 43650-0800 / -0200 | No LCSC code. DigiKey WM2424-ND / WM1860-ND. Mating housings are 43645-0800 / -0200, crimps 43030-0007 (20–24 AWG) and 43030-0038 (18 AWG, for the battery). |
| SWD cable | Tag-Connect TC2030-CTX. A tool, not a BOM item. |

## Remaining ERC items

The final ERC reports 6 messages: 2 errors and 4 warnings.

| Item | Why it remains |
|---|---|
| `F_LED_+` / `R_LED_+` unconnected sheet pins and labels (2 errors, 2 warnings) | The LED Drivers sheet is an unpopulated placeholder. |
| `display.VCC` / `display.GND` multiple net names (2 warnings) | Intended: the display bus aliases those members onto +3V3/GND. |

## Open items

- **Update the PCB.** Run Update PCB from Schematic (F8). The fix commit did not touch the board file. About 40 new parts need placing, and U3/U4 move to the new footprint.
- **CAN short to battery.** The TCAN330 (kept by choice) survives only ±14 V on its bus pins, but CAN shares a harness with 42 V. A harness short to battery will destroy it. TCAN1044AV (±58 V) is the robust option. It is not a drop-in: it needs 5 V VCC, pin 5 becomes VIO, and pin 8 becomes STB with a pull-up.
- **J2 pin order.** +5V is still next to +3V3. Changing it would change the harness.
- **Display SPI resistors.** Series resistors (22–33 Ω on SCK/MOSI) are not fitted. These were optional.
- **Thermal vias.** They use TI's Ø0.2 mm drill. Check that the fab supports it, or increase it to 0.3 mm.
- **Fuse choice.** F1 is a PTC because a fast-blow 1206 fuse (I²t about 0.04 A²s) would likely trip on hot-plug inrush into C37.
- **Cosmetic.** R16's GND label slightly overlaps R11's reference on the MCU sheet.
