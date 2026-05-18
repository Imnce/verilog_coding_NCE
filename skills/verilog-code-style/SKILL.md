---
name: verilog-code-style
description: Write, modify, review, or format any Verilog/SystemVerilog code using the user's Vivado-oriented Verilog style guide. Use this skill for every Verilog/RTL task: creating modules, editing .v/.sv files, implementing FPGA logic, writing test RTL, instantiating modules, cleaning formatting, reviewing style, or generating any Verilog code of any domain. Trigger even if the user only says "按风格", "整理代码", "写个模块", "写 Verilog", "改 RTL", or mentions Vivado/FPGA/RTL.
---

# Verilog Code Style Skill

Synthesizable production RTL for Xilinx and Microchip FPGA. Target: function, timing, area. Zero tolerance for metastability and unsafe CDC. Distinguish simulation-pass from board-stable RTL.

## Workflow

1. Read existing RTL before editing. Preserve behavior unless explicitly changing it.
2. Match this guide for new or substantially modified code. Tiny fixes: keep surrounding style.
3. New modules get a matching testbench unless explicitly scoped out.
4. If no toolchain, state that syntax was not compiled.
5. When unclear, ask: target device, clock frequency, reset scheme, interface protocol, timing/resource budget.
6. Define parameters and ports first. For input ports with handshake timing, register them before decoding.
7. Define one register per output port, then assign output wires from those registers.
8. Write simple register logic before complex control logic.

## 1. Always block discipline

**Space after `always`**: `always @(posedge i_clk)`, not `always@(posedge i_clk)`.

**One register per always block**. Exception: multiple registers with identical reset and update conditions.

```verilog
// Good — single register per always
always @(posedge i_clk) begin
    if(i_rst) r_valid <= 1'd0; else r_valid <= i_valid;
end

always @(posedge i_clk) begin
    if(i_rst) r_data <= 'd0; else if(i_valid) r_data <= i_data;
end

// Allowed exception — identical conditions, simple parallel delay
always @(posedge i_clk) begin
    if(i_rst) begin
        r_u0_ready_1d <= 1'd0;
        r_u1_ready_1d <= 1'd0;
        r_u2_ready_1d <= 1'd0;
    end else begin
        r_u0_ready_1d <= w_u0_ready;
        r_u1_ready_1d <= w_u1_ready;
        r_u2_ready_1d <= w_u2_ready;
    end
end

// Bad — different conditions mixed in one block
always @(posedge i_clk) begin
    if(i_rst) begin
        r_valid <= 1'd0;
        r_data  <= 'd0;
    end else begin
        r_valid <= i_valid;       // condition differs from r_data
        if(i_valid)
            r_data <= i_data;     // extra condition
    end
end
```

### Always block convergence

Every `if` / `else if` chain must close with a final `else` that explicitly assigns the register, even if holding the current value.

```verilog
// Good
always @(posedge i_clk) begin
    if(i_rst)       r_state <= P_IDLE;
    else if(w_trig) r_state <= P_RUN;
    else            r_state <= r_state;     // explicit hold
end

// Bad — missing else, intent unclear
always @(posedge i_clk) begin
    if(i_rst)       r_state <= P_IDLE;
    else if(w_trig) r_state <= P_RUN;
end
```

### Flat else-if rule

Single-register always block with mutually-exclusive conditions: use flat `else if` chain, not `else begin...end` nesting.

```verilog
// Good — flat chain
always @(posedge i_clk) begin
    if(i_rst)       r_aw_en <= 1'b1;
    else if(w_a)    r_aw_en <= 1'b1;
    else if(w_b)    r_aw_en <= 1'b0;
    else            r_aw_en <= r_aw_en;
end

// Bad — nested else begin...end adds indentation without structure
always @(posedge i_clk) begin
    if(i_rst)       r_aw_en <= 1'b1;
    else begin
        if(w_a)     r_aw_en <= 1'b1;
        else if(w_b) r_aw_en <= 1'b0;
        else        r_aw_en <= r_aw_en;
    end
end
```

### Valid flag clear guard

When clearing a registered valid flag on handshake, guard with the flag itself to prevent accidental clear.

```verilog
// Good
else if(i_ready && r_valid) r_valid <= 1'b0;

// Bad — may clear when already low
else if(i_ready)            r_valid <= 1'b0;
```

Applies to: `r_rvalid`, `r_mram_rd_valid`, `r_bvalid`, and any registered valid flag that clears on handshake.

### Explicit bit widths

All literals carry explicit widths matching the target register or expression context.

