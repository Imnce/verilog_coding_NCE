---
name: verilog-code-style
description: Write, modify, review, or format any Verilog/SystemVerilog code using the user's Vivado-oriented Verilog style guide. Use this skill for every Verilog/RTL task: creating modules, editing .v/.sv files, implementing FPGA logic, writing test RTL, instantiating modules, cleaning formatting, reviewing style, or generating any Verilog code of any domain. FIFO/AXI Stream are examples only, not limits. Trigger even if the user only says "按风格", "整理代码", "写个模块", "写 Verilog", "改 RTL", or mentions Vivado/FPGA/RTL.
---

# Verilog Code Style Skill

Use this skill when producing or editing any Verilog/SystemVerilog code for the user. The user's preferred style is Vivado-oriented, Chinese-comment friendly, and emphasizes readable alignment, consistent signal prefixes, explicit timing/reset behavior, CDC correctness, and timing closure discipline. Treat the user as an FPGA digital design engineer working on synthesizable production RTL for Xilinx and Microchip platforms. FIFO/AXI Stream patterns are included as common examples, but the style applies to all Verilog modules and RTL domains.

## Core mindset

Write synthesizable, maintainable RTL that meets function, area, timing, and power goals.

Keep zero-tolerance mindset for metastability and unsafe clock-domain crossings. Distinguish clearly between simulation-pass RTL and board-stable production RTL.

For any non-trivial module, think about these constraints before writing code:

- target device resources: LUT, BRAM, DSP
- clock architecture and clock-domain boundaries
- critical timing paths and pipeline depth
- interface protocol details such as AXI4, AXI4-Lite, and AXI4-Stream

## First steps

1. If editing existing files, read the relevant RTL before changing it and preserve the module's existing behavior unless the user explicitly asks for functional changes.
2. Prefer editing existing files over creating new ones. Only create a new `.v`/`.sv` file when the task requires a new module.
3. Match this style guide for any new or substantially modified RTL. For tiny local fixes, keep surrounding style consistent while avoiding unrelated churn.
4. For new modules, expect a matching testbench. If the user asks for module implementation and no testbench exists, create or update one unless the user explicitly scopes it out.
5. For non-trivial RTL, check syntax or run the available project simulation/build command if the repository provides one. If no toolchain is present, say that syntax was not compiled.
6. When requirements are unclear, ask for target device, clock frequency, reset scheme, interface protocol, and resource/timing budget before locking implementation details.

## File header

For new Vivado-style modules, include a standard header with these fields:

```verilog
`timescale 1ns / 1ps
//////////////////////////////////////////////////////////////////////////////////
// Company: 
// Engineer: 
// 
// Create Date: 
// Design Name: 
// Module Name: module_name
// Project Name: 
// Target Devices: 
// Tool Versions: 
// Description: 
// 
// Dependencies: 
// 
// Revision:
// Revision 0.01 - File Created
// Additional Comments:
// 
//////////////////////////////////////////////////////////////////////////////////
```

Fill `Company`, `Engineer`, `Target Devices`, and `Description` when the user provides the information. Otherwise leave them blank rather than inventing values. Use `timescale 1ns / 1ps`.

When modifying existing file, preserve entire `Revision` history. Append new entry with version, description, date:
```verilog
// Revision:
// Revision 0.01 - File Created
// Revision 0.02 - add burst mode support     // <-- keep old, add new below
// Revision 0.03 - fix spi timing, 2026-05-17
```

Add a short functional description block inside complex modules when it clarifies chip model, command flow, timing requirements, or interface behavior.

## Naming conventions

Use lowercase snake_case for module names, for example `mram_ctrl`, `flash_ctrl`, `top`.

Use these prefixes consistently:

- `i_` input ports
- `o_` output ports
- `r_` register signals
- `w_` combinational wire signals
- `ri_` registered version of an input signal
- `ro_` registered version of an output signal

Use `P_` for parameters and localparams. Keep parameter names uppercase with underscores. Every `parameter` and `localparam` must have an end-of-line comment:

```verilog
parameter  P_RST_CYCLE = 32'd100;  // reset cycle count
localparam P_INIT      = 3'd0;     // init state
localparam P_IDEL      = 3'd1;     // idle state
localparam P_RUN       = 3'd2;     // run state
```

For shared/cross-module parameters, prefer a dedicated parameter file (e.g. `params.vh`) instead of duplicating `parameter` definitions across modules. Use `` `include "params.vh" `` to bring parameters into each module that needs them. Module-local parameters (FSM states, internal constants) stay inside the module.

