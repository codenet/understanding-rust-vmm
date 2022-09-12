# [GDT](../src/vm-vcpu-ref/src/x86_64/gdt.rs)


This file contains useful structs and methods to create, acess, and update a GDT. It also contains methods to write a GDT and IDT to any GuestMemory.

> Constants
```rs
/// The offset at which GDT resides in memory.
pub const BOOT_GDT_OFFSET: u64 = 0x500;
/// The offset at which IDT resides in memory.
pub const BOOT_IDT_OFFSET: u64 = 0x520;
/// Maximum number of GDT entries as defined in the Intel Specification.
pub const MAX_GDT_SIZE: usize = 1 << 13;
```

> Error enum for invalid memory access and exceeding number of entries in GDT.
```rs
pub enum Error {
    /// Invalid memory access.
    GuestMemory(GuestMemoryError),
    /// Too many entries in the GDT.
    TooManyEntries,
}
```

> Helper function to convert GuestMemoryError to Error.
```rs
impl From<GuestMemoryError> for Error {
    fn from(inner: GuestMemoryError) -> Self {
        Error::GuestMemory(inner)
    }
}
```

> Type alias for method return types.
```rs
/// Results corresponding to operations on the GDT.
pub type Result<T> = std::result::Result<T, Error>;
```

## Segment Descriptor
> Struct for an entry in GDT.
```rs
pub struct SegmentDescriptor(pub u64);
```
> It has one field of type u64 and has the following structure.
```rs
// |31                                    16|15                               0|
// |          Base 0:15                     |          Limit 0:15              |
// |63 - - - - - - 56 55 - - 52 51 - -    48 47 - - - - - - 40 39 - - - - - - 32
// |Base 24:31       | Flags   |Limit 16:19 |  Access Bytes   | Base 16:23     |
```

> ByteValued implementation so that individual bytes of the u64 value can be accessed when writing to `GuestMemory`.
```rs
unsafe impl ByteValued for SegmentDescriptor {}
```

### Methods

> Following method constructs a `SegmentDescriptor` from base, flags and limits. Base is the linear address where the GDT entry begins. Flag includes present bit, privelege level bits and so on. Limit is the maximum addressable memory.
```rs
pub fn from(flags: u16, base: u32, limit: u32) -> Self
```

> Following methods are getters for the corresponding fields in a GDT entry such as base, limit, p (present) bit, dpl (privilege level) bits, and so on.
```rs
fn base(&self) -> u64
fn limit(&self) -> u32
fn g(&self) -> u8
fn db(&self) -> u8
fn l(&self) -> u8
fn avl(&self) -> u8
fn p(&self) -> u8
fn dpl(&self) -> u8
fn s(&self) -> u8
fn segment_type(&self) -> u8
```

> Following method constructs a `kvm_segment` from index of entry in GDT.
```rs
fn create_kvm_segment(&self, table_index: usize) -> kvm_segment
```

> `kvm_segment` is a struct which allows easy access to the structure of `SegmentDescriptor` 
```rs
pub struct kvm_segment {
    pub base: __u64,
    pub limit: __u32,
    pub selector: __u16,
    pub type_: __u8,
    pub present: __u8,
    pub dpl: __u8,
    pub db: __u8,
    pub s: __u8,
    pub l: __u8,
    pub g: __u8,
    pub avl: __u8,
    pub unusable: __u8,
    pub padding: __u8,
}
```

## GDT

> Struct for GDT which contains one field which is a vector of segment descriptors.
```rs
pub struct Gdt(Vec<SegmentDescriptor>);
```

### Methods

> The following method creates an empty GDT with 0 entries
```rs
pub fn new()
```

> The following method tries to push one `SegmentDescriptor` into the GDT. It returns a `TooManyEntries` error if the size of GDT is more than `MAX_GDT_SIZE` (8192). Otherwise, it appends the entry and returns `Ok(())`.
```rs
pub fn try_push(&mut self, entry: SegmentDescriptor) -> Result<()>
```

> The following method return a `kvm_segment` for an index in GDT if it exists.
```rs
pub fn create_kvm_segment_for(&self, index: usize) -> Option<kvm_segment>
```

> The following method writes the GDT structure to any `GuestMemory` at `BOOT_GDT_OFFSET` (1280). It can fail if it attempts to write at an invalid guest address in which case it returns an `InvalidGuestAddress`.
```rs
pub fn write_to_mem<Memory: GuestMemory>(&self, mem: &Memory) -> Result<()> {
```

> This provides a default GDT table with 4 entries. A null entry, a code entry, a data entry, and a TSS entry. The code, data, and TSS entries have the same base (0x0) and limit (0xfffff) but different flags. This is a flat memory model.
```rs
impl Default for Gdt {
    fn default() -> Self {
        Gdt(vec![
            SegmentDescriptor::from(0, 0, 0),            // NULL
            SegmentDescriptor::from(0xa09b, 0, 0xfffff), // CODE
            SegmentDescriptor::from(0xc093, 0, 0xfffff), // DATA
            SegmentDescriptor::from(0x808b, 0, 0xfffff), // TSS
        ])
    }
}
```

> This writes 8 bytes of data (val) into the Guest Memory at the `BOOT_IDT_OFFSET` (1312). It returns a Guest Memory Error if invalid memory is accessed.
```rs
pub fn write_idt_value<Memory: GuestMemory>(val: u64, guest_mem: &Memory) -> Result<()>
```

## Tests

> It tests if the `create_kvm_segment_for` method correctly creates a `kvm_segment` struct from an entry of the GDT.
```rs
fn test_kvm_segment_parse()
```

> It tests if the `write_to_mem` method and `write_idt_value` function correctly writes the table to a `GuestMemory` and returns an error if the `GuestMemory` is smaller than the table.
```rs
fn test_write_table()
```

> It tests if the `try_push` method pushes a `SegmentDescriptor` when the size of table is less than `MAX_GDT_SIZE` (8192) and returns an error when its size exceed `MAX_GDT_SIZE`.
```rs
fn test_too_many_entries()
```

> It tests if the size of `SegmentDescriptor` is equal to the size of `u64`.
```rs
fn test_memory_constraints()
```

## Usage

Used in [vcpu](../src/vm-vcpu/src/vcpu/mod.rs) in `configure_sregs` method.
> It creates a default gdt and writes it to guest memory. It also stores the gdt entries in the `kvm_sregs` struct.