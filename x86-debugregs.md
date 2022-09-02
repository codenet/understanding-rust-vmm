# x86-debugregs

From Chapter 17 in Volume 3B, Part II

Intel 64 and IA-32 architectures provide debug facilities for use in debugging
code and monitoring performance. These facilities are valuable for debugging
application software, system software, and multitasking operating systems. Debug
support is accessed using debug registers (DR0 through DR7) and model-specific
registers (MSRs): 

* Debug registers hold the addresses of memory and I/O
locations called breakpoints. Breakpoints are userselected locations in a
program, a data-storage area in memory, or specific I/O ports. They are set
where a programmer or system designer wishes to halt execution of a program and
examine the state of the processor by invoking debugger software. A debug
exception (#DB) is generated when a memory or I/O access is made to a breakpoint
address. 
* MSRs monitor branches, interrupts, and exceptions; they record
addresses of the last branch, interrupt or exception taken and the last branch
taken before an interrupt or exception.


## [kvm_debugregs](kvm-bindings-0.5.0/src/x86/bindings.rs)
```rs
pub struct kvm_debugregs {
    pub db: [__u64; 4usize],
    pub dr6: __u64,
    pub dr7: __u64,
    pub flags: __u64,
    pub reserved: [__u64; 9usize],
}
```

These registers are for setting and managing breakpoints. `db` is an array of 4
registers which seems to be DR0-DR3 registers. We can also do breakpoints using
`INT 3` instruction and let the debugger handle it by inspecting `DB`, `DR6`,
and `DR7` registers.

> Each of the debug-address registers (DR0 through DR3) holds the 32-bit linear
address of a breakpoint (see Figure 17-1). Breakpoint comparisons are made
before physical address translation occurs. The contents of debug register DR7
further specifies breakpoint conditions.

> * Debug exception (#DB) — Transfers program control to a debug procedure or task
  when a debug event occurs. 
> * Breakpoint exception (#BP) — See breakpoint instruction (INT 3) below. 
> * Breakpoint-address registers (DR0 through DR3) — Specifies the addresses of up
 to 4 breakpoints. 
> * Debug status register (DR6) — Reports the conditions that were in effect when
  a debug or breakpoint exception was generated. 
> * Debug control register (DR7) — Specifies the forms of memory or I/O access
 that cause breakpoints to be generated. 
> * Breakpoint instruction (INT 3) — Generates a breakpoint exception (#BP) that
 transfers program control to the debugger procedure or task. This instruction
 is an alternative way to set code breakpoints. It is especially useful when
 more than four breakpoints are desired, or when breakpoints are being placed
 in the source code. 

There are some flags.
> * T (trap) flag, TSS — Generates a debug exception (#DB) when an attempt is made
 to switch to a task with the T flag set in its TSS. 
> * RF (resume) flag, EFLAGS register — Suppresses multiple exceptions to the same
 instruction. 
> * TF (trap) flag, EFLAGS register — Generates a debug exception (#DB) after
 every execution of an instruction. 

## MSRs
> From [Wikipedia](https://en.wikipedia.org/wiki/Model-specific_register), a
model-specific register (MSR) is any of various control registers in the x86
instruction set used for debugging, program execution tracing, computer
performance monitoring, and toggling certain CPU features.

Basically, MSR are "model specific": they are experimental registers. Intel
doesn't yet want to commit to them for future processors.

From 9.4 in Volume 3A, Part 1
> Most IA-32 processors (starting from Pentium processors) and Intel 64
processors contain a model-specific registers (MSRs). A given MSR may not be
supported across all families and models for Intel 64 and IA-32 processors. Some
MSRs are designated as architectural to simplify software programming; a feature
introduced by an architectural MSR is expected to be supported in future
processors. Non-architectural MSRs are not guaranteed to be supported or to have
the same functions on future processors.

> MSRs that provide control for a number of hardware and software-related
features, include: 
> * Performance-monitoring counters (see Chapter 23, “Introduction to Virtual Machine Extensions”). 
> * Debug extensions (see Chapter 23, “Introduction to Virtual Machine Extensions.”). 
> * Machine-check exception capability and its accompanying machine-check
architecture (see Chapter 15, “Machine-Check Architecture”). 
> * MTRRs (see Section 11.11, “Memory Type Range Registers (MTRRs)”). 
> * Thermal and power management. 
> * Instruction-specific support (for example: SYSENTER, SYSEXIT, SWAPGS, etc.). 
> * Processor feature/mode support (for example: IA32_EFER,
IA32_FEATURE_CONTROL). 
>
> The MSRs can be read and
written to using the RDMSR and WRMSR instructions, respectively. When performing
software initialization of an IA-32 or Intel 64 processor, many of the MSRs will
need to be initialized to set up things like performance-monitoring events,
run-time machine checks, and memory types for physical memory.

Chapter 17 in Volume 3B, Part II shows some examples of MSRs used for debugging.
> * Last branch recording facilities — Store branch records in the last branch
 record (LBR) stack MSRs for the most recent taken branches, interrupts, and/or
 exceptions in MSRs. A branch record consist of a branch-from and a branch-to
 instruction address. Send branch records out on the system bus as branch trace
 messages (BTMs).

>  When the LBR flag (bit 0) in the IA32_DEBUGCTL MSR is set, the processor
automatically begins recording branch records for taken branches, interrupts,
and exceptions (except for debug exceptions) in the LBR stack MSRs. When the
processor generates a debug exception (#DB), it automatically clears the LBR
flag before executing the exception handler. This action does not clear
previously stored LBR stack MSRs. A debugger can use the linear addresses in the
LBR stack to re-set breakpoints in the breakpoint address registers (DR0 through
DR3). This allows a backward trace from the manifestation of a particular bug
toward its source.

More details can be found in Chapter 35, “Model-Specific Registers (MSRs)”.