```verilog
// params.vh — 全局参数定义
localparam P_CLK_FREQ     = 50_000_000;  // 系统时钟频率 (Hz)
localparam P_SPI_CLK_DIV  = 4;           // SPI 时钟分频系数
```

For edge detection (rising/falling edge), use `w_xxx_pose` and `w_xxx_nege`:

```verilog
wire w_vld_pose = i_valid && !r_i_valid;  // rising edge detect
wire w_vld_nege = i_ready && !r_i_ready;  // falling edge detect
```

For handshake signals, use `w_xxx_handshake`:

```verilog
wire w_op_handshake = i_op_valid && o_op_ready;  // valid/ready handshake
```

For module instances, use `u0_module_name`, `u1_module_name`, or `ux_module_name` (prefix form). Keep legacy uppercase IP instance style only when the existing project already uses it, e.g. `u0_FIFO_64X256`.

## Port declarations

Align port direction, width, signal name, delimiter, and comment columns. Put one port per line. Use trailing commas except the final port when that is the surrounding style. Add a short end-of-line comment for every `input` and `output` port explaining the signal purpose or timing/handshake meaning.

Preferred pattern:

```verilog
module mram_ctrl#(
    parameter       P_CLK_FREQ          = 32'd50_000_000
)(
    input           i_clk               , // system clock
    input           i_rst               , // active-high reset
    input  [7 :0]   i_cmd_data          , // command byte
    input           i_cmd_valid         , // command valid pulse
    output          o_cmd_ready         , // command accept ready
    output [7 :0]   o_rd_data           , // read data byte
    output          o_rd_valid            // read data valid pulse
);
```

Use spaces inside bit ranges to preserve alignment, especially for narrow widths: `[7 :0]`, `[15:0]`, `[63:0]`. Align wider declarations with nearby declarations.

### Explicit wire for ports under `` `default_nettype none ``

When using `` `default_nettype none ``, every `input` and `output` port must be explicitly declared as `wire`. Without it, synthesis tools report "net type was not explicitly declared" (vlog-2892).

```verilog
// Good — explicit wire under `default_nettype none
module example #(
    parameter      P_WIDTH         = 32'd8
)(
    input  wire                     i_clk               , // system clock
    input  wire                     i_rst               , // active-high reset
    input  wire [P_WIDTH-1 :0]      i_data              , // input data
    output wire                     o_valid             , // output valid
    output wire [P_WIDTH-1 :0]      o_data               // output data
);

// Bad — missing wire under `default_nettype none
module example #(
    parameter      P_WIDTH         = 32'd8
)(
    input                            i_clk               , // ERROR: net type not declared
    input                            i_rst               , // ERROR: net type not declared
    input         [P_WIDTH-1 :0]     i_data              , // ERROR: net type not declared
    output                           o_valid             ,
    output        [P_WIDTH-1 :0]     o_data
);
```

## RTL correctness rules

- Use nonblocking assignments `<=` in sequential logic.
- Use blocking assignments `=` in combinational logic.
- Do not use `initial` blocks in synthesizable RTL unless the target FPGA flow and project style explicitly rely on vendor-supported initialization. Prefer reset logic.
- FSMs need explicit default states and recoverable error/default paths. Avoid unrecoverable stuck states.
- Avoid combinational feedback loops. If feedback is required, break it with a register.
- Register-to-register paths should not contain long combinational chains. If logic approaches more than about 4 LUT levels or misses target frequency, prefer pipeline insertion or logic restructuring instead of relying on synthesis magic.

### Registered output isolation (r_o_xx)

Module output ports should be driven by registered signals, not combinational logic. This prevents long combinational paths from feeding into downstream modules and improves timing closure.

Use `r_o_` prefix for output register, then `assign o_port = r_o_port`:

```verilog
// Good — registered output
reg                                     r_o_rd_ready        ; // registered read ready output
reg                                     r_o_wr_ready        ; // registered write ready output

