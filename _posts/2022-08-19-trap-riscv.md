---
title: Trap handling under RISC-V Architecture 
date: 2022-08-19 10:20:48
tags: RISC-V
copyright: true
---

# **1. Overview**
During user mode program execution, unexpected events, exceptions or interruptions, may occur, leading to a trap. This post's goal is to provide a general view of how the trap is handled, and this section will provide an outline of key concepts. You can skip forward to the next section if you'd rather start with something practical.

## 1.1. Trap
> The transfer of control to a trap handler caused by either an exception or an interrupt. 
---- RISC-V Spec Volume 2^[1]

Any control transfer raised from unprivileged mode (ie. user/application mode) to the operating system can be considered as a trap.

| Type  |  Detailed Description | Execution terminates?  | Software is oblivious?  | Handled by environment?  |
|---|---|---|---|---|
| Contained Trap | Being visible and handled by the software in the environment | No  | No  | No  |
|  Requested Trap | Synchronous action request from software to the environment (*eg. System call which removes the hart*)  | Maybe (when termination is requested)  | No  | Yes  |
| Invisible Trap | Handled transparently by the execution environment and execution resumes normally after the trap is handled (*eg.handling device interrupts*) | No  | Yes  | Yes  |
| Fatal Trap |  fatal failure and causes the execution environment to terminate execution  | Yes  | Maybe  | Yes  |

## 1.2. Exception and Interrupt  
Exception: Asynchronous, program-initiated unexpected events raised within the processor (associated with instructions) which disrupt program execution. *e.g. Divide by zero, Undefined function*

Interrupt: Asynchronous, device-initiated events from user to the OS. Most interrupts are external while there are still a few internal interrupts under RISC-V architecture. *e.g. Clock tick, network packet*

Interrupts and exceptions are **precise** when it is clearly associated with specific instructions^[2]. Therefore, all instructions before the faulty instructions could be completely executed with the faulty instruction address stored in SEPC/MEPC (will be elaborated in 2. RISC-V Implementation). **Imprecise** interrupts and exceptions are not associated with instructions, the faulty instruction will be determined by operating system. Consequently, The address stored in SEPC/MEPC is from the latest instruction in the pipeline.


# **2. Trap Handling Implementation**

## 2.1. CSR - Control/Status Register
To assist the trap handling, a subset of CSRs is utilized to record the trap-related information *(e.g. The cause of trap)*. In this section, CSRs related to trap handling will be listed. Detailed introduction for CSR can be found in [RISC-V Spec Volume 2](https://riscv.org/technical/specifications/).

Note that in RISC-V, each mode has its own set of CSRs (M: Machine; S: Supervisor; U: User/Application), and the bit width of CSR is determined by XLEN macro which has various configurations.
*e.g. MEPC is for M-mode and SEPC is for S-mode. MXLEN for M-mode.*

CSRs relevant to trap handling in this post can be found at the end. Let's trace through the handlers to see how RISC-V handling those unexpected events! 


## 2.2. Basic Trap Handling Process

In general, trap handling begins with its trap entry, which functions like an entrance to the "trap handling" park. The trap entry label is written in assembly code, while the handler is written in C. When the control is transfered to OS (ie. trap triggered), the program counter pointed to the address labeled **trap entry** in assembly.

**Sample code for trap handler in machine mode** is shown below^[3].

RISC-V Assembly interrupt handler to Push and Pop register file
``` assembly
  .align 2
  .global trap_entry
trap_entry:
  addi sp, sp, -16*REGBYTES
  //store ABI Caller Registers
  STORE x1, 0*REGBYTES(sp)
  STORE x5, 2*REGBYTES(sp)
    …
  STORE x30, 14*REGBYTES(sp)
  STORE x31, 15*REGBYTES(sp)
 //call C Code Handler
  call handle_trap
 //restore ABI Caller Registers
  LOAD x1, 0*REGBYTES(sp)
  LOAD x5, 2*REGBYTES(sp)
    …
  LOAD x30, 14*REGBYTES(sp)
  LOAD x31, 15*REGBYTES(sp)
  addi sp, sp, 16*REGBYTES
  mret
```

C Code Handler determines interrupt cause and branches to the appropriate function 
```c
void handle_trap() {
    unsigned long mcause = read_csr(mcause);
    if (mcause & MCAUSE_INT) {
        //mask interrupt bit and branch to handler
        isr_handler[mcause & MCAUSE_CAUSE] ();
    } else {
        //branch to handler
        exception_handler[mcause]();
    }
 }
 //write trap_entry address to mtvec  
 write_csr(mtvec, ((unsigned long)&trap_entry)); 
```

After getting to the "trap handling" park, to preserve user registers' values before the content switching, they are stored into the kernel stack. After getting into the handle_trap() function in C, the cause of the trap is identified based on the value in CSR **mcause**. According to the type of cause, specific handler is called to further handle the trap. When the handler program is done, control is passed back to assembly side to restore the values in user registers. Finally, **mret** instruction is used to go back to user mode execution.

![Trap Handling Flow](/images/trap_flow.png)

## 2.3. Variations
The description of the handling process above is pretty generic, and many detailed configurations of the process are varied. For instance, CSR **mtvec** is holding the base address of the trap entry and configuring the way to access trap handler. When it in direct mode (MODE 0), all exceptions set pc to BASE. When it is vectored mode (MODE 1), asynchronous interrupts set pc to BASE+4×cause. One more example, return instruction **mret** may not jump back to user mode, as it can also switch to another privileged mode based on configuration.

Maybe I will further edit this post one day to introduce more about it or you can checkout those trap-related CSRs in [RISC-V Spec Volume 2](https://riscv.org/technical/specifications/) to explore :D

## 2.4. Relevant CSRs List
- Cause Register (**mcause**/**scause**): The highest bit (mcause[XLEN-1]) indicates this is caused by exception (0) or interrupt (1). The rest of bits (mcause[XLEN-2:0]) are holding the cause of the trap.
- Trap-Vector Base-Address Register (**mtvec**/**stvec**): read/write register that holds trap vector configuration, consisting of a vector base address (BASE) and a vector mode (MODE).
- Machine Exception Program Counter (**mepc**/**sepc**): Holds the address of the instruction which will be executed after current trap handling.

# **3. References**
[1] [Volume 1/2, RISC-V Spec](https://riscv.org/technical/specifications/)
[2] [Computer Organization and Design - The Hardware/Software Interface: RISC-V Edition](https://www.cs.sfu.ca/~ashriram/Courses/CS295/assets/books/HandP_RISCV.pdf)
[3] [An Introduction to the RISC-V Architecture (Online Slides by SiFive) ](https://cdn2.hubspot.net/hubfs/3020607/An%20Introduction%20to%20the%20RISC-V%20Architecture.pdf)

# **4. Helpful Links**
 [Cornell CompArch lecture slides](http://www.cs.cornell.edu/courses/cs316/2007fa/Lectures/Lec21_Interrupts_web.pdf)
[RISC-V Exception and Interrupt implementation (Blog by Lee)](https://mullerlee.cyou/2020/07/09/riscv-exception-interrupt/)
[All Aboard, Part 7: Entering and Exiting the Linux Kernel on RISC-V](https://www.sifive.com/blog/all-aboard-part-7-entering-and-exiting-the-linux-kernel-on-risc-v)