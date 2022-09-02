# [IoManager](vm-device-0.1.0/src/device_manager.rs)

```rs
/// System IO manager serving for all devices management and VM exit handling.
#[derive(Default)]
pub struct IoManager {
    // Range mapping for VM exit pio operations.
    pio_bus: PioBus<Arc<dyn DevicePio + Send + Sync>>,
    // Range mapping for VM exit mmio operations.
    mmio_bus: MmioBus<Arc<dyn DeviceMmio + Send + Sync>>,
}
```

`IoManager` is just a wrapper over `PioBus` and `MmioBus`. It has methods to
register new devices: 

```rs
/// Register a new MMIO device with its allocated resources.
    /// VMM is responsible for providing the allocated resources to virtual device.
    ///
    /// # Arguments
    ///
    /// * `device`: device instance object to be registered
    /// * `resources`: resources that this device owns, might include
    ///                port I/O and memory-mapped I/O ranges, irq number, etc.
    pub fn register_mmio_resources(
        &mut self,
        device: Arc<dyn DeviceMmio + Send + Sync>,
        resources: &[Resource],
    ) -> Result<(), Error> {
```

Similarly there is `register_pio_resources`. A `Resource` is what is owned by
the device: 

```rs
pub enum Resource {
    /// IO Port address range.
    PioAddressRange { base: u16, size: u16 },
    /// Memory Mapped IO address range.
    MmioAddressRange { base: u64, size: u64 },
    /// Legacy IRQ number.
    LegacyIrq(u32),
    /// Message Signaled Interrupt
    MsiIrq {
        ty: MsiIrqType,
        base: u32,
        size: u32,
    },
    /// Network Interface Card MAC address.
    MacAddresss(String),
    /// KVM memslot index.
    KvmMemSlot(u32),
}
```