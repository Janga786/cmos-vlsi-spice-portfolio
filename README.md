# CMOS VLSI SPICE Netlists

Schematic-level CMOS designs from my ENGR 338 course at Fort Lewis
College (fall 2025; the Electric library is named `ENGR-338`). I drew the
schematics in Electric VLSI 9.08 and exported these SPICE decks. They use
BSIM3 models for the AMI C5 process (5 V, 0.6 µm drawn channel length,
`docs/mosis_c5_process_models.txt`). All results are from simulation;
nothing here was laid out for fabrication or built.

## Designs

| Folder / file | What it is | Transistors |
| --- | --- | ---: |
| `building-blocks/inverter.spi` | Inverter with a pulse input, transient run | 2 |
| `building-blocks/nand2.spi`, `nor2.spi` | 2-input NAND and NOR test decks | 4 each |
| `building-blocks/or2_testbench.spi` | OR gate (NOR + inverter) test deck | 6 |
| `building-blocks/xor2.spi`, `xor2_testbench.spi` | Static CMOS XOR, netlist and test deck | 12 |
| `building-blocks/inverter_dc_sweep.spi` | Inverter netlist only (no sources or analysis) | 2 |
| `building-blocks/transistor_iv_curves.spi` | Inverter transient run (deck "Lab4"); not an I-V sweep despite the name | 2 |
| `full-adder/full_adder.spi` | Gate-level full adder (NAND, NOR, XOR, inverter) | 42 |
| `full-adder/high_speed_full_adder.spi` | Mirror full adder | 28 |
| `full-adder/8bit_ripple_adder_high_speed.spi` | 8-bit ripple-carry adder of mirror full adders | 224 |
| `8bit-alu/alu.spi` | 8-bit ALU: AND, OR, add, and subtract (B inverted through a mux) | 438 |
| `8bit-mux/mux_8to1.spi` | 8-bit-wide 2-to-1 transmission-gate mux (not 8-to-1, despite the file name) | 34 |
| `ring-oscillator/ring_oscillator.spi`, `ring_oscillator_bus.spi` | 11-stage inverter ring; the bus version names every stage output | 22 in the loop |
| `r2r-adc-dac/adc_r2r_dac.spi` | Ideal (behavioral) 10-bit ADC feeding a 10-bit R-2R resistor-ladder DAC, 2 MHz sine input | 0 |
| `r2r-adc-dac/adc_decay_test.spi` | Step on the R-2R ladder MSB into a 10 pF load, to see the output settle | 0 |

Transistor counts are per design under test, counted from the netlists.
`ring_oscillator.spi` also contains 20 inverters that are not connected to
the ring.

The two `r2r-adc-dac` decks carry an Electric creation date of May 2008,
so they started from existing course example cells (the `Ideal_*` blocks)
rather than from scratch.

## Running a deck

The decks include the model file as `C5_models.txt` from their own folder.
With ngspice:

```bash
cp docs/mosis_c5_process_models.txt full-adder/C5_models.txt
cd full-adder && ngspice high_speed_full_adder.spi
```

Then `plot` the node of interest. The two `r2r-adc-dac` decks need no model
file. `transistor_iv_curves.spi` names its devices `N` and `P`, while the
model file defines `NMOS` and `PMOS`, so that deck needs the names changed
to run.

## License

[MIT](LICENSE). The C5 model card in `docs/` is third-party process data,
not my work.