assign o_rd_ready = r_o_rd_ready;
assign o_wr_ready = r_o_wr_ready;

// Bad — combinational output
assign o_rd_ready = (r_st_current == P_IDEL) & w_fifo_empty & w_all_ready;
```

Exception: outputs that are constant (e.g. `o_wr_ready = 1'b1`) or already driven by a dedicated register from a sub-module do not need a separate `r_o_` wrapper.

### Register signals driving sub-module instances

Control signals connected to sub-module instance ports (valid, type, op_code, etc.) must be driven by registers, not combinational expressions.

```verilog
// Good — registered operation valid and type
reg                                     r_op_valid          ; // registered operation valid to drivers
reg  [1 :0]                             r_op_type           ; // registered operation type (1:write 2:read)

always @(posedge i_clk) begin
    if(i_rst)
        r_op_valid <= 1'b0;
    else if(r_st_next == P_WR_REQ && r_st_current == P_FIFO_LATCH)
        r_op_valid <= 1'b1;
    else if(r_st_next == P_RD_REQ && r_st_current == P_IDEL)
        r_op_valid <= 1'b1;
    else if(w_all_handshake)
        r_op_valid <= 1'b0;
    else
        r_op_valid <= r_op_valid;
end

mram_driver u0_mram_driver (
    .i_operation_type               (r_op_type                      ),
    .i_operation_valid              (r_op_valid                     ),
    ...
);

// Bad — combinational mux and valid in port map
mram_driver u0_mram_driver (
    .i_operation_type               ((r_st_current == P_WR_REQ) ? 2'd1 : 2'd2),
    .i_operation_valid              (w_op_valid                     ),
    ...
);
```

### FSM edge-triggered state entry detection

When latching data or setting flags on entering a specific FSM state, use the exact transition edge `r_st_next == P_TARGET && r_st_current == P_SOURCE` instead of level-triggered `r_st_current == P_TARGET`. This fires exactly once on the state transition and does not depend on `r_st_cnt`, avoiding counter overflow concerns.

```verilog
// Good — edge-triggered, exact transition
always @(posedge i_clk) begin
    if(i_rst)
        r_op_type <= 2'd2;
    else if(r_st_next == P_WR_REQ && r_st_current == P_FIFO_LATCH)
        r_op_type <= 2'd1;
    else if(r_st_next == P_RD_REQ && r_st_current == P_IDEL)
        r_op_type <= 2'd2;
    else if(r_st_next == P_RD_REQ && r_st_current == P_WR_WAIT_DONE)
        r_op_type <= 2'd2;
    else
        r_op_type <= r_op_type;
end

// Bad — level-triggered, fires every cycle while in state
always @(posedge i_clk) begin
    if(i_rst)
        r_op_type <= 2'd2;
    else if(r_st_current == P_WR_REQ)
        r_op_type <= 2'd1;
    else if(r_st_current == P_RD_REQ)
        r_op_type <= 2'd2;
    else
        r_op_type <= r_op_type;
end

// Bad — uses r_st_cnt, overflow risk at 16'hFFFF -> 0
always @(posedge i_clk) begin
    if(i_rst)
        r_op_type <= 2'd2;
    else if(r_st_current == P_WR_REQ && r_st_cnt == 16'd0)
        r_op_type <= 2'd1;
end
```

When a state has multiple entry paths, list each transition explicitly:

```verilog
else if(r_st_next == P_RD_REQ && r_st_current == P_IDEL)
    ...
else if(r_st_next == P_RD_REQ && r_st_current == P_WR_WAIT_DONE)
    ...
```

### Ready handshake register pattern

For ready/valid handshake output ports, use a register that clears on handshake and asserts on a completion edge — no combinational gating:

