### <ins>0.0 setup and development: </ins>

#### WSL:

- **WSL** stands for windows subsystem for Linux and is a compatibility layer in-built into the windows OS that allows the user to run a Linux environment natively on a windows operating system, without the need for a virtual machine.
- We must use **WSL** since much of the software, particularly, **GTK wave** which we need to use for our wave simulations, only can run via a Linux OS.
- Additionally all our packages and dependencies that allow us to design and write system-Verilog code will be stored in our ubuntu-22.04 Linux subsystem, which therefore means to be able to access these dependencies during our development, we must open our IDE (integrated development environment) in our Ubuntu subsystem.
- This IDE being **visual studio code.

#### labs repositories:
- To do these labs, I will first fork the relevant labs repo into a separate remote repo under my GitHub account.
	- This will allow to add files to repository containing my lab work
- Then clone the repository locally into my ubuntu Linux subsystem
- Open the lab folder in VS code and complete the labs.
- Sync local repo and remote repo by 'committing' changes to GitHub
- These changes will be seen in my forked repo, which our professor Peter Chung will have access to, to monitor our progress.

###### <ins> navigating and opening files in WSL </ins>
```powershell
ahmed@DESKTOP-5A7F7CK:~$ ls #checks files in current directory
Documents
ahmed@DESKTOP-5A7F7CK:~$ cd Documents/ #moves into different directory
ahmed@DESKTOP-5A7F7CK:~/Documents$ ls
iac
ahmed@DESKTOP-5A7F7CK:~/Documents$ cd iac
ahmed@DESKTOP-5A7F7CK:~/Documents/iac$ ls
lab0-devtools
```
###### <ins>cloning and opening lab repo</ins>:
```powershell
ahmed@DESKTOP-5A7F7CK:~/Documents/iac$ git clone https://github.com/Avalors/Lab1-Counter.git
Cloning into 'Lab1-Counter'...
remote: Enumerating objects: 91, done.
remote: Counting objects: 100% (45/45), done.
remote: Compressing objects: 100% (24/24), done.
remote: Total 91 (delta 29), reused 25 (delta 21), pack-reused 46 (from 1)
Receiving objects: 100% (91/91), 4.38 MiB | 7.22 MiB/s, done.
Resolving deltas: 100% (35/35), done.
ahmed@DESKTOP-5A7F7CK:~/Documents/iac$ ls
Lab1-Counter  lab0-devtools
ahmed@DESKTOP-5A7F7CK:~/Documents/iac$ cd Lab1-Counter/
ahmed@DESKTOP-5A7F7CK:~/Documents/iac/Lab1-Counter$ code . #opens file in vscode
```

###### <ins>committing changes after completing lab </ins>
```bash
#within the correct file
git status #checks if any changes have been made
git add . #moves all files into staging area
git status #checks if files are in staging area
git commit -am "msg" #commits all files within the staging area to the local repo
#move file to remote repoo
git push -u origin master #pushes all files in local repo to remote repo
```

### Verilator
- Verilator is a cycle accurate simulator that synthesizes SystemVerilog HDL into compila ble cpp code that can be run.
- Verilator is not an event accurate simulator therefore it cannot handle time delay, however the signal values are correct cycle to cycle, however the length of the cycle cannot be determined due to being cycle accurate.
- It only understands two logic levels 0 and 1, therefore any tristate busses could be difficult to simulate.

### Understanding the work flow:-
- After writing system verilog hardware description ({name}.sv) a testbench of the the file has to be created in Cpp, which describes how the input signals would change such that we can simulate the hardware to see if it works.
- Verilator takes the test bench file ({name}_ tb.cpp) and the vbuddy.cpp file and creates a cpp file with a make file that describes how the compiler can combine these files in such a way that our simulation is what we want and the file synthesizes the hardware properly.
- This executable file created by GNU tool and cpp compiler can then be simulated on the v buddy and also compiled (ran) with its simulated cycle by cycle values visually represented on GTK wave ({name}.vcd) This is the trace file.

---
---
## <ins>task 1: simulating a synchronous 8 bit binary counter</ins>
- **IMPORTANT NOTE:** file name and module name must be identical e.g. counter.sv corresponds to the module counter

```SystemVerilog
module counter #(
	parameter WIDTH = 8
)(
	// interface signals
	input logic clk, //system clock (changes made based on rising edge of this signal)
	input logic rst, //reset signal
	input logic en, //enable signal
	output logic [WIDTH-1:0] count //count signal (counter value)
);

always_ff @(posedge clk)
	if(rst) count <= {WIDTH{1'b0}} //non-blocking assignment
	else count <= count + {{WIDTH-1{1'b0}}, en}

endmodule
```
- the '#' that precedes the parameter list, allows us to change the parameters as we wish when instantiating this module when using structured level hardware descriptions.

