# Task0: setting up Gtest
- Given the cost that comes with manufacturing an ASIC from a HDL design, we must test our designs rigorously through a process called **verification**.
- We are using Gtest which is an industry standard for **Verification** of our HDL.
- Applying this to some random cpp code:
```Cpp
#include "gtest/gtest.h"
int add(int a, int b) { return a + b; }
class TestAdd : public ::testing::Test

{
    void SetUp() override
    {
        // Runs before each test
    }

    void TearDown() override
    {
        // Runs after each test
    }
};

// Don't worry about the syntax here, the TEST_F macro is very complicated.
// Just now that this is how you create a test case.
TEST_F(TestAdd, AddTest)
{
    // This should pass, 2 + 4 = 6
    EXPECT_EQ(add(2, 4), 6);
}
  
TEST_F(TestAdd, AddTest2)
{
    // Create a test case here. Maybe fail this to see what happens?
    EXPECT_EQ(add(2,4),7);
}

int main(int argc, char **argv)
{
    // Standard Google Test main function
    testing::InitGoogleTest(&argc, argv);
    auto res = RUN_ALL_TESTS();
    return res;
}
```

- doit.sh file

```bash
#!/bin/bash
if [ "$(uname -s)" == "Darwin" ]; then
    g++ -std=c++14 -o main main.cpp \
        -isystem /opt/homebrew/Cellar/googletest/1.15.2/include \
        -L/opt/homebrew/Cellar/googletest/1.15.2/lib \
        -lgtest -lgtest_main -pthread
else
    g++ -o main main.cpp -lgtest -lgtest_main -pthread
fi
./main
```
- After running this doit.sh shell script this is the verified output file: 
```powershell
[==========] Running 2 tests from 1 test suite.
[----------] Global test environment set-up.
[----------] 2 tests from TestAdd
[ RUN      ] TestAdd.AddTest
[       OK ] TestAdd.AddTest (0 ms)
[ RUN      ] TestAdd.AddTest2
main.cpp:29: Failure
Expected equality of these values:
  add(2,4)
    Which is: 6
  7
[  FAILED  ] TestAdd.AddTest2 (0 ms)
[----------] 2 tests from TestAdd (0 ms total)

[----------] Global test environment tear-down
[==========] 2 tests from 1 test suite ran. (0 ms total)
[  PASSED  ] 1 test.
[  FAILED  ] 1 test, listed below:
[  FAILED  ] TestAdd.AddTest2
 1 FAILED TEST
```
# Task 1: 4 bit LFSR and PRBS

- LFSR (linear feedback shift register) generates the pseudo-random binary sequence with the primitive polynominal that describes the nth term:
<center>1 + x<sup>3</sup> + x<sup>4</sup></center>
- We wish to implement:
![[Pasted image 20241111150004.png]]

- Implemented in system Verilog the Hardware description looks like this:
```SystemVerilog
module lsfr4(
	input logic clk,
	input logic rst,
	input logic en,
	output logic [4:1] data_out
);

logic [4:1] sreg;

always_ff @(posedge clk, posedge rst)
	if(rst) sreg <= 4'b1;
	else if(en) sreg <=  {sreg[3:1],sreg[4] ^ sreg[3]};

assign data_out = sreg;
```

- Attached test bench cpp script: 
	- By setting it in a class like this, the advantage is that you can run multiple simulations from which multiple verification tests can be made.