```verilog
// Good — handshake-based registered ready
always @(posedge i_clk) begin
    if(i_rst)
        r_o_rd_ready <= 1'b0;
    else if(i_rd_valid && r_o_rd_ready)
        r_o_rd_ready <= 1'b0;
    else if(r_st_current == P_RD_DONE && r_st_next == P_IDEL)
        r_o_rd_ready <= 1'b1;
    else
        r_o_rd_ready <= r_o_rd_ready;
end

// Bad — combinational gating, long path
assign o_rd_ready = (r_st_current == P_IDEL) & w_fifo_empty & w_all_ready;
```

All internal consumers of the ready signal must reference the registered version (`r_o_rd_ready`), not a combinational wire.

## Formatting

- Use 4 spaces for indentation, never tabs.
- Keep `always`, `if/else`, and `case` bodies consistently indented.
- Use nonblocking assignments `<=` in sequential logic.
- Use blocking assignments `=` in combinational logic or continuous assign expressions when appropriate.
- Put spaces around assignment operators and binary operators.
- Keep declarations ordered as: registers, wires, then functional sections.
- Add aligned end-of-line comments for every `reg` and `wire` declaration. The comment should describe the signal role, not restate the name.
- For larger modules with many instances, group declarations by the instance or functional block they serve. Use section dividers or compact comment headers to separate groups, so readers can quickly find the signals that connect to each instance.
- Keep declaration groups visually tidy: align type, width, name, and comment columns across nearby signals.

## Always block discipline

Always add a space between `always` and `@()`: `always @(posedge i_clk)`.

One `always` block should drive **only one register**, except when the reset condition and update condition are identical across multiple registers.

Good — single register per always:

```verilog
always @(posedge i_clk) begin
    if(i_rst)
        r_valid <= 1'd0;
    else
        r_valid <= i_valid;
end

always @(posedge i_clk) begin
    if(i_rst)
        r_data <= 'd0;
    else if(i_valid)
        r_data <= i_data;
end
```

Allowed exception — identical conditions, simple parallel delay:

```verilog
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
```

Bad — different conditions mixed in one block:

```verilog
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

FSM uses three always blocks (see FSM section below): current state register, combinational next state, and state duration counter. These are three separate blocks, not an exception to the one-reg-per-always rule.

### Always block convergence

Every `if` / `else if` chain inside an `always` block must close with a final `else`. The `else` must explicitly assign the register, even if holding the current value. This prevents latch inference in combinational blocks and makes designer intent explicit in sequential blocks.

```verilog
// Good — final else收敛
always @(posedge i_clk) begin
    if(i_rst)
        r_state <= P_IDLE;
    else if(w_trig)
        r_state <= P_RUN;
    else
        r_state <= r_state;     // explicit hold
end

// Bad — 无else，虽综合器可推断保持，但意图不明
always @(posedge i_clk) begin
    if(i_rst)
        r_state <= P_IDLE;
    else if(w_trig)
        r_state <= P_RUN;
end

// Bad — else分支未赋值目标寄存器
always @(posedge i_clk) begin
    if(i_rst)
        r_cnt <= 8'd0;
    else if(w_inc)
        r_cnt <= r_cnt + 8'd1;
    // missing else — r_cnt保持? latch? 不明确
end
```

### Flat else-if rule

Single-register always block with multiple mutually-exclusive conditions must use flat `else if` chain, not `else begin...end` nesting.

```verilog
// Good — flat else if chain
always @(posedge i_clk) begin
    if(i_rst)
        r_aw_en <= 1'b1;
    else if(w_a)
        r_aw_en <= 1'b1;
    else if(w_b)
        r_aw_en <= 1'b0;
    else
        r_aw_en <= r_aw_en;
end

// Bad — nested else begin...end hides same-condition logic
always @(posedge i_clk) begin
    if(i_rst)
        r_aw_en <= 1'b1;
    else begin
        if(w_a)
            r_aw_en <= 1'b1;
        else if(w_b)
            r_aw_en <= 1'b0;
        else
            r_aw_en <= r_aw_en;
    end
