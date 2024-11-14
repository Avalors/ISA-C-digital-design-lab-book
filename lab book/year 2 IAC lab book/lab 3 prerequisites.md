
## Finite state machines
### Mealy synchronous finite state machines
- **Finite state machines** allow us to translate a **digital control specification** to an actual digital circuit.
- For a **synchronous finite state machine** if the current output is dependent on both the current input and the current state it is considered a **mealy** state machine.
- A **Finite state machine** consists of both *combinational* and *sequential* logic circuits.
- For a finite state machine with an n-bit logic the finite state machine can have a maximum of 2<sup>n</sup> states represented using binary state encoding.
- An n-bit register  consists of n number of D-type flip flops.
- To design a reliable FSM:
	- Dont mess with the clock signal by adding combinational logic between it and the register
	- Avoid using asynchronous logic such as S-R latches
	- Initialize FSM to a known intial state on reset or power on.
![[WhatsApp Image 2024-11-04 at 18.51.25_a02e7338.jpg]]

### Moore FSM
- The **Moore** finite state machine is special type of Mealy finite state machine where the output is dependent only on the current state of the FSM.


- For Mealy finite state machines, since the output is dependent on both the input and the current state of the system, the output can change in the middle of a cycle if the input is not synchronized to the clock.

## Analysing Finite state machines

### Using state table
- Each state has its own row (2<sup>n</sup> rows depending on the register bit size)
- Each input combination has its own column
- Each cell contains the next state value of the finite state machine and the output the system based on the current state.
### Defining finite state machine in SV
#### Example 1: Divide by 3 FSM
- This FSM creates a high signal every 3 rising clock edges
```SystemVerilog
module div3FSM(
	input logic clk,
	input logic rst,
	output logic out
);
	//declaration of states
	typedef enum {IDLE, S1, S2} my_state;
	my_state current_state, next_state;

	//state registers
	always_ff @(posedge clk, posedge rst)
		if(rst) current_state <= IDLE;
		else current_state <= next_state;

	//next state logic
	always_comb
		case(current_state)
			IDLE: next_state = S1;
			S1:   next_state = S2;
			S2:   next_state = IDLE;
			default: next_state = IDLE;
		endcase
	//output logic
	assign out = (current_state == IDLE); //goes high when condition is satisfied
endmodule
```
<ins>state diagram:</ins>
![[WhatsApp Image 2024-11-11 at 10.59.37_6472d717.jpg]]
##### Divide by 3 FSM not using states:
- Uses the logic from the clktik module
```SystemVerilog
module Div3(
	input logic clk,
	input logic en,
	input logic rst,
	output logic tick
);

	logic[1:0] count;

	always_ff @(posedge clk)
		if(rst)begin
			tick <= 1'b0;
			count <= 2'b10;
			end
		else if(en) begin
			if(count == 0) begin
				tick <= 1'b1;
				count <= 2'b10;
			end
			else begin
				tick <= 1'b0;
				count <= count - 1;
			end
		end
endmodule

```
#### Example 2: Noise pulse eliminator
- this FSM eliminates any single clock pulse values , each state represents the unique current input histories:
	- state A represents a consistent input of 0 outputting 0
	- state B represents a single cycle input of 1 outputting 0
	- state C represents a consistent input of 1 outputting 1
	- state D represents a single cycle input of 0 outputting 1

```SystemVerilog
module eliminator{
	input logic clk,
	input logic rst,
	input logic in,
	output logic out
}

	//define states
	typedef enum {S_A, S_B, S_C, S_D} my_state;
	my_state current_state, next_state;

	//establish state transition
	always_ff @(posedge clk)
		if(rst) current_state <= S_A;
		else    current_state <= next_state;


	//next_state logic
	always_comb
		case(current_state)
			S_A: if(in == 1'b1)  next_state = S_B;
				 else            next_state = current_state;
			S_B: if(in == 1'b1)  next_state = S_C;
				 else            next_state = S_A;
			S_C: if(in == 1'b0)  next_state = S_D;
				 else            next_state = current_state;
			S_D: if(in == 1'b0)  next_state = S_A;
			     else            next_state = S_C;
			default: next_state = S_A; //required default case
		endcase 
endmodule
```