```cpp
#include "gtest/gtest.h"
#include "Vdut.h"
#include "verilated.h"
#include "verilated_vcd_c.h"
Vdut *top;
VerilatedVcdC *tfp;
unsigned int ticks = 0;
class TestDut : public ::testing::Test

{
public:
    void SetUp() override
    {
        initializeInputs(); // re-initializes the inputs
        runReset(); //resets DUT 
        // runs this prior to the test
    }

    void TearDown() override
    {
    //runs this after the test
    }
    
    void initializeInputs()
    {
        top->clk = 0;
        top->rst = 0;
        top->en = 1;
    }

    void runReset()
    {
        top->rst = 1;
        runSimulation();
        top->rst = 0;
    }

    // Runs the simulation for a clock cycle, evaluates the DUT, dumps waveform.

    void runSimulation()
    {
        for (int clk = 0; clk < 2; clk++)
        {
            top->eval();
            tfp->dump(2 * ticks + clk);
            top->clk = !top->clk;
        }
        ticks++;
        if (Verilated::gotFinish())
        {
            exit(0);
        }
    }
};

  
TEST_F(TestDut, InitialStateTest) //verifies if initial state is 0001 for 4 bit FSM
{
    top->rst = 1;
    runSimulation();
    EXPECT_EQ(top->data_out, 0b0001);
}

  
TEST_F(TestDut, SequenceTestMini)
{
    runSimulation();
    EXPECT_EQ(top->data_out, 0b0010);
}

TEST_F(TestDut, SequenceTest)
{
    std::vector<int> expected = {
        0b0001,
        0b0010,
        0b0100,
        0b1001,
        0b0011,
        0b0110,
        0b1101,
        0b1010,
        0b0101,
        0b1011,
        0b0111,
        0b1111,
        0b1110,
        0b1100,
        0b1000,
        0b0001};

    for (int exp : expected) //runs for each value in the expected vector
    {
        EXPECT_EQ(top->data_out, exp);
        runSimulation();
    }
}

  
int main(int argc, char **argv)
{
    top = new Vdut;
    tfp = new VerilatedVcdC;

  
    Verilated::traceEverOn(true);
    top->trace(tfp, 99);
    tfp->open("waveform.vcd");


    testing::InitGoogleTest(&argc, argv);
    auto res = RUN_ALL_TESTS();

  
    top->final();
    tfp->close();

  
    delete top;
    delete tfp;
    return res;
}
```
- verify.sh shell script:
```powershell
#!/bin/bash
# cleanup
rm -rf obj_dir
rm -f *.vcd

  

# Translate Verilog -> C++ including testbench
verilator   -Wall --trace \
            -cc lfsr_7.sv \
            --exe verify_7.cpp \
            --prefix "Vdut" \
            -o Vdut \
            -CFLAGS "-isystem /opt/homebrew/Cellar/googletest/1.15.2/include"\
            -LDFLAGS "-L/opt/homebrew/Cellar/googletest/1.15.2/lib -lgtest -lgtest_main -lpthread" \

# Build C++ project with automatically generated Makefile
make -j -C obj_dir/ -f Vdut.mk

# Run executable simulation file
./obj_dir/Vdut
```


- Actual test:
```powershell
make: Leaving directory '/home/ahmed/Documents/iac/Lab3-FSM/task1/obj_dir'
[==========] Running 3 tests from 1 test suite.
[----------] Global test environment set-up.
[----------] 3 tests from TestDut
[ RUN      ] TestDut.InitialStateTest
[       OK ] TestDut.InitialStateTest (0 ms)
[ RUN      ] TestDut.SequenceTestMini
[       OK ] TestDut.SequenceTestMini (0 ms)
[ RUN      ] TestDut.SequenceTest
[       OK ] TestDut.SequenceTest (0 ms)
[----------] 3 tests from TestDut (0 ms total)

[----------] Global test environment tear-down
[==========] 3 tests from 1 test suite ran. (0 ms total)
[  PASSED  ] 3 tests.
```
- This shows that during verification the DUT passed all three tests.