end
```

Rationale: nested `else begin...end` adds indentation without expressing additional structure. Flat chain equally correct, more readable. Reserve `else begin...end` for cases where multiple registers share same reset and update conditions.

### Valid flag clear guard

When clearing a registered valid flag on ready handshake, guard with the flag itself (`i_ready && r_valid`) to prevent accidental clear when flag is already low.

```verilog
// Good — guard with r_valid
always @(posedge i_clk) begin
    if(i_rst)
        r_valid <= 1'b0;
    else if(i_ready && r_valid)
        r_valid <= 1'b0;
    else if(w_trig)
        r_valid <= 1'b1;
    else
        r_valid <= r_valid;
end

// Bad — bare i_ready may clear when r_valid is already 0
always @(posedge i_clk) begin
    if(i_rst)
        r_valid <= 1'b0;
    else if(i_ready)          // should be i_ready && r_valid
        r_valid <= 1'b0;
    else if(w_trig)
        r_valid <= 1'b1;
    else
        r_valid <= r_valid;
end
```

Applies to: `r_rvalid`, `r_mram_rd_valid`, `r_bvalid`, and any registered valid flag that clears on handshake.

### Explicit bit widths

All literals must carry explicit bit widths. Never use width-inferred forms like `'d0` or `'d1`. Match the width to the target register or expression context.

```verilog
// Good — explicit widths
reg [7:0] r_cnt;
always @(posedge i_clk) begin
    if(i_rst)
        r_cnt <= 8'd0;
    else if(w_inc)
        r_cnt <= r_cnt + 8'd1;
    else
        r_cnt <= r_cnt;
end

// Bad — width inferred
reg [7:0] r_cnt;
always @(posedge i_clk) begin
    if(i_rst)
        r_cnt <= 'd0;           // width inferred, may cause truncation warning
    else if(w_inc)
        r_cnt <= r_cnt + 'd1;   // same issue
end

// Good — explicit widths, all registers
reg [31:0] r_data;
always @(posedge i_clk) begin
    if(i_rst)
        r_data <= 32'd0;
    else if(w_valid)
        r_data <= i_data;
    else
        r_data <= r_data;
end
```

Exception for `P_` constants declared with explicit widths — those may be used directly without re-specifying width.

## CDC and clock-domain rules

Call out every clock-domain crossing explicitly in code review, design notes, or response text.

For FPGA work:

- Simple single-bit CDC usually uses synchronizer flops.
- Multi-bit data CDC should usually use FIFO, dual-port RAM scheme, handshake, or another architecture-safe transfer method.
- Do not move pulses, counters, or buses across domains with naive single-flop wiring.
- Mention source clock domain and destination clock domain when discussing a crossing.

If a signal crosses from `i_clk_a` to `i_clk_b`, say so directly and show synchronization strategy.

## Reset and clocking style

<!--
AXI bus uses active-low synchronous reset (`i_axi_aresetn`). Use directly in sensitivity list — no synchronizer needed:

```verilog
always @(posedge i_axi_aclk, negedge i_axi_aresetn) begin
    if(!i_axi_aresetn)
        r_axil_reg <= 32'd0;
    else
        r_axil_reg <= i_axil_data;
end
```
-->

For non-AXI modules, use the project's chosen reset style (typically active-high async).

For asynchronous active-high reset, use:

```verilog
always @(posedge i_clk,posedge i_rst) begin
    if(i_rst) begin
        r_cnt <= 'd0;
    end else begin
        r_cnt <= r_cnt + 'd1;
    end
end
```

Reset has highest priority and appears first inside the `always` block. Use `'d0` and `'d1` for reset/default decimal constants where width is inferred.

For derived/internal resets or generated reset signals, use synchronous reset style:

```verilog
always @(posedge i_clk) begin
    if(w_rst) begin
        r_state <= P_INIT;
    end else begin
        r_state <= r_state_next;
    end
