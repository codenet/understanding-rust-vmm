Firstly in this code they strongly suggest that the 'cpuid' be derived from the supported CPUID of the host machine that is currently operating.
use kvm_bindings::CpuId;
use kvm_ioctls::{Cap::TscDeadlineTimer, Kvm};


They have defined cpuid and used it with kvm_bindings and kvm_iocontrols .we are defining some variables with the cpuid bits in ebx,ecx  and edx registers .all most every varibale is unsigned int of 32 bit ex:  EBX_CLFLUSH_CACHELINE: u32,EBX_CPU_COUNT_SHIFT: u32 ,ECX_TSC_DEADLINE_TIMER_SHIFT: u32 etc 

### const EBX_CLFLUSH_CACHELINE: u32 = 8; // Flush a cache line size.
const EBX_CLFLUSH_SIZE_SHIFT: u32 = 8; // Bytes flushed when executing CLFLUSH.
const EBX_CPU_COUNT_SHIFT: u32 = 16; // Index of this CPU.
const EBX_CPUID_SHIFT: u32 = 24; // Index of this CPU.
const ECX_EPB_SHIFT: u32 = 3; // "Energy Performance Bias" bit.
const ECX_TSC_DEADLINE_TIMER_SHIFT: u32 = 24; // TSC deadline mode of APIC timer
const ECX_HYPERVISOR_SHIFT: u32 = 31; // Flag to be set when the cpu is running on a hypervisor.
const EDX_HTT_SHIFT: u32 = 28; // Hyper Threading Enabled.

pub fn filter_cpuid(kvm: &Kvm, vcpu_id: u8, cpu_count: u8, cpuid: &mut CpuId) {
    for entry in cpuid.as_mut_slice().iter_mut() {
        match entry.function {
            0x01 => {
                // X86 hypervisor feature.
                if entry.index == 0 {
                    entry.ecx |= 1 << ECX_HYPERVISOR_SHIFT;
                }
                if kvm.check_extension(TscDeadlineTimer) {
                    entry.ecx |= 1 << ECX_TSC_DEADLINE_TIMER_SHIFT;
                }
                entry.ebx = ((vcpu_id as u32) << EBX_CPUID_SHIFT) as u32
                    | (EBX_CLFLUSH_CACHELINE << EBX_CLFLUSH_SIZE_SHIFT);
                if cpu_count > 1 {
                    entry.ebx |= (cpu_count as u32) << EBX_CPU_COUNT_SHIFT;
                    entry.edx |= 1 << EDX_HTT_SHIFT;
                }
            }
            0x06 => {
                // Clear X86 EPB feature. No frequency selection in the hypervisor.
                entry.ecx &= !(1 << ECX_EPB_SHIFT);
            }
            0x0B => {
                // EDX bits 31..0 contain x2APIC ID of current logical processor.
                entry.edx = vcpu_id as u32;
            }
            _ => (),
        }
    }

##They have written a funciton for filter_cpuid which takes input of vcpu_id and cpu_count,cpuid and for each entry in cpuid we are checking/matching the address(0x06) of x86 hypervisor feature  .
here we are using the temporary registers for data storing and memory access like edx,ebx,ecx etc.
if address matches then it will execute the clear the x86 EPB feature and no frequency selection in the hypervisor is done.In other case if address is (0x0B) then edx 32 bit contains the X2APIC ID of current logical processor as vcpu_id.

In other case of address(0x01) then as we will check the index ,if it is equal to 0,then shift the ECX_HYPERVISOR_SHIFT and in the same address if check_extension is TscDeadlineTimer accordingly as we have declared above the function is performing TSC deadline mode of APIC timer.
IF ebx register  is equals to ,we are flusing the byte which are stored in CACHELINE
In the other case of address (0x01) if cpu_count is greater than one then we are shifting the EBX_CPU_COUNT_SHIFT to the ebx register and EDX_HTT_SHIFT is shifted to edx register accordingly.

At last a bit of an artificial test is given to validate at unit test level. 
