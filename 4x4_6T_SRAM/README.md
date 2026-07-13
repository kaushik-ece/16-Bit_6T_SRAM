# 4×4 6T SRAM Array — README

This document explains the 4×4 6T SRAM memory (4 words, 4 bits per word) built from the schematic: `DECODER + 6T_PC + 6T_CELL + Write_en + Sense_comp`. It covers what every block does, the equations behind it, and how to read the transient waveform.

---

## 1. What this circuit is

It's a small memory array that can store **4 words of 4 bits each (16 bits total)**.

- **Address**: A1, A0 (2 bits → picks 1 of 4 rows)
- **Enable**: E (turns the decoder on/off)
- **Precharge**: PC (readies the bitlines before a read)
- **Write control**: Write_en, Din0–Din3
- **Read control**: Read_en, D0–D3 (Dout of each column)

Rows = word lines (W0–W3), Columns = bits (bit 0 to bit 3). Each row/column intersection is one `6T_CELL`.

---

## 2. Block-by-block explanation

### 2.1 DECODER (row selector)
Takes A1, A0, E and turns ON exactly one word line at a time.

```
W0 = E · A1' · A0'
W1 = E · A1' · A0
W2 = E · A1  · A0'
W3 = E · A1  · A0
```

If E = 0, all word lines are 0 — no row is selected, array is idle.

### 2.2 6T_PC (Precharge circuit — one per column)
Before every read, both bitlines (BL, BLB) must start at the same voltage (VDD), so the cell — not leftover charge — decides which line drops.

```
if PC = 1:  BL = BLB = VDD   (equalized, both PMOS pull up)
if PC = 0:  precharge PMOS off, bitlines float, driven by cell/write driver
```

Precharge must happen **before** the word line turns on, otherwise the cell fights the precharge devices.

### 2.3 6T_CELL (the actual storage cell)
Standard 6-transistor cell: 2 cross-coupled inverters (4 transistors) holding the bit, + 2 NMOS access transistors gated by the word line (WL).

Storage node equations (Q = stored bit, QB = its complement):
```
Q  = NOT(QB)   -- held by inverter 1
QB = NOT(Q)    -- held by inverter 2
```

**Read condition** (WL = 1, PC was 1 just before):
- If Q = 1 → cell pulls BLB down slightly (BL stays high)
- If Q = 0 → cell pulls BL down slightly (BLB stays high)
- The small voltage difference (ΔV) between BL/BLB is what the sense amp detects.

**Read stability rule (cell ratio, CR):**
```
CR = (W/L)_driver-NMOS  ÷  (W/L)_access-NMOS   ≥ ~1.2–2 (typical)
```
This keeps the driver transistor "stronger" than the access transistor so the stored value doesn't accidentally flip during a read.

**Write condition** (WL = 1, bitlines forced by write driver, not precharge):
- To write a 1: BL driven to VDD, BLB driven to 0 → forces Q=1, QB=0
- To write a 0: BL driven to 0, BLB driven to VDD → forces Q=0, QB=1

**Write margin rule (pull-up ratio, PR):**
```
PR = (W/L)_access-NMOS ÷ (W/L)_pull-up-PMOS  ≥ ~1 (typical)
```
This keeps the access transistor "stronger" than the PMOS pull-up so the write driver can successfully overpower and flip the cell.

### 2.4 Write_en block (write driver — one per column)
Only active when the array is being written to.

```
if Write_en = 1 and WL = 1:
    BL  = Din
    BLB = NOT(Din)
else:
    driver is high-impedance (bitlines controlled by precharge/cell instead)
```

### 2.5 Sense_comp (sense amplifier — one per column)
Detects which bitline is lower and produces a clean digital output.

```
if Read_en = 1:
    Dout = 1   if BL > BLB
    Dout = 0   if BL < BLB
else:
    Dout = high-impedance / holds last value
```

The sense amp is needed because the cell only produces a small ΔV on the bitlines (not a full-swing signal) — the sense amp amplifies that small difference into a proper logic level.

---

## 3. Read operation — step by step
1. PC = 1 → BL = BLB = VDD for the selected column(s).
2. PC = 0, Read_en = 1.
3. A1, A0, E select the row → correct WL goes high.
4. Selected cell pulls one bitline down slightly.
5. Sense_comp compares BL vs BLB → outputs D correctly.
6. WL goes low again (row deselected), cycle repeats for next address.

## 4. Write operation — step by step
1. Din is set to the value to be written.
2. Write_en = 1.
3. A1, A0, E select the row → correct WL goes high.
4. Write driver forces BL/BLB to Din / Din′, overpowering the cell and flipping it if needed.
5. WL goes low → new value is now latched (held) by the cross-coupled inverters even with WL off.

---

## 5. Reading the transient waveform

The simulation sweeps through **all 4 addresses (00 → 01 → 10 → 11)** using A1/A0, while E stays enabled. In each address window you can see two sub-events:

| Signal | What to look for |
|---|---|
| **PC** | Fast toggling clock-like pulse — precharges the bitlines every cycle, right before each read |
| **A0, A1** | Step every address window — together they form the 2-bit binary count 00→01→10→11, sweeping through all 4 rows one at a time |
| **E** | Stays active (toggling with PC) — keeps the decoder enabled throughout the test |
| **Write_en** | Pulses high once per address window — marks the write phase for that row |
| **Din0–Din3** | The data pattern applied during the write pulse for that column, held steady until the next write |
| **Read_en** | Pulses high after the write, once bitlines are precharged — marks the read phase |
| **D0–D3** | Follows the corresponding Din after a short delay — this delay is the time taken for precharge → access → sense-amp evaluation |

**How to verify correctness from the waveform:** for every address window, whatever value was written on `Din_i` should reappear on `D_i` during the following read pulse (after PC has toggled and Read_en goes high). If `D_i` matches `Din_i` for all 4 addresses and all 4 columns, the cell is correctly storing and returning data — i.e., read-after-write is functioning.

The narrow spikes you see on some D lines (e.g., D2) are transient glitches during the precharge/access transition, not valid data — only the settled level while Read_en is high should be read as the actual output.

---

## 6. Signal summary

| Signal | Direction | Purpose |
|---|---|---|
| A0, A1 | Input | Row address (selects 1 of 4 word lines) |
| E | Input | Decoder enable |
| PC | Input | Precharge control for bitlines |
| Write_en | Input | Enables write drivers |
| Din0–Din3 | Input | Data to write, one per column |
| Read_en | Input | Enables sense amplifiers |
| D0–D3 (Dout) | Output | Data read out, one per column |
| W0–W3 | Internal | Word lines from decoder to cell rows |
| BL, BLB | Internal | Bitline pair per column |

---

## 7. Quick design checklist
- [ ] Precharge (PC=1) always happens **before** WL goes high for a read.
- [ ] Write_en and Read_en are never high at the same time for the same column.
- [ ] Cell ratio (CR) sized for read stability; pull-up ratio (PR) sized for write-ability.
- [ ] Only one word line (W0–W3) is high at any given time.