```verilog
// Good
reg [7:0] r_cnt;
r_cnt <= 8'd0;
r_cnt <= r_cnt + 8'd1;

reg [31:0] r_data;
r_data <= 32'd0;

// Bad — width inferred, may cause truncation
reg [7:0] r_cnt;
r_cnt <= 'd0;
r_cnt <= r_cnt + 'd1;
```

Exception for `P_` constants with explicit widths — usable directly without re-specifying.

## 2. Timing-critical rules

### Registered output isolation (r_o_xx)

Module output ports driven by registered signals, not combinational logic. Use `r_o_` prefix → `assign o_xxx = r_o_xxx`.

```verilog
// Good
reg  r_o_rd_ready;
assign o_rd_ready = r_o_rd_ready;

// Bad — long combinational path to output
assign o_rd_ready = (r_st_current == P_IDEL) & w_fifo_empty & w_all_ready;
```

Exception: constants (e.g. `o_wr_ready = 1'b1`) or ports already fed by sub-module registers.

### Register sub-module port signals

Control signals to instance ports (valid, type, op_code) must be driven by registers, not combinational expressions or muxes in the port map.

```verilog
// Good
reg         r_op_valid;
reg  [1:0]  r_op_type;

mram_driver u0 (
    .i_operation_valid (r_op_valid),
    .i_operation_type  (r_op_type ),
    ...
);

// Bad — combinational in port map
mram_driver u0 (
    .i_operation_valid (w_op_valid),
    .i_operation_type  ((r_st_current == P_WR_REQ) ? 2'd1 : 2'd2),
    ...
);
```

### FSM edge-triggered state entry

Latch data or set flags on state entry using exact transition edge `r_st_next == TARGET && r_st_current == SOURCE`. Fires once per entry, independent of `r_st_cnt` (no overflow risk).

```verilog
// Good — exact edge, no counter dependency
always @(posedge i_clk) begin
    if(i_rst)                                                     r_op_type <= 2'd2;
    else if(r_st_next == P_WR_REQ && r_st_current == P_FIFO_LATCH)     r_op_type <= 2'd1;
    else if(r_st_next == P_RD_REQ && r_st_current == P_IDEL)           r_op_type <= 2'd2;
    else if(r_st_next == P_RD_REQ && r_st_current == P_WR_WAIT_DONE)   r_op_type <= 2'd2;
    else                                                          r_op_type <= r_op_type;
end

// Bad — level-triggered, fires every cycle in state
else if(r_st_current == P_WR_REQ) r_op_type <= 2'd1;

// Bad — r_st_cnt overflow risk at 16'hFFFF -> 0
else if(r_st_current == P_WR_REQ && r_st_cnt == 16'd0) r_op_type <= 2'd1;
```

When a state has multiple entry paths, list each transition explicitly.

### Ready handshake register pattern

Ready output register: clears on handshake (`i_valid && r_o_ready`), asserts on completion edge. No combinational gating.

```verilog
always @(posedge i_clk) begin
    if(i_rst)                                     r_o_rd_ready <= 1'b0;
    else if(i_rd_valid && r_o_rd_ready)           r_o_rd_ready <= 1'b0;
    else if(r_st_current == P_RD_DONE && r_st_next == P_IDEL) r_o_rd_ready <= 1'b1;
    else                                          r_o_rd_ready <= r_o_rd_ready;
end
```

All internal consumers reference the registered version (`r_o_rd_ready`), not a combinational wire.

## 3. Naming conventions

| Prefix | Meaning | Example |
|--------|---------|---------|
| `i_` | input port | `i_clk` |
| `o_` | output port | `o_valid` |
| `r_` | register | `r_cnt` |
| `w_` | combinational wire | `w_all_ready` |
| `ri_` | registered input | `ri_op_type` |
| `ro_` | registered output | `ro_axi_awready` |
| `r_o_` | output register (→ assign o_xxx) | `r_o_rd_ready` |
| `P_` | parameter / localparam | `P_INIT_CYCLE` |

Module names: lowercase snake_case (`mram_ctrl`, `flash_ctrl`, `top`).

Instance names: `u0_module_name`, `u1_module_name` (prefix form). Legacy IP: `u0_FIFO_64X256`.

Edge detection: `w_xxx_pose` (posedge), `w_xxx_nege` (negedge):
```verilog
wire w_vld_pose = i_valid && !r_i_valid;
wire w_vld_nege = i_ready && !r_i_ready;
```

Handshake: `w_xxx_handshake`:
```verilog
wire w_op_handshake = i_op_valid && o_op_ready;
```

Pipeline delay: `_1d`, `_2d`, `_3d` suffix.

## 4. Parameters

