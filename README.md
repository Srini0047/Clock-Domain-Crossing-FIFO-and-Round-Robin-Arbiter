# cdc-fifo-arbiter

Synthesizable SystemVerilog RTL for two building blocks common in multi-clock ASIC/SoC designs, with formal verification setups for each:

- **Round-robin arbiter**: fair, starvation-free access for `N` requesters, using a rotating-priority mask.
- **Asynchronous FIFO**: safe data transfer between independent clock domains, using Gray-coded pointers and dual-flop synchronizers.

## Features

- Parameterized round-robin arbiter (`N` requesters, default 4) with rotating priority, so no requester can be starved
- Single-cycle, purely combinational grant path with a one-hot (or all-zero) output
- Asynchronous FIFO with parameterizable depth and data width (set in `fifo_pkg`)
- Gray-coded read/write pointers, so only one bit changes per transition
- Dual-flop synchronizers for metastability-safe clock-domain crossing
- Full/empty detection by comparing synchronized Gray pointers
- Modular RTL, with SVA properties and JasperGold scripts included

## Repository layout

```
rtl/
  rr_arbiter.sv           Round-robin arbiter (top)
  arbiter_fx_priority.sv  Fixed-priority arbiter (lowest index wins)
  async_fifo.sv           Async FIFO (top)
  wrt_ptr_handler.sv      Write pointer, Gray encode, full flag (write domain)
  rd_ptr_handler.sv       Read pointer, Gray encode, empty flag (read domain)
  dual_ff_sync.sv         Two-flop synchronizer
  fifo_pkg.sv             FIFO_DEPTH, A_WIDTH, D_WIDTH
formal/
  properties/
    arbiter_properties.sva  SVA for the arbiter (bound to rr_arbiter)
    fifo_properties.sva     SVA for the FIFO (bound to async_fifo)
  scripts/
    arbiter_setup.tcl       JasperGold FPV for the arbiter
    fifo_setup.tcl          JasperGold FPV for the FIFO
    cdc_check.tcl           JasperGold CDC structural check for the FIFO
```

---

## Round-robin arbiter

### Interface

```systemverilog
module rr_arbiter #(parameter N = 4) (
    input  logic         clk,
    input  logic         arst_n,   // async, active-low reset
    input  logic [N-1:0] i_req,    // request vector, one bit per requester
    output logic [N-1:0] o_grant   // one-hot grant (all zeros if no request)
);
```

| Port      | Dir | Description                                        |
|-----------|-----|----------------------------------------------------|
| `clk`     | in  | Arbiter clock                                      |
| `arst_n`  | in  | Asynchronous active-low reset                      |
| `i_req`   | in  | Request vector; bit `i` is requester `i`           |
| `o_grant` | out | Grant vector; at most one bit set, only where `i_req` is set |

### How it works

The arbiter uses the standard **masked fixed-priority** technique. A fixed-priority arbiter (`arbiter_fx_priority`, lowest index wins) is instantiated twice:

1. **Masked arbiter** sees `i_req & mask_reg`. The mask blocks every requester at or below the one granted last.
2. **Unmasked arbiter** sees the raw `i_req`.

The output selects between them:

```
o_grant = (|masked_req) ? masked_grant : priority_sel_grant;
```

- If any requester above the last winner is requesting, the masked arbiter picks the next one in rotation.
- If none is, the round has finished and priority wraps around. The unmasked arbiter picks the lowest-index requester.

`mask_reg` updates every clock. After a grant to requester `i`, `next_mask` is `{N{1'b1}} << (i+1)`, so requesters `0..i` are masked and priority moves to `i+1`. After a grant to the last requester (`N-1`), or when nothing is granted, the mask resets to all ones.

### Example (N = 4)

