## Loading the kernel into the guest memory
__File Path__ - vmm-reference/src/vmm/src/lib.rs

__Application__ - VM Migration

### load_kernel
```rust
let mut kernel_image = File::open(&self.kernel_cfg.path).map_err(Error::IO)?;
let zero_page_addr = GuestAddress(ZEROPG_START);

// Load the kernel into guest memory.
let kernel_load = match Elf::load(
    &self.guest_memory,
    None,
    &mut kernel_image,
    Some(GuestAddress(self.kernel_cfg.load_addr)),
) {
    Ok(result) => result,
    Err(loader::Error::Elf(elf::Error::InvalidElfMagicNumber)) => BzImage::load(
        &self.guest_memory,
        None,
        &mut kernel_image,
        Some(GuestAddress(self.kernel_cfg.load_addr)),
    )
    .map_err(Error::KernelLoad)?,
    Err(e) => {
        return Err(Error::KernelLoad(e));
    }
};
```
The load_kernel function goes through the following steps: 
- The function first reads the kernel image from the provided path.
- Next, the kernel imaage is loaded from the [ELF File Format](https://linuxhint.com/understanding_elf_file_format/) to the guest memory.

```rust
let mut bootparams = build_bootparams(
    &self.guest_memory,
    &kernel_load,
    GuestAddress(self.kernel_cfg.load_addr),
    GuestAddress(MMIO_GAP_START),
    GuestAddress(MMIO_GAP_END),
)
```
- Next, we have the function `build_bootparams` which helps in setting up the boot parameters required by the kernel to properly boot. This includes setting up the memory region where the kernel code can run and getting the header parameters from the kernel. Apart from this, it also helps in setting up the E820 entries in the memory which is basically the reserved regions for MMIO.

```rust
// Add the kernel command line to the boot parameters.
bootparams.hdr.cmd_line_ptr = CMDLINE_START as u32;
bootparams.hdr.cmdline_size = self.kernel_cfg.cmdline.as_str().len() as u32 + 1;

// Load the kernel command line into guest memory.
let mut cmdline = Cmdline::new(4096);
cmdline
    .insert_str(self.kernel_cfg.cmdline.as_str())
    .map_err(Error::Cmdline)?;

load_cmdline(
    &self.guest_memory,
    GuestAddress(CMDLINE_START),
    // Safe because we know the command line string doesn't contain any 0 bytes.
    &cmdline,
)
.map_err(Error::KernelLoad)?;
```

- Now, parameters for modules which are built into the kernel need to be specified on the kernel command line. Here, the same are added to the `bootparams`.

```rust
LinuxBootConfigurator::write_bootparams::<GuestMemoryMmap>(
    &BootParams::new::<boot_params>(&bootparams, zero_page_addr),
    &self.guest_memory,
)
.map_err(Error::BootConfigure)?;
```

- Finally, the boot params are written to the zeropage on address (0x7000) in the Guest Memory. 


### ELF File
- The ELF file format is a standard file format for executables which was first launched during the development of the UNIX operating system. 
- Each ELF file is made up of ELF header, program headers and then the code and data segments. Each segment contains information that is necessary for the run-time execution of the file. e.g. It contains the address where the code and data would reside in the memory while the kernel boots up. 

#### References
- [ELF](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format)