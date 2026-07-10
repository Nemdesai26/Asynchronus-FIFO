# Asynchronous FIFO (Dual-Clock FIFO) in Verilog

A parameterizable, synthesizable **asynchronous FIFO** written in Verilog, using the classic gray-code pointer + two-flip-flop synchronizer technique (Clifford E. Cummings' methodology) for safe clock-domain crossing between independent write and read clocks.

## Overview

An asynchronous FIFO is used to pass data between two clock domains that have no fixed phase relationship. Since binary pointer comparisons are unsafe across clock domains (multiple bits can change at once, causing metastability/incorrect flag generation), this design:

- Represents read/write pointers in **Gray code**, so only one bit changes per increment.
- Synchronizes each pointer into the *other* clock domain using a **2-stage flip-flop synchronizer**.
- Derives `full` and `empty` flags from the synchronized, gray-coded pointers.

## Module Structure

```
FIFO.v            Top-level module — wires all submodules together
├── FIFO_memory.v     Dual-port memory array (the actual data storage)
├── wptr_full.v       Write-pointer logic + full-flag generation (write clock domain)
├── rptr_empty.v      Read-pointer logic + empty-flag generation (read clock domain)
├── two_ff_sync.v      2-flip-flop synchronizer (instantiated twice: r→w and w→r)
└── FIFO_tb.v          Testbench
```

### `FIFO.v` — Top Module
Parameters: `DSIZE` (data width, default 8), `ASIZE` (address width, default 4 → depth 16).
Instantiates two `two_ff_sync` synchronizers (one per direction), the memory, and the read/write pointer-flag modules, and connects them together.

### `FIFO_memory.v`
A simple dual-port RAM (`DEPTH = 2^ADDR_SIZE`). Reads are combinational (`rdata = mem[raddr]`); writes are synchronous to `wclk`, gated by `wclk_en && !wfull`.

### `wptr_full.v`
Maintains the binary write pointer (`wbin`), converts it to Gray code (`wptr`), and computes `wfull` by comparing the next Gray-coded write pointer against the synchronized read pointer (`wq2_rptr`) using the standard "MSBs inverted, rest equal" full-detection condition.

### `rptr_empty.v`
Mirror of `wptr_full.v` for the read side — maintains `rbin`/`rptr`, and computes `rempty` when the next Gray-coded read pointer equals the synchronized write pointer (`rq2_wptr`).

### `two_ff_sync.v`
Generic `SIZE`-bit, 2-stage flip-flop synchronizer used to safely bring a Gray-coded pointer from one clock domain into another.

### `FIFO_tb.v`
Self-contained testbench (`DSIZE=8`, `ASIZE=3`, depth 8) that:
1. Applies reset, then writes 10 words while reading in parallel.
2. Fills the FIFO past capacity to exercise the `wfull` flag.
3. Drains the FIFO past empty to exercise the `rempty` flag.

Write clock period = 10 ns, read clock period = 20 ns (asynchronous, unrelated frequencies — this is the point of the design).

## Block Diagram

```
        WRITE CLOCK DOMAIN                    READ CLOCK DOMAIN
      ┌───────────────────┐                ┌───────────────────┐
      │   wptr_full        │  wptr (gray)   │                   │
wdata │  (wbin, wptr,      │───────────────▶│   two_ff_sync     │
──────▶   wfull)           │                │    (w2r)          │──▶ rq2_wptr
      └─────────┬──────────┘                └─────────┬─────────┘
                │ waddr                                │
                ▼                                       ▼
         ┌─────────────────────────────────────────────────┐
         │              FIFO_memory (dual-port)              │
         └─────────────────────────────────────────────────┘
                │                                       ▲
                │ raddr                                 │
      ┌─────────▼──────────┐                ┌───────────┴─────────┐
      │   two_ff_sync      │◀───────────────│   rptr_empty          │ rdata
      │    (r2w)           │  rptr (gray)   │  (rbin, rptr,          │──▶
      └───────────┬────────┘                │   rempty)              │
                  ▼                          └───────────────────────┘
              wq2_rptr
```

## Simulation

Any Verilog simulator works (Icarus Verilog, Vivado XSim, ModelSim, etc.). Example with Icarus Verilog:

```bash
iverilog -o fifo_sim FIFO.v FIFO_memory.v wptr_full.v rptr_empty.v two_ff_sync.v FIFO_tb.v
vvp fifo_sim
```

To view waveforms, add a `$dumpfile`/`$dumpvars` block to `FIFO_tb.v` and open the resulting `.vcd` in GTKWave:

```verilog
initial begin
    $dumpfile("fifo.vcd");
    $dumpvars(0, FIFO_tb);
end
```

```bash
gtkwave fifo.vcd
```

## Parameters

| Parameter | Description       | Default |
|-----------|--------------------|---------|
| `DSIZE`   | Data bus width      | 8       |
| `ASIZE`   | Address bus width   | 4 (depth = 16) |

Both are set at instantiation of the top-level `FIFO` module and propagate down to all submodules.

## Key Design Notes

- **Gray-code pointers** ensure only a single bit toggles per pointer increment, which is essential for safe multi-bit signal crossing between asynchronous clock domains.
- **Full/empty detection** uses one extra MSB bit on each pointer (`ADDR_SIZE+1` bits total) to disambiguate the full-vs-empty condition when read and write pointers are otherwise equal.
- **Active-low resets** (`wrst_n`, `rrst_n`) are used throughout, one per clock domain.

## License

Add a license of your choice (e.g., MIT) here.
