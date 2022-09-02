# x86-mpstate

The idea is manage the multiprocessing system. x86 goes through a sophisticated
booting mechanism called "Multiple-processor (MP) initialization". From section 8.4 
in Volume 3A, Part 1: 

> The MP initialization protocol defines two classes of processors: the
bootstrap processor (BSP) and the application processors (APs). Following a
power-up or RESET of an MP system, system hardware dynamically selects one of
the processors on the system bus as the BSP. The remaining processors are
designated as APs. 
>
> As part of the BSP selection mechanism, the BSP flag is set in the
IA32_APIC_BASE MSR (see Figure 10-5) of the BSP, indicating that it is the BSP.
This flag is cleared for all other processors. 
>
> The BSP executes the BIOS’s boot-strap code to configure the APIC environment,
sets up system-wide data structures, and starts and initializes the APs. When
the BSP and APs are initialized, the BSP then begins executing the
operating-system initialization code.  
> 
> Following a power-up or reset, the APs complete a minimal self-configuration,
then wait for a startup signal (a SIPI message) from the BSP processor. Upon
receiving a SIPI message, an AP executes the BIOS AP configuration code, which
ends with the AP being placed in halt state. 
> 
> For Intel 64 and IA-32 processors supporting Intel Hyper-Threading Technology,
the MP initialization protocol treats each of the logical processors on the
system bus or coherent link domain as a separate processor (with a unique APIC
ID). During boot-up, one of the logical processors is selected as the BSP and
the remainder of the logical processors are designated as APs.

This state keeps the state of the multiprocessor. Has the processor started or
is waiting for signal?  From
https://www.kernel.org/doc/Documentation/virtual/kvm/api.txt

> 4.38 KVM_GET_MP_STATE
>
> Capability: KVM_CAP_MP_STATE
> Architectures: x86, s390, arm, arm64
> Type: vcpu ioctl
> Parameters: struct kvm_mp_state (out)
> Returns: 0 on success; -1 on error
> struct kvm_mp_state {
	__u32 mp_state;
};
> 
> Returns the vcpu's current "multiprocessing state" (though also valid on
> uniprocessor guests).
> 
> Possible values are:
> 
>  - KVM_MP_STATE_RUNNABLE:        the vcpu is currently running [x86,arm/arm64]
>  - KVM_MP_STATE_UNINITIALIZED:   the vcpu is an application processor (AP)
>                                  which has not yet received an INIT signal [x86]
>  - KVM_MP_STATE_INIT_RECEIVED:   the vcpu has received an INIT signal, and is
>                                  now ready for a SIPI [x86]
>  - KVM_MP_STATE_HALTED:          the vcpu has executed a HLT instruction and
>                                  is waiting for an interrupt [x86]
>  - KVM_MP_STATE_SIPI_RECEIVED:   the vcpu has just received a SIPI (vector
>                                  accessible via KVM_GET_VCPU_EVENTS) [x86]
>  - KVM_MP_STATE_STOPPED:         the vcpu is stopped [s390,arm/arm64]
>  - KVM_MP_STATE_CHECK_STOP:      the vcpu is in a special error state [s390]
>  - KVM_MP_STATE_OPERATING:       the vcpu is operating (running or halted)
>                                  [s390]
>  - KVM_MP_STATE_LOAD:            the vcpu is in a special load/startup state
>                                  [s390]
> 
> On x86, this ioctl is only useful after KVM_CREATE_IRQCHIP. Without an
> in-kernel irqchip, the multiprocessing state must be maintained by userspace on
> these architectures.
> 
> 4.39 KVM_SET_MP_STATE
> 
> Capability: KVM_CAP_MP_STATE
> Architectures: x86, s390, arm, arm64
> Type: vcpu ioctl
> Parameters: struct kvm_mp_state (in)
> Returns: 0 on success; -1 on error
> 
> Sets the vcpu's current "multiprocessing state"; see KVM_GET_MP_STATE for
> arguments.

This seems to give the guest the same environment as of real hardware.  When guest
boots up, only one processor will be marked `KVM_MP_STATE_RUNNABLE`.  Guest may
manage the Application Processors on its own. 

`vmm-reference` does create multiple CPUs but we did not see explicit calls to
`KVM_GET_MP_STATE`.  Perhaps, it is handled automatically when create VCPUs.

https://www.kernel.org/doc/Documentation/virtual/kvm/api.txt mentions
```
4.41 KVM_SET_BOOT_CPU_ID

Capability: KVM_CAP_SET_BOOT_CPU_ID
Architectures: x86
Type: vm ioctl
Parameters: unsigned long vcpu_id
Returns: 0 on success, -1 on error

Define which vcpu is the Bootstrap Processor (BSP).  Values are the same
as the vcpu id in KVM_CREATE_VCPU.  If this ioctl is not called, the default
is vcpu 0.
```