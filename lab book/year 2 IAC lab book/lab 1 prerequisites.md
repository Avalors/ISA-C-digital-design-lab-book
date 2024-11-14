## Important prerequisite theory:

- A **HDL** (hardware description language) specifies the logical function of combinational and sequential logic circuits in digital design.
- **CAD** (computer-aided design) tools can then **synthesize** these descriptions written in a HDL into the corresponding optimal hardware consisting of gates.
- **synthesis** transforms the HDL code into a netlist that describes the hardware, which can then be tested.
- unlike a schematic based way of designing digital circuits, A HDL is used ubiquitously in industry and is much more efficient and easy to use.
- To design circuits using a HDL abstraction of these digital hardware blocks will be used called **RTL** (register transfer level) design.
- **RTL** design describes digital systems as combinational logic sandwiched between synchronous registers.
- By understanding how these registers change as the clock changes, the combinational logic can be discerned.
- **Simulation** allows us to choose inputs values for our digital circuits to test their functionality in a wave simulator like GTK wave, we will do this by setting up our own test bench in cpp.

#### SystemVerilog understanding:

- A SystemVerilog design is made up of basic units of hardware called **modules**.
- **Modules** provide a specific functionality just like functions in traditional programming.
- **Modules** can then be **instantiated** in other modules hierarchically as an instance of the Module, just like **instantiating** an object from a class.
	- In this analogy, the original module is the class and the object is the instantiated module.

- A module can be specified in 2 ways (of abstraction): 
	1. **behaviour level** where SV syntax allows you to describe the abstract functionality of the module from  its constituent hardware.

		```SystemVerilog
module and3(input logic a,b,c output logic y); //establishes inputs and outputs
	assign y = a & b & c;	/* hardware description in terms of gates */
endmodule

module inv(input logic a, output logic y);
	assign y =  ~a
endmodule
		```
	2. **Structural level** where SV syntax allows you instantiate smaller modules into higher level modules for functionality

```SystemVerilog
module nand3(input logic a,b,c output logic y);
	logic n1; //establishes internal signal

	and3 andgate(a,b,c,n1) //instantiating of and3 module n1 is the output of the signal
	inv inverter(n1,y)//instance of inv module
endmodule
```

> when choosing names for modules, signals or instantiated modules remember identifiers are case-sensitive, names cannot start with numbers and white space must be ignored

> <ins>behavioural syntax:</ins>
> 	-  & : AND gate two signals
> 	-  | : OR gate two signals
> 	- ^ : XOR gate two signals


###### <ins>specifying input length:</ins>
Signal length is specified by the brackets [m : n] where m represents the most significant bit (MSB) and n represents the least significant bit (LSB).
- signal bit length = m - n  + 1
- standard convention for a signal with m bits: [n+1 : 0]
```SystemVerilog
module gates(input logic [3:0] a,b, output logic [3:0] y1,y2,y3,y4,y5);
	assign y1 = a & b // performs bit wise operation on multi-bit bus input
	assign y2 = a | b
	assign y3 = a ^ b
	assign y4 = ~(a & b)
	assign y5 = ~(a | b)
endmodule
```

###### <ins>reduction operators:</ins>
By using the &, |, ^ (and their respective inverted equivalents) operations with a single operand and reduction operation is performed where the respective bits of the operand are operated on under a single bus into a single output bit. E.g.

```SystemVerilog
module and8(input logic [7:0] a,b, output logic y);
	assign y = &a;
endmodule
```

###### <ins>conditional assignment:</ins>
In the description below it performs 'if s is true y = d1 otherwise y = d0'
```SystemVerilog
module mux2(input logic [3:0] d0, d1, input logic s, output logic [3:0] y);
	assign y = s ? d1:d0; //this performs the functoinality of a multiplexer
	//here the s is the switching signal for inputs d1 and d2
endmodule
```

###### <ins>precedence of operation:</ins>
From bottom to top operations, describes the order of precedence for systemveriolog

