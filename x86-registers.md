# x86-registers

## [kvm_regs](kvm-bindings-0.5.0/src/x86/bindings.rs)
```rs
pub struct kvm_regs {
    pub rax: __u64,
    pub rbx: __u64,
    pub rcx: __u64,
    pub rdx: __u64,
    pub rsi: __u64,
    pub rdi: __u64,
    pub rsp: __u64,
    pub rbp: __u64,
    pub r8: __u64,
    pub r9: __u64,
    pub r10: __u64,
    pub r11: __u64,
    pub r12: __u64,
    pub r13: __u64,
    pub r14: __u64,
    pub r15: __u64,
    pub rip: __u64,
    pub rflags: __u64,
}
```

From Section 3.4.1 Volume 1.
* EAX — Accumulator for operands and results data 
* EBX — Pointer to data in the DS segment 
* ECX — Counter for string and loop operations 
* EDX — I/O pointer 
* ESI — Pointer to data in the segment pointed to by the DS register; source pointer for string operations 
* EDI — Pointer to data (or destination) in the segment pointed to by the ES register; destination pointer for string operations 
* ESP — Stack pointer (in the SS segment) 
* EBP — Pointer to data on the stack (in the SS segment)

In 64-bit mode, there are 16 general purpose registers and the default operand
size is 32 bits.  If a 32-bit operand size is specified: EAX, EBX, ECX, EDX,
EDI, ESI, EBP, ESP, R8D - R15D are available. If a 64-bit operand size is
specified: RAX, RBX, RCX, RDX, RDI, RSI, RBP, RSP, R8-R15 are available.
R8D-R15D/R8-R15 represent eight new general-purpose registers.  All of these
registers can be accessed at the byte, word, dword, and qword level.

The instruction pointer (EIP) register contains the offset in the current code
segment for the next instruction to be executed. It is advanced from one
instruction boundary to the next in straight-line code or it is moved ahead or
backwards by a number of instructions when executing JMP, Jcc, CALL, RET, and
IRET instructions. In 64-bit mode, the RIP register becomes the instruction
pointer.  From Section 3.4.3 Volume 1 EFLAGS register contains a group of
status flags, a control flag, and a group of system flags.  In 64-bit mode,
EFLAGS is extended to 64 bits and called RFLAGS. The upper 32 bits of RFLAGS
register is reserved. The lower 32 bits of RFLAGS is the same as EFLAGS. Setting
rflags to 2 implies that all EFLAGS registers are clear. See section 3.4.3 for
more details.

## kvm_sregs 

```rs
pub struct kvm_sregs {
    pub cs: kvm_segment,
    pub ds: kvm_segment,
    pub es: kvm_segment,
    pub fs: kvm_segment,
    pub gs: kvm_segment,
    pub ss: kvm_segment,
    pub tr: kvm_segment,
    pub ldt: kvm_segment,
    pub gdt: kvm_dtable,
    pub idt: kvm_dtable,
    pub cr0: __u64,
    pub cr2: __u64,
    pub cr3: __u64,
    pub cr4: __u64,
    pub cr8: __u64,
    pub efer: __u64,
    pub apic_base: __u64,
    pub interrupt_bitmap: [__u64; 4usize],
}
```

From Section 3.4.2, Volume 2.  CS register contains the segment selector for the
code segment, where the instructions being executed are stored. The processor
fetches instructions from the code segment, using a logical address that
consists of the segment selector in the CS register and the contents of the EIP
register. The EIP register contains the offset within the code segment of the
next instruction to be executed. The CS register cannot be loaded explicitly by
an application program. Instead, it is loaded implicitly by instructions or
internal processor operations that change program control (such as, procedure
calls, interrupt handling, or task switching).  The DS, ES, FS, and GS registers
point to four data segments. The availability of four data segments permits
efficient and secure access to different types of data structures. For example,
four separate data segments might be created: one for the data structures of the
current module, another for the data exported from a higher-level module, a
third for a dynamically created data structure, and a fourth for data shared
with another program.

SS register contains the segment selector for the stack segment, where the
procedure stack is stored for the program, task, or handler currently being
executed. All stack operations use the SS register to find the stack segment.
Unlike the CS register, the SS register can be loaded explicitly, which permits
application programs to set up multiple stacks and switch among them.

In 64-bit mode: CS, DS, ES, SS are treated as if each segment base is 0,
regardless of the value of the associated segment descriptor base. This creates
a flat address space for code, data, and stack. FS and GS are exceptions. Both
segment registers may be used as additional base registers in linear address
calculations.  Why are they stored in kvm_segment and not just as _u16?  TR
(Task Register) has to do with Task state segment (TSS). It was built to support
context switching but it is not used in practice.  From
https://en.wikipedia.org/wiki/Task_state_segment The TSS may contain saved
values of all the x86 registers. This is used for task switching. The operating
system may load the TSS with the values of the registers that the new task needs
and after executing a hardware task switch (such as with an IRET instruction)
the x86 CPU will load the saved values from the TSS into the appropriate
registers. Note that some modern operating systems such as Windows and Linux[1]
do not use these fields in the TSS as they implement software task switching.
    
From Section 2.5 in Volume 3A
* CR0 — Contains system control flags that control operating mode and states
of the processor. 
* CR1 — Reserved. 
* CR2 — Contains the page-fault linear address (the linear address that
caused a page fault). 
* CR3 — Contains the physical address of the base of the paging-structure
hierarchy and two flags (PCD and PWT). Only the most-significant bits (less
the lower 12 bits) of the base address are specified; the lower 12 bits of
the address are assumed to be 0. The first paging structure must thus be
aligned to a page (4-KByte) boundary. The PCD and PWT flags control caching
of that paging structure in the processor’s internal data caches (they do
not control TLB caching of page-directory information). When using the
physical address extension, the CR3 register contains the base address of
the page-directory pointer table.  In IA-32e mode, the CR3 register contains
the base address of the PML4 table. See also: Chapter 4, “Paging.” 
* CR4 — Contains a group of flags that enable several architectural
extensions, and indicate operating system or executive support for specific
processor capabilities.
* CR8 — Provides read and write access to the Task Priority Register (TPR).
It specifies the priority threshold value that operating systems use to
control the priority class of external interrupts allowed to interrupt the
processor. This register is available only in 64-bit mode. However,
interrupt filtering continues to apply in compatibility mode.

Very detailed. Of interest to us is CR2, CR3, and CS. Everything is set to
zero.