- The parameter *WIDTH* allows us to change the size of the counter by changing the bitwidth of the count signal.
	- This means that by increasing *WIDTH* we also increase the possible largest count value
	- When instantiating this *counter* module, we can alter the counter by setting the parameter value.

- **Interface signals** is the name given to the signals of a module that we observe when simulating and set in our testbench.
	- If you imagine a System Verilog module as a black box, the interface signals are the respective inputs and outputs that 'interface' with external modules.
- The *clk* signal is the signal for which the hardware is synchronized too. 
	- Only when it encounters a rising edge 0->1 does the statements within the *always_ff* block execute.
		- This is indicated by the fact that the **sensativity-list** of the *always_ff* blocks computes only contains the rising edge of the clk signal and nothing else.
		- This means that even is the reset signal goes high in the middle of a clock pulse it only will be registered and reset the signal back to 0 when the clk edge is high once more.
	- *rst* signal resets the count back to d0, when there is a rising clockedge 
	- *en* determines if the counter is permitted to count up, or remain stationary at the rising clock pulse.

- **Non-blocking assignments** '<=' must be used in synchronous blocks in the *always_ff* block as multiple assigned signals are updated simultaneously, which emulates the behaviour of multiple synchronous registers.

### <ins>Corresponding digital hardware</ins>
![[Pasted image 20241022121558.png]]

### <ins>Creating Cpp testbench</ins>
- To be able to synthesize and simulate our Counter we must create a Cpp test bench that establishes our important interface values that determines how our test will be.

```cpp
//mandatory header files
#include "Vbuddy.cpp" //ensures the compatibility file is linked
#include "Vcounter.h" 
#inlcude "verilated.h"
#include "verilated_vcd_c.h"

int main(int argc, char **argv, char **env){//parameters allow us to configure simulation behaviour at run-time when running test bench via command line
	int i;
	int clk;

	Verilated::commandArgs(argc, argv);
	//--init top verilog instance--
	Vcounter* top = new Vcounter; //instantiates Device under test
	
	//--init trace dump--
	Verilated::traceEverOn(true); //enables tracing of all signal changes to VCD filel
	VerilatedVcdC* tfp = new VerilatedVcdC; //pointer to VCD file handler
	top->trace (tfp, 99); //measures traces of signals 99 levels deep into the hierarchy from the top
	top->open ("counter.vcd"); //opens VCD file for writing

	//--initialize simulation inputs (inteface signals)--
	top->clk = 1;
	top->rst = 1;
	top->en = 0;

	//run simulation for many clock  cycles

	for(i = 0; i < 300; i++){ //300 clock cycles

		//dump variables to VCD trace file and toggle clock
		for(clk = 0; clk<2; clk++){
			tfp->dump (2*i+clk); //dumps interface signal values at both rising and falling edges to VCD file
			top->clk = !top->clk;
			top->eval (); //evaluates interface signal values
		}
		top->rst = (i<2) | (i==15); //keeps active i < 2 and i == 15
		top->en = (i>4); //signal becomes active after 5th cycle
		if(Verilated::gotFinish()) exit(0); //checks simulatoin has finished
	}
	tfp->close(); //closes VCD trace file
	exit(0);
}
```


- **Header files** give us access to external files and utilities that allow us to set up the test bench in the cpp environment.
```cpp
#include "Vbuddy.cpp"
#include "Vcounter.h"
#include "verilated.h"
#inlude "verilated_vcd_c.h"
```
- Vbuddy.cpp allows us to be able to make our executable file created from our test bench and system Verilog description using Verilator compatible to be run physically on the Vbuddy hardware.
- *Vcounter.h* is a file generated by Verilator automatically from our SystemVerilog file.
	- This file outlines a class definition of our module that allows us to make changes and access interface signals as cpp attributes within the test bench.
- *verilated.h* is a compulsory header required to be able to use verilators run-time library to allow verilator to initialize the simulator using these testbench.
- *verilated_vcd_c.h* This header enables VCD (value change dump) trace file generation by storing signal changes during simulation from which a VCD file generated can be viewed in GTK wave to see wave simulation of our hardware from the test-bench stimuli.


- For the test values for our interface signals:
```cpp
top->rst = (i < 2) | (i == 15)
top->en = (i > 4)
```
- For the cycles in which these conditions are satisfied the signals are set to high 


