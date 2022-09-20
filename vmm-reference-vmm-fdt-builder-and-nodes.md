# FDT Builder and its nodes.
**Source File:** `vmm-reference/src/arch/src/lib.rs`

## Explanations
This is used to through suitable exceptions. This helps in better error handling.
```rust
#[derive(Debug)]
pub enum Error {
    Fdt(FdtError),
    Memory(GuestMemoryError),
    MissingRequiredConfig(String),
}

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

This is the structure of the FDT builder. It is implemented below.
```rust
#[derive(Default)]
pub struct FdtBuilder<'a> {
    cmdline: Option<&'a str>,
    mem_size: Option<u64>,
    num_vcpus: Option<u32>,
    serial_console: Option<(u64, u64)>,
    rtc: Option<(u64, u64)>,
}
```

This is the implementation of Fdt Builder. Fdt is the one which will create the root node. Also, it will have information about mem_size and cpu nodes. Which it uses to create memory and cpu nodes respectively.
```rust
impl<'a> FdtBuilder<'a> {
    pub fn new() -> Self {
        FdtBuilder::default()
    }

    ...

    pub fn create_fdt(&self) -> Result<Fdt> {
        let mut fdt = FdtWriter::new()?;

        // The whole thing is put into one giant node with s
        // ome top level properties
        let root_node = fdt.begin_node("")?;
        fdt.property_u32("interrupt-parent", PHANDLE_GIC)?;
        fdt.property_string("compatible", "linux,dummy-virt")?;
        fdt.property_u32("#address-cells", 0x2)?;
        fdt.property_u32("#size-cells", 0x2)?;

        let cmdline = self
            .cmdline
            .ok_or_else(|| Error::MissingRequiredConfig("cmdline".to_owned()))?;
        create_chosen_node(&mut fdt, cmdline)?;

        let mem_size = self
            .mem_size
            .ok_or_else(|| Error::MissingRequiredConfig("memory".to_owned()))?;
        create_memory_node(&mut fdt, mem_size)?;
        let num_vcpus = self
            .num_vcpus
            .ok_or_else(|| Error::MissingRequiredConfig("vcpu".to_owned()))?;
        create_cpu_nodes(&mut fdt, num_vcpus)?;
        create_gic_node(&mut fdt, true, num_vcpus as u64)?;
        if let Some(serial_console) = self.serial_console {
            create_serial_node(&mut fdt, serial_console.0, serial_console.1)?;
        }
        if let Some(rtc) = self.rtc {
            create_rtc_node(&mut fdt, rtc.0, rtc.1)?;
        }
        create_timer_node(&mut fdt, num_vcpus)?;
        create_psci_node(&mut fdt)?;
        create_pmu_node(&mut fdt, num_vcpus)?;

        fdt.end_node(root_node)?;

        Ok(Fdt {
            fdt_blob: fdt.finish()?,
        })
    }
}
```

## Creating memory nodes
We create a memory node here. We verify memory register properties for the same.
```rust
fn create_memory_node(fdt: &mut FdtWriter, mem_size: u64) -> Result<()> {
    let mem_reg_prop = [AARCH64_PHYS_MEM_START, mem_size];

    let memory_node = fdt.begin_node("memory")?;
    fdt.property_string("device_type", "memory")?;
    fdt.property_array_u64("reg", &mem_reg_prop)?;
    fdt.end_node(memory_node)?;
    Ok(())
}
```

## Creating CPU Nodes
For each CPU, a CPU node is created. Also, only arm and arm version 8 are compatible with CPU node creation.
```rust
fn create_cpu_nodes(fdt: &mut FdtWriter, num_cpus: u32) -> Result<()> {
    let cpus_node = fdt.begin_node("cpus")?;
    fdt.property_u32("#address-cells", 0x1)?;
    fdt.property_u32("#size-cells", 0x0)?;

    for cpu_id in 0..num_cpus {
        let cpu_name = format!("cpu@{:x}", cpu_id);
        let cpu_node = fdt.begin_node(&cpu_name)?;
        fdt.property_string("device_type", "cpu")?;
        fdt.property_string("compatible", "arm,arm-v8")?;
        fdt.property_string("enable-method", "psci")?;
        fdt.property_u32("reg", cpu_id)?;
        fdt.end_node(cpu_node)?;
    }
    fdt.end_node(cpus_node)?;
    Ok(())
}
```

## Creating GIC Node
Here, we are constructing the GIC Node. We validate certain characteristics, such as the nullness of the interrupt controller for the gic node. Then, verify this node's phandle, etc. Additionally, suitable architectures for gic version 3 are arm and giv-v2, otherwise arm and cortex-a15-gic. We must also take care of it.
```rust
fn create_gic_node(fdt: &mut FdtWriter, is_gicv3: bool, num_cpus: u64) -> Result<()> {
    let mut gic_reg_prop = [AARCH64_GIC_DIST_BASE, AARCH64_GIC_DIST_SIZE, 0, 0];

    let intc_node = fdt.begin_node("intc")?;
    if is_gicv3 {
        fdt.property_string("compatible", "arm,gic-v3")?;
        gic_reg_prop[2] = AARCH64_GIC_DIST_BASE - (AARCH64_GIC_REDIST_SIZE * num_cpus);
        gic_reg_prop[3] = AARCH64_GIC_REDIST_SIZE * num_cpus;
    } else {
        fdt.property_string("compatible", "arm,cortex-a15-gic")?;
        gic_reg_prop[2] = AARCH64_GIC_CPUI_BASE;
        gic_reg_prop[3] = AARCH64_GIC_CPUI_SIZE;
    }
    fdt.property_u32("#interrupt-cells", GIC_FDT_IRQ_NUM_CELLS)?;
    fdt.property_null("interrupt-controller")?;
    fdt.property_array_u64("reg", &gic_reg_prop)?;
    fdt.property_phandle(PHANDLE_GIC)?;
    fdt.property_u32("#address-cells", 2)?;
    fdt.property_u32("#size-cells", 2)?;
    fdt.end_node(intc_node)?;

    Ok(())
}
```

## Creating PSCI Node
We create the PSCI Node here. Only compatible archs are arm and psci-0.2. We verify that. Also, method used here is hvc instead of smc.  
**Reason:**
> There are two methods: hvc and smc. According to the documentation, PSCI calls between a guest and hypervisor may use HVC rather than SMC. Since we are using kvm, we must employ hvc.
```rust
fn create_psci_node(fdt: &mut FdtWriter) -> Result<()> {
    let compatible = "arm,psci-0.2";
    let psci_node = fdt.begin_node("psci")?;
    fdt.property_string("compatible", compatible)?;
    fdt.property_string("method", "hvc")?;
    fdt.end_node(psci_node)?;

    Ok(())
}
```

## Creating Serial node
Over here, we are creating a serial node. We pass address and size, which are used to create serial register properties. We also have various checks like the arch should be ns16550a, which is the only one compatible, verifying clock names, etc. We have CLK_PHANDLE phandle and irq as the interrupt type. So, serial node will also contain these informations.
```rust
fn create_serial_node(fdt: &mut FdtWriter, addr: u64, size: u64) -> Result<()> {
    let serial_node = fdt.begin_node(&format!("uart@{:x}", addr))?;
    fdt.property_string("compatible", "ns16550a")?;
    let serial_reg_prop = [addr, size];
    fdt.property_array_u64("reg", &serial_reg_prop)?;

    const CLK_PHANDLE: u32 = 24;
    fdt.property_u32("clocks", CLK_PHANDLE)?;
    fdt.property_string("clock-names", "apb_pclk")?;
    ...
    fdt.end_node(serial_node)?;
    Ok(())
}
```

## Creating Timer Node
Here we are creating a timer node. While doing so, we ensure that we are using a compatible architecture, such as arm or armv8-timer. Timer devices have a fixed number of interrupt types. So, we also generate cpu mask, which is utilised to generate timer reg cells for interrupt verification.
```rust
fn create_timer_node(fdt: &mut FdtWriter, num_cpus: u32) -> Result<()> {
    let irqs = [13, 14, 11, 10];
    let compatible = "arm,armv8-timer";
    let cpu_mask: u32 =
        (((1 << num_cpus) - 1) << GIC_FDT_IRQ_PPI_CPU_SHIFT) & GIC_FDT_IRQ_PPI_CPU_MASK;

    let mut timer_reg_cells = Vec::new();
    for &irq in &irqs {
        timer_reg_cells.push(GIC_FDT_IRQ_TYPE_PPI);
        timer_reg_cells.push(irq);
        timer_reg_cells.push(cpu_mask | IRQ_TYPE_LEVEL_LOW);
    }

    let timer_node = fdt.begin_node("timer")?;
    fdt.property_string("compatible", compatible)?;
    fdt.property_array_u32("interrupts", &timer_reg_cells)?;
    fdt.property_null("always-on")?;
    fdt.end_node(timer_node)?;

    Ok(())
}
```

## Creating PMU Node
Here, we're generating the PMU Node and adding it using FDT Writer. We also need the num cpu option to correctly generate the cpu mask, which is then used to determine irq, the pmu interrupt type. which is the pmu interrupt type.
```rust
fn create_pmu_node(fdt: &mut FdtWriter, num_cpus: u32) -> Result<()> {
    let compatible = "arm,armv8-pmuv3";
    let cpu_mask: u32 =
        (((1 << num_cpus) - 1) << GIC_FDT_IRQ_PPI_CPU_SHIFT) & GIC_FDT_IRQ_PPI_CPU_MASK;
    let irq = [
        GIC_FDT_IRQ_TYPE_PPI,
        AARCH64_PMU_IRQ,
        cpu_mask | IRQ_TYPE_LEVEL_HIGH,
    ];

    let pmu_node = fdt.begin_node("pmu")?;
    fdt.property_string("compatible", compatible)?;
    fdt.property_array_u32("interrupts", &irq)?;
    fdt.end_node(pmu_node)?;
    Ok(())
}
```

## Creating RTC Node
We are creating a clock node and a RTC node. Main goal is to create a RTC node but we are also creating a clock node because the kernel driver for pl030 needs a clock node to be connected to an Advanced microcontroller bus architecture device, or else it won't be able to probe.
```rust
line let clock_node = fdt.begin_node("apb-pclk")?;
```
This clock node will connect rtc node and a handle with a unique phandle value.
Also, while creating clock node we are verifying various properties like cpu frequency. Then we create the rtc node at the end using rtc address passed (rtc_addr) and size.
```rust
fn create_rtc_node(fdt: &mut FdtWriter, rtc_addr: u64, size: u64) -> Result<()> {
    const CLK_PHANDLE: u32 = 24;
    let clock_node = fdt.begin_node("apb-pclk")?;
    fdt.property_u32("#clock-cells", 0)?;
    ...
    fdt.property_u32("clock-frequency", 24_000_000)?;
    ...
    fdt.end_node(clock_node)?;

    let rtc_name = format!("rtc@{:x}", rtc_addr);
    let reg = [rtc_addr, size];
    let irq = [GIC_FDT_IRQ_TYPE_SPI, 33, IRQ_TYPE_LEVEL_HIGH];

    let rtc_node = fdt.begin_node(&rtc_name)?;
    fdt.property_string_list(
        "compatible",
        vec![String::from("arm,pl031"), String::from("arm,primecell")],
    )?;
    fdt.property_array_u64("reg", &reg)?;
    fdt.property_array_u32("interrupts", &irq)?;
    ...
    fdt.end_node(rtc_node)?;
    Ok(())
}
```

## Unit Testing
Following code is used to unit test the functions that were made above.
```rust
#[cfg(test)]
mod tests {
    use crate::FdtBuilder;

