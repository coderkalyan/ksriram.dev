+++
title = "Formal, part 2: combinational and sequential model checks"
date = "2025-03-06T01:20:49-06:00"
author = ""
authorTwitter = "" #do not include @
cover = ""
coverCaption = ""
tags = ["", ""]
keywords = ["", ""]
description = ""
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
+++

This post is the second in a series introducing formal verification of hardware circuits in Verilog. Take a look at [part 1](https://ksriram.dev/posts/formal-1/) for an introduction to what formal is and how to set up a verification flow.

In this article, we will start by verifying a slightly more complex combinational design - an integer ALU - and then introduce a clock to look at how to verify synchronous circuits.

# The ALU Design
Let's create a simple ALU that performs addition, subtraction, bitwise and, and bitwise xor on two 8-bit inputs and produces an 8 bit output (2's complement). To make the simulation more interesting, let's write it out logically as two chained 4-bit carry lookahead adders (in real life, this is rarely a good idea as `+` is simple enough for your synthesis tool to do a good job). I'm doing this only to motivate the fact that your *DUT* is usually implementing a complex circuit in an efficient way while your *formal properties* are verifying simple and straight forward properties about your design as succinctly as possible to avoid errors (synthesis isn't an issue here).

```verilog
module cla_4bit (
    input  wire [3:0] i_a,
    input  wire [3:0] i_b,
    input  wire       i_cin,
    output wire [3:0] o_sum,
    output wire       o_cout
);
    wire [3:0] cg = i_a & i_b;
    wire [3:0] cp = i_a ^ i_b;

    wire [4:0] carry;
    assign carry[0] = i_cin;
    assign carry[1] = cg[0] | (carry[0] & cp[0]);
    assign carry[2] = cg[1] | (cg[0] & cp[1]) | (carry[0] & cp[0] & cp[1]);
    assign carry[3] = cg[2] | (cg[1] & cp[2]) | (cg[0] & cp[1] & cp[2]) | (carry[0] & cp[0] & cp[1] & cp[2]);
    assign carry[4] = cg[3] | (cg[2] & cp[3]) | (cg[1] & cp[2] & cp[3]) | (cg[0] & cp[1] & cp[2] & cp[3]) | (carry[0] & cp[0] & cp[1] & cp[2] & cp[3]);

    wire [3:0] sum = i_a ^ i_b ^ carry;

    assign o_sum  = sum;
    assign o_cout = carry[4];
endmodule

module alu (
    input  wire [7:0] i_a,
    input  wire [7:0] i_b,
    input  wire [1:0] i_opsel,
    output wire [7:0] o_result
);
    // conditionally invert second operand and carry in to subtract
    wire       op_sub = i_opsel == 2'h1;
    wire [7:0] op_b   = {8{op_sub}} ^ i_b;

    wire [7:0] add_result;
    wire       carry;
    cla_4bit cla0 (.i_a(i_a[3:0]), .i_b(op_b[3:0]), .i_cin(op_sub), .o_sum(add_result[3:0]), .o_cout(carry));
    cla_4bit cla1 (.i_a(i_a[7:4]), .i_b(op_b[7:4]), .i_cin(carry), .o_sum(add_result[7:4]), .o_cout());

    wire [7:0] and_result = i_a & i_b;
    wire [7:0] xor_result = i_a ^ i_b;
    wire [7:0] result = i_opsel[1] ? (i_opsel[0] ? xor_result : and_result) : add_result;
    assign o_result = result;
endmodule
```

Formally verifying this is just like our full adder example. When verifying combinational designs, our `asserts` go in `always @(*)` blocks. It is perfectly legal to use your typical verilog constructs, such as `if`, `else`, and `case` as we'll see below. I'll also explain how conditionals get translated into SMT algebra. It's also a good idea to define temporary nets with `wire` and `reg` to avoid unwieldy asserts.

```verilog
module alu (
    input  wire [7:0] i_a,
    input  wire [7:0] i_b,
    input  wire [1:0] i_opsel,
    output wire [7:0] o_result
);
    // our real design here

`ifdef FORMAL
    wire [7:0] f_add = i_a + i_b;
    wire [7:0] f_sub = i_a - i_b;
    wire [7:0] f_and = i_a & i_b;
    wire [7:0] f_xor = i_a ^ i_b;

    always @(*) begin
        case (i_opsel)
            2'h0: assert (o_result == f_add);
            2'h1: assert (o_result == f_sub);
            2'h2: assert (o_result == f_and);
            2'h3: assert (o_result == f_xor);
        endcase
    end
`endif
endmodule
```

In this case, we can think about the case statement being expanded into a set of if statements:
```verilog
if (i_opsel == 2'h0)
    assert (o_result == f_add);
if (i_opsel == 2'h1)
    assert (o_result == f_sub);
if (i_opsel == 2'h2)
    assert (o_result == f_and);
if (i_opsel == 2'h3)
    assert (o_result == f_xor);
```
But wait, how can we express conditionals (control flow) without an event driven simulator? The key to this lies in the least used, and most underappreciated, logical operator in discrete math: the **implication**. The implication operator (denoted `→` and read *implies*) has the following truth table:

|a|b|a → b|
|-|-|-----|
|T|T|T    |
|T|F|F    |
|F|T|T    |
|F|F|T    |

The implication operator is only false when `a` is true but `b` is false. If `a` is false, no constraint is placed on `b`, and it always returns true. This is convenient, because we can transform code of the form:
```verilog
always @(*) begin
    if (i_opsel == 2'h0)
        assert (o_result == f_add);
end
```

into (this is pseudocode, SystemVerilog does have an implication operator but its used differently):

```verilog
always @(*) begin
    assert ((i_opsel == 2'h0) -> (o_result == f_add));
end
```

This allows us to effectively disable the assert if the condition is not met, since the implication will trivially return true!

> Try to think about how to extend this principle to implement if/else, chained if statements, and nested ifs. Note that chained if/else if statements have an implied order, unlike case statements.

# Adding a Clock

Now that we have our ALU working, let's modify the design into a synchronous one. To start simple, lets say that our design should always return the result of the operation presented on the previous clock cycle at each rising edge. We will ignore resets for now and come back to them later.

```verilog
module alu (
    input  wire       i_clk,
    input  wire [7:0] i_a,
    input  wire [7:0] i_b,
    input  wire [1:0] i_opsel,
    output wire [7:0] o_result
);
    // conditionally invert second operand and carry in to subtract
    wire       op_sub = i_opsel == 2'h1;
    wire [7:0] op_b   = {8{op_sub}} ^ i_b;

    wire [7:0] add_result;
    wire       carry;
    cla_4bit cla0 (.i_a(i_a[3:0]), .i_b(op_b[3:0]), .i_cin(op_sub), .o_sum(add_result[3:0]), .o_cout(carry));
    cla_4bit cla1 (.i_a(i_a[7:4]), .i_b(op_b[7:4]), .i_cin(carry), .o_sum(add_result[7:4]), .o_cout());

    wire [7:0] and_result = i_a & i_b;
    wire [7:0] xor_result = i_a ^ i_b;
    wire [7:0] next_result = i_opsel[1] ? (i_opsel[0] ? xor_result : and_result) : add_result;

    reg [7:0] result;
    always @(posedge i_clk)
        result <= next_result;

    assign o_result = result;

`ifdef FORMAL
    wire [7:0] f_add = i_a + i_b;
    wire [7:0] f_sub = i_a - i_b;
    wire [7:0] f_and = i_a & i_b;
    wire [7:0] f_xor = i_a ^ i_b;

    always @(*) begin
        case (i_opsel)
            2'h0: assert (o_result == f_add);
            2'h1: assert (o_result == f_sub);
            2'h2: assert (o_result == f_and);
            2'h3: assert (o_result == f_xor);
        endcase
    end
`endif
endmodule
```

If you run the verification now, you will see that it now fails. This is not surprising, as the `o_result` output port now only gets updated on each rising clock edge. So all the prover has to do is keep the clock low, change the input values, and complain that the output value doesn't match the expectation.

To solve this, we will update our formal property to be aware of the synchronous behavior. Recall that the `@(posedge i_clk)` register assignment we have means "on the rising clock edge, take the value of `next_result` before the clock edge and write it to the register `result`". We need to make two changes to our formal property:
1. Instead of a continuous block, we should only assert the result on the clock edge.
2. Inside a `@(posedge i_clk)` block, any wires we refer to in right-hand side expressions will refer to the values *after* the clock edge. We want to refer to the values *before* the clock edge.

The first point is trivial, as we can just change our `always @(*)` assert block into `always @(posedge i_clk)`. The second requires us to use new syntax: `$past`. In a formal context, `$past(next_result)` refers to the value that `next_result` had on the *previous clock cycle*. More generally, `$past(next_result, n)` refers to the value of `next_result` `n` cycles in the past, with `n` defaulting to 1. For now, we will only use one cycle past, but will come back to multiple cycles in the next article. Let's make the modifications:

```verilog
`ifdef FORMAL
    wire [7:0] f_add = i_a + i_b;
    wire [7:0] f_sub = i_a - i_b;
    wire [7:0] f_and = i_a & i_b;
    wire [7:0] f_xor = i_a ^ i_b;

    always @(posedge i_clk) begin
        case ($past(i_opsel))
            2'h0: assert (o_result == $past(f_add));
            2'h1: assert (o_result == $past(f_sub));
            2'h2: assert (o_result == $past(f_and));
            2'h3: assert (o_result == $past(f_xor));
        endcase
    end
`endif
```

Notice that we are 
// TODO: make sure the result is fully synchronous with assert