### <ins>Compiling systemVerilog with test bench: </ins>
- Within the directory containing your SV and test bench file on your linux command line run:
```powershell
verilator -Wall --cc --trace counter.sv --exe counter_tb.cpp
```
- This command runs Verilator, breaking it down:
	- -Wall: report all warnings
	- --cc: generate C++ instead of systemC codes
	- --trace: produce trace for the signals (VCD files)
	- counter.sv --exe counter_tb.cpp: produce an executable model of counter.sv using counter_tb.cpp as a test bench
- The C++ file generated by Verilator and the .mk file is stored a new file called obj_dir/
- the .mk file contains instructions on how to build the simulation.

- The executable model is then made by running: 
```powershell
make -j -C obj_dir/ -f Vcounter.mk Vcounter
```
 - The *make* reads the composition instruction of the make file (that describes the relationship between the test bench and other files) and the C++ file generated model by Verilator to make a compiled file that can simulate our DUT.
 - *-j* enables parallel compilation, to compile as many files as possible in parallel to speed up the compilation process
 - *-C obj_dir/* means make changes to this directory (in this case the executable file will be saved here, which constitutes the changes being referenced).
 - *-f Vcounter.mk* specifies that Vcounter.mk is the makefile that should be used to compile the C++ model created by Verilator
 - *Vcounter* is the name of the executable file, which will be stored inside the obj_dir/ folder.

- run the compiled model:
```powershell
obj_dir/Vcounter
```
- This will generate the Vcounter.vcd trace file from which we can view the simulation in GTK wave.

### <ins>Observing simulation using GTK wave:</ins>
- simply open the .vcd file in GTK wave and click on teh signals we wish to observe
![[Pasted image 20241022143448.png]]
- analysing wave simulation of counter.sv synthesis
![[WhatsApp Image 2024-10-22 at 15.02.30_34dd88dc.jpg]]

### <ins>Creating shell script</ins>
- creating a shell script allows you to complete the entire process of creating your executable model of your digital design through the execution of a single script.
```shell
#!/bin/sh #establishes which unix shell should be used to interpret the script
#sh is a lightweight version of bash

#cleanup
rm -rf obj_dir #removes previous obj_dir folder
rm -f counter.vcd #removes previous vcd file

#run verilator to turn Verilog into C++ including testbench
verilator -Wall --cc --trace counter.sv --exe counter_tb.cpp

#build C++ executable file from verilator cpp and make file
make -j -C obj_dir/ -f Vcounter.mk Vcounter

#run executable simulation file
obj_dir/Vcounter
```

- By saving this shell script in the current directory, it can be easily ran by doing:
```shell
source ./doit.sh
```
### <ins>CHALLENGE</ins>
#### <ins>(1) make count reach 9 and then pause for 3 cycles, then resume:</ins>
- change signal values for en and rst:
```cpp
top->rst = (i < 2);
top->en = (i > 4) & ((i < 14) | (i > 16));
```
- *&* represents the intersection set operator
- *|* represents the union set operator

##### GTK wave simulation:
![[Pasted image 20241022202734.png]]
- This is the expected behaviour as the counter stops counting once it reaches 9 and remains at 9 for 3 clock cycles, before rising again.
#### <ins>(2) Applying Asynchronous reset counter</ins>

- To implement the Asynchronous counter, I added the positive edge of the rst signal to the sensitivity list for the sequential always block:
```SystemVerilog
module counter #(
	parameter WIDTH = 8
	)(
	input logic clk,
	input logic rst,
	input logic en,
	output logic [WIDTH-1:0] count
	);

	always_ff @(posedge clk, posedge rst)
		if(rst) count <= {WIDTH{1'b0}}
		else count <= count + {{WIDTH-1{1'b0}},en}
endmodule
```
- I also altered the test bench for counter by changing the signals for the reset and enable signals to show this change in functionality compared to the synchronous counter
```cpp
top->rst = (i < 2) | (i == 14);
top->en = (i > 4)
```

##### GTK wave simulation
![[Pasted image 20241022212916.png]]
- Since Verilator is a cycle accurate simulator, the values of the signals are measured twice within each cycle at both the rising and falling edge of a clock cycle (as these values are dumpeed to the VCD (value change dump)  file)
- Therefore, unlike for the synchrounous reset timer, the reset is applied to the count at the falling edge of the clock cycle for which the reset goes active, instead of waiting on the following rising edge of the next clock cycle.
- Additionally The cycle doesn't start rising again until the next rising edge of the clock when the reset is not active.
## <ins>Task 2: Using the Vbuddy</ins>
-  run this in the ubuntu terminal to connect the WSL to the port in which the V buddy is connected using usbipd
```shell
~/Documents/iac/lab0-devtools/tools/attach_usb.sh
```
- This is then used to find the name of the device
```shell
ls /dev/ttyU*
```
The device name is outputted:
```shell
/dev/ttyUSB0
```
This is then saved in vbuddy.cfg file

- Carrige return endings have to be added to the end of the .cfg file so that the Vbuddy can be used: 
```shell
#replaces file with CR endings
q
#tests to see if CR endings used if line ends with 0d
xxd vbuddy.cfg
```

- changes made to test bench to enable vbuddy functionality:![[Pasted image 20241022223824.png]]

- *vbdHex({display number}, {4 bit number to display})* in the test bench it takes the count value right shifts it such that each nibble is represented in the least 4 bits of the binary number, then uses bitwise AND with 0xF, which preserves only the least 4 significant bits of the remaining number.
	- This number is then displayed in Hexidecimal at the {display number} virtual 7 segment display
- The interface signal of en is controlled by *vbdFlag()* which flips from on to off depending on if the rotary button is pressed.

#### <ins>Running program on Vbuddy</ins>
-![[WhatsApp Video 2024-10-23 at 08.56.34_4a50eabb.mp4]]
- replacing the vbdHex block with: 
```cpp
vbdPlot(int(top->count),0,255);
```
- This will plot the count on the display as a string of connected dots for each cycle, by changing EN by toggling the rotary switch we can also see it increase.
- 0 sets the minimum value that will be plotted
- 255 sets the maximum value that will be plotted, given that the width of the count is 8 bits, this is the maximum value that can possibly be plotted.
- When simulating this, I increased the number of cycles to 1000 since plotting is much quicker than displaying the values on a hex 7 seg display.
![[WhatsApp Video 2024-10-23 at 09.35.32_e9b839ca.mp4]]

### <ins>CHALLANGE</ins>: make the EN signal determine the direction if active +1 if off -1

- changed SystemVerilog module:
```SystemVerilog
module counter #(
	parameter WIDTH = 8
)(
	input logic clk,
	input logic rst,
	input logic en,
	output logic [WIDTH-1:0] count
)

	always_ff @(posedge clk)
		if(rst) count <= {WIDTH{1'b0}}; //reset condition
		else if (en) count <= count + {{WIDTH-1{1'b0}},en}; //count up condition
		else count <= count - {{WIDTH-1{1'b0}},1'b1} //count down condition
endmodule
```

- Testing on Vbuddy using vbdPlot:
![[WhatsApp Video 2024-10-23 at 10.00.11_75b41dbc.mp4]]
- Testing on Vbuddy using hex 7 segment  display
![[WhatsApp Video 2024-10-23 at 10.12.38_18fba0a5.mp4]]


## <ins>task 3: Setting interface signal parameter using rotary button on Vbuddy</ins>
- The aim of this task is to be able to preload a interface signal parameter V associated with the rotary (EC11) button. As seen on the bottom right of the TFT screen in both hex and decimal values.
- After loading, whenever the Rotary flag is pressed the count value is set to the preloaded V value parameter.
- Otherwise the count continues counting.

- we implement this in SystemVerilog by altering the counter.sv file:
```SystemVerilog
module counter #(
	parameter WIDTH = 8;
)(
	input logic clk,
	input logic rst,
	input logic ld,
	input logic[WIDTH-1:0] v,
	input logic[WIDTH-1:0] count
)

always_ff @(posedge clk)
	if(rst) count <= {WIDTH{1'b0}};
	else count <= ld ? v: count + {{WIDTH-1{1'b0},1'b1}} //if v is high count = v otherwise count increments by 1 like before
endmodule
```

- To implement this we change the mode on the *vbdFlag()* by enabling a one-shot functionality, which means that once the EC11 rotary encoder is pressed the flag rises but then immediately falls once *vbdFlag()* reads the flags high signal. This is done by using *vbdSetMode(1)* in the counter test bench.
- We also must change our initial interface signal.
```cpp
top->clk = 1;
top->rst = 1;
top->ld = 0; //the signal that sets our values
vbdSetMode(1);
```


- Video test: 
![[WhatsApp Video 2024-10-24 at 10.05.27_8e6251ab.mp4]]

### task 2: Single stepping count by pressing EC11 rotary encoder to create a single clock pulse


- To implement this we recreate the hardware from the task 2 the only difference is the test bench for the counter contains the *vbdSetMode(1)*, which creates the one-shot functionality of the flag.

- Hardware description:
```SystemVerilog
module counter #(
	parameter WIDTH = 8
)(
	input logic clk,
	input logic rst,
	input logic en, //will be controlled by oneshot singal of count
	output logic [WIDTH-1:0] count
)

always_ff @(posedge clk)
	if(rst) count <= {WIDTH{1'b1}}
	else    count <= count + {{WIDTH-1{1'b0}},en}

endmodule
```

- test bench:
```cpp
//initialize simulation inputs

    top->clk = 1;
    top->rst = 1;
    top->en = 0;
    vbdSetMode(1);

    //run simulation for many clockcycles
    for(i = 0; i<300; i++){

        //dump variables into VCD file and toggle clock
        for(clk = 0; clk < 2; clk++){
            tfp->dmp (2*i+clk);
            top->clk = !top->clk;
            top->eval ();
        }

        vbdHex(4,int(top->count >> 16) & 0xF);
        vbdHex(3,int(top->count >> 8) & 0xF);
        vbdHex(2,int(top->count >> 4) & 0xF);
        vbdHex(1,int(top->count) & 0xF);
        vbdCycle(i+1);

  

        top->rst = (i<2);
        top->en = vbdFlag();
        if(Verilated::gotFinish()) exit(0);
    }

    vbdClose();
    tfp->close();
    exit(0);

}
```

## <ins>Task 4: Displaying count as BCD (binary coded decimal)</ins>
- By creating a *top.sv* structural file, which instantiates *counter.sv*  with *bin2BCD.sv* behaviour descriptions of hardware, we can create a hardware description, where the counter counts up on the virtual 4x7 seg display of the TFT screen in denary and no longer only in hex.


- **top.sv**: structural description that is the highest in the hierarchy of modules:
```SystemVerilog
module top #(
	parameter WIDTH = 8
)(
	input logic clk,
	input logic rst,
	input logic en,
	input logic [WIDTH-1:0] v,
	output logic [11:0] bcd
)
	logic [WIDTH-1:0] count

counter myCounter ( //creates instance of counter module
	.clk (clk),
	.rst (rst),
	.en (en),
	.count (count),
);

bin2bcd myDecoder( //creates instance of bin2bcd module
	.x (count),
	.BCD (bcd)
);
```

- the interface signals of the respective sub-modules in this structural design are connected via ports, which make up the signals of the top module.
- Within each of the instance calls within the top module, the signal preceded by the .{signal} is an interface signal within the submodule#
- The ({signal}) is the port signal that is from the top module, that is used to connect corresponding signals across modules.

- To allow this to run, changes had to be made to the test bench, in which, all instances of Vcounter are replaced by Vtop, the name of the test bench is also changed to top_tb.cpp.
- The commands in the doit.sh shell file must also change since all compilation, synthesis and simulation will be performed on the top.sv file not the counter.sv file.
```shell
#!/bin/sh
#vbuddy connection established

~/Documents/iac/lab0-devtools/tools/attach_usb.sh

#cleanup

rm -rf obj_dir

rm -f counter.vcd

#verilator that generates mk file and Vcounter.cpp file

verilator -Wall --cc --trace top.sv -exe top_tb.cpp

#compiles mk file and cpp file into executable model of digital design

make -j -C obj_dir/ -f Vtop.mk Vtop

#run executable file

obj_dir/Vtop
```

### test on Vbuddy:
![[WhatsApp Video 2024-10-24 at 19.34.10_3969fce1.mp4]]

the bin2bcd. module can be implemented in both a structural and behavioural approach, the behavioural approach is much more efficient and nice.

```SystemVerilog
//behavioural apprach
module bin2bcd (
   input  logic [7:0]   x,       // value ot be converted
   output logic [11:0]  BCD     // BCD digits
);
    // Concatenation of input and output
   logic  [19:0] result;  // no of bits = no_of_bit of x + 4* no of digits
   integer i;
   
   always_comb
   begin
   
      result[19:0] = 0;
      result[7:0] = x;     // bottom 8 bits has input value

      for (i=0; i<8; i=i+1) begin
         // Check if unit digit >= 5
         if (result[11:8] >= 5)
            result[11:8] = result[11:8] + 4'd3;
         // Check if ten digit >= 5
         if (result[15:12] >= 5)
            result[15:12] = result[15:12] + 4'd3;
         // Shift everything left
         result = result << 1;
      end
      // Decode output from result
      BCD = result[19:8];
   end
endmodule
```

- The structural approach makes use of 4 bit size sub-modules that perform the checking of values and adding of 3 due to shifting of the original binary value, if the module contains a value greater than 5.
![[Pasted image 20241024193821.png]]
- Each rectangle labelled with A... represents a module that is necessary to perform the regular checks

- implemented in SystemVerilog it looks like this:

![[Pasted image 20241024193934.png]]