## challenge: create 7 bit LFSR
- The 7-bit LFSR (linear feedback shift register) is implemented in the file *LFSR_7.sv* implementing a primitive polynomial:
<center>1 + X<sup>3</sup> + X<sup>7</sup></center>
- From the primitive polynomial you can see that the number of registers we will need corresponds to the X term with the greatest exponent which is 7.
- Additionally since there is both a 3 and 7 exponent that means the outputs of both the 3rd and 7th register in sequence must be fedback into the first input through a XOR gate.
- LFSR_7.sv
```systemVerilog
module lfsr_7 (
    input   logic       clk,
    input   logic       rst,
    input   logic       en,
    output  logic [6:0] data_out
);

logic [7:1] sreg;
  
always_ff @(posedge clk, posedge rst)
    if(rst) sreg <= 7'b1;
    else if(en) sreg <= {sreg[6:1], sreg[7] ^ sreg[3]};
  
assign data_out = sreg;

endmodule
```
- Corresponding testbech for LFSR_7.sv verification
```cpp
#include "gtest/gtest.h"
#include "Vdut.h"
#include "verilated.h"
#include "verilated_vcd_c.h"
#include <cmath>

Vdut *top;
VerilatedVcdC *tfp;
unsigned int ticks = 0;

class TestDut : public ::testing::Test
{
public:
    void SetUp() override
    {
        initializeInputs();
        runReset();
    }

    void TearDown() override
    {
        top->final();
        tfp->close();
    }

    void initializeInputs()
    {
        top->clk = 0;
        top->rst = 0;
        top->en = 1;
    }
  
    void runReset()
    {
        top->rst = 1;
        runSimulation();
        top->rst = 0;
    }

    // Runs the simulation for a clock cycle, evaluates the DUT, dumps waveform.
    void runSimulation()
    {
        for (int clk = 0; clk < 2; clk++)
        {
            top->eval();
            tfp->dump(2 * ticks + clk);
            top->clk = !top->clk;
        }
        ticks++;
        if (Verilated::gotFinish())
        {
            exit(0);
        }
    }
};

  
TEST_F(TestDut, InitialStateTest)
{
    top->rst = 1;
    runSimulation();
    EXPECT_EQ(top->data_out, 0b0001);
}

  
/**
 * Here's to anyone who actually reads this code.
 * The answer to the LSFR7 lies here. You're welcome.
 * Thank you for taking your time to understand good testing.
 */
int generateLFSR7(int state)
{
    int b0 = (state >> 0) & 1;
    int b1 = (state >> 1) & 1;
    int b2 = (state >> 2) & 1;
    int b3 = (state >> 3) & 1;
    int b4 = (state >> 4) & 1;
    int b5 = (state >> 5) & 1;
    int b6 = (state >> 6) & 1;
    return (b5 << 6 | b4 << 5 | b3 << 4 | b2 << 3 | b1 << 2 | b0 << 1 |
            (b2 ^ b6));

}

TEST_F(TestDut, SequenceTest)
{
    int exp = 0b000'0001;

    for (int i = 0; i < pow(2, 7); i++)
    {
        exp = generateLFSR7(exp);
        runSimulation();
        EXPECT_EQ(top->data_out, exp);
    }
}

  

int main(int argc, char **argv)
{
    top = new Vdut;
    tfp = new VerilatedVcdC;

    Verilated::traceEverOn(true);
    
    top->trace(tfp, 99);
    tfp->open("waveform.vcd");

    testing::InitGoogleTest(&argc, argv);
    auto res = RUN_ALL_TESTS();

    top->final();
    tfp->close();
    
    delete top;
    delete tfp;
    return res;
}
```

- Verify_7.sh shell file for verification
```powershell
#!/bin/bash

# cleanup
rm -rf obj_dir
rm -f *.vcd

# Translate Verilog -> C++ including testbench
verilator   -Wall --trace \
            -cc lfsr_7.sv \
            --exe verify_7.cpp \
            --prefix "Vdut" \
            -o Vdut \
            -CFLAGS "-isystem /opt/homebrew/Cellar/googletest/1.15.2/include"\
            -LDFLAGS "-L/opt/homebrew/Cellar/googletest/1.15.2/lib -lgtest -lgtest_main -lpthread" \ 

# Build C++ project with automatically generated Makefile
make -j -C obj_dir/ -f Vdut.mk

# Run executable simulation file

./obj_dir/Vdut
```

- verifiacation outputs:
```bash
[==========] Running 2 tests from 1 test suite.
[----------] Global test environment set-up.
[----------] 2 tests from TestDut
[ RUN      ] TestDut.InitialStateTest
[       OK ] TestDut.InitialStateTest (0 ms)
[ RUN      ] TestDut.SequenceTest
[       OK ] TestDut.SequenceTest (0 ms)
[----------] 2 tests from TestDut (0 ms total)

[----------] Global test environment tear-down
[==========] 2 tests from 1 test suite ran. (0 ms total)
[  PASSED  ] 2 tests.
```
- Our Gtest verification shows that my DUT LFSR_7 outputs the exepcted sequence and results.

# Task 2:  Formula 1 light sequence
- To implement this I create the SystemVerilog hardware description from the FSM state diagram that describes the functionality of this Formula 1 light sequence.
![[Pasted image 20241112102508.png]]

- To create the functionality of this FSM in SystemVerilog we create distinct blocks: 
	- create enum data structure for all possible states
	- use sequential block to instanitate state changes on rising clock edge
	- use combinational block for next_state logic.
	- use combinational block for output logic.

- This F1 FSM is a Moore machine since its output does not depend on any input and solely depends on the current_state.

