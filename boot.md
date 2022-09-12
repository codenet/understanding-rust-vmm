# [BootManager](../src/vmm/src/boot.rs)

There are 3 terms related to memory being used here:
* `Guest Memory`: The memory available to the VM.
* `High Memory`: The guest memory is divided into 2 parts: Low Memory and High Memory. The kernel uses the Low Memory and the user uses High Memory. PTEs for High Memory are not maintained permanently, but rather as they are needed.
* `Memory Mapped for I/O`: This is the memory mapped for I/O devices.

```rs
pub enum Error {
    /// Invalid E820 configuration.
    E820Configuration,
    /// Highmem start address is past the guest memory end.
    HimemStartPastMemEnd,
    /// Highmem start address is past the MMIO gap start.
    HimemStartPastMmioGapStart,
    /// The MMIO gap end is past the guest memory end.
    MmioGapPastMemEnd,
    /// The MMIO gap start is past the gap end.
    MmioGapStartPastMmioGapEnd,
}
```

The above code lists some errors that may occur while initializing memory during bootup. The VM uses E820 RAM type. 
* `E820Configuration`: This error denotes that the E820 configuration is invalid.
* `HimemStartPastMemEnd`: This error denotes that the starting address of high memory is greater than the last guest memory address. High memory is one for which kernel doesn't maintain PTEs permanently, but on fly as needed. The address space for users generally lies in this memory region.
* `MmioGapPastMemEnd`: This error denotes that MMIO gap's ending address is greater than the guest memory's ending address.
* `MmioGapStartPastMmioGapEnd`: This error denotes that the starting address of MMIO is greater than its ending address.

```rs
fn add_e820_entry(
    params: &mut boot_params,
    addr: u64,
    size: u64,
    mem_type: u32,
) -> result::Result<(), Error> {
    if params.e820_entries >= params.e820_table.len() as u8 {
        return Err(Error::E820Configuration);
    }

    params.e820_table[params.e820_entries as usize].addr = addr;
    params.e820_table[params.e820_entries as usize].size = size;
    params.e820_table[params.e820_entries as usize].type_ = mem_type;
    params.e820_entries += 1;

    Ok(())
}
```

The above method is used to add entries to e820_table in boot parameters and returns error if the table cannot fit another entry. The table stores the valid address ranges that can be used by the user. An address A is valid for the user if:
1) It lies in the range [High Memory Start, Guest Memory End]
2) It does not lie in the range [MMIO start, MMIO end]
The below method implements this.

```rs
/// Build boot parameters for ELF kernels following the Linux boot protocol.
///
/// # Arguments
///
/// * `guest_memory` - guest memory.
/// * `kernel_load` - result of loading the kernel in guest memory.
/// * `himem_start` - address where high memory starts.
/// * `mmio_gap_start` - address where the MMIO gap starts.
/// * `mmio_gap_end` - address where the MMIO gap ends.
pub fn build_bootparams(
    guest_memory: &GuestMemoryMmap,
    kernel_load: &KernelLoaderResult,
    himem_start: GuestAddress,
    mmio_gap_start: GuestAddress,
    mmio_gap_end: GuestAddress,
) -> result::Result<boot_params, Error> {
    if mmio_gap_start >= mmio_gap_end {
        return Err(Error::MmioGapStartPastMmioGapEnd);
    }

    let mut params = boot_params::default();

    if let Some(hdr) = kernel_load.setup_header {
        params.hdr = hdr;
    } else {
        params.hdr.boot_flag = KERNEL_BOOT_FLAG_MAGIC;
        params.hdr.header = KERNEL_HDR_MAGIC;
        params.hdr.kernel_alignment = KERNEL_MIN_ALIGNMENT_BYTES;
    }
    // If the header copied from the bzImage file didn't set type_of_loader,
    // force it to "undefined" so that the guest can boot normally.
    // See: https://github.com/cloud-hypervisor/cloud-hypervisor/issues/918
    // and: https://www.kernel.org/doc/html/latest/x86/boot.html#details-of-header-fields
    if params.hdr.type_of_loader == 0 {
        params.hdr.type_of_loader = KERNEL_LOADER_OTHER;
    }

    // Add an entry for EBDA itself.
    add_e820_entry(&mut params, 0, EBDA_START, E820_RAM)?;

    // Add entries for the usable RAM regions (potentially surrounding the MMIO gap).
    let last_addr = guest_memory.last_addr();
    if last_addr < mmio_gap_start {
        add_e820_entry(
            &mut params,
            himem_start.raw_value(),
            // The unchecked + 1 is safe because:
            // * overflow could only occur if last_addr - himem_start == u64::MAX
            // * last_addr is smaller than mmio_gap_start, a valid u64 value
            // * last_addr - himem_start is also smaller than mmio_gap_start
            last_addr
                .checked_offset_from(himem_start)
                .ok_or(Error::HimemStartPastMemEnd)?
                + 1,
            E820_RAM,
        )?;
    } else {
        add_e820_entry(
            &mut params,
            himem_start.raw_value(),
            mmio_gap_start
                .checked_offset_from(himem_start)
                .ok_or(Error::HimemStartPastMmioGapStart)?,
            E820_RAM,
        )?;

        if last_addr > mmio_gap_end {
            add_e820_entry(
                &mut params,
                mmio_gap_end.raw_value(),
                // The unchecked_offset_from is safe, guaranteed by the `if` condition above.
                // The unchecked + 1 is safe because:
                // * overflow could only occur if last_addr == u64::MAX and mmio_gap_end == 0
                // * mmio_gap_end > mmio_gap_start, which is a valid u64 => mmio_gap_end > 0
                last_addr.unchecked_offset_from(mmio_gap_end) + 1,
                E820_RAM,
            )?;
        }
    }

    Ok(params)
}
```

It sets the header field of boot parameters. If the type of loader was unspecified, it forces the type to be undefined so that guest can boot it normally. It first adds an entry for EBDA (Extended Bios Data Area). After that, it adds the address ranges that the user can use according to the following conditions:
* Guest Memory end < MMIO start: Add [High Memory start, Guest Memory end]
* MMIO start <= Guest Memory end <= MMIO end: Add [High Memory start, MMIO start)
* Guest Memory end > MMIO end: Add [High Memory start, MMIO start) and (MMIO end, Guest Memory end]

While doing so, it checks for appropriate errors.
