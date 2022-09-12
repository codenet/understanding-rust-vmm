Firstly in this code they strongly suggest that the 'cpuid' be derived from the supported CPUID of the host machine that is currently operating.

They have defined cpuid and used it with kvm_bindings and kvm_iocontrols .we are defining some variables with the cpuid bits in ebx,ecx  and edx registers .all most every varibale is unsigned int of 32 bit ex:  EBX_CLFLUSH_CACHELINE: u32,EBX_CPU_COUNT_SHIFT: u32 ,ECX_TSC_DEADLINE_TIMER_SHIFT: u32 etc 

pub fn filter_cpuid(kvm: &Kvm, vcpu_id: u8, cpu_count: u8, cpuid: &mut CpuId) { //this function description starts here 

They have written a funciton for filter_cpuid which takes input of vcpu_id and cpu_count,cpuid and for each entry in cpuid we are checking/matching the address(0x06) of x86 hypervisor feature  .
here we are using the temporary registers for data storing and memory access like edx,ebx,ecx etc.
if address matches then it will execute the clear the x86 EPB feature and no frequency selection in the hypervisor is done.In other case if address is (0x0B) then edx 32 bit contains the X2APIC ID of current logical processor as vcpu_id.

In other case of address(0x01) then as we will check the index ,if it is equal to 0,then shift the ECX_HYPERVISOR_SHIFT and in the same address if check_extension is TscDeadlineTimer accordingly as we have declared above the function is performing TSC deadline mode of APIC timer.
IF ebx register  is equals to ,we are flusing the byte which are stored in CACHELINE
In the other case of address (0x01) if cpu_count is greater than one then we are shifting the EBX_CPU_COUNT_SHIFT to the ebx register and EDX_HTT_SHIFT is shifted to edx register accordingly.

At last a bit of an artificial test is given to validate at unit test level. 