| Cycle | `i_req` | `mask_reg` | `o_grant` | Notes                                      |
|-------|---------|------------|-----------|--------------------------------------------|
| 0     | `1111`  | `1111`     | `0001`    | Req 0 wins (lowest index)                  |
| 1     | `1111`  | `1110`     | `0010`    | Mask excludes req 0, so req 1 wins         |
| 2     | `1111`  | `1100`     | `0100`    | Req 2 wins                                 |
| 3     | `1111`  | `1000`     | `1000`    | Req 3 wins                                 |
| 4     | `1111`  | `1111`     | `0001`    | Mask reset, priority wraps back to req 0   |

### Fairness

Each grant moves the lowest-priority slot to the requester just served. A persistent requester therefore waits at most `N-1` grants behind the others, which prevents starvation.

### Usage

```systemverilog
rr_arbiter #(.N(8)) u_arb (
    .clk    (clk),
    .arst_n (arst_n),
    .i_req  (req),
    .o_grant(grant)
);
```

---

## Asynchronous FIFO

The FIFO has independent write (`wclk`/`wrst_n`) and read (`rclk`/`rrst_n`) domains.

- **Pointers** are `A_WIDTH+1` bits. The extra MSB distinguishes full from empty.
- **Write side** (`wrt_ptr_handler`): keeps a binary pointer for addressing and a Gray pointer for crossing. `full` is set when the next Gray write pointer equals the synchronized Gray read pointer with its top two bits inverted.
- **Read side** (`rd_ptr_handler`): the mirror image. `empty` is set when the Gray read pointer equals the synchronized Gray write pointer.
- **Synchronizers** (`dual_ff_sync`): each Gray pointer is passed through two flops in the destination domain.
- **Memory**: `fifo` is written synchronously in `wclk` when `w_en && !full`. The read is asynchronous (combinational) from the binary read pointer.

Gray coding means only one pointer bit changes per increment. A synchronizer that samples mid-transition therefore yields either the old or the new value, never a corrupt one. `full` and `empty` stay conservative: they may assert slightly late to deassert, but never incorrectly allow an overflow or underflow.

### Configuration

Edit `rtl/fifo_pkg.sv`:

```systemverilog
parameter int unsigned FIFO_DEPTH = 16;          // must be a power of two
parameter int unsigned A_WIDTH    = $clog2(FIFO_DEPTH);
parameter int unsigned D_WIDTH    = 8;
```

---

## Formal verification

The properties are written in SVA and attached with `bind`, so the RTL stays untouched.

**Arbiter** ([arbiter_properties.sva](formal/properties/arbiter_properties.sva))

| Property          | Meaning                                                              |
|-------------------|----------------------------------------------------------------------|
| `a_onehot`        | At most one grant at a time                                          |
| `a_no_spurious`   | Grant is only given to a requester that is requesting                |
| `a_bounded_wait`  | A held request is granted within `N` cycles (starvation freedom)     |
| `a_no_deadlock`   | Any request produces a grant within `N-1` cycles                     |
| mask rotation     | After granting `i`, `mask_reg` equals the expected rotated mask      |

The bounded-wait check assumes a requester holds its request until it is granted.

**FIFO** ([fifo_properties.sva](formal/properties/fifo_properties.sva)): reset values, pointer increment and hold (overflow/underflow protection), one-bit-per-step Gray transitions, Gray/binary consistency, full/empty flag correctness, and cover points for full and pointer wrap-around.

### Running (Cadence JasperGold)

Run from the repo root:

```sh
# Arbiter functional proofs
jaspergold -batch -tcl formal/scripts/arbiter_setup.tcl

# FIFO functional proofs
jaspergold -batch -tcl formal/scripts/fifo_setup.tcl

# FIFO structural CDC check
jaspergold -batch -tcl formal/scripts/cdc_check.tcl
```

`cdc_check.tcl` writes its report to `docs/proofs/`, so create that directory first.

## Design notes

- The arbiter is a single-clock block with no CDC concerns.
- Both resets are asynchronous and active-low.
- The arbiter has a combinational request-to-grant path. Registering `o_grant` is a drop-in change if timing requires it.
- Each FIFO reset is local to its domain. Assert both resets together.
- FIFO depth must be a power of two, since the Gray-pointer full test relies on binary wrap-around.
