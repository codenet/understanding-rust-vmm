Here is a brief description of how msrs.rs works from "vmm-reference/src/vm-vcpu-ref/src/x86_64/msrs.rs"
MSRS is used to store a half done process' state and continue with some other process and then come back to it, restore it and continue wth the work.

struct MsrRange {
    /// Base MSR address
    base: u32,
    /// Number of MSRs
    nmsrs: u32,
}
This part of the code describes msr range, what address the registers start from and how many there are. We can find out where the msrs end from this.

impl MsrRange {
    /// Returns whether `msr` is contained in this MSR range.
    fn contains(&self, msr: u32) -> bool {
        self.base <= msr && msr < self.base + self.nmsrs
    }
}
This part is used to check whether the msr address we are trying to access is actually an msr(in the address range where msrs are)

macro_rules! SINGLE_MSR {
    ($msr:expr) => {
        MsrRange {
            base: $msr,
            nmsrs: 1,
        }
    };
}
This is used to define just a single register at a particular address. Therefore nmsrs is 1.

macro_rules! MSR_RANGE {
    ($first:expr, $count:expr) => {
        MsrRange {
            base: $first,
            nmsrs: $count,
        }
    };
}
This part uses the structure above to define a msrs range with fixed address and fixed number of registers.

static ALLOWED_MSR_RANGES: &[MsrRange] = &[......]
This part defines a list of msrs registers one by one. It contains many functions of the from
SINGLE_MSR!(MSR_IA32_P5_MC_ADDR)
The arguments are addresses and the function purpose is defined above. It creates a single msrs at the given address.

fn msr_should_serialize(index: u32) -> bool {
    // Denied MSRs not exported by Linux: IA32_FEATURE_CONTROL and IA32_MCG_CTL
    if index == MSR_IA32_FEATURE_CONTROL || index == MSR_IA32_MCG_CTL {
        return false;
    };
    ALLOWED_MSR_RANGES.iter().any(|range| range.contains(index))
}
This part takes an indexof msrs present and says if it can be serialized. 2 registers are explicitly removed from this list as linux does not support its serialization.

fn test_create_boot_msrs() {
    // This is a rather dummy test to check that creating the MSRs that we
    // need for booting can be initialized into the `Msrs` type without
    // yielding any error.
    let kvm = Kvm::new().unwrap();
    let vm = kvm.create_vm().unwrap();
    let vcpu = vm.create_vcpu(0).unwrap();

    let boot_msrs = create_boot_msr_entries().unwrap();
    assert!(vcpu.set_msrs(&boot_msrs).is_ok())
}
This part tests if all the msrs can be started during boot without errors.

let mut msrs =
    Msrs::new(msr_list.as_fam_struct_ref().nmsrs as usize).map_err(|_| Error::CreateMsrs)?;
let indices = msr_list.as_slice();
let msr_entries = msrs.as_mut_slice();
// We created the msrs from the msr_list. If the size is not the same,
// there is a fatal programming error.
assert_eq!(indices.len(), msr_entries.len());
for (pos, index) in indices.iter().enumerate() {
    msr_entries[pos].index = *index;
}
This part creates all the msrs from the list. After creating it gives everyone their proper index and checks if the number of msrs created is the same as the list. If not it throws error.