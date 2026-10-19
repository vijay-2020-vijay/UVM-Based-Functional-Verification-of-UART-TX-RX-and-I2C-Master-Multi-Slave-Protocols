# UVM-Based Functional Verification of Serial Communication Protocols: UART (TX/RX) and I2C (1 Master, 3 Slaves)

Complete **UVM (Universal Verification Methodology)** testbenches in **SystemVerilog** for two widely used serial protocols:

| Design | DUT | Verification focus |
|---|---|---|
| **UART** | `uart_top` = `clk_gen` + `uart_tx` + `uart_rx` (TX looped back to RX) | Frame format, parity, stop bits, baud rates, 16x oversampling |
| **I2C** | `i2c_top` = 1 `i2c_master` + 3 slaves on one shared SCL/SDA bus | Address-based slave select, write/read, ACK/NACK, START/STOP timing |

Both environments include constrained-random and directed sequences, a reference-model scoreboard, **SystemVerilog Assertions (SVA)** and cover properties in the interface, **functional coverage**, and a detailed `report_phase` summary.

---

## Table of contents
1. [Repository structure](#repository-structure)
2. [Part 1: UART (TX and RX)](#part-1-uart-tx-and-rx)
3. [Part 2: I2C (1 master, multiple slaves)](#part-2-i2c-1-master-multiple-slaves)
4. [How to run](#how-to-run)
5. [Results](#results)
6. [Tools](#tools)
7. [Author](#author)

---

## Repository structure

```
.
├── uart/
│   ├── rtl/            uart_dut_sv.sv            (clk_gen, uart_tx, uart_rx, uart_top)
│   ├── tb/             uart_uvm_tb_v2.sv         (UVM testbench + interface with SVA)
│   └── docs/           block diagram, UVM architecture diagram, waveform diagram
├── i2c/
│   ├── rtl/            i2c DUT                   (i2c_master, i2c_slave, slave1/2/3, i2c_top)
│   ├── tb/             i2c_uvm_tb_v3.sv          (UVM testbench + interface with SVA)
│   └── docs/           block diagram, UVM architecture diagram, waveform diagram
├── LICENSE
└── README.md
```
*(Rename the files above to match your repository.)*

---

# Part 1: UART (TX and RX)

## 1.1 What is verified
UART is an asynchronous serial protocol. Every frame is sent as:

```
idle(1) | START (0) | D0 D1 ... D(n-1)  (LSB first) | PARITY (optional) | STOP (1) [STOP (1)] | idle(1)
```

The testbench checks that the byte sent by `uart_tx` is received correctly by `uart_rx` for different **baud rates**, **data lengths (5 to 8 bits)**, **parity modes (none / odd / even)** and **stop bits (1 or 2)**.

## 1.2 DUT architecture

![UART block diagram](uart/docs/uart_top_block_diagram.svg)

`uart_top` contains three blocks. The serial output of the transmitter is looped back to the receiver.

| Block | Role | Inputs | Outputs |
|---|---|---|---|
| `clk_gen` | Divides `clk` into two baud clocks from the `baud` input | `clk`, `rst`, `baud[16:0]` | `tx_clk`, `rx_clk` |
| `uart_tx` | Serialises `tx_data` into a frame | `tx_clk`, `tx_start`, `rst`, `tx_data[7:0]`, `length[3:0]`, `parity_type`, `parity_en`, `stop2` | `tx`, `tx_done`, `tx_err` |
| `uart_rx` | Samples the serial line and rebuilds the byte | `rx_clk`, `rx_start`, `rst`, `rx`, `length[3:0]`, `parity_type`, `parity_en`, `stop2` | `rx_out[7:0]`, `rx_done`, `rx_error` |

### Clocks and 16x oversampling
`clk_gen` toggles `tx_clk` and `rx_clk` when a counter passes the value selected by `baud`. `rx_clk` runs about **16 times faster** than `tx_clk`, so the receiver gets 16 clock ticks for every transmitted bit.

| baud | rx_max | tx_max |
|---|---|---|
| 4800 | 651 | 10416 |
| 9600 (default) | 325 | 5208 |
| 14400 | 217 | 3472 |
| 19200 | 163 | 2604 |
| 38400 | 81 | 1302 |
| 57600 | 54 | 868 |
| 115200 | 27 | 434 |
| 128000 | 24 | 392 |

### uart_tx FSM
`idle -> start_bit -> send_data -> send_parity -> send_first_stop -> send_sec_stop -> done`

- A new bit is placed on `tx` at every `tx_clk` edge, **LSB first**.
- The parity bit is the XOR of the data bits when `parity_type = 1`, and the XNOR when `parity_type = 0`.
- `tx_done` pulses when the last stop bit has been sent.

### uart_rx FSM
`idle -> start_bit -> recv_data -> check_parity -> check_first_stop -> check_sec_stop -> done`

- The receiver counts `0..15` per bit and **samples at count = 7 (the middle of the bit)**.
- At count = 7 of the start bit it checks that `rx` is still 0 (rejects glitches).
- Received bits are shifted into `datard`; after the last data bit `rx_out` is loaded, right-aligned for the selected `length`.
- Parity and stop bit(s) are checked, and `rx_error` flags a mismatch.
- `rx_done` pulses when the frame is complete.

## 1.3 Signal waveform

![UART waveform](uart/docs/uart_waveform.svg)

The waveform shows one complete frame:

1. The testbench applies `tx_data`, `baud`, `length`, parity and stop settings, then raises `tx_start` and `rx_start`.
2. `uart_tx` leaves `idle` on the next `tx_clk` edge and `tx` falls to 0 (start bit).
3. `uart_rx` sees `rx = 0`, enters `start_bit` and starts counting `0..15` on `rx_clk`.
4. `uart_tx` shifts out one data bit per `tx_clk` period, LSB first, then parity and the stop bit.
5. Each bit lasts 16 `rx_clk` periods, so `uart_rx` samples it at count 7 and shifts it into `datard`.
6. After the last data bit, `rx_out` is loaded. `tx_done` and `rx_done` pulse, and the testbench drops `tx_start` and `rx_start`.
7. The scoreboard compares `rx_out` with `tx_data` (masked to `length` bits).

## 1.4 UVM environment

![UART UVM architecture](uart/docs/uart_uvm_tb_diagram.svg)

| Component | Description |
|---|---|
| `uart_config` | Configuration object (agent active/passive) |
| `oper_mode` / `transaction` | Enum for the operation, and the sequence item carrying `baud`, `length`, `parity_type`, `parity_en`, `stop2`, `tx_data`, results and status |
| **10 sequences** | Constrained-random and directed sequences of 5 frames each, covering different baud rates, lengths, parity and stop-bit settings |
| `driver` | Resets the DUT, applies the configuration, raises `tx_start` and `rx_start`, and waits for `tx_done` and `rx_done` |
| `monitor` | Samples the interface on `rx_done` and sends the transaction through an analysis port |
| `scoreboard` | Checks `tx_data` (masked to `length`) equals `rx_out` and that `rx_err` is not set; collects coverage |
| `agent`, `env`, `test` | Standard UVM hierarchy; the test starts all sequences |
| `uart_if` | Virtual interface with SVA, cover properties and a report function |

**Driver handshake:** reset (5 clk) -> release -> wait -> apply config -> `tx_start`/`rx_start` = 1 -> drop each when its `done` pulses -> wait for both to fall -> gap -> next frame.

## 1.5 Assertions and coverage
- **SVA** in the interface check the handshake and pulse behaviour of the DUT signals (for example that `done` signals are short pulses and that `rx_done` arrives after `tx_start`).
- **Cover properties** confirm that the key scenarios actually happened.
- **Functional coverage** (scoreboard covergroup) samples baud rate, data length, parity setting, stop bits, and the results.
- **`report_phase`** prints the scoreboard summary, coverage and the assertion report.

## 1.6 Design notes found during review
These are observations from reading the RTL; confirm them in simulation:
- `tx_err` is declared but never set by `uart_tx`.
- In `uart_rx` the parity is computed over the whole 8-bit `datard`, so for `length < 8` with parity enabled it may not match the transmitter's parity (which uses only `length` bits).
- At 115200 and 128000 baud the rounded divider values make the 16x ratio only approximate.

---

# Part 2: I2C (1 master, multiple slaves)

## 2.1 What is verified
I2C is a two-wire, open-drain bus (**SCL** clock and **SDA** data). The testbench verifies one master talking to **three slaves** on the same bus:

- A 7-bit address `{slave_id[1:0], location[4:0]}`: ID `01` = slave1, `10` = slave2, `11` = slave3, `00` = **no slave** (must NACK).
- Single-byte **write** (`wr = 1`) and single-byte **read** (`wr = 0`).
- START and STOP conditions, address ACK, data ACK and the master NACK at the end of a read.

## 2.2 DUT architecture

![I2C block diagram](i2c/docs/i2c_top_block_diagram.svg)

| Block | Role |
|---|---|
| `i2c_master` | FSM: `IDLE, START, ADDR, ACK1, WDATA, ACK2, RDATA, MACK, STOP, DONE`. A tick happens every `DIV = 4` clocks, with 4 phases per SCL bit. SDA is driven open-drain (`sda_low`). |
| `i2c_slave` (x3) | Generic core with a 32 x 8 memory. A slave responds only when `addr[6:5]` equals its `ID`. |
| `slave1`, `slave2`, `slave3` | IDs `2'b01` (0x20-0x3F), `2'b10` (0x40-0x5F), `2'b11` (0x60-0x7F) |
| `i2c_top` | One master and three slaves sharing SCL and SDA (`tri1`, pulled up) |

**Top-level ports:** inputs `clk`, `rst`, `wr`, `addr[6:0]`, `din[7:0]`; outputs `datard[7:0]`, `done` (1-clock pulse), `err` (1 = NACK).

**Behaviour:**
- The master latches `wr`, `addr` and `din` at the start of each transfer, so the testbench can change them right after `done`.
- A write sends address + data and expects an ACK after each byte; a read receives one byte into `datard`.
- If no slave answers the address, the master sets `err = 1` and ends the transfer with a STOP.
- Slave memories are cleared by reset.

## 2.3 Signal waveform

![I2C waveform](i2c/docs/i2c_waveform.svg)

The waveform shows the START condition (SDA falls while SCL is high), the address and `wr` bit sent MSB first, the slave ACK, the data byte, the final ACK or master NACK, the STOP condition (SDA rises while SCL is high), and the `done` pulse.

## 2.4 UVM environment

![I2C UVM architecture](i2c/docs/i2c_uvm_tb_diagram.svg)

| Component | Description |
|---|---|
| `transaction` | Fields `op`, `wr`, `addr`, `din`, `datard`, `done`, `err`, with field macros so `print()` works. `addr[6:5]` = slave ID, `addr[4:0]` = location. |
| **Sequences** | `reset_dut`, `write_data`, `read_data`, `wr_rd_slave` (write then read back, one slave at a time, run for all 3 slaves), `write_err` and `read_err` (address with no slave) |
| `driver` | Reset, write and read tasks; waits for `vif.done` |
| `monitor` | Samples `wr`, `addr`, `din` and, after `done`, `err` and `datard` |
| `scoreboard` | Reference memory model (cleared on reset); checks NACK for `sid = 00`, ACK for valid slaves, and `datard == expected` on every read; per-slave counters |
| `agent`, `env`, `test` | Standard UVM hierarchy |
| `i2c_i` | Virtual interface with SVA, cover properties and a report function; `scl` and `sda` are observed from the DUT |

## 2.5 Assertions and coverage
- **9 assertions** in the interface: reset state, `done` is a one-clock pulse, bus released when `done`, START followed by SCL falling, STOP followed by `done`, START-to-STOP transaction length (ACK vs NACK), NACK followed by `done`, and no X/Z on outputs or inputs.
- **Cover properties:** reset, START, STOP, done, NACK, full ACK transaction, full NACK transaction, back-to-back transactions, reset after done.
- **Functional coverage** (`cg`): operation (write/read), slave ID (none, slave1/2/3), memory-location ranges, ACK/NACK, and a cross of slave x operation.
- **`report_phase`** prints: transaction counts, a per-slave table, every failed transaction (using `print()`), coverage per coverpoint with the holes named, a boxed SVA report (pass/fail counts, first failing time, cover hits), and a final PASS/FAIL verdict.

---

## How to run

**Vivado (xsim):**
1. Create a project and add the RTL file and the testbench file of the design you want to run.
2. Set the testbench module as the simulation top: `uart_top_TB` for UART, `tb` for I2C.
3. Run behavioural simulation. The UVM report appears in the Tcl console.

**Command line (xsim):**
```bash
xvlog -sv -L uvm <rtl_file>.sv <tb_file>.sv
xelab -L uvm <top_module> -s sim
xsim sim -R
```
> Make sure the old `interface` definition is not duplicated in both the RTL and testbench files.

## Results

Fill in after your runs:

| Design | Functional coverage | Assertions passed / failed | Scoreboard mismatches |
|---|---|---|---|
| UART | [ ] % | [ ] / [ ] | [ ] |
| I2C | [ ] % | [ ] / [ ] | [ ] |

## Tools
SystemVerilog, UVM, Vivado xsim [edit to match yours]

## Author
[Your name] · [LinkedIn link] · [Email]

## License
MIT [or your choice]
