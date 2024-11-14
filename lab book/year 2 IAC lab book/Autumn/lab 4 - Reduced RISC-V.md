- Implemented simplified RISC-V architecture
![[Pasted image 20241114105813.png]]
- My responsibility was to implement the *program_counter.sv*
	- Width is 32 bits as RISCV is a 32 bit architecture
```SystemVerilog
module program_counter#(
    parameter WIDTH = 32
)(
    input logic            clk,
    input logic            rst,
    input logic            PCsrc,
    input logic [WIDTH-1:0]     ImmOp,
    output logic[WIDTH-1:0]     PC

);

    logic [WIDTH-1:0] inc_PC, branch_PC, next_PC;
    
    always_ff @(posedge clk)
        if(rst) PC <= {WIDTH{1'b0}};
        else PC <= next_PC;
        
    always_comb begin
        branch_PC = PC + ImmOp;
        inc_PC = inc_PC + 32'd4;
        next_PC = PCsrc ? branch_PC:inc_PC;
    end
endmodule

```

- To add my work onto the shared git hub repo, I used branches to prevent any conflicts between my peers:
![[Pasted image 20241114205918.png]]


- In terms of commands it looked like this:
```bash
git clone https://github.com/aa6dcc/RISC-V-Team2 #clones repo to my current directory (only need to do this once)
git checkout -b pc #creates temporary branch program counter
code . # opens file in vs code to start work
git status # checks if any commitable changes have been made
git add . #adds all files to staging area in branch
git commit -m "program counter module complete" #commits files in staging area to local repo in pc branch
git branch # shows available branches and current branch with astriks
git checkout main # switches branch to main
git pull origin main # ensures updated main branch from remote repo
git merge pc # merges changes from pc to main when on main branch
git push origin main # pushes main branch from local repo to remote repo
git branch -d pc #deletes temporary pc branch
git push orign --delete pc #deltes pc branch on remote repo
```