Every `parameter` and `localparam` has end-of-line comment. Use `P_` prefix, uppercase with underscores.

```verilog
parameter  P_RST_CYCLE = 32'd100;  // reset cycle count
localparam P_INIT      = 4'd0;     // init state
localparam P_IDEL      = 4'd1;     // idle state
```

Shared/cross-module parameters → dedicated `params.vh` file with `` `include "params.vh" ``. Module-local parameters (FSM states, constants) stay inside the module.

## 5. `default_nettype none + explicit wire ports

Every file: `` `default_nettype none `` at top, `` `default_nettype wire `` at bottom.

Under `` `default_nettype none ``, ALL `input`/`output` ports MUST declare `wire` explicitly (vlog-2892):

```verilog
`default_nettype none

module example #(
    parameter P_WIDTH = 32'd8
)(
    input  wire               i_clk,
    input  wire               i_rst,
    input  wire [P_WIDTH-1:0] i_data,
    output wire               o_valid,
    output wire [P_WIDTH-1:0] o_data
);
...
endmodule

`default_nettype wire
```

## 6. Port declarations

One port per line. Align direction, width, name, delimiter, comment. Trailing comma except last port.

```verilog
module mram_ctrl #(
    parameter P_CLK_FREQ = 32'd50_000_000
)(
    input  wire         i_clk        , // system clock
    input  wire         i_rst        , // active-high reset
    input  wire [7 :0]  i_cmd_data   , // command byte
    input  wire         i_cmd_valid  , // command valid
    output wire         o_cmd_ready  , // command ready
    output wire [7 :0]  o_rd_data      // read data
);
```

Bit range spacing: `[7 :0]`, `[15:0]`, `[63:0]`. Align with nearby declarations.

## 7. File header + Revision

New modules: Vivado-style header. Fill `Company`, `Engineer`, `Target Devices`, `Description` when known; leave blank rather than inventing.

```verilog
`timescale 1ns / 1ps
//////////////////////////////////////////////////////////////////////////////////
// Company:
// Engineer:
// Create Date:
// Module Name:
// Target Devices:
// Description:
//
// Dependencies:
//
// Revision:
// Revision 0.01 - File Created
// Additional Comments:
//////////////////////////////////////////////////////////////////////////////////
```

When modifying: preserve ALL `Revision` history. Append new entry with version, description, date:
```verilog
// Revision 0.02 - add burst mode support
// Revision 0.03 - fix spi timing, 2026-05-17
```

## 8. Formatting & comments

- 4 spaces indentation, never tabs.
- `<=` in sequential logic, `=` in combinational logic.
- Spaces around assignment and binary operators.
- Declaration order: parameters → wires → registers → assign → always blocks → instances.

Every `input`, `output`, `reg`, and `wire` must have aligned end-of-line comment describing role.

```verilog
reg                                     r_cmd_valid     ; // command valid flag
reg  [7 :0]                             r_cmd_data      ; // command byte buffer
```

Section dividers:
```verilog
//----------------------------------------------------------------
// FSM: state register
//----------------------------------------------------------------
```

Common sections: `param`, `FSM`, `register`, `wire`, `module instance`.

For large modules, group registers/wires by instance with labels:
```verilog
// u0_spi_drive signal
reg  [31:0]  r_spi_op_data   ; // SPI command field
wire         w_spi_ready     ; // SPI operation ready

// u0_fifo_wr signal
wire         w_fifo_wr_en    ; // FIFO write enable
wire [7 :0]  w_fifo_wr_data  ; // FIFO write data
```

## 9. Module instantiation

Named port connections, one per line. Parameters via `#()`. Align names.

```verilog
mram_driver #(
    .P_INIT_CYCLE       (P_INIT_CYCLE       ),
    .P_OP_LEN           (P_OP_LEN           )
) u0_mram_driver (
    .i_clk              (i_clk              ),
    .i_rst              (i_rst              ),
    .i_operation_valid  (r_op_valid         )
);
```

Vendor IP preserves uppercase module name:
```verilog
FIFO_64X256 u0_FIFO_64X256 (
    .clk    (i_clk  ),
    .srst   (i_rst  ),
    ...
);
```

## 10. FSM (three-block style)

Required for >4 states or complex transitions.

- **Block 1** (seq): state register — `r_st_current <= r_st_next`
- **Block 2** (combo): `always @(*)` next-state logic — **must use `=`** (blocking), must have `default`
- **Block 3** (seq): state duration counter `r_st_cnt`

