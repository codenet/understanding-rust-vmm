# [gdt.rs](https://github.com/rust-vmm/vmm-reference/blob/main/src/vm-vcpu-ref/src/x86_64/gdt.rs)


## GDT in intel x86 

`GDT` (Global Descriptor Table) is one the many tables(LDT, IDT) defined by the intel x86 architecture.   
The GDT contains several 8 byte entries telling the CPU about memory segments or system wide resources.   

Each entry should have the following format:
   
> `base`: A 32-bit value containing the linear address where the segment begins.   
> `limit`: A 20-bit value (the most significant 12 bits are ignored from the value) that tells the maximum addressable unit.
 > `flags`: Depending on the segment type, the flags set up various properties such as the privilege level, type, and granularity.

In 64-bit mode, the Base and Limit values are ignored, each descriptor covers the entire linear address space regardless of what they are set to. 

The GDT itself is not a segment and the base and bound of the GDT in the memory is stored in a special `GDTR` register.   

The GDT is used to manage system wide resource and unlike the LDT(which is present for each task), only one instance of the GDT is present in a system.   

The first entry of the GDT must be NULL(zeroed out).   
The next two entries contain the description of the CS and DS. These can be used for segmentation in the OS.   

The GDT also contains the entries for the segments containing the TSSs and the LDTs.   

Other than these, it can also contain information for system wide resources such as the keyboard input buffer.   

The file [gdt.rs](https://github.com/rust-vmm/vmm-reference/blob/main/src/vm-vcpu-ref/src/x86_64/cpuid.rs) contains an `impl SegmentDescriptor` which contains functions for creating a segment descriptor from given base, bound and flags.  
It also contains functions to get and set the base, limit or flags.   

It also contains functions to get and set individual flags-   

**g**: Granularity flag, indicates the size the Limit value is scaled by. If clear (0), the Limit is in 1 Byte blocks (byte granularity). If set (1), the Limit is in 4 KiB blocks (page granularity).  

**l**: Long-mode code flag. If set (1), the descriptor defines a 64-bit code segment. When set, DB should always be clear. For any other type of segment (other code types or any data segment), it should be clear (0). 

**avl**: Ignored by the hardware.

**p**: Present bit. Allows an entry to refer to a valid segment. Must be set (1) for any valid segment  

**dpl**: Descriptor privilege level field. Contains the CPU Privilege level of the segment. 0 = highest privilege (kernel), 3 = lowest privilege (user applications).   

**s**: Descriptor type bit. If clear (0) the descriptor defines a system segment (eg. a Task State Segment). If set (1) it defines a code or data segment.  

Finally, it contains a function `create_kvm_segment`  
which returns a `kvm_segment` struct from the `Segment Descriptor`.   

This struct is present the in `kvm_sregs` struct  
```c
truct kvm_sregs {
	struct kvm_segment cs, ds, es, fs, gs, ss;
	struct kvm_segment tr, ldt;
	struct kvm_dtable gdt, idt;
	__u64 cr0, cr2, cr3, cr4, cr8;
	__u64 efer;
	__u64 apic_base;
	__u64 interrupt_bitmap[(KVM_NR_INTERRUPTS + 63) / 64];
};
``` 
and is passed to the KVM API for `vCPU` initialization using the `KVM_SET_SREGS` call.    


The file also contains a `Gdt` struct which can be used to manage the guest's `GDT`.    

It contains  method `create_kvm_segment_for` to create a segment using KVM in a more controlled way,  
and a method `write_to_mem` to directly write the GDT to the guest memory.   



The default implementation for the `GDT` is
```rust
SegmentDescriptor::from(0, 0, 0),            // NULL
SegmentDescriptor::from(0xa09b, 0, 0xfffff), // CODE
SegmentDescriptor::from(0xc093, 0, 0xfffff), // DATA
SegmentDescriptor::from(0x808b, 0, 0xfffff), // TSS
```
which contains the segments descriptors needed to [safely enter ring3](https://wiki.osdev.org/Getting_to_Ring_3).  
