# Tutorial: Hacking SimpleScalar

## Table of Contents

* [Introduction](#introduction)
* [Problem Statement](#problem-statement)
  * [Task 1](#task-1)
  * [Task 2](#task-2)
  * [Task 3](#task-3)
* [Tutorial](#tutorial)
  * [Instrumenting the simulator to measure conditional branches](#instrumenting-the-simulator-to-measure-conditional-branches)
  * [The functional simulation loop](#the-functional-simulation-loop)
  * [Debugging primer](#debugging-primer)
  * [Notes about the compilation environment](#notes-about-the-compilation-environment)

## Introduction

In this tutorial you will become familiar with the simulation infrastructure we will be using in this course. We will use a subset of the SimpleScalar toolset, a simplified version of simulators used in academic research and industry.

The variant we use in this assignment is a functional simulator. This means the code simulates every instruction executed by the processor, but does not simulate every microarchitectural component (e.g., cache hits and misses); as a result, simulated program functionality will be correct but the exact timing will be wrong.

After completing this tutorial, you should feel comfortable with the SimpleScalar infrastructure.

## Problem Statement

You will measure several quantities separately for each of the four benchmarks provided.

### Task 1

Measure and report the percentage of all executed instructions that are:

- conditional branches
- unconditional branches to a PC-relative offset (also called “direct jumps”)
- floating-point instructions
- store instructions
- load instructions
- instructions with immediate operands

### Task 2

Measure the frequency of different offsets between conditional branch instructions and the target of the conditional branch. Measure the offset as the number of _instructions_ (rather than bytes) between the branch and the target.

Plot a histogram of your measurements, where the _x_-axis (histogram bins) is the number of bits required to represent the offset (_including_ the sign bit), and the _y_-axis is the percentage of all conditional branch instructions executed that fall into a given histogram bin. Note that the PC is incremented by 8 for each sequential instruction in SimpleScalar—how is this different from RISC-V and why is the value 8?

### Task 3

Measure the average number of bits inside a register that change when a general-purpose register (R1–R31) is written to (ignore R0). Assume initially all registers contain zero. For example, consider the sequence:
```
    DADDI R1,R0,#1 ; R1 = 1 dec = 0... 0001 binary
      1
    DADDI R2,R0,#7 ; R2 = 7 dec = 0... 0111 binary
    DADDI R1,R0,#3 ; R1 = 3 dec = 0... 0011 binary
```
The first instruction changes one bit of R1. The second instruction changes three bits of R2 and the third instruction changes one bit of R1. On average, there are (1+3+1)/3 = 1.667 bits changed per instruction that writes to a register.


## Tutorial

### Instrumenting the simulator to measure conditional branches

Using your UBC ECE department account, log in to one of the department SSH servers (i.e., SSH to ssh-linux.ece.ubc.ca). The simulator should work on most variants of Linux, but we have only tested it on the ECE servers, so YMMV.

Check out the GitHub repository for the assignment (`git clone repository-url`). Enter the `src` folder and build the simulator by using the command `make`. Finally, test the simulator by using the command `make sim-tests` (this test checks to ensure the functional simulator is executing instructions correctly).

Open `sim-safe.c` in a text editor and add the following line of code at the beginning of the file after the `#include` statements:
```
static counter_t g_total_cond_branches;
```
This declares a counter variable we will use to measure the total number of conditional branches.  To register this counter so it will be printed out, add the following code to `sim_reg_stats()`:
```
stat_reg_counter(sdb, "sim_num_cond_branches" /* label for printing */,
                 "total conditional branches executed" /* description */,
                 &g_total_cond_branches /* pointer to the counter */, 
                 0 /* initial value for the counter */, NULL);
stat_reg_formula(sdb, "sim_cond_branch_freq",
                 "relative frequency of conditional branches",
                 "sim_num_cond_branches / sim_num_insn", NULL);
```

Next, find the function `sim_main()` in `sim-safe.c`. Add the following lines after the `switch` statement, but before the end of the `while` loop.
```
if (MD_OP_FLAGS(op) & F_COND)
    g_total_cond_branches++;
```
This code increments the counter each time a conditional branch is encountered.

Rebuild the simulator using `make`, enter the `benchmarks/fpppp` folder, and run the benchmark `fpppp` with the command:
```
../../src/sim-safe fpppp.ss < ./natoms-input.in
```
This should generate output very similar to:
```
sim_num_cond_branches       1571430 # total number of conditional branches …
sim_cond_branch_freq         0.0105 # relative frequency of conditional …
```
Note that your output [might not _exactly_ match ours](http://www.simplescalar.com/faq.html#Q2).


### The functional simulation loop

To execute an individual instruction, the following steps must occur in order:

1. the instruction must be obtained from program storage;
2. the actions to be carried out by the instruction must be determined;
3. any required data must be located and retrieved from storage;
4. the value or status generated by the instruction must be computed;
5. the results must be deposited in storage for later use;
6. the next instruction must be determined. 

Referring to the `while` loop inside `sim_main()`, the simulator performs these operations as follows:

Instruction fetch (step 1) is carried out by the lines:
```
/* get the next instruction to execute */
        MD_FETCH_INST(inst, mem, regs.regs_PC);
```

Instruction decode (step 2) is carried out by the lines:
```
/* decode the instruction */
        MD_SET_OPCODE(op, inst);
```

The remaining steps are carried out by the following 17 lines of C code:
```
/* execute the instruction */
      switch (op)
        {
#define DEFINST(OP,MSK,NAME,OPFORM,RES,FLAGS,O1,O2,I1,I2,I3)            \
        case OP:                                                        \
          SYMCAT(OP,_IMPL);                                             \
          break;
#define DEFLINK(OP,MSK,NAME,MASK,SHIFT)                                 \
        case OP:                                                        \
          panic("attempted to execute a linking opcode");
#define CONNECT(OP)
#define DECLARE_FAULT(FAULT)                                            \
          { fault = (FAULT); break; }
#include "machine.def"
        default:
          panic("attempted to execute a bogus opcode");
      }
```

Understanding the above lines of code can be a little challenging since these lines use the C `#define` macro operator in a way you may not have seen before. To understand the code, first consider the following code:
```
#define FOO(X,Y,Z) X=Y+Z;
int a, b=1, c=2, i, j=3, k=4;
FOO(a,b,c)
FOO(i,j,k)
```

The third line will be replaced with `a=b+c;` by the C compiler’s macro preprocessor and the fourth line will be replaced with `i=j+k;`.

Next, consider the line
```
#include "machine.def"
```

If you open the file `machine.def` in a text editor, you will find that it contains many instances of `DEFINST()`. You can think of these as the multiple instances of `FOO()` in the example above.  First, the `#include "machine.def"` statement is replaced by the C macro preprocessor with the contents of `machine.def`; then, the definition from `#define DEFINST` is used to replace each occurrence of `DEFINST()` copied from `machine.def` into `sim-safe.c`.

You can see the code generated for `sim-safe.c` by the C macro preprocessor by running:
```
gcc `./sysprobe -ﬂags` -DDEBUG -O0 -g -Wall -E -c sim-safe.c > sim-safe.i
```

The -E flag tells C compiler to send the output of the C macro preprocessor to standard output. The command line above instead redirects this output to the file `sim-safe.i`, which you can examine in a text editor.


### Debugging primer

If you encounter bugs (and you will) you may wonder how it is possible to debug anything on a linux machine over a text-only connection. The answer is `gdb`. Here is a quick tutorial:

Go to the `benchmarks/gcc` folder, and start the simulator in `gdb` using the following commands: 
```
cd gcc
gdb --args ../../src/sim-safe -max:inst 200000000 cc1.ss -O2 integrate.i
```

After a short while, you should see the gdb command prompt `(gdb)`. Set a breakpoint at the beginning of the main simulator loop function `sim_main()` by entering `b sim_main` at the `gdb` command prompt:
```
(gdb) b sim_main
Breakpoint 1 at 0x804986f: file sim-safe.c, line 285.
```

Next, run the program by typing `r`. The simulator will stop at the breakpoint we just set above:
```
(gdb) r
...
Breakpoint 1, sim_main () at sim-safe.c:285
285       fprintf(stderr, "sim: ** starting functional simulation **\n");
```

Now you can examine the values of any variable that is in scope by typing `p` followed by the name of the C variable you want to examine. For example, to print out the current number of instructions that have been simulated (which at this point should be zero), examine the variable `sim_num_insn`:
```
(gdb) p sim_num_insn
$1 = 0
```

You can also list some lines of code around the place where `gdb` has stopped the program (and therefore see what it will execute next) by typing `l`. The next line to execute is always in the middle of the lines that are displayed. The numbers on the left hand side are line numbers in the source program. Here, the next line to execute is 285:
```
(gdb) l
280       register md_addr_t addr;
281       enum md_opcode op;
282       register int is_write;
283       enum md_fault_type fault;
284
285       fprintf(stderr, "sim: ** starting functional simulation **\n");
286
287       /* set up initial default next PC */
288       regs.regs_NPC = regs.regs_PC + sizeof(md_inst_t);
289
```

Now, single-step over the next line of code by typing `n`:
```
(gdb) n
sim: ** starting functional simulation **
288       regs.regs_NPC = regs.regs_PC + sizeof(md_inst_t);
```
(typing `s` is similar, but will follow into a function call).

You can also set breakpoints by filename and line number. Set another breakpoint, this time at line 332, inside the main simulation loop:
```
(gdb) b sim-safe.c:332
Breakpoint 2 at 0x8051e7c: file sim-safe.c, line 332.
```

Finally, allow the simulator to continue running until the next breakpoint is reached by typing `c`:
```
(gdb) c
Continuing.
Breakpoint 2, sim_main () at sim-safe.c:332
332           if (fault != md_fault_none)
```

We have shown how to set a breakpoint, run the program, and continue after a breakpoint. Other gdb commands you may find useful for debugging your modified simulator can be found by the “help” command in gdb.  

### Notes about the compilation environment

If you are using Ubuntu and get a compilation error about a missing `sys/cdefs.h`, you are probably missing the package `g++-multilib`.
