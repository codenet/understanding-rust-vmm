# [[Vmm]](vmm-reference/src/vmm/src/lib.rs)

```rs 
/// A live VMM.
pub struct Vmm {
    vm: KvmVm<WrappedExitHandler>,
    kernel_cfg: KernelConfig,
    guest_memory: GuestMemoryMmap,
    // The `device_mgr` is an Arc<Mutex> so that it can be shared between
    // the Vcpu threads, and modified when new devices are added.
    device_mgr: Arc<Mutex<IoManager>>,
    // Arc<Mutex<>> because the same device (a dyn DevicePio/DeviceMmio from IoManager's
    // perspective, and a dyn MutEventSubscriber from EventManager's) is managed by the 2 entities,
    // and isn't Copy-able; so once one of them gets ownership, the other one can't anymore.
    event_mgr: EventManager<Arc<Mutex<dyn MutEventSubscriber + Send>>>,
    exit_handler: WrappedExitHandler,
    block_devices: Vec<Arc<Mutex<Block>>>,
    net_devices: Vec<Arc<Mutex<Net>>>,
    // TODO: fetch the vcpu number from the `vm` object.
    // TODO-continued: this is needed to make the arm POC work as we need to create the FDT
    // TODO-continued: after the other resources are created.
    #[cfg(target_arch = "aarch64")]
    num_vcpus: u64,
}
```

Vmm is the top-level class to interact with a live VMM. It has configurations for: 
* [[vmm-reference-kvmvm]] which internally holds states for all vCPUs.
* GuestMemoryMmap
* [[vmm-reference-iomanager]] device_mgr
  * `block_devices`
  * `net_devices`
* event_mgr
* WrappedExitHandler: We can just start the VMM with our own WrappedExitHandler
  perhaps to override the behavior of VMM.

`Vmm` has methods to `load_kernel`, `add_serial_console`, `add_i8042_device`,
`add_block_device`, and `add_net_device`. There are also other helper methods
like `check_kvm_capabilities`, `compute_kernel_load_addr`.

## **`load_kernel`** 
It just reads the kernel image from a file, writes to Guest memory, 
  prepares boot parameters and writes them to "zero page".

```rs 
/// Address of the zeropage, where Linux kernel boot parameters are written.
#[cfg(target_arch = "x86_64")]
const ZEROPG_START: u64 = 0x7000;
```

## **`run`** 
It just loads the kernel and calls `run` on KvmVm.

## **`add_serial_console`** 

It creates a serial console to take inputs from user and to print stuff on the
screen. There are only two noteworthy details 

1. `0x3f8` is the port IO address being registered for the serial device. We
have seen `0x3f8` in basic KVM implementation as well. There, we were able to
use `OUT` instruction to address `0x3f8`. This gave us an exit and we were able
to print it on the screen.

Registering `0x3f8`:
```rs
        // Put it on the bus.
        // Safe to use unwrap() because the device manager is instantiated in new(), there's no
        // default implementation, and the field is private inside the VMM struct.
        #[cfg(target_arch = "x86_64")]
        {
            let range = PioRange::new(PioAddress(0x3f8), 0x8).unwrap();
            self.device_mgr
                .lock()
                .unwrap()
                .register_pio(range, serial.clone())
                .unwrap();
        }
```

Run loop of [[vmm-reference-kvmvcpu]]
```rs
 VcpuExit::IoOut(addr, data) => {
    if (0x3f8..(0x3f8 + 8)).contains(&addr) {
        // Write at the serial port.
        if self.device_mgr
            .lock()
            .unwrap()
            .pio_write(PioAddress(addr), data)
            .is_err()
        {
            debug!("Failed to write to serial port");
        }
    }
```
This `pio_write` shall eventually write to stdout.

2. The code maps an event file to interrupt number 4 of the guest. IRQ 4 is the
"serial port 1".

```rs 
let interrupt_evt = EventFdTrigger::new(libc::EFD_NONBLOCK).map_err(Error::IO)?;
let serial = Arc::new(Mutex::new(SerialWrapper(Serial::new(
    interrupt_evt.try_clone().map_err(Error::IO)?,
    stdout(),
))));

// Register its interrupt fd with KVM. IRQ line 4 is typically used for serial port 1.
// See more IRQ assignments & info: https://tldp.org/HOWTO/Serial-HOWTO-8.html
self.vm.register_irqfd(&interrupt_evt, 4)?;
```

Serial device just triggers the interrupt in guest by triggering on eventfd.
```rs
impl<T: Trigger, EV: SerialEvents, W: Write> Serial<T, EV, W> {
    /// Creates a new `Serial` instance which writes the guest's output to
    /// `out`, uses `trigger` object to notify the driver about new
    /// events, and invokes the `serial_evts` implementation of `SerialEvents`
    /// during operation.
    ///
    /// # Arguments
    /// * `trigger` - The `Trigger` object that will be used to notify the driver
    ///               about events.
    /// * `serial_evts` - The `SerialEvents` implementation used to track the occurrence
    ///                   of significant events in the serial operation logic.
    /// * `out` - An object for writing guest's output to. In case the output
    ///           is not of interest,
    ///           [std::io::Sink](https://doc.rust-lang.org/std/io/struct.Sink.html)
    ///           can be used here.
    pub fn with_events(trigger: T, serial_evts: EV, out: W) -> Self { .. }

    fn trigger_interrupt(&mut self) -> Result<(), T::E> {
        self.interrupt_evt.trigger()
    }
```

From https://tldp.org/HOWTO/Serial-HOWTO-8.html
```
Standard IRQ assignments:

        IRQ  0    Timer channel 0 (May mean "no interrupt".  See below.)
        IRQ  1    Keyboard
        IRQ  2    Cascade for controller 2
        IRQ  3    Serial port 2
        IRQ  4    Serial port 1
        IRQ  5    Parallel port 2, Sound card
        IRQ  6    Floppy diskette
        IRQ  7    Parallel port 1
        IRQ  8    Real-time clock
        IRQ  9    Redirected to IRQ2
        IRQ 10    not assigned
        IRQ 11    not assigned
        IRQ 12    not assigned
        IRQ 13    Math co-processor
        IRQ 14    Hard disk controller 1
        IRQ 15    Hard disk controller 2
```

* **`add_i8042_device`** is similar. Mapping eventfd to an interrupt line.

* **`add_net_device`** and **`add_block_device`** would require understanding
  [virtio](https://github.com/rust-vmm/vm-virtio) first.
