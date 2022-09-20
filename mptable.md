# [mptable](mptable.rs)

## Introduction 

A multiprocessor table is one of the two ways defined in the [MultiProcessor Specification V1.4](https://pdos.csail.mit.edu/6.828/2008/readings/ia32/MPspec.pdf) through which an operating system can get information about the underlying multiprocessor configuration. Floating-pointer structure is the minimal for this where as Table is the maximal data structure representing this. 

From Chapter 4 of the MP Specification: 
> This table is optional. The table is composed of a base section and an extended section. The base section contains entries that are completely backwards compatible with previous versions of this specification. The extended section contains additional entry types. 
> 
> The MP configuration table contains explicit configuration information about APICs, processors, buses, and interrupts. The table consists of a header, followed by a number of entries of various types. The format and length of each entry depends on its type. When present, this configuration table must be stored either in a non-reported system RAM or within the BIOS read-only memory space.
>
> Systems that support a variable number of installed processors must supply a complete MP configuration table that indicates the correct number of installed processors. 
>

## MpTable in Rust
Within the rust code for implementing this structure we have: 
```rust
pub struct MpTable {
    irq_num: u8,
    cpu_num: u8,
}
```
`cpu_num` allows configuring of the number of vCPUs. `irq_num` refers to the number of interrupt requests allowed within the configuration.


## Writing the MP Table in guest memory.
The key to mptable is writing it in the memory for each of the guest operating systems. 

```rust
pub fn write<M: GuestMemory>(&self, mem: &M) -> Result<()> {
    let mut base_mp = GuestAddress(MPTABLE_START);
    let mp_size = self.size();
```
Here the `base_mp` keeps track of the next record in the table, it is intialised with a predefined base value. `mp_size` records the total size of the table.  It is then checked if there is enough memory to allocate the table. 

```rust
    let mut checksum: u8 = 0;
    let max_ioapic_id = self.cpu_num + 1;
```

`checksum` is allocated once the mptable has been finalised. `max_ioapic_id` is defined as the maximum value the I/O APIC is allowed to take. It is later checked while setting up the kvm interrupt routing. 

> I/O APICs contain a redirection table, which is used to route the interrupts it receives from peripheral buses to one or more local APICs. 

A memory object is further initialised with relevant parameters. A `MpfIntel` object is defined using predefined values for `signature`, `length` and `specification`. Checksum for this allocation is calculated and stored. The object is written in the memory and the memory pointer moved to the next location. We note the starting point here with the following statements. A space is left in the middle for the header which will be added after the rest of the configuration.

```rust
let table_base = base_mp;
base_mp = base_mp.unchecked_add(mem::size_of::<MpcTable>() as u64);
```

Base pointer for the table is now set where all the specification of the virtualised resources shall be saved. Each of the vCPUs when stored in the table are allocated with the following data as per the intel spec. This is written into the current `base_mp` value and the value is moved to the next available space.
```rust
for cpu_id in 0..self.cpu_num {
    let mpc_cpu = MpcCpu(mpspec::mpc_cpu {
        type_: mpspec::MP_PROCESSOR as u8, // ENTRY TYPE
        apicid: cpu_id, // LOCAL APIC ID
        apicver: APIC_VERSION, // LOCAL APIC VERSION #
        cpuflag: mpspec::CPU_ENABLED as u8 //CPU FLAGS: EN
            | if cpu_id == 0 {
                mpspec::CPU_BOOTPROCESSOR as u8
            } else {
                0
            },
        cpufeature: CPU_STEPPING,  // CPU FLAGS: BP
        featureflag: CPU_FEATURE_APIC | CPU_FEATURE_FPU, //FEATURE FLAGS 8
        ..Default::default()
});
```
These are defined as follows 
| Field | Description| 
| ----------- | ----------- |
| ENTRY TYPE | A value of 0 identifies a processor entry. |
| LOCAL APIC ID | The local APIC ID number for the particular processor. | 
| LOCAL APIC VERSION #| Bits 0–7 of the local APIC’s version register. |
| CPU FLAGS: EN| If zero, this processor is unusable, and the operating system should not attempt to access this processor.|
| CPU FLAGS: BP| If zero, this processor is unusable, and the operating system should not attempt to access this processor.|
|FEATURE FLAGS  |The feature definition flags for the processor as returned by the CPUID instruction. Refer to Table 4-6 for values. If the processor does not have a CPUID instruction, the BIOS must assign values to FEATURE FLAGS according to the features that it detects.|

The bus entires are defined the next. The specification document explains them as 
> Bus entries identify the kinds of buses in the system. Because there may be more than one bus in a system, each bus is assigned a unique bus ID number by the BIOS. The bus ID number is used by the operating system to associate interrupt lines with specific buses.

```rust 
let size = mem::size_of::<MpcBus>() as u64;
let mpc_bus = MpcBus(mpspec::mpc_bus {
    type_: mpspec::MP_BUS as u8,
    busid: 0,
    bustype: BUS_TYPE_ISA,
});
``` 

| Field | Description| 
| ----------- | ----------- |
| ENTRY TYPE |Entry type 1 identifies a bus entry. |
| BUS ID | An integer that identifies the bus entry. The BIOS assigns identifiers sequentially, starting at zero.| 
| BUS TYPE STRING| A string that identifies the type of bus. Refer to Table 4-8 for values. These are 6-character ASCII (blank-filled) strings used by the MP specification.|

As per the Intel specification and the constants pre-defined in [mpspec.rs](mpspec.rs) the following entires are added to the `mptable`: 
1. I/O APIC Entries
2. I/O Interrupt Assignment Entires 
    > These entries indicate which interrupt source is connected to each I/O APIC interrupt input. There is one entry for each I/O APIC interrupt input that is connected.
3. Local Interrupt Assignment Entries
    > These configuration table entries tell what interrupt source is connected to each local interrupt input of each local APIC.


This constitues Base MP Configuration Entires. We have memory allocation for all the necessary components a Guest OS will need for execution apart from the MP Configuration Table Header. This table stores metadata about the information we have recorded so far. This structure is slotted in the enmpty space we had left earlier in the Guest OS memory. 

```rust 
let mut mpc_table = MpcTable(mpspec::mpc_table {
    signature: MPC_SIGNATURE,
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
```
The header can have the following values, 

| Field | Description | 
| --- | --- | 
| SIGNATURE| The ASCII string representation of “PCMP,” which confirms the presence of the table.| 
| BASE TABLE LENGTH |The length of the base configuration table in bytes, including the header, starting from offset 0. This field aids in calculation of the checksum. | 
| SPEC_REV | The version number of the MP specification. A value of 01h indicates Version 1.1. A value of 04h indicates Version 1.4.|
| CHECKSUM | A checksum of the entire base configuration table. All bytes, including CHECKSUM and reserved bytes, must add up to zero.|
| OEM ID | A string that identifies the manufacturer of the system hardware.| 
| PRODUCT ID| A string that identifies the product family. |
| OEM TABLE POINTER| A physical-address pointer to an OEM-defined configuration table. This table is optional. If not present, this field is zero. | 
| OEM TABLE SIZE | The size of the base OEM table in bytes. If the table is not present, this field is zero. |
| ENTRY COUNT| The number of entries in the variable portion of the base table. This field helps the software identify the end of the table when stepping through the entries. | 
| ADDRESS OF LOCAL APIC| The base address by which each processor accesses its local APIC. | 
| EXTENDED TABLE LENGTH| Length in bytes of the extended entries that are located in memory at the end of the base configuration table. A zero value in this field indicates that no extended entries are present. | 
| EXTENDED TABLE CHECKSUM| Checksum for the set of extended table entries, including only extended entries starting from the end of the base configuration table. When no extended table is present, this field is zero.| 

--- 

With writing all these values in the memory, more than two virtual CPU allocations can be made on available to the Guest OS. 
