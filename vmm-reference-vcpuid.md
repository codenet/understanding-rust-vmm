
# [cpudid.rs](https://github.com/rust-vmm/vmm-reference/blob/main/src/vm-vcpu-ref/src/x86_64/cpuid.rs)

## `CPUID` instruction in x86

Can be used by the software to determine the type of CPU it is running on and what features it provides.

The instruction does not have an explicit parameter. Instead EAX must be set before calling the instruction to indicate a `function`.  The information returned in stored in the `EBX, EDX, ECX` registers.   


Some examples of `functions` are:

**EAX = 0:**
Stores the manufacturer's id as ASCII string in the registers EBX, EDX, ECX and sets EAX to be highest calling parameter value that the CPU supports.   
Common common manufacturer id values are:

"GenuineIntel": Intel
"AuthenticAMD": AMD
"AMDisbetter!": early AMD chips

"KVMKVMKVM   ": KVM 
"VMwareVMware": VMware

**EAX = 1:**

returns the family and model of the CPU in EAX.   
returns the features supported by the cpu such as hyper-threading, vmx, debug instructions, lapic, simd etc..

**EAX = 2:**
Cache and TLB Description

**EAX = 6:**
Power & Thermal Management features


For a VM most of the cpu id information is set by KVM.    

The file [cpuid.rs](https://github.com/rust-vmm/vmm-reference/blob/main/src/vm-vcpu-ref/src/x86_64/cpuid.rs) contains a function `filter_cpuid` the overrides some of the values in used by the default KVM. The top of the file contains the positions of `CPUID` bits for specific information that the filter function may override.   


This function is used in the file [vcpu/mod.rs](https://github.com/rust-vmm/vmm-reference/blob/ad371898afe7d1d230ab4f8a5148d1aa511eed69/src/vm-vcpu/src/vcpu/mod.rs) while creating config for the vcpus of a vm.  

```rust
        for index in 0..num_vcpus {
            // Set CPUID.
            #[cfg(target_arch = "x86_64")]
            let mut cpuid = base_cpuid.clone();
            #[cfg(target_arch = "x86_64")]
            vm_vcpu_ref::x86_64::cpuid::filter_cpuid(_kvm, index, num_vcpus, &mut cpuid);

            #[cfg(target_arch = "x86_64")]
            let vcpu_config = VcpuConfig {
                cpuid,
                id: index,
                msrs: supported_msrs.clone(),
            };
```

The config is then used in a `KVM_SET_CPUID` call before the cpu is run.   
Whenever the `vCPU` calls the `CPUDID` instruction, a `VM_EXIT` happens and it is handled by the `KVM kernel exit handler`.  