```verilog
// Block 1: state register
always @(posedge i_clk) begin
    if(i_rst) r_st_current <= P_INIT; else r_st_current <= r_st_next;
end

// Block 2: next state logic (combinational)
always @(*) begin
    case(r_st_current)
        P_INIT   : r_st_next = w_all_ready ? P_IDEL : P_INIT;
        P_IDEL   : r_st_next = w_trig     ? P_RUN  : P_IDEL;
        P_RUN    : r_st_next = w_done     ? P_IDEL : P_RUN;
        default  : r_st_next = P_INIT;
    endcase
end

// Block 3: state duration counter
always @(posedge i_clk) begin
    if(i_rst)                    r_st_cnt <= 16'd0;
    else if(r_st_current != r_st_next) r_st_cnt <= 16'd0;
    else                         r_st_cnt <= r_st_cnt + 16'd1;
end
```

- State names: `P_` prefix. Register: `r_st_current`. Next state: `r_st_next` (wire or reg, combo-driven).
- Complex transition conditions → extract to `assign` wires before case block.
- Simple flag/counter state machines also acceptable. Three-block required for >~4 states.

## 11. RTL correctness

- No `initial` blocks in synthesizable RTL (prefer reset logic).
- FSM must have explicit `default` state and recoverable paths.
- No combinational feedback loops. Break with register if needed.
- Register-to-register paths: keep ≤4-6 LUT levels. Insert pipeline if exceeding.

## 12. Reset style

Project default: active-high async reset.
```verilog
always @(posedge i_clk or posedge i_rst) begin
    if(i_rst) r_cnt <= 'd0; else r_cnt <= r_cnt + 'd1;
end
```

For derived/internal resets, use synchronous:
```verilog
always @(posedge i_clk) begin
    if(w_rst) r_state <= P_INIT; else r_state <= r_state_next;
end
```

Reset highest priority, always first in block.

<!--
AXI bus uses active-low synchronous reset (`i_axi_aresetn`):
always @(posedge i_axi_aclk) begin
    if(!i_axi_aresetn) r_axil_reg <= 32'd0; else r_axil_reg <= i_axil_data;
end
-->

## 13. CDC rules

- Call out every clock-domain crossing.
- Single-bit: synchronizer flops.
- Multi-bit: FIFO, dual-port RAM, or handshake.
- Name source and destination clock domains.

## 14. IP usage

Do not hand-roll vendor IP blocks (FIFO, BRAM, DSP, PLL). Leave placeholder instantiation with comments. User generates via Libero/Vivado.

```verilog
// FIFO_256x56: Libero SmartDesign COREFIFO, 256×56
// User: generate this IP and wire below
FIFO_256X56 u0_fifo_256x56 (
    .CLK    (i_clk                              ),
    .DATA   ({i_wr_addr[23:0], i_wr_data[31:0]} ),
    .WE     (i_wr_valid && o_wr_ready           ),
    .RE     (r_fifo_re                          ),
    .Q      (w_fifo_q[55:0]                     ),
    .EMPTY  (w_fifo_empty                       ),
    .FULL   (                                   )
);
```

Vendor IP uses device-specific primitives (BRAM, DSP, carry chains) hand-RTL cannot reliably infer.

## 15. Numeric literals

Explicit widths with readable separators:
- Decimal: `32'd49_999_999`, `8'd0`, `'d0`, `'d1`
- Hex: `8'hFF`, `16'h0800`
- Binary: `8'b1111_1111`
- Always include apostrophe: `8'hFF` not `8hFF`.

## 16. LUT architecture reference

| Device | LUT | Safe levels | Notes |
|--------|-----|-------------|-------|
| Xilinx 7-series | LUT6 | ≤5 | Two LUT5 per LUT6 |
| Xilinx UltraScale+ | LUT6 | ≤5 | LUT+FF per slice |
| Microchip PolarFire | LUT4 | ≤4 | LUT4C + carry chain |
| Microchip RTG4 | LUT4 | ≤4 | Similar to PolarFire |

Common costs: 32b adder (carry chain), 32b comparator (~3-4 levels), 32b MUX (1 level), barrel shifter (1-2), CRC tree (`ceil(log2(w))`).

## 17. Verification

- Self-checking testbench for new/changed RTL.
- Cover boundary + error paths, not just golden path.
- AXI interfaces: verify `valid`/`ready`, `last`, `keep`, metadata.
- Prefer assertions for interface handshake timing (SVA/PSL).
- Targets: line >95%, branch >90%, FSM state 100%.
- For important blocks, note if post-synthesis or gate-level sim is needed.

## Communication

- Precise timing: "2 cycles" not "quickly".
- Quantify: LUT, BRAM18K, DSP blocks.
- Flag CDC hazards, combinational feedback, inferred latch immediately.
- Chinese response when user writes Chinese.
