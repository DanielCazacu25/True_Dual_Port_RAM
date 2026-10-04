# True Dual-Port RAM: Architecture and Write-First Behaviour

## Project Overview
This project implements a **True Dual-Port RAM (8x8 bits)** in Verilog: one storage array shared by two fully independent ports, each on its own clock, with **write-first** behaviour on each port.

It was built as one of the steps towards understanding an asynchronous FIFO, alongside a synchronous FIFO ([RAM_FIFO_Project](https://github.com/DanielCazacu25/RAM_FIFO_Project)) and the Gray-code converters in the [ALU](https://github.com/DanielCazacu25/UVM_based_ALU_testbench). Along the way my focus shifted from design to verification, and the asynchronous FIFO was eventually approached from the verification side instead ([UVM_Based_Async_FIFO_Testbench](https://github.com/DanielCazacu25/UVM_Based_Async_FIFO_Testbench)), on a separate design. Note that the RAM itself does not perform clock-domain-crossing synchronization: it only provides storage that both clock domains can access; in an asynchronous FIFO the synchronization (Gray-coded pointers, two-flip-flop synchronizers) lives outside the memory.

## Simple vs. True Dual-Port RAM (Architecture Comparison)
This project is a significant architectural upgrade from the previous `Classic_RAM8_8` design (in [RAM_FIFO_Project](https://github.com/DanielCazacu25/RAM_FIFO_Project)):
* **Port Autonomy:** The previous version was a *Simple Dual-Port RAM*, featuring one dedicated write port and one dedicated read port. This new *True Dual-Port* architecture features two completely independent ports (Port A and Port B). Both ports can read and write to any memory address simultaneously.
* **Clock Domains:** The old design shared a single global clock. This true dual-port implementation supports two independent clocks (`clk_a` and `clk_b`), allowing the memory to bridge two systems running at different frequencies.
* **Initialization:** The legacy design used a global asynchronous reset (`rst`) that cleared the whole memory array. This project removes it, mirroring FPGA Block RAM (BRAM), whose contents cannot be cleared by a reset signal: a BRAM is initialised by the bitstream instead (zeros by default), and a reset that touches every word would prevent the array from mapping onto a BRAM at all. As a consequence, the system must write an address before reading it.

## Features & Hardware Logic

### Verilog Design
* **Write-First per port:** each port has a single address and either reads or writes on a given edge. On a write, the port's output register is loaded directly from `data_in` (bypass), so the output shows the newly written data instead of the location's previous contents. (A read-first variant would load `data_out` from `memory[addr]` in the write branch.)
* **No Global Reset:** in simulation the memory starts as undefined (`X`), so every location must be written before it is read. (On real hardware an ASIC SRAM powers up with arbitrary contents, while an FPGA BRAM starts from its bitstream initial values.)

### Testbench Strategy
* **Asynchronous Clocks:** The testbench drives `clk_a` with a 10ns period and `clk_b` with a 16ns period to verify independent operation.
* **Cross-port communication:** one port writes an address and the other reads it later on its own clock (e.g. Port A writes `AA` to address 2, Port B reads it back; Port B writes `99` to address 7, Port A reads it back).
* **Write-first check:** each port writes at least once, so the bypass behaviour is visible on its output.
* **Negative Edge Stimuli:** inputs are driven on the falling edge (`negedge clk`) so that they are stable well before the rising edge that samples them, which avoids races between the testbench and the design in simulation (RTL simulation has no setup time as such).
* **Directed, non-self-checking:** results are judged from the `$monitor` log and the waveform; there is no automatic comparison against expected values.

## Waveform Analysis

![Waveform Simulation](results/waveforms.png)
![TCL Console](results/TCL_Console.png)

The waveform simulation effectively demonstrates the correct behavior of the dual-port architecture, specifically highlighting the following hardware realities:

1. **Initial Undefined States (`XX` / Unknowns):** At the beginning of the simulation, the read outputs (`data_out_a` and `data_out_b`) frequently display `XX` (highlighted in red). This happens because there is intentionally no global reset. Therefore, any attempt to read an address *before* explicitly writing data to it will route this "garbage" data to the output. The outputs only resolve to valid hexadecimal values once a Write Enable (`we`) signal explicitly overwrites the `X` states at those specific addresses.
2. **Write-First Bypass (Port B):** When Port B writes `BB` to address 5, `data_out_b` shows `BB` from that same clock edge, instead of the location's previous contents (`XX`). This is the write-first bypass at work.
3. **Seamless Cross-Port Communication:** The waveform shows Port B writing the value `99` to address `7`. Shortly after, Port A reads address `7` and successfully outputs `99`. This works seamlessly because both ports, despite having completely independent clocks and control logic, can access the exact same physical storage matrix (`reg [7:0] memory [0:7]`).

## Project Structure

| Folder/File | Description |
| :---------- | :---------- |
| `Design/` | The True Dual-Port design implementation. |
| `Testbench/` | Contains the testbench simulation. |
| `results/` | Contains waveforms and TCL Console images. |

## How to Run

1. Open Vivado and create a new project.
2. Add `Design/RAM.v` as a design source and `Testbench/RAM_tb.v` as a simulation source.
3. Run behavioral simulation; the testbench ends itself with `$finish` (about 0.6 µs). Check the Tcl Console for the `$monitor` log and the waveform viewer for the signals.

## Known limitations

* **Cross-port collisions are not handled.** If both ports write the same address on the same instant, the two `always` blocks race: in simulation the last update wins, depending on the simulator, and on a real BRAM the stored data is undefined. Likewise, a read on one port of an address being written by the other port at the same instant returns an undefined value on hardware. Typical solutions are a fixed port priority, a system-level rule that forbids the case, or an assertion that flags it. With the testbench's clock periods (10 ns and 16 ns) the rising edges never coincide, so these cases are not exercised.
* **The testbench is directed and not self-checking,** with a fixed scenario of eight accesses.