```SystemVerilog
module f1_fsm (
    input   logic       rst,
    input   logic       en,
    input   logic       clk,
    output  logic [7:0] data_out

);
    //initilize all states
    typedef enum {S0, S1, S2, S3, S4, S5, S6, S7, S8} mystate;
    mystate current_state, next_state;
    
    //intantiates state change
    always_ff @(posedge clk, posedge rst)
        if(rst) current_state <= S0;
        else if(en) current_state <= next_state;

    //sets next_state logic
    always_comb
        case(current_state)
            S0: next_state = S1;
            S1: next_state = S2;
            S2: next_state = S3;
            S3: next_state = S4;
            S4: next_state = S5;
            S5: next_state = S6;
            S6: next_state = S7;
            S7: next_state = S8;
            S8: next_state = S0;
            default next_state = S0;
        endcase

    always_comb
        case(current_state)
            S0: data_out = 8'b0;
            S1: data_out = 8'b1;
            S2: data_out = 8'b11;
            S3: data_out = 8'b111;
            S4: data_out = 8'b1111;
            S5: data_out = 8'b11111;
            S6: data_out = 8'b111111;
            S7: data_out = 8'b1111111;
            S8: data_out = 8'b11111111;
            default data_out = 8'b0;
        endcase
endmodule

```


- The test bench runs two distinct tests: 
	- To test the initial output is 8'b0
	- To ensure that all following outputs are correct

```cpp
#include "gtest/gtest.h"
#include "Vdut.h"
#include "verilated.h"
#include "verilated_vcd_c.h"

Vdut *top;
VerilatedVcdC *tfp;
unsigned int ticks = 0;

class TestDut : public ::testing::Test
{
public:
    void SetUp() override
    {
        initializeInputs();
        runReset();
    }

    void TearDown() override
    {
    }

    void initializeInputs()
    {
        top->clk = 0;
        top->rst = 0;
        top->en = 1;
    }

    void runReset(
    {
        top->rst = 1;
        runSimulation();
        top->rst = 0;
    }

    // Runs the simulation for a clock cycle, evaluates the DUT, dumps waveform.
    void runSimulation()
    {
        for (int clk = 0; clk < 2; clk++)
        {
            top->eval();
            tfp->dump(2 * ticks + clk);
            top->clk = !top->clk;
        }
        ticks++;

        if (Verilated::gotFinish())
        {
            exit(0);
        }
    }
};

TEST_F(TestDut, InitialStateTest)
{
    top->rst = 1;
    runSimulation();
    EXPECT_EQ(top->data_out, 0x00);
}

TEST_F(TestDut, FSMTest)
{
    top->rst = 1;
    runSimulation();
    EXPECT_EQ(top->data_out, 0b0000);
    
    top->rst = 0;
    
    std::vector<int> expected = {
        0b0000'0000,
        0b0000'0001,
        0b0000'0011,
        0b0000'0111,
        0b0000'1111,
        0b0001'1111,
        0b0011'1111,
        0b0111'1111,
        0b1111'1111,
        0b0000'0000};

    for (int exp : expected)
    {
        EXPECT_EQ(top->data_out, exp);
        runSimulation();
    }
}

int main(int argc, char **argv)
{
    top = new Vdut;
    tfp = new VerilatedVcdC;
    Verilated::traceEverOn(true);
    top->trace(tfp, 99);
    tfp->open("waveform.vcd");

    testing::InitGoogleTest(&argc, argv);
    auto res = RUN_ALL_TESTS();

    top->final();
    tfp->close();
  

    delete top;
    delete tfp;

    return res;

}
```

- Verify.sh shell script performs the synthesis and verification of the DUT, which in this case is our F1 lights FSM.
```powershell
#!/bin/bash
# cleanup
rm -rf obj_dir
rm -f *.vcd

# Translate Verilog -> C++ including testbench

verilator   -Wall --trace \
            -cc f1_fsm.sv \
            --exe verify.cpp \
            --prefix "Vdut" \
            -o Vdut \
            -CFLAGS "-isystem /opt/homebrew/Cellar/googletest/1.15.2/include"\
            -LDFLAGS "-L/opt/homebrew/Cellar/googletest/1.15.2/lib -lgtest -lgtest_main -lpthread" \

# Build C++ project with automatically generated Makefile
make -j -C obj_dir/ -f Vdut.mk

# Run executable simulation file
./obj_dir/Vdut
```

- output after running Gtest verification: 
	- This shows that the F1_FSM module is functional.
```powershell
[==========] Running 2 tests from 1 test suite.
[----------] Global test environment set-up.
[----------] 2 tests from TestDut
[ RUN      ] TestDut.InitialStateTest
[       OK ] TestDut.InitialStateTest (0 ms)
[ RUN      ] TestDut.FSMTest
[       OK ] TestDut.FSMTest (0 ms)
[----------] 2 tests from TestDut (0 ms total)

[----------] Global test environment tear-down
[==========] 2 tests from 1 test suite ran. (0 ms total)
[  PASSED  ] 2 tests.
```

