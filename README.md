# UART Controller — Verilog Implementation

[![Language](https://img.shields.io/badge/Language-Verilog-blue)](https://img.shields.io/badge/Language-Verilog-blue) [![Simulator](https://img.shields.io/badge/Simulator-Icarus%20Verilog-green)](https://img.shields.io/badge/Simulator-Icarus%20Verilog-green) [![Status](https://img.shields.io/badge/Status-Verified-brightgreen)](https://img.shields.io/badge/Status-Verified-brightgreen) [![Baud Rate](https://img.shields.io/badge/Baud%20Rate-9600-orange)](https://img.shields.io/badge/Baud%20Rate-9600-orange)

A fully functional **UART (Universal Asynchronous Receiver Transmitter)** controller implemented in Verilog HDL. The design includes a baud rate generator, FSM-based transmitter, independent-counter-based receiver with middle sampling, top-level integration module, and a self-checking testbench. Verified using Icarus Verilog simulator with waveform analysis in GTKWave.

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Module Description](#module-description)
- [Design Specifications](#design-specifications)
- [Key Design Decisions](#key-design-decisions)
- [Simulation Results](#simulation-results)
- [How to Run](#how-to-run)
- [Directory Structure](#directory-structure)
- [Tools Used](#tools-used)
- [Synthesis Results](#synthesis-results)
- [Future Improvements](#future-improvements)
- [Waveform](#waveform)

---

## Overview

UART is a serial communication protocol that transmits data one bit at a time over a single wire. Unlike synchronous protocols (SPI, I2C), UART requires no shared clock — both transmitter and receiver agree on a baud rate in advance and maintain independent timing.

This implementation follows the standard UART frame format:

```
IDLE → START(0) → D0 → D1 → D2 → D3 → D4 → D5 → D6 → D7 → STOP(1) → IDLE
  1  →    0    → LSB                                       → MSB →   1
```

- **START bit** — always 0, signals beginning of frame
- **8 data bits** — LSB transmitted first
- **STOP bit** — always 1, signals end of frame
- **IDLE** — line held HIGH when no transmission

---

## Architecture

```
               ┌─────────────────────────────────────────┐
               │              uart_top.v                 │
               │                                         │
    clk ──────→│──→ baud_rate ──baud_tick──→ uart_tx ────│──→ tx_line ──┐
  start ──────→│                         ↑               │              │
tx_data ──────→│                      clk, start         │              │
               │                      tx_data            │              │
               │                                         │              │
rx_data ←──────│←── uart_rx ←──────────────────────────← │←─────────────┘
   done ←──────│         ↑                               │
               │      clk, rx_line                       │
               │   (independent counter)                 │
               └─────────────────────────────────────────┘
```

**Key design choice:** The receiver uses an **independent internal counter** that resets on start bit detection, enabling accurate middle-of-bit sampling without depending on the transmitter's baud_tick signal.

---

## Module Description

### 1. `uart_baud_rate.v` — Baud Rate Generator

Generates a `baud_tick` pulse every 5208 clock cycles (9600 baud at 50MHz).

```
cycles_per_bit = clock_frequency / baud_rate
               = 50,000,000 / 9600
               = 5208 cycles
```

| Port        | Direction | Width | Description                   |
| ----------- | --------- | ----- | ----------------------------- |
| `clk`       | input     | 1     | 50MHz system clock            |
| `baud_tick` | output    | 1     | Pulses HIGH every 5208 cycles |

---

### 2. `uart_tx.v` — UART Transmitter

FSM-based transmitter with 11 states. Converts 8-bit parallel data into a serial bit stream.

| Port        | Direction | Width | Description                           |
| ----------- | --------- | ----- | ------------------------------------- |
| `clk`       | input     | 1     | System clock                          |
| `baud_tick` | input     | 1     | Timing pulse from baud rate generator |
| `start`     | input     | 1     | Trigger signal to begin transmission  |
| `data`      | input     | 8     | Parallel data to transmit             |
| `tx_line`   | output    | 1     | Serial output line                    |
| `idle`      | output    | 1     | HIGH when transmitter is free         |

**FSM States:**

```
State 0  → IDLE   : tx_line = 1, wait for start=1
State 1  → START  : tx_line = 0, UART start bit
State 2  → D0     : tx_line = data[0]
...
State 9  → D7     : tx_line = data[7]
State 10 → STOP   : tx_line = 1, UART stop bit → return to IDLE
```

---

### 3. `uart_rx.v` — UART Receiver (Independent Counter)

Receives serial data and reconstructs 8-bit parallel output. Uses an **independent internal counter** for accurate middle-of-bit sampling.

| Port      | Direction | Width | Description                  |
| --------- | --------- | ----- | ---------------------------- |
| `clk`     | input     | 1     | System clock                 |
| `rx_line` | input     | 1     | Serial input line            |
| `data`    | output    | 8     | Reconstructed parallel data  |
| `done`    | output    | 1     | HIGH when full byte received |

**Middle Sampling Strategy:**

```
Start bit detected (rx_line=0)
        ↓
rx_count resets to 0
        ↓
rx_count == 2603 → sample (middle of bit) ✅
rx_count == 5207 → advance state, reset counter
```

---

### 4. `uart_top.v` — Top Level Integration

| Port      | Direction | Width | Description             |
| --------- | --------- | ----- | ----------------------- |
| `clk`     | input     | 1     | System clock            |
| `start`   | input     | 1     | Transmission trigger    |
| `tx_data` | input     | 8     | Data to transmit        |
| `rx_data` | output    | 8     | Received data           |
| `done`    | output    | 1     | Reception complete flag |

---

### 5. `uart_tb.v` — Testbench

Self-checking testbench that generates 50MHz clock, drives `tx_data = 8'b00010110`, triggers transmission, monitors `done` and `rx_data`, and dumps VCD for GTKWave analysis.

---

## Design Specifications

| Parameter       | Value                   |
| --------------- | ----------------------- |
| Clock Frequency | 50 MHz                  |
| Baud Rate       | 9600                    |
| Data Bits       | 8                       |
| Stop Bits       | 1                       |
| Parity          | None                    |
| Cycles per Bit  | 5208                    |
| Bit Duration    | 104.16 µs               |
| Frame Duration  | 1,145,760 ns (~1.14 ms) |
| HDL Standard    | Verilog                 |

---

## Key Design Decisions

### FSM for Transmitter
The transmitter uses an 11-state FSM driven by `baud_tick`. Each state holds the TX line value for exactly one bit period.

### Independent Counter for Receiver
The receiver maintains its own 13-bit counter (`rx_count`) that resets upon detecting the falling edge of `rx_line`. Samples at `rx_count == 2603` (halfway through each bit).

### Non-Blocking Assignments
All sequential logic uses `<=` inside `always @(posedge clk)` blocks, correctly modeling flip-flop behavior.

---

## Simulation Results

**Test vector:** `tx_data = 8'b00010110 (0x16)`

```
Transmitted : 8'b00010110
Received    : 8'b00010110  ✅ MATCH
done        : 1            ✅ ASSERTED at ~1,145,810 ns
```

---

## How to Run

### Compile
```
iverilog -o uart_tb uart_tb.v uart_top.v uart_tx.v uart_rx.v uart_baud_rate.v
```

### Simulate
```
vvp uart_tb
```

### View Waveforms
```
gtkwave uart_tb.vcd
```

---

## Directory Structure

```
uart_project/
├── uart_baud_rate.v   # Baud rate generator (9600 @ 50MHz)
├── uart_tx.v          # FSM-based UART transmitter
├── uart_rx.v          # Independent counter UART receiver
├── uart_top.v         # Top-level integration module
├── uart_tb.v          # Self-checking testbench
└── README.md          # This file
```

---

## Tools Used

| Tool           | Version | Purpose                        |
| -------------- | ------- | ------------------------------ |
| Icarus Verilog | 12.0    | HDL compilation and simulation |
| GTKWave        | 3.3.108 | Waveform visualization         |
| VS Code        | Latest  | Code editor                    |

---

## Synthesis Results

**Target Device:** Xilinx Artix-7 `xc7a35tcpg236-1` | **Tool:** Vivado 2025.1

| Resource | Used | Available | Utilization |
| --- | --- | --- | --- |
| Slice LUTs | 43 | 20800 | <1% |
| Slice Registers (FFs) | 53 | 41600 | <1% |
| Bonded IOB | 19 | 106 | <1% |
| BUFGCTRL | 1 | 32 | <1% |

**Timing Summary**
- Failing Endpoints: **0**
- Total Negative Slack (TNS): 0.000 ns
- Worst Negative Slack (WNS): inf

### Schematic
<img width="1083" height="612" alt="uart_schematic" src="https://github.com/user-attachments/assets/dd4de691-a3c9-42e4-9eb6-b3d254699b09" />


### Utilization Report
<img width="1920" height="1020" alt="uart_utilization" src="https://github.com/user-attachments/assets/964a40b8-5ffa-4657-99f3-75162c6836d2" />


### Timing Report
<img width="1920" height="1026" alt="uart_timing" src="https://github.com/user-attachments/assets/c03145dd-1cb8-4193-8e99-0234c0038120" />


---

## Future Improvements

- [ ] Add configurable baud rate via parameter
- [ ] Add parity bit support (even/odd/none)
- [ ] Add FIFO buffer for continuous data streaming
- [ ] Implement flow control (RTS/CTS)
- [ ] Add reset signal for FPGA deployment
- [ ] Synthesize and test on Basys3/Nexys4 FPGA board
- [ ] Upgrade testbench to SystemVerilog with assertions

---

## Waveform

![uart](images/waveform.png)

---

## Author

**Tejaswi** ECE Student | Hardware Design Enthusiast  
Building skills in VLSI, FPGA, and SoC design

---

*Built from scratch as a learning project — every module written, debugged, and verified independently.*
