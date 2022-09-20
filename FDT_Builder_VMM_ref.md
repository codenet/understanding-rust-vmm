# [FDT Builder](./../src/arch/src/lib.rs)


Enumerating the errors that can be encountered while creating the FDT 
```rs
#[derive(Debug)]
pub enum Error {
    Fdt(FdtError),
    Memory(GuestMemoryError),
    MissingRequiredConfig(String),
}
```

This structure represents the configuration for the FDT that we're going to use. cmdline is the kernel command line, which is a string, and mem_size is the size of the memory that we're going to pass to the kernel in bytes , num_vcpus is the number of vcpus that we're going to pass to the kernel, rtc is the RTC(real time clock) (base and address) device , serial is the serial port device (base and address) 
```rs
pub struct FdtBuilder<'a> {
    cmdline: Option<&'a str>,
    mem_size: Option<u64>,
    num_vcpus: Option<u32>,
    serial_console: Option<(u64, u64)>,
    rtc: Option<(u64, u64)>,
}
```

Impelmenting the From trait for the Error enum 
```rs
impl From<FdtError> for Error {
    fn from(inner: FdtError) -> Self {
        Error::Fdt(inner)
    }
}

impl From<GuestMemoryError> for Error {
    fn from(inner: GuestMemoryError) -> Self {
        Error::Memory(inner)
    }
}
```

This is the implementation of the FdtBuilder structure.(In comments explained what the function is doing )

```rs
impl<'a> FdtBuilder<'a>{
    //This is the constructor for the FdtBuilder structure
    pub fn new() -> Self {
        FdtBuilder::default()
    }
    // This function is used to set the command line for the FDT 
    pub fn with_cmdline(&mut self, cmdline: &'a str) -> &mut Self {
    }
    // This function is used to set the memory size for the FDT
    pub fn with_mem_size(&mut self, mem_size: u64) -> &mut Self {
    }
    // This function is used to set the number of vcpus for the FDT
    pub fn with_num_vcpus(&mut self, num_vcpus: u32) -> &mut Self {
    }
    // This function is used to set the serial console for the FDT
    pub fn with_serial_console(&mut self, addr: u64, size: u64) -> &mut Self {
    }
    // This function is used to set the RTC device for the FDT 
    pub fn with_rtc(&mut self, addr: u64, size: u64) -> &mut Self {
    }
    // This function is used to create the FDT takes the fdtbuilder structure as a parameter and returns a Result type
    pub fn create_fdt(&self) -> Result<Fdt> {
    }
}
```

This function is used to write psci node and its properties to the FDT. The psci node contains the psci compatible and the psci method.
```rs
fn create_psci_node(fdt: &mut FdtWriter) -> Result<()> {
    let compatible = "arm,psci-0.2";
    let psci_node = fdt.begin_node("psci")?;
    fdt.property_string("compatible", compatible)?;
    // Two methods available: hvc and smc.
    // As per documentation, PSCI calls between a guest and hypervisor may use the HVC conduit instead of SMC.
    // So, since we are using kvm, we need to use hvc.
    fdt.property_string("method", "hvc")?;
    fdt.end_node(psci_node)?;

    Ok(())
}
```
This function is used to write the serial node and its properties to the FDT. The serial node contains the serial interrupt type and the serial interrupt phandle.
```rs
fn create_serial_node(fdt: &mut FdtWriter, addr: u64, size: u64) -> Result<()> {
    let serial_node = fdt.begin_node(&format!("uart@{:x}", addr))?;
    fdt.property_string("compatible", "ns16550a")?;
    let serial_reg_prop = [addr, size];
    fdt.property_array_u64("reg", &serial_reg_prop)?;

    // fdt.property_u32("clock-frequency", AARCH64_SERIAL_SPEED)?;
    const CLK_PHANDLE: u32 = 24;
    fdt.property_u32("clocks", CLK_PHANDLE)?;
    fdt.property_string("clock-names", "apb_pclk")?;
    let irq = [GIC_FDT_IRQ_TYPE_SPI, 4, IRQ_TYPE_EDGE_RISING];
    fdt.property_array_u32("interrupts", &irq)?;
    fdt.end_node(serial_node)?;
    Ok(())
}
```
This function is used to write the timer node and its properties to the FDT. The timer node contains the timer interrupt type and the timer interrupt phandle. (some explanation provided in the comments )

