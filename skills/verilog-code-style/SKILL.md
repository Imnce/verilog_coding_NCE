---
name: verilog-code-style
description: Write, modify, review, or format any Verilog/SystemVerilog code using the user's Vivado-oriented Verilog style guide. Use this skill for every Verilog/RTL task: creating modules, editing .v/.sv files, implementing FPGA logic, writing test RTL, instantiating modules, cleaning formatting, reviewing style, or generating any Verilog code of any domain. Trigger even if the user only says "按风格", "整理代码", "写个模块", "写 Verilog", "改 RTL", or mentions Vivado/FPGA/RTL.
---

# Verilog Code Style Skill

Synthesizable production RTL for Xilinx and Microchip FPGA. Target: function, timing, area. Zero tolerance for metastability and unsafe CDC.

## Workflow

1. Read existing RTL before editing. Preserve behavior unless explicitly changing it.
2. Match this guide for new or substantially modified code. Tiny fixes: keep surrounding style.
3. New modules get a matching testbench unless explicitly scoped out.
4. If no toolchain, state that syntax was not compiled.
5. When unclear, ask: target device, clock, reset scheme, interface, timing budget.

## 1. Naming conventions

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

- Module names: lowercase snake_case (`mram_ctrl`, `flash_ctrl`).
- Instance names: `u0_module_name`, `u1_module_name` (prefix form). Legacy uppercase IP: `u0_FIFO_64X256`.
- Edge detection: `w_xxx_pose` (posedge), `w_xxx_nege` (negedge).
- Handshake: `w_xxx_handshake`.
- Pipeline delay: `_1d`, `_2d` suffix.

Every `parameter` / `localparam` has end-of-line comment:
```verilog
parameter  P_RST_CYCLE = 32'd100;  // reset cycle count
localparam P_INIT      = 4'd0;     // init state
```

Shared parameters go in `params.vh`, not duplicated across modules. Module-local parameters stay inside.

## 2. Always block discipline

**Space after `always`**: `always @(posedge i_clk)`.

**One register per always block**. Exception: multiple registers with identical reset + update conditions.

```verilog
// Good — single register per always
always @(posedge i_clk) begin
    if(i_rst) r_valid <= 1'd0; else r_valid <= i_valid;
end

// Allowed exception — identical conditions
always @(posedge i_clk) begin
    if(i_rst) begin
        r_u0_ready_1d <= 1'd0;
        r_u1_ready_1d <= 1'd0;
    end else begin
        r_u0_ready_1d <= w_u0_ready;
        r_u1_ready_1d <= w_u1_ready;
    end
end
```

**Always close with `else`**: every `if`/`else if` chain must end with explicit `else` assigning the register (even if holding).

```verilog
// Good
always @(posedge i_clk) begin
    if(i_rst)       r_state <= P_IDLE;
    else if(w_trig) r_state <= P_RUN;
    else            r_state <= r_state;
end
```

**Flat else-if**: single-register blocks use flat `else if` chain, not `else begin...end` nesting.

```verilog
// Good — flat
always @(posedge i_clk) begin
    if(i_rst)       r_aw_en <= 1'b1;
    else if(w_a)    r_aw_en <= 1'b1;
    else if(w_b)    r_aw_en <= 1'b0;
    else            r_aw_en <= r_aw_en;
end
```

**Valid flag clear guard**: when clearing on handshake, guard with `i_ready && r_valid` to prevent accidental clear.

```verilog
// Good
else if(i_ready && r_valid) r_valid <= 1'b0;
// Bad
else if(i_ready)            r_valid <= 1'b0;  // may clear when already low
```

**Explicit bit widths**: all literals carry explicit widths. `32'd0` not `'d0`.

```verilog
// Good
r_data <= 32'd0;
r_cnt  <= r_cnt + 8'd1;

// Bad
r_data <= 'd0;
r_cnt  <= r_cnt + 'd1;
```

## 3. Timing-critical rules

### Registered output isolation (r_o_xx)

Module output ports driven by registers, not combinational logic. Use `r_o_` prefix → `assign o_xxx = r_o_xxx`.

```verilog
// Good
reg  r_o_rd_ready;
assign o_rd_ready = r_o_rd_ready;

// Bad
assign o_rd_ready = (r_st_current == P_IDEL) & w_fifo_empty & w_all_ready;
```

### Register sub-module port signals

Control signals to instances (valid, type) driven by registers, not combinational mux.

```verilog
// Good
reg  r_op_valid;
reg  [1:0] r_op_type;

mram_driver u0 ( .i_operation_valid(r_op_valid), .i_operation_type(r_op_type) );

// Bad
mram_driver u0 ( .i_operation_valid(w_op_valid),   // combinational
                 .i_operation_type((r_st_current == P_WR_REQ) ? 2'd1 : 2'd2) );
```

### FSM edge-triggered state entry

Latch data or set flags using exact transition edge `r_st_next == TARGET && r_st_current == SOURCE`, not level-triggered `r_st_current == TARGET`. Fires once per entry, independent of `r_st_cnt`.

```verilog
// Good — exact edge, no counter dependency
always @(posedge i_clk) begin
    if(i_rst)                        r_op_type <= 2'd2;
    else if(r_st_next == P_WR_REQ && r_st_current == P_FIFO_LATCH)     r_op_type <= 2'd1;
    else if(r_st_next == P_RD_REQ && r_st_current == P_IDEL)           r_op_type <= 2'd2;
    else if(r_st_next == P_RD_REQ && r_st_current == P_WR_WAIT_DONE)   r_op_type <= 2'd2;
    else                              r_op_type <= r_op_type;
end

// Bad — level-triggered, fires every cycle in state
else if(r_st_current == P_WR_REQ) r_op_type <= 2'd1;

// Bad — r_st_cnt overflow risk
else if(r_st_current == P_WR_REQ && r_st_cnt == 16'd0) r_op_type <= 2'd1;
```

