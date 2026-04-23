# Verilog Counter Smoke Test Implementation Plan

> Status: Historical implementation plan. The work described here may already
> be partially or fully implemented. Use the corresponding architecture page and
> current source tree as the source of truth.

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a minimal Verilog counter example and testbench that can be compiled and simulated locally to confirm the toolchain works.

**Architecture:** Create one RTL module, `counter.v`, that exposes `clk`, `rst_n`, and a 4-bit `count`. Create one self-contained testbench, `tb_counter.v`, that generates a clock, drives reset, monitors the output, and ends the simulation after a short run.

**Tech Stack:** Verilog HDL, `iverilog`, `vvp`

---

### Task 1: Add the failing simulation harness

**Files:**
- Create: `tb_counter.v`

- [ ] **Step 1: Write the failing test**

```verilog
`timescale 1ns/1ps

module tb_counter;
    reg clk;
    reg rst_n;
    wire [3:0] count;

    counter uut (
        .clk(clk),
        .rst_n(rst_n),
        .count(count)
    );

    initial begin
        clk = 1'b0;
        forever #5 clk = ~clk;
    end

    initial begin
        rst_n = 1'b0;
        #12;
        rst_n = 1'b1;
        #60;
        $finish;
    end

    initial begin
        $monitor("time=%0t rst_n=%b count=%d", $time, rst_n, count);
    end
endmodule
```

- [ ] **Step 2: Run test to verify it fails**

Run: `iverilog -o sim.out .\tb_counter.v`
Expected: FAIL because module `counter` is undefined

### Task 2: Add the minimal RTL

**Files:**
- Create: `counter.v`
- Test: `tb_counter.v`

- [ ] **Step 1: Write minimal implementation**

```verilog
module counter (
    input wire clk,
    input wire rst_n,
    output reg [3:0] count
);
    always @(posedge clk or negedge rst_n) begin
        if (!rst_n)
            count <= 4'b0000;
        else
            count <= count + 4'b0001;
    end
endmodule
```

- [ ] **Step 2: Run simulation to verify it passes**

Run: `iverilog -o sim.out .\counter.v .\tb_counter.v`
Expected: compile succeeds

Run: `vvp .\sim.out`
Expected: output shows `count` incrementing after reset is released
