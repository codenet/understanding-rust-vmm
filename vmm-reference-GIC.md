# Generic Interupt Controller

## kvm-bindings
Rust FFI generated using
[bindgen](https://crates.io/crates/bindgen) are used for bindings KVM. It currently has support for the
following target architectures:
- x86
- x86_64
- arm
- arm64

The bindings exported by this crate are statically generated using header files associated with a specific kernel version, and are not automatically synced with the kernel version running on a particular host. 

In this case kvm_bindings is using some defined structure (like "kvm_create_device") and public CONSTANT (like "KVM_DEV_ARM_VGIC_GRP_ADDR") which is used for bindings.



```rust
  use kvm_bindings::{
      kvm_create_device, kvm_device_attr, kvm_device_type_KVM_DEV_TYPE_ARM_VGIC_V2,
      kvm_device_type_KVM_DEV_TYPE_ARM_VGIC_V3, KVM_DEV_ARM_VGIC_CTRL_INIT,
      KVM_DEV_ARM_VGIC_GRP_ADDR, KVM_DEV_ARM_VGIC_GRP_CTRL, KVM_DEV_ARM_VGIC_GRP_NR_IRQS,
      KVM_VGIC_V2_ADDR_TYPE_CPU, KVM_VGIC_V2_ADDR_TYPE_DIST, KVM_VGIC_V3_ADDR_TYPE_DIST,
      KVM_VGIC_V3_ADDR_TYPE_REDIST,
  };
```

## kvm-ioctls

The kvm-ioctls crate provides safe wrappers over the
[KVM API](https://www.kernel.org/doc/Documentation/virtual/kvm/api.txt), a set
of ioctls used for creating and configuring Virtual Machines (VMs) on Linux.
The ioctls are accessible through four structures:
- `Kvm` - wrappers over system ioctls
- `VmFd` - wrappers over VM ioctls
- `VcpuFd` - wrappers over vCPU ioctls
- `DeviceFd` - wrappers over device ioctls



### Supported Platforms

The kvm-ioctls can be used on x86_64 and aarch64. Right now the aarch64 support
is considered and hence only "DeviceFd" and "VmFd" is used here.


```rust
use kvm_ioctls::{DeviceFd, VmFd};


/// The minimum number of interrupts supported by the GIC.
// This is the minimum number of SPI interrupts aligned to 32 + 32 for the
// PPI (16) and GSI (16).
```

Some constant related to "GIC Architecture" is defined below.


```rust

pub const MIN_NR_IRQS: u32 = 64;

const AARCH64_AXI_BASE: u64 = 0x40000000;

// These constants indicate the address space used by the ARM vGIC.
// TODO: find a way to export the registers base & lengths as part of the GIC.
const AARCH64_GIC_DIST_SIZE: u64 = 0x10000;
const AARCH64_GIC_CPUI_SIZE: u64 = 0x20000;

// These constants indicate the placement of the GIC registers in the physical
// address space.
const AARCH64_GIC_DIST_BASE: u64 = AARCH64_AXI_BASE - AARCH64_GIC_DIST_SIZE;
const AARCH64_GIC_CPUI_BASE: u64 = AARCH64_GIC_DIST_BASE - AARCH64_GIC_CPUI_SIZE;
const AARCH64_GIC_REDIST_SIZE: u64 = 0x20000;

```


## Basic features and working of GIC (Generic Interrupt Controller)

> * It is an interrupt controller for aarch64. 
> * The GIC Architecture has a mechanism that allows a Hypervisor to programmatically generate Virtual Interrupts (vIRQs) that are sent to the Guest OS running at EL1.
> * This achieved by writing to the List Registers (GICH_LR<n> or ICH_LR<n>_EL2) that are part of the GIC Virtual Interface Control.



```rust 

/// High level wrapper for creating and managing the GIC device.
#[derive(Debug)]
pub struct Gic {
    version: GicVersion,
    device_fd: DeviceFd,
    num_irqs: u32,
    num_cpus: u8,
}

```

We can see from above Structure definition of Gic that there are "four attributes" let explain in brief about them one by one.

- GicVersion : We can see that V2 and V3 version is used for GicVersion and there value we are accesing from kvm_bindings constant which we have already defined above.


``` rust
/// Specifies the version of the GIC device
#[derive(Debug, PartialEq, Clone, Copy)]
pub enum GicVersion {
    // The following casts are safe because the device type has small values (i.e. 5 and 7)
    /// GICv2 identifier.
    V2 = kvm_device_type_KVM_DEV_TYPE_ARM_VGIC_V2 as isize,
    /// GICv3 identifier.
    V3 = kvm_device_type_KVM_DEV_TYPE_ARM_VGIC_V3 as isize,
}

```


- DeviceFd : It is a wrappers over "device iocatls" and it is also defined above in kvm_ioctls.

- num_irqs : Number of IRQs that this GIC supports.

- num_cups : Number of CPUs that this GIC supports.



Let's see default configuration of GIC.


```rust
impl Default for GicConfig {
    fn default() -> Self {
        GicConfig {
            num_irqs: MIN_NR_IRQS,
            num_cpus: 0,
            version: None,
        }
    }
}
```


Configuration details of GIC.

```rust
/// Configuration of the GIC device.
///
/// # Example
/// ```rust
/// use vm_vcpu_ref::aarch64::interrupts::{GicConfig, GicVersion};
///
/// // Create a default configuration for GICv2. We only care about setting the version.
/// let config = GicConfig {
///     version: Some(GicVersion::V2),
///     ..Default::default()
/// };
///
/// // Create a default configuration for GICv3. We also need to setup the cpu_num.
/// let config = GicConfig {
///     version: Some(GicVersion::V3),
///     num_cpus: 1,
///     ..Default::default()
/// };
///
/// // Create a default configuration for the 4 cpus. When creating the `Gic` from this
/// // configuration the GIC version will be selected depending on what's supported on the host.
/// let config = GicConfig {
///     num_cpus: 4,
///     ..Default::default()
/// };
/// ```
#[derive(Debug, Clone)]
pub struct GicConfig {
    /// Number of IRQs that this GIC supports. The IRQ number can be a number divisible by 32 in
    /// the interval [[MIN_NR_IRQS], 1024].
    pub num_irqs: u32,
    /// Number of CPUs that this GIC supports. This is not used when configuring
    /// a GICv2.
    pub num_cpus: u8,
    /// Version of the GIC. When no version is specified, we try to create a GICv3, and fallback
    /// to GICv2 in case of failure.
    pub version: Option<GicVersion>,
}
```


## GIC Architecture 

> * The GIC Architecture allows for maintaining individual states for over a 1000 physical interrupts if LPIs are not used. If LPIs are present this number increases dramatically.
> * The GIC architecture allows for up to 16 list registers and typically implementations choose to have less, with four being a common number. 
> *  Typically fewer interrupts are live at any one time than can be represented as virtual interrupts in the list registers but a scenario cannot be ruled out where there are more live physical interrupts. This can be managed by the Hypervisor maintaining the state of any extra virtual interrupts in memory and swapping their state in and out of the list registers when the Guest OS needs to interact with them through the Virtual CPU interface.
>  *  The GIC allows the configuration of a number of Maintenance Interrupts that can be signalled to the Hypervisor when certain conditions are met in the List Registers to allow the Hypervisor to manage the list.
> * Maintenance Interrupt are enabled using the Interrupt Controller Hyp Control Register (ICH_HCR_EL2 or ICH_HCR).

## Some Important methods implemented for GIC

```rust
impl Gic {
    /// Create a new GIC based on the passed configuration.
    ///
    /// # Arguments
    /// * `config`: The GIC configuration. More details are available in the
    ///             [`GicConfig`] definition.
    /// * `vm_fd`: A reference to the KVM specific file descriptor. This is used
    ///            for creating and configuring the GIC in KVM.
    /// # Example
    /// ```rust
    /// use kvm_ioctls::Kvm;
    /// use vm_vcpu_ref::aarch64::interrupts::{Gic, GicConfig};
    ///
    /// let kvm = Kvm::new().unwrap();
    /// let vm = kvm.create_vm().unwrap();
    /// let _vcpu = vm.create_vcpu(0).unwrap();
    ///
    /// let gic_config = GicConfig {
    ///     num_cpus: 1,
    ///     ..Default::default()
    /// };
    ///
    /// let gic = Gic::new(gic_config, &vm).unwrap();
    /// let device_fd = gic.device_fd();
    /// ```
    pub fn new(config: GicConfig, vm_fd: &VmFd) -> Result<Gic> {
        let (version, device_fd) = match config.version {
            Some(version) => (version, Gic::create_device(vm_fd, version)?),
            None => {
                // First try to create a GICv3, if that does not work, update the version
                // and try to create a V2 instead.
                let mut version = GicVersion::V3;
                let device_fd = Gic::create_device(vm_fd, GicVersion::V3).or_else(|_| {
                    version = GicVersion::V2;
                    Gic::create_device(vm_fd, GicVersion::V2)
                })?;
                (version, device_fd)
            }
        };

        let mut gic = Gic {
            num_irqs: config.num_irqs,
            num_cpus: config.num_cpus,
            version,
            device_fd,
        };

        gic.configure_device()?;
        Ok(gic)
    }


```

- new : This method used to Create a new GIC based on the passed "config" and "vm_fd" parameters . Also in this method a structure of Gic is created by of "gic" whose attributes is decided by  "config" parameter attribute's value. 

- new method return value is "Result<Gic>" which is defined as follows :

```rust 
/// Specialized result type for operations on the GIC.
pub type Result<T> = std::result::Result<T, Error>;
```

Some other methods

```rust 
    // Helper function that sets the required attributes for the device.
    fn configure_device(&mut self) -> Result<()> {
        match self.version {
            GicVersion::V2 => {
                self.set_distr_attr(KVM_VGIC_V2_ADDR_TYPE_DIST as u64)?;
                self.set_cpu_attr()?;
            }
            GicVersion::V3 => {
                self.set_redist_attr()?;
                self.set_distr_attr(KVM_VGIC_V3_ADDR_TYPE_DIST as u64)?;
            }
        }
        self.set_interrupts_attr()?;
        self.finalize_gic()
    }

```

- This above method configure_device sets the attributes for device depending of version assigned to it i.e V2 or V3.


```rust

    // Create the device FD corresponding to the GIC version specified as parameter.
    fn create_device(vm_fd: &VmFd, version: GicVersion) -> Result<DeviceFd> {
        let mut create_device_attr = kvm_create_device {
            type_: version as u32,
            fd: 0,
            flags: 0,
        };
        vm_fd
            .create_device(&mut create_device_attr)
            .map_err(Error::CreateDevice)
    }
```



```rust 
    // Helper function for setting the _ADDR_TYPE_DIST. The attribute type depends on the
    // type of GIC, so we are passing it as a parameter so that we don't need to do
    // a match on the version again.
    fn set_distr_attr(&mut self, attr_type: u64) -> Result<()> {
        let dist_if_addr: u64 = AARCH64_GIC_DIST_BASE;
        let raw_dist_if_addr = &dist_if_addr as *const u64;
        let dist_attr = kvm_device_attr {
            group: KVM_DEV_ARM_VGIC_GRP_ADDR,
            addr: raw_dist_if_addr as u64,
            attr: attr_type,
            flags: 0,
        };
        self.device_fd
            .set_device_attr(&dist_attr)
            .map_err(|e| Error::SetAttr("dist", e))
    }

    // Helper function for setting the `KVM_VGIC_V2_ADDR_TYPE_CPU`. This can only be used with
    // VGIC 2. We're not doing a check here because KVM will return an error if used with V3,
    // and this function is private, so calling it with a v3 GIC can only happen if we have
    // a programming error.
    fn set_cpu_attr(&mut self) -> Result<()> {
        let cpu_if_addr: u64 = AARCH64_GIC_CPUI_BASE;
        let raw_cpu_if_addr = &cpu_if_addr as *const u64;
        let cpu_if_attr = kvm_device_attr {
            group: KVM_DEV_ARM_VGIC_GRP_ADDR,
            attr: KVM_VGIC_V2_ADDR_TYPE_CPU as u64,
            addr: raw_cpu_if_addr as u64,
            flags: 0,
        };

        self.device_fd
            .set_device_attr(&cpu_if_attr)
            .map_err(|e| Error::SetAttr("cpu", e))
    }

    fn set_redist_attr(&mut self) -> Result<()> {
        // The following arithmetic operations are safe because the `num_cpus` can be maximum
        // u8::MAX, which multiplied by the `AARCH64_GIC_REDIST_SIZE` results in a number that is
        // an order of maginitude smaller that `AARCH64_GIC_DIST_BASE`.
        let redist_addr: u64 =
            AARCH64_GIC_DIST_BASE - (AARCH64_GIC_REDIST_SIZE * self.num_cpus as u64);
        let raw_redist_addr = &redist_addr as *const u64;
        let redist_attr = kvm_device_attr {
            group: KVM_DEV_ARM_VGIC_GRP_ADDR,
            attr: KVM_VGIC_V3_ADDR_TYPE_REDIST as u64,
            addr: raw_redist_addr as u64,
            flags: 0,
        };

        self.device_fd
            .set_device_attr(&redist_attr)
            .map_err(|e| Error::SetAttr("redist", e))
    }

```

```rust
    fn set_interrupts_attr(&mut self) -> Result<()> {
        let nr_irqs_ptr = &self.num_irqs as *const u32;
        let nr_irqs_attr = kvm_device_attr {
            group: KVM_DEV_ARM_VGIC_GRP_NR_IRQS,
            addr: nr_irqs_ptr as u64,
            ..Default::default()
        };

        self.device_fd
            .set_device_attr(&nr_irqs_attr)
            .map_err(|e| Error::SetAttr("irq", e))
    }

```

- Here GCI attribute is value is set as per defined/default configuration and structure of GCI as well as device.

```rust

    fn finalize_gic(&mut self) -> Result<()> {
        let init_gic_attr = kvm_device_attr {
            group: KVM_DEV_ARM_VGIC_GRP_CTRL,
            attr: KVM_DEV_ARM_VGIC_CTRL_INIT as u64,
            ..Default::default()
        };

        self.device_fd
            .set_device_attr(&init_gic_attr)
            .map_err(|e| Error::SetAttr("finalize", e))
    }

```

- This will return specialized result type for operation on the   finalize Generic Interrupt Controller (GIC) 


```rust

    /// Return the `DeviceFd` associated with this GIC.
    pub fn device_fd(&self) -> &DeviceFd {
        &self.device_fd
    }

    /// Returns the version of this GIC.
    pub fn version(&self) -> GicVersion {
        self.version
    }

```

## Error handlinng in GIC

```rust

/// Errors associated with operations related to the GIC.
#[derive(Debug, PartialEq, thiserror::Error)]
pub enum Error {
    /// Error calling into KVM ioctl.
    #[error("Error calling into KVM ioctl: {0}")]
    Kvm(kvm_ioctls::Error),
    /// Error creating the GIC device.
    #[error("Error creating the GIC device: {0}")]
    CreateDevice(kvm_ioctls::Error),
    /// Error setting an attribute for the GIC device.
    #[error("Error setting an attribute ({0}) for the GIC device: {1}")]
    SetAttr(&'static str, kvm_ioctls::Error),
}

impl From<kvm_ioctls::Error> for Error {
    fn from(inner: kvm_ioctls::Error) -> Self {
        Error::Kvm(inner)
    }
}

/// Specialized result type for operations on the GIC.

```


### Conclusion :  This README file go in details about definition(including GCI basic architecture) and implementation of GIC (Generic Interrupt Handler) along with Error Handling. 