## Step 3: connecting FSM to vbuddy
![[Pasted image 20241112121924.png]]
- To do this, I had to create a new test bench without the verification tools: *f1_fsm_tb.cpp*
	- This test bench allows sets the input from the vbdFlag() to one shot mode, which allows me to set the *en* enable single to high when its pressed.
	- This allows me to cycle through the states of the FSM one state at a time and see it appear on the **neopixel** strip.
	- The *vbdBar()* takes an unsigned 8 bit integer parameter which is why data_out is masked out using & 0xFF.
```cpp
#include "Vf1_fsm.h"
#include "verilated.h"
#include "verilated_vcd_c.h"
#include "vbuddy.cpp"
  

int main(int argc, char **argv, char **env){
    int i;
    int clk;

    Verilated::commandArgs(argc, argv);

    Vf1_fsm* top = new Vf1_fsm;

    Verilated::traceEverOn(true);
    VerilatedVcdC* tfp = new VerilatedVcdC;
    top->trace(tfp,99);
    tfp->open("counter.vcd");

  

    //init buddy
    if(vbdOpen() != 1) return(-1);
    vbdHeader("Lab3: FSM");

    //initialize simulation inputs
    top->clk = 1;
    top->rst = 1;
    top->en = 0;
    vbdSetMode(1);

    //run simulation for this many clock cycles:
    for(int i = 0; i < 1000; i++){
        for(clk = 0; clk < 2; clk++){
            tfp->dump(2*i*clk);
            top->clk = !top->clk;
            top->eval();
        }

        //displays output on neopixel strip
        vbdBar(top->data_out & 0xFF);
        vbdCycle(i+1);

        //change input stimuli
        top->rst = (i < 2);
        top->en = vbdFlag();

        if(Verilated::gotFinish() || (vbdGetkey() == 'q')) exit(0);
    }

    vbdClose();
    tfp->close();
    exit(0);

}
```

- Also created a new shell script to perform the same synthesis and simulation like before *doit.sh*.
```powershell
#!/bin/sh

#vbuddy connection established
~/Documents/iac/lab0-devtools/tools/attach_usb.sh

#cleanup
rm -rf obj_dir
rm -f f1_fsm.vcd

#verilator that generates mk file and Vcounter.cpp file
verilator -Wall --cc --trace f1_fsm.sv -exe f1_fsm_tb.cpp

#compiles mk file and cpp file into executable model of digital design
make -j -C obj_dir/ -f Vf1_fsm.mk Vf1_fsm

#run executable file
obj_dir/Vf1_fsm
```

#### <ins>test:</ins>
![[WhatsApp Video 2024-11-12 at 12.18.34_f3ef71e6.mp4]]

# Task 3: exploring the clktick.sv and delay.sv modules
- The *clktick.sv* module generates a high pulse every N+1 cycles, this allows us to create a new clock signal that can be adjusted to a particular frequency or period.
- We did this by synthesizing and simulating the clk tick module to turn on the neopixel bar when high.
- We then adjusted the N value by adjusting the *vbdValue()* by turning the EC11 rotary switch and by listening to a 60bpm metronome attempted to sync the flashing lights to the metronome, ensuring that the *clktick.sv* module has the period of 1s.

- *clktick.sv* module:
```systemVerilog
module clktick #(
    parameter WIDTH = 16
)(
  // interface signals
  input  logic             clk,      // clock
  input  logic             rst,      // reset
  input  logic             en,       // enable signal
  input  logic [WIDTH-1:0] N,        // clock divided by N+1
  output logic             tick      // tick output
);

logic [WIDTH-1:0] count;

always_ff @ (posedge clk)
    if (rst) begin
        tick <= 1'b0;
        count <= N;  
        end
    else if (en) begin
        if (count == 0) begin
            tick <= 1'b1;
            count <= N;
            end
        else begin
            tick <= 1'b0;
            count <= count - 1'b1;
            end
        end
endmodule

```