end
```

## Timing closure rules

When timing is tight, prefer pipeline insertion, retiming-friendly structure, or logic reduction over hoping the tool will fix it.

For path analysis and implementation guidance:

- identify critical paths early
- keep I/O constraints consistent with external device timing data
- call out external `set_input_delay` and `set_output_delay` requirements when relevant
- keep resource usage within budget and leave headroom for later feature growth
- when discussing timing, quantify it with cycles, MHz, slack, LUT depth, or resource counts instead of vague language

## Numeric literals

Use Verilog-sized literals with readable separators:

- Decimal: `32'd49_999_999`, `8'd0`, `'d0`, `'d1`
- Hex: `8'hFF`, `16'h0800`
- Binary: `8'b1111_1111`

Do not use malformed shorthand like `8hFF`; include the apostrophe.

## Section comments

Use section dividers to separate functional areas:

```verilog
//----------------------------------------------------------------
// param
//----------------------------------------------------------------
```

Common section names include `param`, `FSM`, `rst_gen instance`, `register`, `wire`, `cmd_ctrl`, `data_path`, and `module instance`.

When a file contains many module instances, split the `register` and `wire` sections by instance or subsystem using clear comments. Prefer concise labels such as `// u0_spi_drive signal`, `// u0_fifo_wr signal`, or Chinese equivalents that match the file. Keep each group close in naming and purpose, and avoid mixing unrelated instance signals in one block.

Use single-line `//` comments only where they clarify meaning that is not obvious from names. Chinese comments are acceptable and preferred when they match the surrounding file. For packed metadata fields, use compact bit layout comments such as:

```verilog
// 1bMF,16dlen,1bsplit,8dtype,13doffset,16dID
```

## Declaration comments and grouping

Every `input`, `output`, `reg`, and `wire` declaration should include a short aligned end-of-line comment. Use the comment to capture purpose, valid/ready direction, byte/bit meaning, clock domain, reset behavior, or the connected instance when that is the clearest explanation.

```verilog
//----------------------------------------------------------------
// register
//----------------------------------------------------------------
reg                                     r_cmd_valid                         ; // command valid flag
reg  [7 :0]                             r_cmd_data                          ; // command byte buffer

// u0_spi_drive signal
reg  [31:0]                             r_spi_op_data                       ; // SPI command/address field
wire                                    w_spi_ready                         ; // SPI operation ready
wire [7 :0]                             w_spi_read_data                     ; // SPI read byte

// fifo_wr_u0 signal
wire                                    w_fifo_wr_en                        ; // FIFO write enable
wire [7 :0]                             w_fifo_wr_data                      ; // FIFO write data
```

For small modules, one `register` block and one `wire` block is enough. For larger modules, group signals by instance (`u0_spi_drive`, `u0_fifo_wr`, etc.) or by subsystem before writing the logic, and use comment dividers between groups. Keep the grouping consistent with the later module instance order when possible.

## Module instantiation

Pass parameters with `#()` syntax. Align parameter names and signal connections. Use explicit named port connections, one port per line.

```verilog
udp_rx#(
    .P_SOURCE_PORT      (P_SOURCE_PORT          ),
    .P_TARGET_PORT      (P_TARGET_PORT          )
)
u0_udp_rx
(
    .i_clk              (i_clk                  ),
    .i_rst              (i_rst                  ),
    .i_set_source_port  (i_set_source_port      )
);
```

For vendor/IP modules whose names are uppercase in the project, preserve that name:

```verilog
FIFO_64X256 u0_FIFO_64X256
(
    .clk                (i_clk                  ),
    .srst               (i_rst                  )
);
```

## Common RTL patterns

### AXI Stream

Use standard AXI Stream signal concepts: `data`, `user`, `keep`, `last`, `valid`, `ready`. Data width is commonly 64 bits. `user` may carry metadata such as length, type, MAC address, split flag, offset, or ID.

When adding AXI Stream logic:

- Preserve valid/ready handshakes.
- Register outputs when needed using `ro_` prefixes.
- Avoid dropping `last`, `keep`, or `user` metadata across FIFOs or pipeline stages.

### FIFO

