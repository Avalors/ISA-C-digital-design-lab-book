
### behavioural description of binary to BCD 8 bit conversion

- This behavioural description looks like a programming solution to this conversion, however when synthesized by Verilator into a model it will be represented by optimised digital hardware that performs the necessary expected function.

```SystemVerilog
module bin2BCD(
	input logic [7:0] x, //8bit binary
	output logic [11:0] BCD //12 bit BCD (due to adding operations is greater)
);

	logic [19:0] result; //20 bits includes the width of x and the corresponding BCD output bits
	integer i;

	always_comb
	begin
		result[19:0] = 0;
		result[7:0] = x; //first 8 bits of result is the orignal binary value

		for(i = 0; i <8; i=i+1) begin  //iterates till all 8 original bits are in the remaining 3 nibbles (that represent the BCD values)
		//
		if(result[11:8] >= 5)
			result[11:8] = result[11:8] + 4'd3;
		//
		if(result[15:12] >= 5)
			result[15:12] = result[15:12] + 4'd3;
		//
		result = result << 1; //left shifts by 1
	end

endmodule
		
```


### clktik module
- A *clock tick* module generates a single pulse every N+1 rising clock pulses of the system clock
- It is a key component creates a timing circuit.
- It contains an *en* (enable) signal, which when inactive will result in the system clock pulses being ignored by not triggering a clock pulse when the N+1th rising clock edge has occured.
```SystemVerilog
module clktick #(
	parameter WIDTH = 16 //sets the parameter for the word length of the N value
)(
	input logic clk,
	input logic rst,
	input logic en,
	input logic [WIDTH-1:0] N,
	output logic tick
)

	logic [WIDTH-1:0] count;//internal signal

	always_ff @(posedge clk)
		if (rst) begin
			tick <= 1'b0;
			count <= N;
			end
		else if(en) begin
			if(count == 0) begin //once 0 generates tick
				tick <= 1'b1;
				count <= N;
					end
			else begin
				tick <= 1'b0
				count <= count - 1'b1; //counts down
					end
				end
endmodule
```

### Cascading counters
- By then connecting this in series with our familiar counter module, you can construct a digital timer.
- As assuming the clktick creates a rising edge every N+1 cycles, knowing the clock frequency of the system clock:
> 1/(**SystemClock (Hz)/(N+1)**) = **Period (t)** of clock tick module
- To then take this as a clock input for our timer would then cause the counter to increment after this time elapses, hence creating a timer.

- Counter module:
```SystemVerilog
module counter #(
	parameter WIDTH = 16
)(
	input logic clk,
	input logic rst,
	input logic en,
	output logic [WIDTH-1:0] count
)

	always_ff @(posedge clk)
		if(rst) count <= {WIDTH{1'b0}};
		else count <= count + 1;

endmodule
```
- Timer module:
```SystemVerilog
module timer #(
	parameter WIDTH = 16
)(
	input logic clk,
	input logic rst,
	input logic en,
	input logic [WIDTH-1:0] N,
	output logic [WIDTH-1:0] t
)

	logic [WIDTH-1:0] tick

clktick myclktick(
	.clk (clk),
	.rst (rst),
	.en (en),
	.N (N),
	.tick (tick)
)

counter myTimer(
	.clk (clk),
	.rst (rst),
	.en (tick),
	.count (t)
)

endmodule

```


### Clock divider 
- A clock divider digital design generates a digital clock signal that divides the input clock frequency by  2*(K+1).
- The output of a clock divider circuit is a usable clock signal
```SystemVerilog
module clkdiv #(
	parameter WIDTH = 16
)(
	input logic clkin, //input clk singal to be divided
	input logic en, //enables clock divider
	input logic [WIDTH-1:0] K, //half of output clock period is k+1 cycles
	output logic clkout //output clock signal
);

logic [WIDTH-1:0] count;

initial clkout = 1'b0;
initial count = {WIDTH{1'b0}}

	always_ff @(posedge clk)
		if(en == 1'b1)
			if(count == {WIDTH{1'b0}})begin
				clkout <= ~clkout;
				count <= K;
				end
			else
				count <= count - 1;
endmodule

```


### implementing synchronous shift register

![[WhatsApp Image 2024-10-25 at 12.32.24_adc96478.jpg]]

- SV description:
```SystemVerilog
module sreg4 (
	input logic clk,
	input logic rst,
	input logic data_in,
	output logic data_out
)

	logic [4:1] reg;
	
	always_ff @(posedge clk)
		if(rst)
			sreg <= 4'b0;
		else 
			sreg <= {sreg[3:1], data_in}; //performs shift through concatenatoin since sreg[3:1] becomes the new MSB's and sreg[4] is outputted to data_out
	assign data_out = sreg[4];
endmodule
```

