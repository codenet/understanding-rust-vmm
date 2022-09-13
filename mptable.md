Description of [Mptable](../src/vm-vcpu-ref/src/x86_64/mptable.rs)

mptable allows to configure a multiprocessor configuration table given the number of vcpus.

```rust
pub struct MpTable {
    irq_num: u8,
    cpu_num: u8,
}
```

mptable has two attributes.
irq_num (interrupt request number) and cpu_num.

```rust
fn write_mptable() -> Result<()> {
    let num_cpus = 4;
    let mptable = MpTable::new(num_cpus)?;
    let mem: GuestMemoryMmap =
        GuestMemoryMmap::from_ranges(&[(GuestAddress(0), 1024 << 20)]).unwrap();
    mptable.write(&mem)
}
```

example code to create a mptable for 4 vcpus.

```rust
pub fn new(cpu_num: u8) -> Result<MpTable> {
    if cpu_num > MAX_SUPPORTED_CPUS {
        return Err(Error::TooManyCpus);
    }

    Ok(MpTable {
        irq_num: IRQ_MAX + 1,
        cpu_num,
    })
}
```

irq_num is set to 24 when created since 23 is the last usable IRQ ID for virtio device interrupts on x86_64.

Since we don't know the exact length of MpcTable to be created, all the requrired structs are created first,
added to a base address and final length is determined.

All the structs that are going to be created are defined [here](../src/vm-vcpu-ref/src/x86_64/mpspec.rs).

```rust
{
    let size = mem::size_of::<MpfIntel>() as u64;
    let mut mpf_intel = MpfIntel(mpspec::mpf_intel {
        signature: SMP_MAGIC_IDENT,
        length: 1,
        specification: MPC_SPEC as u8,
        // The conversion to u32 is safe because the Base MP address is the
        // MPTABLE_START = 0x9fc00 and the size of MpfIntel is 16 bytes.
        // This value is much smaller that u32::MAX.
        physptr: (base_mp.raw_value() + size) as u32,
        ..Default::default()
    });
    mpf_intel.0.checksum = mpf_intel_compute_checksum(&mpf_intel.0);
    mem.write_obj(mpf_intel, base_mp)
        .map_err(|_| Error::WriteMpfIntel)?;
    base_mp = base_mp.unchecked_add(size);
}
```
A MpfIntel struct is being created.
This is being added to Mptable.

```rust
let table_base = base_mp;
base_mp = base_mp.unchecked_add(mem::size_of::<MpcTable>() as u64);

{
    let size = mem::size_of::<MpcCpu>() as u64;
    for cpu_id in 0..self.cpu_num {
        let mpc_cpu = MpcCpu(mpspec::mpc_cpu {
            type_: mpspec::MP_PROCESSOR as u8,
            apicid: cpu_id,
            apicver: APIC_VERSION,
            cpuflag: mpspec::CPU_ENABLED as u8
                | if cpu_id == 0 {
                    mpspec::CPU_BOOTPROCESSOR as u8
                } else {
                    0
                },
            cpufeature: CPU_STEPPING,
            featureflag: CPU_FEATURE_APIC | CPU_FEATURE_FPU,
            ..Default::default()
        });
        mem.write_obj(mpc_cpu, base_mp)
            .map_err(|_| Error::WriteMpcCpu)?;
        base_mp = base_mp.unchecked_add(size);
        checksum = checksum.wrapping_add(compute_checksum(&mpc_cpu.0));
    }
}
```

An instance of MpcCpu is being created for each vcpu.
These instances are being added to mptable after creation.