Common generated FIFO IP names include `FIFO_64X256`, `FIFO_8X32`, and `FIFO_80X32`. Separate data FIFO, user/control FIFO, and keep FIFO when the design already uses that pattern.

### IP usage

When a design requires common IP blocks (FIFO, BRAM, dual-port RAM, DSP macro, PLL, transceiver, etc.), **do not hand-roll them in RTL**. Tell the user which IP is needed (type, width, depth, ports) and leave a placeholder instantiation with clear comments. Let the user generate the IP through the vendor tool (Libero SmartDesign, Vivado IP Catalog, etc.) and wire it in.

```verilog
// FIFO_256x32: 256-deep x 32-bit FIFO, generated via Libero SmartDesign
// User: generate this IP and connect ports below
FIFO_256x32 u0_FIFO_256x32
(
    .clk    (i_clk      ),
    .srst   (i_rst      ),
    .din    (w_din      ),
    .wr_en  (w_wr_en    ),
    .rd_en  (w_rd_en    ),
    .dout   (w_dout     ),
    .full   (w_full     ),
    .empty  (w_empty    )
);
```

Rationale: vendor IP blocks use device-specific primitives (BRAM, DSP, carry chains) that hand-RTL cannot infer reliably. Placeholder keeps the interface clear without guessing synthesis results.

### Pipeline

Use `_1d`, `_2d`, `_3d` suffixes for delayed pipeline stages. Use `generate` for repeated structures when it reduces duplication without making the RTL harder to read.

### FSM

Use three-block FSM style for all explicit state machines:

- **Block 1** — sequential: state register (`r_st_current <= r_st_next`)
- **Block 2** — combinational: next state logic (`always @(*)`), must use `=` (blocking assignment)
- **Block 3** — sequential: state duration counter (`r_st_cnt`)

```verilog
//----------------------------------------------------------------
// FSM: state register
//----------------------------------------------------------------
always @(posedge i_clk) begin
    if(i_rst)
        r_st_current <= P_R_REG;
    else
        r_st_current <= r_st_next;
end

//----------------------------------------------------------------
// FSM: next state logic (combinational)
//----------------------------------------------------------------
always @(*) begin
    case(r_st_current)
        P_R_REG     : r_st_next = w_io_drive_act ? P_REG_CHECK : P_R_REG;
        P_REG_CHECK : r_st_next = i_reg_valid ? (i_reg_data[0] ? P_REG_WAIT : (!r_w_reg_flag ? P_W_EN : P_IDEL)) : P_REG_CHECK;
        P_REG_WAIT  : r_st_next = (r_st_cnt == 16'd255) ? P_R_REG : P_REG_WAIT;
        P_IDEL      : r_st_next = w_operation_active ? P_RUN : P_IDEL;
        P_RUN       : r_st_next = (ri_operation_type == P_OP_READ) ? P_R_INS : P_W_EN;
        P_W_EN      : r_st_next = w_io_drive_act ? (!r_w_reg_flag ? P_W_REG : (ri_operation_type == P_OP_WRITE ? P_W_INS : P_IDEL)) : P_W_EN;
        P_W_INS     : r_st_next = w_io_drive_act ? P_W_DATA : P_W_INS;
        P_W_DATA    : r_st_next = i_user_op_ready ? P_R_REG : P_W_DATA;
        P_R_INS     : r_st_next = w_io_drive_act ? P_R_DATA : P_R_INS;
        P_R_DATA    : r_st_next = i_user_op_ready ? P_IDEL : P_R_DATA;
        P_W_REG     : r_st_next = w_io_drive_act ? P_R_REG : P_W_REG;
        default     : r_st_next = P_R_REG;
    endcase
end

//----------------------------------------------------------------
// FSM: state duration counter
//----------------------------------------------------------------
always @(posedge i_clk) begin
    if(i_rst)
        r_st_cnt <= 16'd0;
    else if(r_st_current != r_st_next)
        r_st_cnt <= 16'd0;
    else
        r_st_cnt <= r_st_cnt + 16'd1;
end
```

