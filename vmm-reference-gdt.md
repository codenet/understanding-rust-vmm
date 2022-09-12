# [GDT](../vmm-reference/src/vm-vcpu-ref/src/x86_64/gdt.rs)
This module provides abstractions for building a Global Descriptor Table (GDT).

## Segment Descriptors
Segment Descriptors for the GDT have been defined as follows:
```rs
pub struct SegmentDescriptor(pub u64);
```

The following function creates a `SegmentDescriptor` given base, limit and flags.
The way to convert these numbers to a 64 - bit number has been described [here](https://wiki.osdev.org/Global_Descriptor_Table#Segment_Descriptor). 

`base` is a 32 bit value and specifies the address where this segment starts.

`limit` is a 32 bit value, of which only the least significant 20 bits are used. It tells the maximum addressable unit.

`flags` is a 16 bit value, of which 12 are used. It has two parts. The last 8 bits are referred to as Access Byte. The first 4 bits are some more flags.

```rs
    pub fn from(flags: u16, base: u32, limit: u32) -> Self;
```
<table class="wikitable">
<caption> Segment Descriptor
</caption>
<tbody><tr>
<th style="width: 20%; text-align: left;">63&nbsp;&nbsp;&nbsp;<span style="float: right;">56</span>
</th>
<th style="width: 12.5%; text-align: left;">55&nbsp;&nbsp;&nbsp;<span style="float: right;">52</span>
</th>
<th style="width: 12.5%; text-align: left;">51&nbsp;&nbsp;&nbsp;<span style="float: right;">48</span>
</th>
<th style="width: 25%; text-align: left;">47&nbsp;&nbsp;&nbsp;<span style="float: right;">40</span>
</th>
<th style="width: 25%; text-align: left;">39&nbsp;&nbsp;&nbsp;<span style="float: right;">32</span>
</th></tr>
<tr>
<td><b>Base</b><br>31&nbsp;&nbsp;&nbsp;<span style="float: right;">24</span>
</td>
<td><b>Flags</b><br>3&nbsp;&nbsp;&nbsp;<span style="float: right;">0</span>
</td>
<td><b>Limit</b><br>19&nbsp;&nbsp;&nbsp;<span style="float: right;">16</span>
</td>
<td><b>Access Byte</b><br>7&nbsp;&nbsp;&nbsp;<span style="float: right;">0</span>
</td>
<td><b>Base</b><br>23&nbsp;&nbsp;&nbsp;<span style="float: right;">16</span>
</td></tr>
<tr>
<th colspan="3" style="text-align: left;">31 &nbsp;&nbsp;<span style="float: right;">16</span>
</th>
<th colspan="2" style="text-align: left;">15 &nbsp;&nbsp;<span style="float: right;">0</span>
</th></tr>
<tr>
<td colspan="3"><b>Base</b><br>15&nbsp;&nbsp;&nbsp;<span style="float: right;">0</span>
</td>
<td colspan="2"><b>Limit</b><br>15&nbsp;&nbsp;&nbsp;<span style="float: right;">0</span>
</td></tr></tbody><div></div></table>

The following methods extract base and limit from a `SegmentDescriptor`.
```rs
    fn base(&self) -> u64;
    fn limit(&self) -> u32;
```
These extract some flags from the segment.
```rs
    fn g(&self) -> u8;
    fn db(&self) -> u8;
    fn l(&self) -> u8;
    fn avl(&self) -> u8;
```
The following values are extracted from the access byte poriton of the flags. Description available [here](https://wiki.osdev.org/Global_Descriptor_Table#Segment_Descriptor).
```rs
    fn p(&self) -> u8;
    fn dpl(&self) -> u8;
    fn s(&self) -> u8;
    fn segment_type(&self) -> u8;
```
`create_kvm_segment` creates a `kvm_segment` from the `SegmentDescriptor`. 

`kvm_segment` is the data type of all the segment registers (`cs`, `ds`, etc.) in a KVM vCPU.

`table_index` is the index of the entry in the gdt table.
```rs
fn create_kvm_segment(&self, table_index: usize) -> kvm_segment {
        kvm_segment {
            base: self.base(),
            limit: self.limit(),
            // The multiplication is safe because the table_index can be maximum
            // `MAX_GDT_SIZE`. The conversion is safe because the result fits in u16.
            selector: (table_index * 8) as u16,
            type_: self.segment_type(),
            present: self.p(),
            dpl: self.dpl(),
            db: self.db(),
            s: self.s(),
            l: self.l(),
            g: self.g(),
            avl: self.avl(),
            padding: 0,
```
The `p` bit in flags specifies whether the segment is valid or not. `p` is set to 1 for a valid segment.
``` rs
            unusable: match self.p() {
                0 => 1,
                _ => 0,
            },
        }
    }
```



## GDT
Implemented as a vector of segment descriptors.
```rs
pub struct Gdt(Vec<SegmentDescriptor>);
```
`try_push` tries to push an entry into the `GDT`. An error is thron if the number of entries is greater than $2^{13} - 1$   (max 12 bits).
```rs
pub fn try_push(&mut self, entry: SegmentDescriptor) -> Result<()>
```

`create_kvm_segment_for` returns a `kvm_segment` corresponding to the GDT entry at `index`.
```rs
pub fn create_kvm_segment_for(&self, index: usize) -> Option<kvm_segment>
```

This method writes the GDT into Guest Memory. 
For each entry, the address is calculated first. The base of the GDT is at `boot_gdt_addr`. Each entry is placed at the end of the previous entry.
```rs
pub fn write_to_mem<Memory: GuestMemory>(&self, mem: &Memory) -> Result<()>;
```

A default implementation of the GDT with default segment descriptors has been provided at the line 257. In the default table, for all the segments, base is 0 and limit is $2^{20}$. 
```rs
impl Default for Gdt {
    fn default() -> Self {
        Gdt(vec![
            SegmentDescriptor::from(0, 0, 0),            // NULL
            SegmentDescriptor::from(0xa09b, 0, 0xfffff), // CODE
            SegmentDescriptor::from(0xc093, 0, 0xfffff), // DATA
            SegmentDescriptor::from(0x808b, 0, 0xfffff), // TSS - holds information about the task's state
        ])
    }
}
```


## IDT
This module also has a function to set a value at the IDT offset. It has only been used once, to create an empty IDT entry.
```rs
pub fn write_idt_value<Memory: GuestMemory>(val: u64, guest_mem: &Memory) -> Result<()>;
```


<br /><br /><br />

This module has been used in `KvmVcpu` in the `configure_sregs` method to set the base and limit of the GDT, the IDT and to setup the segment registers.
```rs
let mut sregs = self.vcpu_fd.get_sregs().map_err(Error::KvmIoctl)?;

// Global descriptor tables.
let gdt_table = Gdt::default();

// The following unwraps are safe because we know that the default GDT has 4 segments.
let code_seg = gdt_table.create_kvm_segment_for(1).unwrap();
let data_seg = gdt_table.create_kvm_segment_for(2).unwrap();
let tss_seg = gdt_table.create_kvm_segment_for(3).unwrap();

// Write segments to guest memory.
gdt_table.write_to_mem(guest_memory).map_err(Error::Gdt)?;
sregs.gdt.base = BOOT_GDT_OFFSET as u64;
sregs.gdt.limit = std::mem::size_of_val(&gdt_table) as u16 - 1;

write_idt_value(0, guest_memory).map_err(Error::Gdt)?;
sregs.idt.base = BOOT_IDT_OFFSET as u64;
sregs.idt.limit = std::mem::size_of::<u64>() as u16 - 1;

sregs.cs = code_seg;
sregs.ds = data_seg;
sregs.es = data_seg;
sregs.fs = data_seg;
sregs.gs = data_seg;
sregs.ss = data_seg;
sregs.tr = tss_seg;
```