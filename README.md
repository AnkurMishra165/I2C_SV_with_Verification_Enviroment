# I2C Master–Slave (SystemVerilog) — Single‑Master, 7‑bit, 100 kHz

A compact **SystemVerilog** I²C bundle with a synthesizable *single‑master* controller, a simple memory‑backed *slave*, a top wrapper that wires them together, and a convenience `interface` for testbenches.

> **Files in this snippet**
> - `i2c_master` — 7‑bit address master, single‑byte R/W, start/stop/ACK handling
> - `i2c_Slave` — simple 128‑byte memory device (addressed by 7‑bit `addr`), R/W
> - `i2c_top` — connects master ↔ slave (shared `sda`, `scl` from master)
> - `i2c_if` — a SystemVerilog interface for easy TB connectivity

---

## ✨ Features

- **7‑bit I²C** addressing, **single‑master**, single‑byte transfer per transaction
- **Start / Stop / ACK / NACK** generation and detection
- **Parametric timing**: `sys_freq` (default 40 MHz) → `i2c_freq` (default 100 kHz)
- **Busy / Done / Ack‑error** handshakes for simple host control
- **Behavioral slave** with **128‑byte** memory initialised to `mem[i] = i`
- **SystemVerilog enums & interfaces** for readable state machines and clean TB wiring

> ⚠️ **Educational model**: simplifies the physical open‑drain bus. See “Hardware notes.”

---

## 🧱 Modules Overview

### `i2c_master`
```sv
module i2c_master (
  input  clk, rst,         // system clock & sync reset
  input  newd,             // pulse: start a new transaction
  input  [6:0] addr,       // 7-bit I2C address
  input  op,               // 0=WRITE, 1=READ (becomes R/W bit)
  inout  sda,              // serial data (shared)
  output scl,              // serial clock (driven by master)
  input  [7:0] din,        // data to write (if op=0)
  output [7:0] dout,       // data read   (if op=1)
  output reg busy,         // high during transaction
  output reg ack_err,      // slave NACK observed
  output reg done          // 1-cycle pulse when complete
);
```
- **Parameters:**  
  `sys_freq=40_000_000`, `i2c_freq=100_000` → internal quarter‑phase tick via `clk_count1 = (sys_freq/i2c_freq)/4`.
- **Protocol flow:**  
  `newd` → `start` → send `{addr,op}` → `ACK` → *if write*: send `din` → `ACK` → `stop`.  
  *if read*: shift in 8 bits → master NACK (releases SDA high) → `stop`.
- **Handshake:** `busy=1` during transfer, `done=1` one cycle at end, `ack_err=1` if NACK where ACK expected.

### `i2c_Slave`
```sv
module i2c_Slave (
  input  scl, clk, rst,
  inout  sda,
  output reg ack_err, done
);
```
- Detects **START** (`SDA↓ while SCL=1`) and **STOP** sequences, receives the 8‑bit address+R/W, ACKs if addressed, then either:
  - **Read direction (R/W=1):** drives 8 data bits from `mem[addr]`, samples master ACK/NACK.
  - **Write direction (R/W=0):** receives 8 data bits into `mem[addr]` and ACKs.
- Memory initialisation on reset: `mem[i] = i` for `i ∈ [0,127]`.

### `i2c_top`
```sv
module i2c_top (
  input  clk, rst, newd, op,
  input  [6:0] addr,
  input  [7:0] din,
  output [7:0] dout,
  output busy, ack_err, done
);
```
- Wires internal `scl` and shared `sda` between master and slave.  
- `ack_err` = OR of master/slave error flags; `done` = master’s `done`.

### `i2c_if` (SystemVerilog interface)
```sv
interface i2c_if;
  logic clk, rst, newd, op;
  logic [6:0] addr;
  logic [7:0] din, dout;
  logic done, busy, ack_err;
endinterface
```
Use in testbenches to group signals cleanly.

---

## 🔌 Top‑Level Ports (Host View)

| Signal  | Dir | Width | Master/Top | Meaning |
|--------:|:---:|:-----:|:-----------|:--------|
| `clk`   | in  | 1 | all | System clock (40 MHz default timing in code) |
| `rst`   | in  | 1 | all | Sync reset |
| `newd`  | in  | 1 | master/top | Pulse high to start a transaction |
| `addr`  | in  | 7 | master/top | I²C 7‑bit device address |
| `op`    | in  | 1 | master/top | `0`=WRITE, `1`=READ |
| `din`   | in  | 8 | master/top | Data to write (when `op=0`) |
| `dout`  | out | 8 | master/top | Data read (when `op=1`) |
| `busy`  | out | 1 | master/top | Transfer in progress |
| `done`  | out | 1 | master/top | 1‑cycle pulse at end of transfer |
| `ack_err` | out | 1 | master/top | Slave NACK detected |

---

## ⏱️ Timing Engine (Quarter‑phase “pulse”)

Both master and slave derive an internal **quarter‑period tick** `pulse ∈ {0,1,2,3}` from `clk` using:
```
clk_count4 = sys_freq / i2c_freq;  // 400 at 40MHz→100kHz
clk_count1 = clk_count4 / 4;       // 100
```
They step actions on specific `pulse` values (e.g., launch data while `SCL=0`, sample during `SCL=1`). Several places sample at `count1==200` etc. (mid‑high window). If you **change `sys_freq`/`i2c_freq`**, verify sample points remain centered.

