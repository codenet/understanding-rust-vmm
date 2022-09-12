# Flatenned Device Tree
## Description
VMM keeps track of the virtio devices  in a data structure called Flatenned Device Tree. FDT as the name suggests is a Tree where each node is a device or collection of devices. There can be different types of nodes in the FDT which can be identified using the attributes of the node. Each node contains a key-value pair mapping which determine the type of the node and configuration of that node.\
This arrangement abstracts out the hardware specifications from the code and the makes the code shareable while making the hardware specifications more accesible and uniform. Arrangement of the hardware resources in the form of the tree also helps in easier reolution of resource ownership.

## Implementation
The implementation of the FDT for arm64 architecture in `vmm-reference` is done using rust crate `vm_fdt` and `vm_memory` so are the following import statements.
```rs
pub use vm_fdt::{Error as FdtError, FdtWriter};
use vm_memory::{guest_memory::Error as GuestMemoryError, Bytes, GuestAddress, GuestMemory};
```
Then some constants are set up which include the max size of the FDT, start of DRAM in physical address space, base for MMIO and AXI ports and set up the virtual `Generic Interrupt Controller`.\
Then the Error handler is set up with default error handler of both vm_memory and vm_fdt crates in addition to a error handler for cases of missing required configuration.
```rs
#[derive(Debug)]
pub enum Error {
    Fdt(FdtError),
    Memory(GuestMemoryError),
    MissingRequiredConfig(String),
}
```
The structure of the FdtBuilder is defined as 
```rs
#[derive(Default)]
pub struct FdtBuilder {
    cmdline: Option<String>,
    mem_size: Option<u64>,
    num_vcpus: Option<u32>,
    serial_console: Option<(u64, u64)>,
    rtc: Option<(u64, u64)>,
    virtio_devices: Vec<DeviceInfo>,
}
```
where each device in virtio_devices vector is in the form of DeviceInfo struct defined as follows
```rs
struct DeviceInfo {
    addr: u64,
    size: u64,
    irq: u32,
}
```

FdtBuilder implementation provides with methods to create a new node in te FDT and then configure that node. It also provides methods to query number of virtio devices in that node and add or a device to them.

Fdt struct is a vector of 8-bit unsigned integers
```rs
pub struct Fdt {
    fdt_blob: Vec<u8>,
}
```
while implementaion fpr Fdt includes method which writes this fdt_blob as slice in the guest memory.

```rs
impl Fdt {
    pub fn write_to_mem<T: GuestMemory>(&self, guest_mem: &T, fdt_load_offset: u64) -> Result<()> {
        let fdt_address = GuestAddress(AARCH64_PHYS_MEM_START + fdt_load_offset);
        guest_mem.write_slice(self.fdt_blob.as_slice(), fdt_address)?;
        Ok(())
    }
}
```

Further we are provided with functions to create all the reuired types of nodes with thier neccessory configurations. THe supported types of nodes include chosen node, memory node, cpu node, gic (Generic Interrupt Controller) node, psci (Power State Coordination Interface) node, serial node, timer node, pmu (Performance Monitor Unit) node, rtc (Real Time Clock) node and virtio node.\
For example to create a chosen node, which does not represent a device but serves in passing arguments between firmware and operating system, is createdd as follows.
```rs
fn create_chosen_node(fdt: &mut FdtWriter, cmdline: &str) -> Result<()> {
    let chosen_node = fdt.begin_node("chosen")?;
    fdt.property_string("bootargs", cmdline)?;
    fdt.end_node(chosen_node)?;

    Ok(())
}
```

## Unit Testing
Testing includes creating a new FdtBuilder object and then check creating fdt without specifying one of the configurations required. In the case when that left out configuration in neccessory we should get MissingRequiredConfig error. \
The addition of new virtio device in a node is also tested. 