### Linear feedback shift regsiter (LFSR)
- A linear feedback shift register contains multiple registers in series just like reg4, however also has feedback in place to the data_in from the output of its subsequent registers.
- At this output this generates a sequence of values that repeats 2<sup>N</sup>-1 cycles where N is the number bits 
- The sequence produced is called pseudo random binary sequence (PRBS)
- The sequence can be described by a primitive polynomial since its the maximum number cycles required to return to its orignal value


- Implementing 4bit LFSR of the primitive polynomial: 1 + X<sup>3</sup> + X<sup>4</sup> 

```SystemVerilog
module lsfr4(
	input logic clk,
	input logic rst,
	output logic[4:1] data_out //pseudo random binary sequence
)

always_ff @(posedge clk)
	if(rst)
		sreg <= 4'b0001;
	else
		sreg <= {sreg[3:1], sreg[4] ^ sreg[3]}; // sreg[4] XOR sreg[3]
	
assign data_out = sreg;
endmodule
```

### 256x8 ROM module
```SystemVerilog
module rom #(
	parameter   ADDRESS_WIDTH = 8;
				DATA_WIDTH = 8;
)(
	input logic clk,
	input logic [ADDRESS_WIDTH-1:0] addr,
	output logic [DATA_WIDTH-1:0] dout
)

logic [DATA_WIDTH-1:0] rom_array [2**ADDRESS_WIDTH-1:0]; //establishes 256 addressable locations each location storing 8 bits

initial begin
		$display("Loading rom."); //whilst this is happening this is showing in the command line
		$readmemh("sinerom.mem", rom_array); //loads sinerom.mem stores values of sine wave in 16x16 matrix in rom
end;

always_ff @(posedge clk)
	//output is synchronous
	dout <= rom_array [addr];

endmodule
```

- In a structural file, for which we wish to use a ROM of a particular size, we can specify the parameters on the ROM module to get a ROM to our required specification.
```SystemVerilog
rom #(10,9) sineRom_1024x9 (.....)
```

- This python script generates the corresponding sine wave that will be downloaded by the ROM
```python
import math
import string
f = open("sinerom.mem","w")
for i in range(256):
	v = int(math.cos(2*3.1415*i/256)*127+127)
	if(i+1)%16 == 0:
		s = "{hex:2X}\n"
	else:
		s = "{hex:2X}"
	f.write(s.format(hex=v))

f.close()
```

- By using a structural description we can create a wave form generator by having a counter module access the addresses of the ROM incrementally.
```SystemVerilog
module sinegen #(
	parameter   A_WIDTH = 8,
				D_WIDTH = 8
)(
	input logic clk,
	input logic rst,
	input logic en,
	input logic [D_WIDTH-1:0] incr,
	input logic [D_WIDTH-1:0] dout
);

	logic [A_WIDTH-1:0] address;

counter addrCounter(
	.clk (clk),
	.rst (rst),
	.en (en),
	.incr (incr), //by altering the incrementing of the counter we can increase the freq
	.count (address)
);

rom SineRom (
	.clk(clk),
	.addr (address),
	.dout(dout)
);

endmodule

```

> *f<sub>out</out>* = *f<sub>clk</sub> * incr*/ 256


### dual port rom

```SystemVerilog
module rom2ports #(
	parameter   ADDRESS_WIDTH = 8,
				DATA_WIDTH = 8
)(
	input logic clk,
	input logic [ADDRESS_WIDTH-1:0] addr1, addr2,
	output logic [DATAWIDTH-1:0] dout1, dout2
)

	logic [DATA_WIDTH-1:0] rom_array [2**ADDRESS_WIDTH-1:0];

inital begin
	@display("loading rom.");
	@readmemh("sinerom.mem", rom_array);
end;

always_ff @(posedge clk) begin
	dout1 <= rom_array[addr1];
	dout2 <= rom_array[addr2];
	end

endmodule
```


### dual port RAM
```SystemVerilog
module ram2ports #(
	parameter   ADDRESS_WIDTH = 8,
				DATA_WIDTH = 8
)(
	input logic clk,
	input logic wr_en, rd_en,
	input logic [ADDRESS_WIDTH-1:0] wr_addr, rd_addr,
	input logic [DATA_WIDTH-1:0] din,
	output logic[DATA_WIDTH-1:0] dout
);

logic [DATA_WIDTH-1:0] ram_array [2**ADDRESS_WIDTH-1:0];

always_ff @(posedge clk) begin
	if (wr_en == 1'b1)
		ram_array[wr_addr] <= din;
	if (rd_en == 1'b1)
		dout = ram_array[rd_addr];
	end

endmodule
```