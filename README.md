<div align="center">

# CMOS VLSI SPICE Portfolio

**Transistor-level CMOS designs in MOSIS 0.6 µm SCMOS, exported as Berkeley-SPICE 3 / NgSpice / HSPICE-compatible netlists from Electric VLSI: an 8-bit ALU, a high-speed 8-bit ripple-carry adder, an R-2R DAC + flash ADC, a ring oscillator, an 8-to-1 MUX, plus the canonical building blocks (inverter, NAND, NOR, XOR with both static-CMOS and pass-transistor implementations).**

[![SPICE](https://img.shields.io/badge/format-Berkeley%20SPICE%203-3a86ff.svg)](https://en.wikipedia.org/wiki/SPICE_(simulator))
[![Process](https://img.shields.io/badge/process-MOSIS%200.6%20%C2%B5m%20SCMOS-9b59b6.svg)](#)
[![Tool: Electric VLSI](https://img.shields.io/badge/tool-Electric%20VLSI%209.08-orange.svg)](https://www.staticfreesoft.com/)
[![License: MIT](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)

</div>

---

## Contents

- [Overview](#overview)
- [The designs](#the-designs)
- [The MOSIS C5 process](#the-mosis-c5-process)
- [Running a simulation](#running-a-simulation)
- [Repository layout](#repository-layout)
- [License](#license)

---

## Overview

Every design in this repository is described at the **transistor
level** — every NMOS / PMOS device is sized explicitly with `L=0.6U`
(channel length) and `W` (channel width) parameters, every internal
node is named, and every test bench instantiates real device models
rather than ideal switches.  This is the level at which a chip
design actually exists; everything above it (gate-level Verilog,
RTL, behavioural HDL) is an abstraction that the synthesis tools
ultimately compile back down to *exactly this representation*.

Designs were captured schematically in **Electric VLSI Design System
9.08** (free, open-source IC layout tool from Static Free Software)
targeting the **MOSIS C5 0.6 µm SCMOS** PDK, then exported as
plain-text SPICE netlists.  The netlists in this repository are
self-contained and run on any SPICE 3-compatible simulator —
Berkeley SPICE, NgSpice, LTspice, or HSPICE — provided you point
them at a transistor model file (the included
`docs/mosis_c5_process_models.txt` is the corresponding C5 model
card).

---

## The designs

### 1. `building-blocks/` — static-CMOS gate library

Every higher-level circuit in this repository is composed of these
six gates:

| Cell | Topology | Transistors | What you can verify |
| --- | --- | ---: | --- |
| `inverter.spi`               | classic CMOS inverter (1 PMOS + 1 NMOS) | 2 | rail-to-rail output, sub-ns prop. delay |
| `inverter_dc_sweep.spi`      | inverter with a DC sweep test bench       | 2 | the famous V<sub>in</sub>/V<sub>out</sub> transfer curve |
| `nand2.spi`                  | 2-input NAND (2 series NMOS, 2 parallel PMOS) | 4 | symmetric rise/fall, V<sub>OL</sub> ≈ 0 |
| `nor2.spi`                   | 2-input NOR (2 parallel NMOS, 2 series PMOS) | 4 | inherently slower P-side stack |
| `xor2.spi`                   | 2-input XOR using pass-transistor logic   | 8 | non-rail-to-rail behaviour, threshold drop |
| `xor2_testbench.spi`         | 2-input XOR with a 4-corner test bench     | — | full truth-table verification |
| `or2_testbench.spi`          | 2-input OR (built from NOR + inverter)     | 6 | propagation through 2 stages |
| `transistor_iv_curves.spi`   | single NMOS/PMOS DC sweep                  | 1 | I<sub>D</sub>-V<sub>DS</sub> family of curves at varying V<sub>GS</sub> |

Every gate uses the **standard ratio** of W<sub>P</sub> = 2 × W<sub>N</sub> = 6 µm to compensate for the
~2× lower hole mobility in the PMOS device, giving roughly equal
rise- and fall-times.

### 2. `full-adder/` — three increasingly performant adders

| File | Architecture | Notes |
| --- | --- | --- |
| `full_adder.spi`                       | gate-level full adder (sum = A ⊕ B ⊕ Cin via XOR cascade; carry-out from majority gate) | Reference implementation. ~28 transistors. |
| `high_speed_full_adder.spi`            | mirror-adder topology — sum and carry computed in *parallel* from the same input minterms, both rail-to-rail in 14 transistors | About 35 % faster than the gate-level version at the same node sizes. |
| `8bit_ripple_adder_high_speed.spi`     | 8-bit ripple chain of the mirror adders | Demonstrates worst-case carry-propagation delay (the "longest path" through the adder). |

The high-speed full adder is the standout — at 14 transistors it
beats the textbook 28-transistor gate-level adder both on speed and
on layout area.  The trick is sharing input minterms between the
sum and carry-out paths instead of computing them separately.

### 3. `8bit-alu/` — an 8-bit Arithmetic Logic Unit

`alu.spi` — a fully-elaborated 8-bit ALU built bottom-up from the
gates above.  Operations include:

- bitwise **AND**, **OR**, **XOR**, **NOR** (parallel 8-bit gates);
- **add** and **subtract** (sub uses an inverted B-input + Cin = 1, the
  classical two's-complement trick);
- **invert** and **shift left**.

Total transistor count is well into the four-digit range and the
SPICE deck is ~9 kB once expanded.  This file demonstrates how a
designer composes hierarchical SPICE: the top-level ALU instantiates
the 8-bit ripple adder, which instantiates eight high-speed full
adders, each of which instantiates two ANDs + an XOR + a majority
gate, etc.

### 4. `r2r-adc-dac/` — mixed-signal: an R-2R DAC and a flash ADC

| File | Block | What it shows |
| --- | --- | --- |
| `adc_r2r_dac.spi`           | full **A/D + D/A** conversion chain — sample & hold, comparator-bank ADC, R-2R ladder DAC, ideal op-amps for buffer stages | classical resistor-only DAC: each successive bit-cap is *exactly* twice the previous, no precision-trim needed |
| `adc_decay_test.spi`        | the same chain stressed with a slowly-varying analog input to expose **leakage and decay** in the sample-and-hold capacitor | how reliable is the conversion when the hold time exceeds the design margin? |

### 5. `ring-oscillator/` — process-corner characterisation

`ring_oscillator.spi` — a classic odd-stage inverter ring (e.g. 5
or 7 inverters in a closed loop, output of the last fed back to
the input of the first).  The natural oscillation frequency is

$$
f_{\mathrm{osc}} \;=\; \frac{1}{2 N \cdot t_{\mathrm{p}}}
$$

where *N* is the number of inverter stages and *t*<sub>p</sub> is the
mean inverter propagation delay.  Sweeping *N* on a fabricated
ring lets you back-out the actual *t*<sub>p</sub> of the process — the
canonical first-pass characterisation for a new wafer.

`ring_oscillator_bus.spi` is a 7-stage variant with the internal
nodes broken out as a bus, so a single transient sweep visualises
all seven phases simultaneously (a 7-phase clock generator).

### 6. `8bit-mux/` — a 1-of-8 selector

`mux_8to1.spi` — an 8-to-1 multiplexer built as a tree of three
levels of 2-to-1 transmission-gate muxes.  Demonstrates the
trade-off between transmission-gate logic (smaller, faster, but
non-restoring — long chains of TGs accumulate threshold drops)
and static-CMOS logic.

---

## The MOSIS C5 process

All designs target the **MOSIS C5** SCMOS PDK — a 0.6 µm 2-poly
3-metal CMOS process.  Key parameters from the
`docs/mosis_c5_process_models.txt` model card:

| Parameter | NMOS | PMOS |
| --- | ---: | ---: |
| Threshold voltage V<sub>T</sub>     | ~0.7 V | ~−0.9 V |
| Mobility μ<sub>n,p</sub>            | ~ 600 cm²/V·s | ~ 250 cm²/V·s |
| Gate oxide t<sub>OX</sub>           | ~ 14 nm |
| Channel length L<sub>min</sub>      | 0.6 µm |
| Supply V<sub>DD</sub>                | 5 V |

This process is the standard educational target — old enough to
hand-layout in Electric without a 100 GB PDK, modern enough that
all the design rules an engineer needs to know (W/L sizing,
parasitic extraction, body-bias effects, transmission-gate
non-idealities) carry over directly to deep-submicron nodes.

---

## Running a simulation

Any SPICE 3-compatible simulator works.  With **NgSpice** (free,
open-source, available via Homebrew):

```bash
brew install ngspice         # macOS
sudo apt install ngspice     # Debian / Ubuntu

# point the netlist at the process model card and run a transient
ngspice -b -i building-blocks/inverter.spi  -o out.txt
ngspice -b -i 8bit-alu/alu.spi               -o alu_results.txt
```

To **plot** waveforms interactively, drop the `-b` flag:

```bash
ngspice ring-oscillator/ring_oscillator.spi
ngspice> plot v(out)
```

For **LTspice**, open the `.spi` file directly — LTspice parses the
Berkeley-SPICE netlist syntax used here without modification.

To **edit** the schematics, install **Electric VLSI 9.08+** (free,
Java-based, runs anywhere) and open the `.jelib` library files
(included alongside `Saved_VLSI_Lab2/` in the original Electric
project, not redistributed here because they reference the MOSIS
PDK by file path).  Re-export Spice 3 deck after edits via
*Tools → Simulation (Spice) → Write Spice Deck*.

---

## Repository layout

```
cmos-vlsi-spice-portfolio/
├── building-blocks/
│   ├── inverter.spi                       # classic 2-transistor CMOS inverter
│   ├── inverter_dc_sweep.spi              # inverter with DC sweep test bench
│   ├── nand2.spi                          # 2-input NAND
│   ├── nor2.spi                           # 2-input NOR
│   ├── xor2.spi                           # 2-input XOR (pass-transistor)
│   ├── xor2_testbench.spi                 # XOR with 4-corner test bench
│   ├── or2_testbench.spi                  # OR (NOR + inverter cascade)
│   └── transistor_iv_curves.spi           # single-device DC characterisation
│
├── full-adder/
│   ├── full_adder.spi                      # gate-level reference adder
│   ├── high_speed_full_adder.spi           # mirror-adder, 14 transistors
│   └── 8bit_ripple_adder_high_speed.spi    # 8-bit ripple chain
│
├── 8bit-alu/
│   └── alu.spi                             # AND/OR/XOR/NOR/ADD/SUB/INV/SHL
│
├── r2r-adc-dac/
│   ├── adc_r2r_dac.spi                     # full ADC + DAC chain
│   └── adc_decay_test.spi                  # sample-and-hold leakage test
│
├── ring-oscillator/
│   ├── ring_oscillator.spi                 # standard odd-stage ring
│   └── ring_oscillator_bus.spi             # 7-stage with broken-out nodes
│
├── 8bit-mux/
│   └── mux_8to1.spi                        # 3-level transmission-gate tree
│
├── docs/
│   └── mosis_c5_process_models.txt         # MOSIS C5 NMOS/PMOS model cards
│
├── README.md
└── LICENSE
```

---

## License

[MIT](LICENSE) — netlists are free for educational, academic and
commercial reference use.

The MOSIS C5 process model card is © MOSIS / Static Free Software
and redistributed here under the standard educational-use terms.
