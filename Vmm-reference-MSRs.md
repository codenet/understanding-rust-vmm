# [MSRs](../src/vm-vcpu-ref/src/x86_64/msrs.rs)

`MSR` stands for model specific registers. 
> From [reference](https://github.com/codenet/understanding-rust-vmm/blob/main/x86-debugregs.md#msrs) msrs are control registers used to measure performance , execution tracing and controlling cpu specifics.

This module provides abstractions for the following:
* Creating and populating Msrs at boot time
* Creating Msr range
* Checking if Msr is serializable
* Getting list of serializable Msrs

## Code Explanation
```rs
pub enum Error {
    /// Failed to initialize MSRS.
    CreateMsrs,
    /// Failed to get supported MSRs.
    GetSupportedMSR(kvm_ioctls::Error),
}
/// Specialized result type for operations on MSRs.
pub type Result<T> = std::result::Result<T, Error>;
````
The above piece of code defines a enum of the possible error generated from this module. The `CreateMsrs` error is returned when there is error in intializing the msrs at boot time, used in fuction `create_boot_msr_entries()` described below. While the `GetSupportedMSR` error is returned when there is error while getting list of serializable msrs, see ` supported_guest_msrs()`.

```rs
/// Base MSR for APIC
const APIC_BASE_MSR: u32 = 0x800;

/// Number of APIC MSR indexes
const APIC_MSR_INDEXES: u32 = 0x400;

/// Custom MSRs fall in the range 0x4b564d00-0x4b564dff
const MSR_KVM_WALL_CLOCK_NEW: u32 = 0x4b56_4d00;
const MSR_KVM_SYSTEM_TIME_NEW: u32 = 0x4b56_4d01;
const MSR_KVM_ASYNC_PF_EN: u32 = 0x4b56_4d02;
const MSR_KVM_STEAL_TIME: u32 = 0x4b56_4d03;
const MSR_KVM_PV_EOI_EN: u32 = 0x4b56_4d04;

/// Taken from arch/x86/include/asm/msr-index.h
const MSR_IA32_SPEC_CTRL: u32 = 0x0000_0048;
const MSR_IA32_PRED_CMD: u32 = 0x0000_0049;
```
The above piece of code defines some constant used in the the below. Here we have only defined list of the index of the custom msrs.
The list of other msrs index is given in file name '[msr_index.rs](../src/vm-vcpu-ref/src/x86_64/msrs_index.rs)' 

Now,as described above this module provides abstraction for creating and populating MSR entries at boot time, the below fuction actually does this:
```rs
pub fn create_boot_msr_entries() -> Result<Msrs>
```
This fucntion returs list of Msrs, let's try to understand that. The kvm holds the all the msrs using following structs defined in `[binding.rs](kvm-bindings-0.5.0/src/x86/bindings.rs)`
```rs
pub struct kvm_msr_entry {
    pub index: __u32,
    pub reserved: __u32,
    pub data: __u64,
}
pub struct kvm_msrs {
    pub nmsrs: __u32,
    pub pad: __u32,
    pub entries: __IncompleteArrayField<kvm_msr_entry>,
}
```
The struct `kvm_msr_entry` holds single register with member `index`,`reserved` and `data`. 
The struct `kvm_msrs` holds all the msrs using an flexible array. 
The type `Msrs` is used to provide safe access to this flexible array of msrs. 

Now let's see the implementation of function ` create_boot_msr_entries()`
```rs
 let msr_entry_default = |msr| kvm_msr_entry {
        index: msr,
        data: 0x0,
        ..Default::default()
    };

    let raw_msrs = vec![
        msr_entry_default(MSR_IA32_SYSENTER_CS),
        msr_entry_default(MSR_IA32_SYSENTER_ESP),
        msr_entry_default(MSR_IA32_SYSENTER_EIP),
        // x86_64 specific msrs, we only run on x86_64 not x86.
        msr_entry_default(MSR_STAR),
        msr_entry_default(MSR_CSTAR),
        msr_entry_default(MSR_KERNEL_GS_BASE),
        msr_entry_default(MSR_SYSCALL_MASK),
        msr_entry_default(MSR_LSTAR),
        // end of x86_64 specific code
        msr_entry_default(MSR_IA32_TSC),
        kvm_msr_entry {
            index: MSR_IA32_MISC_ENABLE,
            data: u64::from(MSR_IA32_MISC_ENABLE_FAST_STRING),
            ..Default::default()
        },
    ];

    Msrs::from_entries(&raw_msrs).map_err(|_| Error::CreateMsrs)
```
Here first a closure `msr_entry_default` is defined, which simply takes msr index and return a object of `kvm_msr_entry`. Then we define a vector of some msrs and map those to the error and return them.

Next a struct `MsrRange` is defined. This struct is used to club the msrs with consecutive index. 
```rs
struct MsrRange {
    /// Base MSR address
    base: u32,
    /// Number of MSRs
    nmsrs: u32,
}

impl MsrRange {
    /// Returns whether `msr` is contained in this MSR range.
    fn contains(&self, msr: u32) -> bool {
        self.base <= msr && msr < self.base + self.nmsrs
    }
}
```
The struct `MsrRange` has two fields `base`, denoting starting index , and `numsrs`, denoting number of consecutive msrs. Now, to create Msr ranges for all the serializable msr indices given in '[msr_index.rs](../src/vm-vcpu-ref/src/x86_64/msrs_index.rs)' , we define two macros as follows:
```rs
macro_rules! SINGLE_MSR {
    ($msr:expr) => {
        MsrRange {
            base: $msr,
            nmsrs: 1,
        }
    };
}

macro_rules! MSR_RANGE {
    ($first:expr, $count:expr) => {
        MsrRange {
            base: $first,
            nmsrs: $count,
        }
    };
}
```
As the name suggest macro `SINGLE_MSR` is used to make Msr range for single msr while MSR_RANGE is used to make msr range with multiple msrs.

Then,  we create an list of MSRS range, for all the serializable msrs indices listed in '[msr_index.rs](../src/vm-vcpu-ref/src/x86_64/msrs_index.rs)' and the custom msrs defined above.
```rs
static ALLOWED_MSR_RANGES: &[MsrRange]=&[
  SINGLE_MSR!(MSR_IA32_P5_MC_ADDR),
  ...
  MSR_RANGE!(
        // IA32_MTRR_PHYSBASE0
        0x200, 0x100
    ),
  ...
]
```

Then, below is the fuction to check whether a msrs index is present in the list `ALLOWED_MSR_RANGES`.
```rs
fn msr_should_serialize(index: u32) -> bool {
    // Denied MSRs not exported by Linux: IA32_FEATURE_CONTROL and IA32_MCG_CTL
    if index == MSR_IA32_FEATURE_CONTROL || index == MSR_IA32_MCG_CTL {
        return false;
    };
    ALLOWED_MSR_RANGES.iter().any(|range| range.contains(index))
}
```

Now, the last function is given a kvm and it returns all its serializable msrs.
```rs
pub fn supported_guest_msrs(kvm_fd: &Kvm) -> Result<Msrs>
```
This fuction takes argument of a structure that holds KVM's fd and return object of MSRs. Here is the implementation

```rs
    let mut msr_list = kvm_fd
        .get_msr_index_list()
        .map_err(Error::GetSupportedMSR)?;

    msr_list.retain(|msr_index| msr_should_serialize(*msr_index));

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

    Ok(msrs)
```
First, we etract msr list from the kvm, then we remove those msrs which are not present in `ALLOWED_MSR_RANGES` list, then return the msrs list.