<ins>state diagram:</ins>
![[WhatsApp Image 2024-11-11 at 10.57.59_21aaa300.jpg]]
#### Example 3: pulse generator
- This FSM generates a singe clock cycle long high signal  on the rising edge of the input signal.
- This system has 3 states:
	- IDLE: waiting for rising edge of input signal
	- IN_HIGH: high output when rising edge achieved
	- WAIT_LOW: waits for input signal to go low again before returning to IDLE state ,otherwise the perpetual high state would generate a high clock signal greater than a single cycle.
```SystemVerilog
module pulse_gen {
	input logic clk,
	input logic rst,
	input logic in,
	output logic pulse
};

	//define our states
	typedef enum{IDLE, IN_HIGH, WAIT_LOW} my_state;
	my_state current_state, next_state;

	//state transition
	always_ff @(posedge clk)
		if(rst) current_state <= IDLE;
		else    current_state <= next_state;

	//next state logic
	always_comb
		case(current_state)
			IDLE: if(in == 1'b1) next_state = IN_HIGH;
				  else:          next_state = current_state;
			IN_HIGH: if(in == 1'b0) next_state = IDLE;
			         else:          next_state = WAIT_LOW;
			WAIT_LOW: if(in == 1'b0) next_state = IDLE;
			          else:          next_state = WAIT_LOW;
			default: next_state = IDLE;
		endcase
	//output logic (since moore FSM only depends on current state value)

	always_comb
		case(current_state)
			IDLE: pulse = 1'b0;
			IN_HIGH: pulse = 1'b1;
			WAIT_LOW: pulse = 1'b0;
			default: pulse = 1'b0;
		endcase
endmodule

```
![[WhatsApp Image 2024-11-05 at 17.04.32_ffd2a7f6.jpg]]
#### Example 4: the delay module
- The delay module is a variation of the pulse generator that would have the counter in-built into the module, that acts as a delay mechanism for the output pulse by k clock cycles. 
- To implement this, the state diagram changes: 
	- by Adding the additional state of the counting clock that persists on mainting an output of 0 until the count reaches 0.
![[WhatsApp Image 2024-11-05 at 17.04.59_e84bc88e.jpg]]
- Implenting in systemVerilog looks like this: 

```SystemVerilog
module delay #(
	parameter WIDTH = 7
)(
	input logic clk,
	input logic rst,
	input logic trigger,
	input logic [WIDTH-1:0] k,
	input logic time_out
);

	logic [WIDTH-1:0] count = {WIDTH{1'b0}};

	//define states
	typedef enum {IDLE, COUNTING, TIME_OUT, WAIT_LOW} my_state;
	my_state current_state, next_state;

	//next state logic
	always_comb
		case(current_state)
			IDLE:  if(trigger == 1'b1)  next_state = COUNTING;
				   else                 next_state = current_state;
			COUNTING: if(count == {WIDTH{1'b0}} next_state = TIME_OUT;
					 else   next_state = current_state;
			TIME_OUT: if (trigger == 1'b1) next_state = WAIT_lOW;
					  else:                next_state = IDLE;
			WAIT_LOW: if (trigger == 1'b0) next_state = IDLE;
					  else                 next_state = current_state;
			default: next_state = IDLE;
		endcase
	
	//output logic input has no bearing therefore implying its a moore FSM
	always_comb
		case (current_state)
			IDLE: time_out = 1'b0;
			COUNTING: time_out = 1'b0;
			TIME_OUT: time_out = 1'b1;
			WAIT_LOW: time_out = 1'b0;
			default: time_out t= 1'b0
		endcase

	//counter
	always_ff @(posedge clk)
		if(rst | current_state == IDLE) count <= k = 1'b1;
		else if(current_state == COUNTING) count <= count - 1'b1;

	//applies state transisition
	always_ff @(posedge clk)
		if (rst)   current_state <= IDLE;
		else       current_state <= next_state;
	

endmodule
```

- each block runs in parallel with each other, therefore there is no need to order them in a certain way sequentially like a programming language like C.
- 