    #[test]
    fn test_create_fdt() {
        let fdt_ok = FdtBuilder::new()
            .with_cmdline("reboot=t panic=1 pci=off")
            .with_num_vcpus(8)
            .with_mem_size(4096)
            .with_serial_console(0x40000000, 0x1000)
            .with_rtc(0x40001000, 0x1000)
            .create_fdt();
        assert!(fdt_ok.is_ok());

        let fdt_no_cmdline = FdtBuilder::new()
            .with_num_vcpus(8)
            .with_mem_size(4096)
            .with_serial_console(0x40000000, 0x1000)
            .with_rtc(0x40001000, 0x1000)
            .create_fdt();
        assert!(fdt_no_cmdline.is_err());

        let fdt_no_num_vcpus = FdtBuilder::new()
            .with_cmdline("reboot=t panic=1 pci=off")
            .with_mem_size(4096)
            .with_serial_console(0x40000000, 0x1000)
            .with_rtc(0x40001000, 0x1000)
            .create_fdt();
        assert!(fdt_no_num_vcpus.is_err());

        let fdt_no_mem_size = FdtBuilder::new()
            .with_cmdline("reboot=t panic=1 pci=off")
            .with_num_vcpus(8)
            .with_serial_console(0x40000000, 0x1000)
            .with_rtc(0x40001000, 0x1000)
            .create_fdt();
        assert!(fdt_no_mem_size.is_err());
    }
}
```