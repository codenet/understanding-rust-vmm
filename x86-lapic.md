# course-col732-x86-lapic

From Volume 3A, Part 1, Ch 10

> The local APIC performs two primary functions for the processor: 
> * It receives interrupts from the processor’s interrupt pins, from internal
sources and from an external I/O APIC (or other external interrupt controller).
It sends these to the processor core for handling. 
> * In multiple processor (MP) systems, it sends and receives interprocessor
interrupt (IPI) messages to and from other logical processors on the system bus.
IPI messages can be used to distribute interrupts among the processors in the
system or to execute system wide functions (such as, booting up processors or
distributing work among a group of processors).

From Volume 3A, Part 1, 10.4.1
> APIC registers are memory-mapped to a 4-KByte region of the processor’s
physical address space with an initial starting address of FEE00000H. For
correct APIC operation, this address space must be mapped to an area of memory
that has been designated as strong uncacheable (UC).

It is indeed kept as 4KB region `usize` = 4 Bytes.

```rs
pub struct kvm_lapic_state {
    pub regs: [::std::os::raw::c_char; 1024usize],
}
```

These can be [set or get using ioctl calls](kvm-ioctls-0.11.0/src/kvm_ioctls.rs):
```rs
/* Available with KVM_CAP_IRQCHIP */
#[cfg(any(target_arch = "x86", target_arch = "x86_64"))]
ioctl_ior_nr!(KVM_GET_LAPIC, KVMIO, 0x8e, kvm_lapic_state);
/* Available with KVM_CAP_IRQCHIP */
#[cfg(any(target_arch = "x86", target_arch = "x86_64"))]
ioctl_iow_nr!(KVM_SET_LAPIC, KVMIO, 0x8f, kvm_lapic_state);
```

Table 10-1 shows the the meaning of each address in the 4KB space. More
structure could be imposed on top of `kvm_lapic_state` based on the table. 
`vmm-reference` seems to have very limited support to configure LAPIC. [It only
allows setting delivery
mode](vmm-reference/src/vm-vcpu-ref/src/x86_64/interrupts.rs).

```rs
/// Set the Local APIC delivery mode. Returns an error when the register configuration
/// is invalid.
///
/// **Note**: setting the Local APIC (using the kvm function `set_lapic`) MUST happen
/// after creating the IRQ chip. Otherwise, KVM returns an invalid argument (errno 22).
///
/// # Arguments
/// * `klapic`: The corresponding `kvm_lapic_state` for which to set the delivery mode.
/// * `reg_offset`: The offset that identifies the register for which to set the delivery mode.
///                 Available options exported by this module are: [APIC_LVT0_REG_OFFSET] and
///                 [APIC_LVT1_REG_OFFSET].
/// * `mode`: The APIC mode to set.
///
/// # Example - Configure LAPIC with KVM
/// ```rust
/// # use kvm_ioctls::Kvm;
/// use kvm_ioctls::{Error, VcpuFd};
/// use vm_vcpu_ref::x86_64::interrupts::{
///     set_klapic_delivery_mode, DeliveryMode, APIC_LVT0_REG_OFFSET, APIC_LVT1_REG_OFFSET,
/// };
///
/// fn configure_default_lapic(vcpu_fd: &mut VcpuFd) -> Result<(), Error> {
///     let mut klapic = vcpu_fd.get_lapic()?;
///
///     set_klapic_delivery_mode(&mut klapic, APIC_LVT0_REG_OFFSET, DeliveryMode::ExtINT).unwrap();
///     set_klapic_delivery_mode(&mut klapic, APIC_LVT1_REG_OFFSET, DeliveryMode::NMI).unwrap();
///     vcpu_fd.set_lapic(&klapic)
/// }
///
/// # let kvm = Kvm::new().unwrap();
/// # let vm = kvm.create_vm().unwrap();
/// # vm.create_irq_chip().unwrap();
/// # let mut vcpu = vm.create_vcpu(0).unwrap();
/// # configure_default_lapic(&mut vcpu).unwrap();
/// ```
```

Directly setting/getting other registers are possible, but it is not
particularly convenient.  Imposing additional structure using the table would
have been nice. Currently, programmer needs to refer to Table 10-1 to understand
the meaning of the register.

```rs
pub fn get_klapic_reg(klapic: &kvm_lapic_state, reg_offset: usize) -> Result<i32>
pub fn set_klapic_reg(klapic: &mut kvm_lapic_state, reg_offset: usize, value: i32) -> Result<()>
```