| syntax       | meaning            |
| ------------ | ------------------ |
| ~            | NOT                |
| *, / , %     | MULT, DIV, MOD     |
| +, -         | ADD, SUB           |
| <<, >>       | SHIFT              |
| <<<, >>>     | ARITHMETIC SHIFT   |
| <, <=, >, >= | COMPARISONS        |
| == , !=      | EQUAL OR NOT EQUAL |
| &, ~&        | AND, NAND          |
| ^, ~^        | XOR, XNOR          |
| \|, ~\|      | OR, NOR            |
| ? :          | TERNARY OPERATOR   |
###### <ins>number formatting:</ins>

- N'B is not compulsory and N is the number of bits required to store the value, and B is the base.
- B = {b - binary, d - denary, h - hexadecimal}, default without N'B is denary representation.
- the value must be represented in the base elucidated by B
- without N'B clarified, the number is assumed to be a 32bit denary number
- underscores for the binary values in the values section are used for demarkation and ease of reading, verilog does not read them
> '{N'B}{value}'


###### <ins>sequential logic using idioms</ins>
- SV uses idioms (patterns) to describe sequential logic 
- e.g a d type flip flop can be described as such: 
```SystemVerilog
module flop(input logic clk, input logic [3:0] d, output logic [3:0] q);
	always_ff @(posedge clk) //special syntax for flip flop always'_ff'
		q <= d //non-blocking assignment
endmodule
//general form for synchronous idiomatic blokc

module flop(input logic clk, input logic [3:0 d, output logic [3:0] q]);
	always @(<sensativity-list>) //for whatever conditions in the sensativity list
		<statement> //the statement will be executed
endmodule

//creating asynchronous reset
module flopr(input logic clk, input logic reset, input logic [3:0] d, output logic [3:0] q);
	always_ff @(posedge clk, posedge reset)
		if (reset) q <= 4'b0;
		else q <= d;
endmodule

//creating synchronous reset

module flopr_sync(input logic clk, input logic reset, input logic [3:0] d, output logic [3:0] q);
	always_ff @(posedge clk)
		if(reset) q <= 4'b0; //nonblocking assignment q gets 0000
		else q <= d;
endmodule

//creating asyncrhonous sr d type flip flop

module flopsr_async(input logic clk, input logic reset, input logic set, input logic[3:0] d, output logic[3:0] q);
	always_ff @(posedge clk)
		if(reset & ~set) q <= 4'b0;
		else if(set & ~reset) q <= 4'b1111;
		else q <= d;
endmodule
```

###### <ins>combinational logic using idioms</ins>
using the always block for combinational logic allows you to not need to write the 'assign' keyword
```SystemVerilog
module gates(input logic[3:0] a, b,
			output logic[3:0] y);
			always_comb
				begin
				y = a & b;
				end
endmodule
```

using case statement in always_comb statement to create 7 segment decoder, a 4 bit input signal is used to drive a 7 segment display in which each LED corresponds to a bit input from the 7 bit output signal.
the outputs are low-active, meaning that the LED turns on when the input to the LED is 0 and vise versa.
Any truth table can be specified this way, but its long
```SystemVerilog
module hex_to_7seg(input logic[3:0] in, 
				   output logic[6:0] out);
				   always_comb
					   case (in)
					   4'h0: out = 7'b1000000;
					   4'h1: out = 7'b1111001;
					   //...
					   endcase
endmodule

```

priority encoder example:

```SystemVerilog
module priorityckt(input logic[3:0] a,
				  output logic[3:0] y);
				always_comb
					if(a[3]) y = 4'b1000; //checks highest priority bit for active signal and sets output high if so
					else if(a[2]) y = 4'b0100;
					else if(a[3]) y = 4'b0010;
					else if(a[4]) y = 4'b0001;
					else y = 4'b0000;
endmodule

```

priority encoder using casez:

```SystemVerilog
module priority_casez(input logic[3:0] a,
					 output logic[3:0] y);

	always_comb
		casez(a)
			4'b1???: y = 4'b1000; //? represents a dont care in which those bits are irrelevent to making a choice
			4'b01??: y = 4'b0100;
			4'b001?: y = 4'b0010;
			4'b0001: y = 4'b0001;
			default: y = 4'b0000;
		endcase
endmodule
```


- non-blocking assignment operators <= implies assignment simultaneously, which means they should be used when  describing sequential circuits in systemVerilog
- blocking assignment operators = are assigned in sequentially in the order they are written, hence in sequential circuits they should not be used as they could create timing issues.