```rust
{
    let size = mem::size_of::<MpcBus>() as u64;
    let mpc_bus = MpcBus(mpspec::mpc_bus {
        type_: mpspec::MP_BUS as u8,
        busid: 0,
        bustype: BUS_TYPE_ISA,
    });
    mem.write_obj(mpc_bus, base_mp)
        .map_err(|_| Error::WriteMpcBus)?;
    base_mp = base_mp.unchecked_add(size);
    checksum = checksum.wrapping_add(compute_checksum(&mpc_bus.0));
}
{
    let size = mem::size_of::<MpcIoapic>() as u64;
    let mpc_ioapic = MpcIoapic(mpspec::mpc_ioapic {
        type_: mpspec::MP_IOAPIC as u8,
        apicid: max_ioapic_id,
        apicver: APIC_VERSION,
        flags: mpspec::MPC_APIC_USABLE as u8,
        apicaddr: IO_APIC_DEFAULT_PHYS_BASE,
    });
    mem.write_obj(mpc_ioapic, base_mp)
        .map_err(|_| Error::WriteMpcIoapic)?;
    base_mp = base_mp.unchecked_add(size);
    checksum = checksum.wrapping_add(compute_checksum(&mpc_ioapic.0));
}
for i in 0..self.irq_num {
    let size = mem::size_of::<MpcIntsrc>() as u64;
    let mpc_intsrc = MpcIntsrc(mpspec::mpc_intsrc {
        type_: mpspec::MP_INTSRC as u8,
        irqtype: mpspec::mp_irq_source_types_mp_INT as u8,
        irqflag: mpspec::MP_IRQDIR_DEFAULT as u16,
        srcbus: 0,
        srcbusirq: i,
        dstapic: max_ioapic_id,
        dstirq: i,
    });
    mem.write_obj(mpc_intsrc, base_mp)
        .map_err(|_| Error::WriteMpcIntsrc)?;
    base_mp = base_mp.unchecked_add(size);
    checksum = checksum.wrapping_add(compute_checksum(&mpc_intsrc.0));
}
{
    let size = mem::size_of::<MpcLintsrc>() as u64;
    let mpc_lintsrc = MpcLintsrc(mpspec::mpc_lintsrc {
        type_: mpspec::MP_LINTSRC as u8,
        irqtype: mpspec::mp_irq_source_types_mp_ExtINT as u8,
        irqflag: mpspec::MP_IRQDIR_DEFAULT as u16,
        srcbusid: 0,
        srcbusirq: 0,
        destapic: 0,
        destapiclint: 0,
    });

    mem.write_obj(mpc_lintsrc, base_mp)
        .map_err(|_| Error::WriteMpcLintsrc)?;
    base_mp = base_mp.unchecked_add(size);
    checksum = checksum.wrapping_add(compute_checksum(&mpc_lintsrc.0));
}
{
    let size = mem::size_of::<MpcLintsrc>() as u64;
    let mpc_lintsrc = MpcLintsrc(mpspec::mpc_lintsrc {
        type_: mpspec::MP_LINTSRC as u8,
        irqtype: mpspec::mp_irq_source_types_mp_NMI as u8,
        irqflag: mpspec::MP_IRQDIR_DEFAULT as u16,
        srcbusid: 0,
        srcbusirq: 0,
        destapic: 0xFF, /* to all local APICs */
        destapiclint: 1,
    });

    mem.write_obj(mpc_lintsrc, base_mp)
        .map_err(|_| Error::WriteMpcLintsrc)?;
    base_mp = base_mp.unchecked_add(size);
    checksum = checksum.wrapping_add(compute_checksum(&mpc_lintsrc.0));
}

```

Instances of MpcBus and MpcIoapic are created and added to mptable.
Instances for more structs regarding interrupts are being created and added to mptable.

```rust
let table_end = base_mp;

{
    let mut mpc_table = MpcTable(mpspec::mpc_table {
        signature: MPC_SIGNATURE,
        // It's safe to use unchecked_offset_from because
        // table_end > table_base. Also, the conversion to u16 is safe
        // because the length of the table is in the order of thousands
        // for the maximum number of cpus and maximum number
        // of IRQs that we allow ( length = 5328), which fits in a u16.
        length: table_end.unchecked_offset_from(table_base) as u16,
        spec: MPC_SPEC,
        oem: MPC_OEM,
        productid: MPC_PRODUCT_ID,
        lapic: APIC_DEFAULT_PHYS_BASE,
        ..Default::default()
    });
    checksum = checksum.wrapping_add(compute_checksum(&mpc_table.0));
    mpc_table.0.checksum = (!checksum).wrapping_add(1) as i8;
    mem.write_obj(mpc_table, table_base)
        .map_err(|_| Error::WriteMpcTable)?;
}
```

Since we have the final length of MpcTable, the final struct for mptable is being created. 