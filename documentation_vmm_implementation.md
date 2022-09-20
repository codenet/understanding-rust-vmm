# Serial Console

```rs
    fn add_serial_console(&mut self) -> Result<()>
```
It emulates the Linux Serial Console by emulating a Serial COM Port.

```rs
    let interrupt_evt = EventFdTrigger::new(libc::EFD_NONBLOCK).map_err(Error::IO)?;
    let serial = Arc::new(Mutex::new(SerialWrapper(Serial::new(
        interrupt_evt.try_clone().map_err(Error::IO)?,
        stdout(),
    ))));
```

The `Serial` device is created by passing a `Stdout` wrapper object that implements the `Write` trait. The `Serial` device is wrapped in an `Arc` and `Mutex` to make it shareable and safe to pass around. The `interrupt_evt` helps in the interaction between the driver and device by using a `Trigger` object for notifications. 

```rs
    self.kernel_cfg
        .cmdline
        .insert_str("console=ttyS0")
        .map_err(Error::Cmdline)?;

    #[cfg(target_arch = "aarch64")]
    self.kernel_cfg
        .cmdline
        .insert_str(&format!("earlycon=uart,mmio,0x{:08x}", AARCH64_MMIO_BASE))
        .map_err(Error::Cmdline)?;

    #[cfg(target_arch = "x86_64")]
    {
        let range = PioRange::new(PioAddress(0x3f8), 0x8).unwrap();
        self.device_mgr
            .lock()
            .unwrap()
            .register_pio(range, serial.clone())
            .unwrap();
    }
    #[cfg(target_arch = "aarch64")]
    {
        let range = MmioRange::new(MmioAddress(AARCH64_MMIO_BASE), 0x1000).unwrap();
        self.device_mgr
            .lock()
            .unwrap()
            .register_mmio(range, serial.clone())
            .unwrap();
    }
```

The `Serial` device is registered on the bus by using the `register_pio` or `register_mmio` methods. The `register_pio` method takes a `PioRange` object and the `Serial` device. The `register_mmio` method takes a `MmioRange` object and the `Serial` device. The `PioRange` and `MmioRange` objects are created by passing the base address and the size of the range. The `register_pio` and `register_mmio` methods are defined in the `DeviceManager` struct. `0x3f8` is the port IO address being registered for the serial device, Using it, we can print stuff on the screen.

<br>
<br>

# i8042 Device
    
```rs
    fn add_i8042_device(&mut self) -> Result<()> {
        let reset_evt = EventFdTrigger::new(libc::EFD_NONBLOCK).map_err(Error::IO)?;
        let i8042_device = Arc::new(Mutex::new(I8042Wrapper(I8042Device::new(
            reset_evt.try_clone().map_err(Error::IO)?,
        ))));
        self.vm.register_irqfd(&reset_evt, 1)?;
    }
```

It emulates a minimal i8042 controller just enough to shutdown the machine. It implements a `Trigger` object `reset_evt` which is used for `notifying` the `VMM` about the CPU `reset` event.

<br>
<br>

# rtc Device

```rs
    #[cfg(target_arch = "aarch64")]
    fn add_rtc_device(&mut self) {
        let rtc = Arc::new(Mutex::new(RtcWrapper(Rtc::new())));
        let range = MmioRange::new(MmioAddress(AARCH64_MMIO_BASE + 0x1000), 0x1000).unwrap();
    }
```

It implements a `PL031` Real Time Clock (RTC) that `emulates` a long time base counter.

<br>
<br>

# Setting up the FDT

```rs
fn setup_fdt(&mut self) -> Result<()> {
        let mem_size: u64 = self.guest_memory.iter().map(|region| region.len()).sum();
        let fdt_offset = mem_size - AARCH64_FDT_MAX_SIZE - 0x10000;
        let fdt = FdtBuilder::new()
            .with_cmdline(self.kernel_cfg.cmdline.as_str())
            .with_num_vcpus(self.num_vcpus.try_into().unwrap())
            .with_mem_size(mem_size)
            .with_serial_console(0x40000000, 0x1000)
            .with_rtc(0x40001000, 0x1000)
            .create_fdt()
            .map_err(Error::SetupFdt)?;
        fdt.write_to_mem(&self.guest_memory, fdt_offset)
            .map_err(Error::SetupFdt)?;
        Ok(())
    }
```

 This function sets up the FDT(File Descriptor Table) for the guest. The FDT is a data structure that describes the hardware configuration of the guest. It is used by the guest kernel to initialize the hardware. The FDT is written to the guest memory at the offset `fdt_offset`. The `fdt_offset` is calculated by subtracting the `AARCH64_FDT_MAX_SIZE` and `0x10000` from the `mem_size`. The `AARCH64_FDT_MAX_SIZE` is the maximum size of the FDT and `0x10000` is the offset from the end of the guest memory. The `fdt_offset` is used to write the FDT to the guest memory. The `fdt` is created by using the `FdtBuilder` struct. The `FdtBuilder` struct is used to create the FDT by adding the nodes and properties to it. The `with_cmdline` method adds the `cmdline` property to the FDT. The `with_num_vcpus` method adds the `num_vcpus` property to the FDT. The `with_mem_size` method adds the `mem_size` property to the FDT. The `with_serial_console` method adds the `serial_console` node to the FDT. The `with_rtc` method adds the `rtc` node to the FDT. The `write_to_mem` method writes the FDT to the guest memory at the offset `fdt_offset`.