- clktick_tb.cpp:
```cpp
#include "Vclktick.h"
#include "verilated.h"
#include "verilated_vcd_c.h"
#include "../vbuddy.cpp" // include vbuddy code
#define MAX_SIM_CYC 100000

int main(int argc, char **argv, char **env)
{
    int simcyc;     // simulation clock count
    int tick;       // each clk cycle has two ticks for two edges
    int lights = 0; // state to toggle LED lights

    Verilated::commandArgs(argc, argv);
    // init top verilog instance
    Vclktick *top = new Vclktick;
    // init trace dump
    Verilated::traceEverOn(true);
    VerilatedVcdC *tfp = new VerilatedVcdC;
    top->trace(tfp, 99);
    tfp->open("clktick.vcd");
    
    // init Vbuddy
    if (vbdOpen() != 1)
        return (-1);
    vbdHeader("L3T2:Clktick");
    vbdSetMode(1); // Flag mode set to one-shot

    // initialize simulation inputs
    top->clk = 1;
    top->rst = 0;
    top->en = 0;
    top->N = vbdValue();

    // run simulation for MAX_SIM_CYC clock cycles
    for (simcyc = 0; simcyc < MAX_SIM_CYC; simcyc++)
    {

        // dump variables into VCD file and toggle clock
        for (tick = 0; tick < 2; tick++)
        {
            tfp->dump(2 * simcyc + tick);
            top->clk = !top->clk;
            top->eval();
        }
        // Display toggle neopixel
        if (top->tick)
        {
            vbdBar(lights);
            lights = lights ^ 0xFF;
        }
        // set up input signals of testbench
        top->rst = (simcyc < 2); // assert reset for 1st cycle
        top->en = (simcyc > 2);
        top->N = vbdValue();
        vbdCycle(simcyc);

        if (Verilated::gotFinish() || vbdGetkey() == 'q')
            exit(0);
    }
  
    vbdClose(); // ++++
    tfp->close();
    exit(0);
}
```

### live simulation results
- Since verilator does not perform the simulations in real-time, the time that each cycle takes is dependant on the hardware on it is run.
- So therefore to get a consistent clk frequency or period calibration of  N must be made.
- Through my experimentation N = 48 works for me, meaning that the *clktick.sv* module generates a high pulse every 49 clock cycles.

![[WhatsApp Video 2024-11-12 at 13.20.25_c6b7d952.mp4]]

## Challenge:
- Create a top level design that incorporates both the F1 FSM and clk_tick module with N value. such that the lights turn on at 1 second intervals.
![[Pasted image 20241112134651.png]]
- To do this I created a *top.sv* module that contains both *f1_fsm.sv* and *clktick.sv* as submodules 
```SystemVerilog
module top(
    input logic[15:0] n,
    input logic clk,
    input logic en,
    input logic rst,
    output logic [7:0] data_out
);


logic ien;

clktick timer(
    .clk(clk),
    .rst(rst),
    .en(en),
    .N(n),
    .tick(ien)
);

f1_fsm timed_f1_fsm(
    .clk(clk),
    .en(ien),
    .rst(rst),
    .data_out(data_out)
);

endmodule

```

- creating the correpsonding test_bench in Cpp that most notably sets my n value to 48 cycles: *top_tb.cpp*
```cpp
#include "Vtop.h"
#include "verilated.h"
#include "verilated_vcd_c.h"
#include "vbuddy.cpp"

int main(int argc, char **argv, char **env){
    int i;
    int clk;
    Verilated::commandArgs(argc, argv);


    Vtop* top = new Vtop;


    Verilated::traceEverOn(true);
    VerilatedVcdC* tfp = new VerilatedVcdC;
    top->trace(tfp,99);
    tfp->open("top.vcd");
  
    //init buddy
    if(vbdOpen() != 1) return(-1);
    vbdHeader("Lab3: FSM");

    //initialize simulation inputs
    top->n = 48; //1 second period for clktick high output
    top->clk = 1;
    top->rst = 1;
    top->en = 0;
  

    //run simulation for this many clock cycles:
    for(int i = 0; i < 1000; i++){
        for(clk = 0; clk < 2; clk++){
            tfp->dump(2*i*clk);
            top->clk = !top->clk;
            top->eval();
        }

        //displays output on neopixel strip
        vbdBar(top->data_out & 0xFF);
        vbdCycle(i+1);
  
        //change input stimuli
        top->rst = (i < 2);
        top->en = 1; //repeats the light output
  
        if(Verilated::gotFinish() || (vbdGetkey() == 'q')) exit(0);
    }

    vbdClose();
    tfp->close();
    exit(0);
}
```

### live simulation results:
![[WhatsApp Video 2024-11-12 at 13.57.17_b98476e3.mp4]]


# Task 4: 