---

## ▶️ Quick Start (Simulation)

### Testbench skeleton
```sv
`timescale 1ns/1ps
module tb;
  logic clk=0, rst=1, newd=0, op=0;
  logic [6:0] addr=7'h12;
  logic [7:0] din=8'hA5, dout;
  logic busy, ack_err, done;

  // 40 MHz system clock
  always #12.5 clk = ~clk;

  i2c_top dut(.*);

  initial begin
    repeat(5) @(posedge clk);
    rst <= 0;

    // WRITE: addr=0x12, data=0xA5
    op <= 1'b0; // write
    din <= 8'hA5;
    addr <= 7'h12;
    @(posedge clk); newd <= 1;
    @(posedge clk); newd <= 0;
    wait(done);
    $display("WRITE done, ack_err=%0b", ack_err);

    // READ: addr=0x12 → dout should return 0xA5
    op <= 1'b1; // read
    @(posedge clk); newd <= 1;
    @(posedge clk); newd <= 0;
    wait(done);
    $display("READ done: dout=0x%02h, ack_err=%0b", dout, ack_err);

    $finish;
  end

  initial begin
    $dumpfile("waves.vcd");
    $dumpvars(0,tb);
  end
endmodule
```
> By default, the slave memory is initialised to `mem[i]=i`. After a write, a subsequent read returns the written byte.

### Compile & run (examples)
- **Verilator (recommended for SV):**  
  ```bash
  verilator --sv -Wall --trace -cc i2c_top.sv --exe tb.cpp   # or use tb.sv via --exe + verilated wrapper
  ```
- **Questa/ModelSim:** add all SV files, set `-sv` compile switch, run `vsim tb`.

> Icarus Verilog’s SystemVerilog support may be limited depending on version; use `-g2012` and check for enum/interface support.

---

## 🧪 Transaction Details

- **START**: Master drives `SDA: 1→0` while `SCL=1` (`start` state).
- **Address + R/W**: Master shifts `{addr, op}` MSB‑first while toggling `SCL`.
- **ACK (from slave)**: Slave pulls `SDA=0` during the 9th clock. Master samples and sets `ack_err` on NACK.
- **WRITE byte**: Master shifts `din` and expects a slave ACK.
- **READ byte**: Slave shifts `mem[addr]` to master; master issues **NACK** after 8 bits (to finish).
- **STOP**: Master drives `SDA: 0→1` while `SCL=1` (`stop` state).
- **Host handshake**: `busy↑` at start, `done↑` for 1 cycle at end. Keep `newd` a **single‑cycle pulse**.

---

## 🔧 Parameters & Customisation

In `i2c_master`:
```sv
parameter int sys_freq = 40_000_000;  // Hz (system clock)
parameter int i2c_freq = 100_000;     // Hz (I²C SCL)
```
Override at instantiation:
```sv
i2c_master #(.sys_freq(50_000_000), .i2c_freq(400_000)) u_master (...);
```
> If you change frequencies, ensure sampling instants (e.g., `count1==200`) still sit mid‑high for `SCL`.

---

## 🧯 Common Gotchas

- **Open‑drain modelling:** The code **actively drives ‘1’** on `SDA` in places (e.g. master’s `assign sda = sda_en ? (sda_t ? 1 : 0) : 1'bz;`). For real hardware you must use **open‑drain** semantics: *drive `0` or `Z` only*, with pull‑ups on the board. For synthesis, prefer:
  ```sv
  assign sda = (drive_low) ? 1'b0 : 1'bz;
  // Add IOBUF / OBUFT for FPGAs and external pull-ups.
  ```
- **Clock stretching:** Not implemented. A real slave might hold `SCL=0` to stretch; here master unilaterally drives SCL.
- **Single‑byte only:** Each `newd` triggers exactly one address + 1 data byte. Extend FSMs for multi‑byte bursts / register pointers.
- **Multi‑master arbitration:** Not supported.
- **Tight timing constants:** Sampling conditions like `count1==200` assume the default divisors. Adjust carefully if you change frequencies.

---

## 🗂️ Suggested Repo Layout

```
├─ src/
│  ├─ i2c_master.sv
│  ├─ i2c_Slave.sv
│  ├─ i2c_top.sv
│  └─ i2c_if.sv
├─ tb/
│  └─ tb_i2c_top.sv
├─ sim/
│  └─ waves.vcd   (generated)
└─ README.md
```

---

## 📈 Roadmap / Ideas

- True open‑drain SDA with I/O buffers & external pull‑ups
- Clock stretching support
- Repeated‑START + sub‑address register pointer
- Multi‑byte bursts (write/read)
- Multi‑master arbitration
- Self‑checking SystemVerilog testbench using `i2c_if`
- Parameterized mid‑bit sample points (remove magic numbers)

---

## 🙌 Notes

This design is intentionally straightforward for learning I²C and finite state machines in SystemVerilog. It’s a solid base to add repeated‑start, burst transfers, and proper open‑drain synthesis patterns.