### Ready handshake register pattern

Ready output: register clears on handshake, asserts on completion edge. No combinational gating.

```verilog
always @(posedge i_clk) begin
    if(i_rst)                                     r_o_rd_ready <= 1'b0;
    else if(i_rd_valid && r_o_rd_ready)           r_o_rd_ready <= 1'b0;
    else if(r_st_current == P_RD_DONE && r_st_next == P_IDEL) r_o_rd_ready <= 1'b1;
    else                                          r_o_rd_ready <= r_o_rd_ready;
end
```

All internal consumers reference the registered version (`r_o_rd_ready`).

## 4. `default_nettype none + explicit wire ports

Every file starts with `` `default_nettype none `` and ends with `` `default_nettype wire ``.

Under `` `default_nettype none ``, all `input`/`output` ports must declare `wire` explicitly (vlog-2892):

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

## 5. Port declarations

One port per line. Align direction, width, name, comment. Trailing comma except last.

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

## 6. File header + Revision

New modules: Vivado-style header with `Company`, `Engineer`, `Target Devices`, `Description`.

When editing: preserve all `Revision` history. Append new entry:
```verilog
// Revision:
// Revision 0.01 - File Created
// Revision 0.02 - add burst mode support
// Revision 0.03 - fix spi timing, 2026-05-17
```

## 7. Formatting & section comments

- 4 spaces indentation, never tabs.
- `<=` in sequential, `=` in combinational.
- Spaces around operators.
- Declaration order: parameters → wires → registers → assign → always blocks → instances.

Every `reg` and `wire` has aligned end-of-line comment describing role.

Section dividers:
```verilog
//----------------------------------------------------------------
// FSM: state register
//----------------------------------------------------------------
```

Common sections: `param`, `FSM`, `register`, `wire`, `module instance`. For large modules, group registers/wires by instance with labels like `// u0_spi_drive signal`.

## 8. Module instantiation

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

Vendor IP preserves uppercase module name: `FIFO_64X256 u0_FIFO_64X256 (...)`.

## 9. FSM (three-block style)

Required for >4 states or complex transitions:

- **Block 1** (seq): `r_st_current <= r_st_next`
- **Block 2** (combo): `always @(*)`, next-state logic, **must use `=`**, must have `default`
- **Block 3** (seq): state duration counter `r_st_cnt`

```verilog
// Block 1
always @(posedge i_clk) begin
    if(i_rst) r_st_current <= P_INIT; else r_st_current <= r_st_next;
end

// Block 2
always @(*) begin
    case(r_st_current)
        P_INIT : r_st_next = w_all_ready ? P_IDEL : P_INIT;
        P_IDEL : r_st_next = w_trig     ? P_RUN  : P_IDEL;
        P_RUN  : r_st_next = w_done     ? P_IDEL : P_RUN;
        default: r_st_next = P_INIT;
    endcase
end

// Block 3
always @(posedge i_clk) begin
    if(i_rst)                    r_st_cnt <= 16'd0;
    else if(r_st_current != r_st_next) r_st_cnt <= 16'd0;
    else                         r_st_cnt <= r_st_cnt + 16'd1;
end
```

Complex transition conditions → extract to `assign` wires before the case block.

## 10. RTL correctness

- No `initial` blocks in synthesizable RTL (use reset logic).
- FSM must have explicit `default` and recoverable paths.
- No combinational feedback loops.
- Register-to-register paths: keep ≤4-6 LUT levels. Insert pipeline if exceeding.

## 11. Reset style

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

Reset highest priority, first in always block.

## 12. CDC rules

- Call out every clock-domain crossing.
- Single-bit: synchronizer flops.
- Multi-bit: FIFO, dual-port RAM, or handshake.
- Name source and destination domains.

## 13. IP usage

Do not hand-roll vendor IP blocks (FIFO, BRAM, DSP, PLL). Leave placeholder instantiation with comments. Let user generate via Libero/Vivado.

```verilog
// FIFO_256x32: Libero SmartDesign COREFIFO, 256×32
// User: generate this IP and wire below
FIFO_256x32 u0_FIFO_256x32 (
    .clk  (i_clk  ),
    .din  (w_din  ),
    .dout (w_dout ),
    ...
);
```

## 14. Numeric literals

Explicit widths with separators:
- Decimal: `32'd49_999_999`, `8'd0`
- Hex: `8'hFF`, `16'h0800`
- Binary: `8'b1111_1111`
- Always include apostrophe: `8'hFF` not `8hFF`.

## 15. LUT architecture reference

| Device | LUT | Levels (safe) | Notes |
|--------|-----|---------------|-------|
| Xilinx 7-series | LUT6 | ≤5 | Two LUT5 per LUT6 |
| Xilinx UltraScale+ | LUT6 | ≤5 | LUT+FF per slice |
| Microchip PolarFire | LUT4 | ≤4 | LUT4C + carry chain |
| Microchip RTG4 | LUT4 | ≤4 | Similar to PolarFire |

Common LUT costs: 32b adder (carry chain), 32b comparator (~3-4 levels), 32b MUX (1 level), barrel shifter (1-2), CRC tree (`ceil(log2(w))`).

## 16. Verification

- Self-checking testbench for new/changed RTL.
- Cover boundary + error paths, not just golden.
- AXI interfaces: verify valid/ready, last, keep, metadata.
- Targets: line >95%, branch >90%, FSM state 100%.

## Communication

- Precise timing: "2 cycles", not "quickly".
- Quantify: LUT count, BRAM, DSP blocks.
- Flag CDC, combinational feedback, unsafe patterns immediately.
- Chinese response when user writes Chinese.
