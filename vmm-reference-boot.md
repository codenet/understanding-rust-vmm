# [Boot](https://github.com/codenet/vmm-reference/blob/main/src/vmm/src/boot.rs)

This module uses [Linux-loader](https://github.com/rust-vmm/linux-loader) and [vm-memory](https://github.com/rust-vmm/vm-memory) crates to provide functionality for building boot parameters for ELF kernels following the [Linux boot protocol](https://www.kernel.org/doc/Documentation/x86/boot.txt).

Described below are the constant headers of the Boot parameters -
```rs
const KERNEL_BOOT_FLAG_MAGIC: u16 = 0xaa55;
```
Header field: `boot_flag`. Must contain 0xaa55. This is the closest thing old Linux kernels have to a magic number. The 0xAA55 signature is the last two bytes of the first sector of your bootdisk (bootsector/Master Boot Record/MBR). If it is 0xAA55, then the BIOS will try booting the system. If it's not found (garbled or 0x0000), you'll get an error message from your BIOS that it didn't find a bootable disk.

```rs
const KERNEL_HDR_MAGIC: u32 = 0x5372_6448;
```
Header field: `header`. Must contain the magic number `HdrS` (0x5372_6448).


```rs
const KERNEL_LOADER_OTHER: u8 = 0xff;
```
Header field: `type_of_loader`. Unless using a pre-registered bootloader (which we aren't), this field must be set to 0xff.


```rs
const KERNEL_MIN_ALIGNMENT_BYTES: u32 = 0x0100_0000;
```
Header field: `kernel_alignment`. Alignment unit required by a relocatable kernel.

```rs
const EBDA_START: u64 = 0x0009_fc00;
```
Start address for the EBDA (Extended Bios Data Area). Older computers (like the one this VMM emulates) typically use 1 KiB for the EBDA, starting at 0x9fc00.

* MEMORY LAYOUT

The traditional memory map for the kernel loader, used for Image or
zImage kernels, typically looks like:

```
	    |			            |                                           
0A0000	+------------------------+
	    |  Reserved for BIOS	|	Do not use.  Reserved for BIOS EBDA.    
09A000	+------------------------+                                          
	    |  Command line		    |                                           
	    |  Stack/heap		    |	For use by the kernel real-mode code.   
098000	+------------------------+	                                        
	    |  Kernel setup		    |	The kernel real-mode code.              
090200	+------------------------+                                          
	    |  Kernel boot sector	|	The kernel legacy boot sector.          
090000	+------------------------+                                          
	    |  Protected-mode kernel|	The bulk of the kernel image.           
010000	+------------------------+                                          
	    |  Boot loader		    |	<- Boot sector entry point 0000:7C00    
001000	+------------------------+                                          
	    |  Reserved for MBR/BIOS|                                           
000800	+------------------------+                                          
	    |  Typically used by MBR|                                           
000600	+------------------------+                                          
	    |  BIOS use only	    |                                           
000000	+------------------------+                                          
```


```rs
const E820_RAM: u32 = 1;
```
RAM memory type.
e820 is shorthand for the facility by which the BIOS of x86-based computer systems reports the memory map to the operating system or boot loader.

```rs
pub enum Error {
    E820Configuration, /// Invalid E820 configuration.
    HimemStartPastMemEnd, /// Highmem start address is past the guest memory end.
    HimemStartPastMmioGapStart, /// Highmem start address is past the MMIO gap start.
    MmioGapPastMemEnd, /// The MMIO gap end is past the guest memory end.
    MmioGapStartPastMmioGapEnd, /// The MMIO gap start is past the gap end.
}
```
These are different types of errors that can occur during the boot parameter setup.

```rs
fn add_e820_entry(
    params: &mut boot_params,
    addr: u64,
    size: u64,
    mem_type: u32,
) -> result::Result<(), Error>
```
This function sets the address, size and memory type in the e820_table of the boot parameters. It will be used in the `build_bootparams` function described next.

```rs
pub fn build_bootparams(
    guest_memory: &GuestMemoryMmap,
    kernel_load: &KernelLoaderResult,
    himem_start: GuestAddress,
    mmio_gap_start: GuestAddress,
    mmio_gap_end: GuestAddress,
) -> result::Result<boot_params, Error> 
```

This function builds boot parameters for ELF kernels following the Linux boot protocol. It takes the following arguements :
* `guest_memory` - guest memory.
* `kernel_load` - result of loading the kernel in guest memory.
* `himem_start` - address where high memory starts.
* `mmio_gap_start` - address where the MMIO gap starts.
* `mmio_gap_end` - address where the MMIO gap ends.

It checks whether `mmio_gap_start < mmio_gap_end` otherwise throws the `MmioGapStartPastMmioGapEnd` error. \
First `params` is declared as the default for `boot_params` from the linux-loader.
Then we set the headers in `params` using the constants declared earlier.

If the header copied from the bzImage file didn't set type_of_loader, we force it to "undefined"(KERNEL_LOADER_OTHER) so that the guest can boot normally.

Next we call the `add_e820_entry` function to add an entry for EBDA itself.
Now we have to add entries for the usable RAM regions (potentially surrounding the MMIO gap).\
There can be 2 cases here :
* `guest_memory.last_addr() < mmio_gap_start`, then we add an e820 entry with RAM type 1 starting at `himem_start` and ending at `last_addr+1`, if possible, otherwise throw the `HimemStartPastMemEnd` error. The unchecked + 1 is safe because, overflow could only occur if `last_addr - himem_start == u64::MAX` and since
`last_addr` is smaller than `mmio_gap_start`, a valid u64 value
`last_addr - himem_start` is also smaller than `mmio_gap_start`

* `guest_memory.last_addr() >= mmio_gap_start`, then we add an e820 entry with RAM type 1 starting at `himem_start` and ending at `mmio_gap_start`, if possible, otherwise throw the `HimemStartPastMmioGapStart` error.\
If the `last_addr` is even past the `mmio_gap_end`, then we add an e820 entry with RAM type 1 starting at `mmio_gap_end` and ending at `last_addr+1`. The unchecked_offset_from is safe, guaranteed by the `if` condition above. The unchecked + 1 is safe because overflow could only occur if `last_addr == u64::MAX` and `mmio_gap_end == 0`
and since `mmio_gap_end > mmio_gap_start`, which is a valid u64 => `mmio_gap_end > 0`

The file also contains 2 tests, `test_build_bootparams` to test the working of `build_bootparams` function and `test_add_e820_entry` to test the working of `add_e820_entry` function. The tests use various assertions to validate that the `build_bootparams` function sets the boot parameters correctly or otherwise throws the appropriate error.

