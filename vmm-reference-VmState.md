# Saving and Restoring State of VM
__File Path__ - vmm-reference/src/vm-vcpu/src/vm.rs

__Application__ - VM Migration

### VmState
```rust
pub struct VmState {
    pub pitstate: kvm_pit_state2,
    pub clock: kvm_clock_data,
    pub pic_master: kvm_irqchip,
    pub pic_slave: kvm_irqchip,
    pub ioapic: kvm_irqchip,
    pub config: VmConfig,
    pub vcpus_state: Vec<VcpuState>,
}
```
- __pitstate__    - Wrapper for struct `kvm_pit_state2` containing PIT(Programmable Interrups) configuration provided by [KVM API](https://www.kernel.org/doc/Documentation/virtual/kvm/api.txt).
- __clock__       - Wrapper for struct `kvm_clock_data` containing kvmclock information provided by [KVM API](https://www.kernel.org/doc/Documentation/virtual/kvm/api.txt).
- __pic_master__  - Wrapper for Kvm interrupt controller configuration for chip id `KVM_IRQCHIP_PIC_MASTER` provided by [KVM API](https://www.kernel.org/doc/Documentation/virtual/kvm/api.txt).
- __pic_slave__   - Wrapper for Kvm interrupt controller configuration for chip id `KVM_IRQCHIP_PIC_SLAVE` provided by [KVM API](https://www.kernel.org/doc/Documentation/virtual/kvm/api.txt).
- __ioapic__      - Wrapper for Kvm interrupt controller configuration for chip id `KVM_IRQCHIP_PIC_IOAPIC` provided by [KVM API](https://www.kernel.org/doc/Documentation/virtual/kvm/api.txt).
- __config__      - Configuration of VM stored in `VmConfig`.
- __vcpus_state__ - Vector of `VcpuState`'s which stores configuration for all Vcpus associated with vm.


### Configuring kernel interrupt controller
```rust
fn setup_irq_controller(&mut self) -> Result<()>
```
- __create_irq_chip()__ - Creates in-kernel interrupt controller by ioctl call to `KVM_CREATE_IRQCHIP` provided by KVM API.
- __create_pit2()__ - Creates in-kernel device model for i8254 PIT by ioctl call to `KVM_CREATE_PIT2` provided by KVM API with `KVM_PIT_SPEAKER_DUMMY` flag to inject timer interrupt directly to per-VM kernel thread.
>__Note:__ On x86_64 architecture, create_pit2() should only be called after setting up interrupt controller via create_irq_chip().

### Retrieving the state of a `paused` VM
```rust
pub fn save_state(&mut self) -> Result<VmState>
```
- __get_pit2()__ - Retrieves state of in-kernel PIT model by ioctl call to `KVM_GET_PIT2` provided by [KVM API](https://www.kernel.org/doc/Documentation/virtual/kvm/api.txt).
- __get_clock()__ - Gets the current timestamp of kvmclock as seen by the current guest by ioctl call to `KVM_GET_CLOCK` provided by [KVM API](https://www.kernel.org/doc/Documentation/virtual/kvm/api.txt).
- __get_irqchip()__ - Reads the state of kernel interrupt controller created with `create_irq_chip()` by ioctl call to `KVM_GET_IRQCHIP` provided by [KVM API](https://www.kernel.org/doc/Documentation/virtual/kvm/api.txt).
- __KvmVcpu.save_state()__ - Returns `VcpuState` which contains current state of paused Vcpu.

### Creating a VM from previously saved state
```rust
pub fn from_state<M: GuestMemory>(
    kvm: &Kvm,
    state: VmState,
    guest_memory: &M,
    exit_handler: EH,
    bus: Arc<Mutex<IoManager>>,
) -> Result<Self>
```
- __create_vm()__ - Creating VM fd by using `KVM_CREATE_VM` ioctl call on kvm fd provided by [KVM API](https://www.kernel.org/doc/Documentation/virtual/kvm/api.txt).
- __setup_irq_controller()__ - Setting up in-kernel interrupt controller.
- __set_state()__ - Restoring previously saved state of in-kernel interrupt controller.
- __create_vcpus_from_state()__ - Create Vcpus associated with this VM using previously saved state.


### Setting up state of this VM
```rust
fn set_state(&mut self, state: VmState) -> Result<()>
```
Setting up state of VM based on stored configuration in supplied `state` object.
- __set_pit2()__ - Set the state of in-kernel PIT by ioctl call to `KVM_SET_PIT2` provided by [KVM API](https://www.kernel.org/doc/Documentation/virtual/kvm/api.txt).
- __set_irqchip()__ - Set state of kernel interrupt controller by ioctl call to `KVM_SET_IRQCHIP` provided by [KVM API](https://www.kernel.org/doc/Documentation/virtual/kvm/api.txt).

--- 

### APIC - Advanced Programmable Interrupt Controller

> __PIC__ - Integrated circuit helps microprocesser handle interrupt requests (IRQs) from multiple sources which may happen simultaneously. It helps in priortizing IRQs so that CPU switches to most appropriate interrupt handler (ISR) after the PIC assesses the IRQs relative priorities.

Intel's APIC is a family of interrupt controllers, more advanced than 8259 PIC enabling construction of microprocessor systems.

It can handle large amount of interrupts, to allow each of these to be programatically routed to a specific set of available CPUs, to support inter CPU communication and to remove need of large no. of devices sharing a single interrupt line.

Uses a combination of local APIC built into each CPU, and one I/O APIC associated with each peripheral bus in the system. Uses system bus for communication between APIC components.

When hardware device generates interrupt, it is detected by I/O APIC it is connected to and then routed across system APIC bus to a particular CPU. OS knows which I/O APIC is connected to which device and to which interrupt line within that device because of combination of information sources.

#### References
- [Multiprocessor Specification](https://pdos.csail.mit.edu/6.828/2008/readings/ia32/MPspec.pdf)
- [APIC](https://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller)


    