```rs
fn create_timer_node(fdt: &mut FdtWriter, num_cpus: u32) -> Result<()> {
    // These are fixed interrupt numbers for the timer device.
    let irqs = [13, 14, 11, 10];
    let compatible = "arm,armv8-timer";
    let cpu_mask: u32 =
        (((1 << num_cpus) - 1) << GIC_FDT_IRQ_PPI_CPU_SHIFT) & GIC_FDT_IRQ_PPI_CPU_MASK;
    // The timer node is created for each cpu.
    let mut timer_reg_cells = Vec::new();
    for &irq in &irqs {
        timer_reg_cells.push(GIC_FDT_IRQ_TYPE_PPI);
        timer_reg_cells.push(irq);
        timer_reg_cells.push(cpu_mask | IRQ_TYPE_LEVEL_LOW);
    }
    // The timer node contains the timer interrupt type and the timer interrupt phandle.
    let timer_node = fdt.begin_node("timer")?;
    fdt.property_string("compatible", compatible)?;
    fdt.property_array_u32("interrupts", &timer_reg_cells)?;
    fdt.property_null("always-on")?;
    fdt.end_node(timer_node)?;

    Ok(())
}
```
This function is used to write pmu node and its properties to the FDT. The pmu node contains the pmu interrupt type.
```rs
fn create_pmu_node(fdt: &mut FdtWriter, num_cpus: u32) -> Result<()> {
    let compatible = "arm,armv8-pmuv3";
    let cpu_mask: u32 =
        (((1 << num_cpus) - 1) << GIC_FDT_IRQ_PPI_CPU_SHIFT) & GIC_FDT_IRQ_PPI_CPU_MASK;
    let irq = [
        GIC_FDT_IRQ_TYPE_PPI,
        AARCH64_PMU_IRQ,
        cpu_mask | IRQ_TYPE_LEVEL_HIGH,
    ];
    // The pmu node contains the pmu interrupt type.
    let pmu_node = fdt.begin_node("pmu")?;
    fdt.property_string("compatible", compatible)?;
    fdt.property_array_u32("interrupts", &irq)?;
    fdt.end_node(pmu_node)?;
    Ok(())
}
```
This function is used to create rtc (clock) node and its properties to the FDT. The rtc node contains the rtc interrupt type. The rtc interrupt is used to wake up the guest from sleep. The rtc interrupt is also used to set the time in the guest.
```rs
fn create_rtc_node(fdt: &mut FdtWriter, rtc_addr: u64, size: u64) -> Result<()> {
    // the kernel driver for pl030 really really wants a clock node
    // associated with an AMBA device or it will fail to probe, so we
    // need to make up a clock node to associate with the pl030 rtc
    // node and an associated handle with a unique phandle value.
    const CLK_PHANDLE: u32 = 24;
    let clock_node = fdt.begin_node("apb-pclk")?;
    fdt.property_u32("#clock-cells", 0)?;
    fdt.property_string("compatible", "fixed-clock")?;
    fdt.property_u32("clock-frequency", 24_000_000)?;
    fdt.property_string("clock-output-names", "clk24mhz")?;
    fdt.property_phandle(CLK_PHANDLE)?;
    fdt.end_node(clock_node)?;
    // the rtc node
    let rtc_name = format!("rtc@{:x}", rtc_addr);
    let reg = [rtc_addr, size];
    let irq = [GIC_FDT_IRQ_TYPE_SPI, 33, IRQ_TYPE_LEVEL_HIGH];
    // The rtc node contains the rtc interrupt type. The rtc interrupt is used to wake up the guest from sleep. The rtc interrupt is also used to set the time in the guest.
    let rtc_node = fdt.begin_node(&rtc_name)?;
    fdt.property_string_list(
        "compatible",
        vec![String::from("arm,pl031"), String::from("arm,primecell")],
    )?;
    // const PL030_AMBA_ID: u32 = 0x00041030;
    // fdt.property_string("arm,pl031", PL030_AMBA_ID)?;
    fdt.property_array_u64("reg", &reg)?;
    fdt.property_array_u32("interrupts", &irq)?;
    fdt.property_u32("clocks", CLK_PHANDLE)?;
    fdt.property_string("clock-names", "apb_pclk")?;
    fdt.end_node(rtc_node)?;
    Ok(())
}
```

Tests for the FDT builder functions

```rs
mod tests {
    use crate::FdtBuilder;

    #[test]
    // function to test the FDT builder functions.
    fn test_create_fdt() {
        // this is build correctly and should not panic.
        let fdt_ok = FdtBuilder::new()
            .with_cmdline("reboot=t panic=1 pci=off")
            .with_num_vcpus(8)
            .with_mem_size(4096)
            .with_serial_console(0x40000000, 0x1000)
            .with_rtc(0x40001000, 0x1000)
            .create_fdt();
        assert!(fdt_ok.is_ok());
        // this is build incorrectly (no cmdline provided) and should give an error and thus assert would be true.
        let fdt_no_cmdline = FdtBuilder::new()
            .with_num_vcpus(8)
            .with_mem_size(4096)
            .with_serial_console(0x40000000, 0x1000)
            .with_rtc(0x40001000, 0x1000)
            .create_fdt();
        assert!(fdt_no_cmdline.is_err());
        // this is build incorrectly (no num_vcpu provided) and should give an error and thus assert would be true.
        let fdt_no_num_vcpus = FdtBuilder::new()
            .with_cmdline("reboot=t panic=1 pci=off")
            .with_mem_size(4096)
            .with_serial_console(0x40000000, 0x1000)
            .with_rtc(0x40001000, 0x1000)
            .create_fdt();
        assert!(fdt_no_num_vcpus.is_err());
        // this is build incorrectly (no mem_size provided) and should give an error and thus assert would be true.
        let fdt_no_mem_size = FdtBuilder::new()
            .with_cmdline("reboot=t panic=1 pci=off")
            .with_num_vcpus(8)
            .with_serial_console(0x40000000, 0x1000)
            .with_rtc(0x40001000, 0x1000)
            .create_fdt();
        assert!(fdt_no_mem_size.is_err());
        
    }
}