Rules:
- Block 2 must use blocking assignment `=`, not `<=`. This is a combinational block — `always @(*)` infers no latches when all outputs are assigned in every branch.
- Block 2 must close with `default` in the case statement.
- State names use `P_` prefix (localparam), state register uses `r_st_current`, next state uses `r_st_next` (wire or reg, but must be driven combinationally).
- Transition conditions in Block 2 should be compact; complex multi-line conditions go in separate `assign` wires before the case block.
- Simple flag/counter state machines are also acceptable. Three-block style is required when the FSM has more than ~4 states or complex transition conditions.

## Logic levels and LUT architecture

Target devices use different LUT architectures. Know the target before coding:

| Device Family | LUT Type | Inputs/LUT | Notes |
|---------------|----------|------------|-------|
| Xilinx 7-series | LUT6 | 6 | Two LUT5 share one LUT6 |
| Xilinx UltraScale/+ | LUT6 | 6 | Combined LUT+FF per CLB slice |
| Microchip PolarFire | LUT4 | 4 | LUT4C with carry chain |
| Microchip RTG4 | LUT4 | 4 | Similar to PolarFire |
| Intel Cyclone V | ALM | 8 (adaptive) | Fracturable into two LUT4 |

Logic level guidelines:

- Single-cycle register-to-register path: **max 6 LUT levels** for typical FPGA fabrics at 100-200 MHz
- For LUT4 architectures (PolarFire, RTG4): aim **≤4 levels** in a single cycle at higher clock frequencies
- For LUT6 architectures (7-series, UltraScale): **≤5 levels** is safe zone at 150+ MHz
- Count levels as number of LUTs on the longest combinational path between two registers
- When estimating: `LUT_levels ≈ ceil(log2(signal_width))` for wide combinational ops (reductions, comparators)
- If a path exceeds the limit, insert pipeline registers or restructure logic

Common high-level patterns and their typical LUT cost:

- 32-bit adder: 8-10 LUT4 levels or 5-6 LUT6 levels (carry chain bypasses LUTs)
- 32-bit comparator (==,<,>): 3-4 LUT levels (reduction tree)
- Priority encoder (32→5): 2-3 LUT levels
- 32-bit MUX (2:1): 1 LUT level
- Barrel shifter: 1-2 LUT levels
- CRC/XOR tree: `ceil(log2(width))` levels

## Verification expectations

For new or meaningfully changed RTL:

- Provide or update a self-checking testbench when practical.
- Cover boundary conditions and abnormal/error paths, not only the golden path.
- Prefer assertions for interface handshake timing when the project uses SVA/PSL.
- For AXI-style interfaces, verify `valid`/`ready`, `last`, `keep`, and metadata preservation.
- If coverage tools are available, aim for line coverage >95%, branch coverage >90%, and FSM state coverage 100%.
- For important blocks, mention whether post-synthesis or gate-level simulation is needed for signoff.

## Coding approach

When writing RTL from scratch:

1. Define parameters and ports first.
2. For input ports that represent commands or handshakes, register them for one clock before decoding if timing or edge detection requires it.
3. Define one register for each output port first, then assign output wires from those registers.
4. Write simple register logic before complex control logic.
5. For edge detection, use `w_xxx_pose` / `w_xxx_nege` naming, e.g. `assign w_sig_pose = i_sig && !r_i_sig;`.
6. Prefer explicit FSMs for complex multi-cycle flows.

## Communication style

When discussing FPGA/RTL work with the user:

- Be precise about timing: say "`valid` high to `ready` response is at most 2 clock cycles" instead of "quickly".
- Quantify resources when known or estimable: LUT, BRAM18K, DSP48E2/DSP blocks.
- Mark CDC hazards immediately and name both clock domains.
- Flag dangerous designs plainly, such as combinational feedback, unsafe CDC, inferred latch, or handshake data loss.
- Use Chinese for final response when the user writes Chinese.

## Final response

When reporting back to the user, mention the files changed and whether syntax/simulation was run. Keep the response concise and in Chinese if the user is using Chinese.
