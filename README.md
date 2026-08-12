<div align="center">

# 🔢 Radix-2 Booth Multiplier

## ✨ What is this?

A clean, classic implementation of the **Radix-2 Booth Multiplication algorithm**, split into a textbook FSM **controller** and **datapath** — the same architecture taught in every computer arithmetic course, built and verified end-to-end:

- ✅ Signed 8×8 → 16-bit multiplication
- ✅ Handles every two's-complement edge case (including `-128`)
- ✅ Self-checking testbench — directed + randomized tests
- ✅ Synthesizes and runs on real hardware (Nexys A7-100T)

---

## 🧭 Architecture

```
                 ┌───────────────────────────┐
   start ───────►│                           │
   Q0, Qm1 ─────►│      booth_controller      │──► load, op_sel,
   count_zero ──►│           (FSM)            │    shift_en, count_dec, done
                 └─────────────┬─────────────┘
                                │  control signals
                                ▼
                 ┌───────────────────────────┐
   M_in, Q_in ──►│                           │
                 │       booth_datapath       │──► product (2N-bit)
                 │   A / Q / M / Qm1 / count   │
                 └───────────────────────────┘
```

Both blocks live inside **`booth_top`** — the module tested by the testbench and wrapped for the FPGA.

---

## 📁 Project Structure

| File | Role |
|---|---|
| 🧠 `booth_controller.v` | FSM sequencing the algorithm — load → decode → add/sub → shift → check → done |
| ⚙️ `booth_datapath.v` | The registers and math: `A`, `Q`, `M`, `Q₋₁`, and the iteration counter |
| 🔗 `booth_top.v` | Wires controller + datapath together |
| 🧪 `booth_tb.v` | Self-checking testbench — directed edge cases + 30 randomized tests |
| 🎛️ `nexys4ddr_wrapper.v` | FPGA wrapper — maps switches/buttons/LEDs to the multiplier |
| 📌 `nexys4ddr.xdc` | Pin constraints for the Nexys A7-100T / Nexys4 DDR |

---

## ⚡ How It Works

**Radix-2 Booth Algorithm**, one bit at a time:

| Step | Action |
|---|---|
| 1️⃣ Load | `A = 0`, `Q = Q_in`, `M = M_in`, `Q₋₁ = 0`, counter = `N` |
| 2️⃣ Decode | Look at `{Q0, Q₋₁}` → `01` add `M`, `10` subtract `M`, else nothing |
| 3️⃣ Shift | Arithmetic right-shift `{A, Q, Q₋₁}` by 1 bit |
| 4️⃣ Repeat | Decrement counter, loop until zero |
| ✅ Done | Result = `{A, Q}`, a full `2N`-bit signed product |

### 🎛️ Controller FSM

```
IDLE → LOAD_S → DECODE ⇄ ADD_SUB → SHIFT → CHECK ─┬─► DECODE (loop)
                                                    └─► DONE_S
```

| State | What happens |
|---|---|
| `IDLE` | Waiting for `start` |
| `LOAD_S` | Latches `M_in` / `Q_in` into the datapath |
| `DECODE` | Reads `{Q0, Qm1}` to decide the next op |
| `ADD_SUB` | Sets `op_sel`: `01`=add, `10`=subtract, `00`=none |
| `SHIFT` | Shifts the combined register, decrements the counter |
| `CHECK` | Loop back, or fall through to `DONE_S` when `count_zero` |
| `DONE_S` | Asserts `done`, holds until `start` drops |

---

## 🧪 Verification

`booth_tb.v` drives `booth_top` and checks every result against Verilog's own `m_val * q_val`:

- 🎯 **Directed cases:** zero, pos×pos, neg×pos, pos×neg, neg×neg, max positive, max negative (`-128`), and both extremes multiplied together
- 🎲 **30 randomized tests** for extra coverage
- 📋 Console output: `[PASS]` / `[FAIL]` per test, plus a final tally
- ⏱️ Built-in simulation timeout + `.vcd` waveform dump for GTKWave

### ▶️ Run it

```bash
iverilog -o booth_sim booth_controller.v booth_datapath.v booth_top.v booth_tb.v
vvp booth_sim
gtkwave booth_tb.vcd   # optional — inspect waveforms
```

Expected output ends with:
```
=========================================
ALL 42 TESTS PASSED
=========================================
```

---

## 🖥️ Run on Hardware — Nexys A7-100T

`nexys4ddr_wrapper.v` puts the multiplier directly on your board's switches and LEDs:

| Board Control | Signal | Purpose |
|---|:---:|---|
| 🕐 `clk100mhz` | `clk` | 100 MHz system clock |
| 🔴 `btnC` | `rst` | Reset |
| 🟢 `btnU` | `start` | Kick off a multiplication |
| 🎚️ `sw[15:8]` | `M_in` | Multiplicand (signed, 8-bit) |
| 🎚️ `sw[7:0]` | `Q_in` | Multiplier (signed, 8-bit) |
| 💡 `led[15:0]` | `product[15:0]` | Lower 16 bits of the result |
| 💚 `led16_g` (RGB LED0) | `done` | Lights up green when finished |

### 🚀 Build & Program (Vivado)

1. New project → part `xc7a100tcsg324-1`
2. Add sources: `booth_controller.v`, `booth_datapath.v`, `booth_top.v`, `nexys4ddr_wrapper.v`
3. Add constraints: `nexys4ddr.xdc`
4. Set **top module** → `nexys4ddr_wrapper`
5. Synthesis → Implementation → Generate Bitstream → Program Device
6. Set switches → pulse `btnC` (reset) → pulse `btnU` (start) → watch LED0 turn green 🟢 and read the product off the LEDs

---

## 🔧 Extending This Project

- `N` is a parameter (default `8`) on `booth_datapath` / `booth_top` — retarget to wider operands by overriding it
- Only the lower 16 bits are shown on-board (16 LEDs available); add a 7-segment display or UART to see the full `2N`-bit product for larger `N`
- Could be extended to **Radix-4 Booth** for fewer cycles per multiply

---

<div align="center">

Built with ❤️ using Verilog · Tested with a self-checking bench · Runs on real silicon 🔩

</div>
