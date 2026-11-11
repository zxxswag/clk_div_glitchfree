# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

Single-module Verilog RTL: `clk_div_glitchfree.v`, a glitch-free clock divider with seamless ratio switching.
- Ports: `clk`, `rst_n` (async, active low), `div_sel[2:0]` (= N-1, gives ÷1 to ÷8), `clk_div`.
- No testbench, Makefile, or git repo yet.

## Tools (host eda-aiagent, user aiagent, no sudo)

- Synopsys VCS / Verdi Y-2026.03-SP2 are on PATH (`vcs`, `verdi`). iverilog, verilator, and yosys are **not** installed.
- Lint/compile (run in a scratch dir so build artifacts stay out of the source dir):
  ```
  mkdir -p /tmp/cdg_build && cd /tmp/cdg_build
  vcs -full64 -sverilog +lint=all ~/clk_div_glitchfree/clk_div_glitchfree.v -o simv
  ```
- Simulating with a testbench: `vcs -full64 -sverilog -debug_access+all -kdb tb.sv clk_div_glitchfree.v -o simv && ./simv`, then view waveforms with `verdi -ssf novas.fsdb`.

## Architecture (how the design works)

The datapath runs as a pipeline in this order; the stages depend on each other:
1. **Reset synchronizer** (`rst_sync1/2` → `rstn`): async assert, sync release. Note that `rstn` is currently generated but **unused**: every other always block still resets on the raw `rst_n`.
2. **div_sel 2-flop synchronizer** (`sel_meta` → `sel_sync`): treats `div_sel` as quasi-static config. Multi-bit sync has no gray/handshake protection, so `div_sel` must stay stable across the sync window.
3. **Counter + boundary load**: `cnt` counts 0..`ratio`; on `wrap` (`cnt==ratio`) it loads `ratio <= sel_sync`. This is the seamless-switch mechanism: the ratio only changes at a period boundary.
4. **Registered enable**: `en <= (cnt_nxt < half)`, `half = ceil(N/2) = (ratio>>1)+1`. Output is high for the first H clk periods of each N-period window. Duty cycle is not 50%.
5. **Latch-based ICG**: a low-transparent latch (`en_l`) + AND gives `clk_div = clk & en_l`. This is what guarantees no glitches. For ASIC, replace it with a library ICG (e.g. CKLNQ); for FPGA, it maps to BUFGCE.

## Conventions

- Comments in the RTL are in Chinese; keep that style when editing.
- Keep the latch + AND as an explicit, recognizable ICG pattern. Do not "optimize" it into combinational gating.
