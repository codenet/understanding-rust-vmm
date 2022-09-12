# [Block Devices](https://github.com/codenet/vmm-reference/blob/main/src/devices/src/virtio/block/device.rs)

A brief description of how block devices are implemented in [vmm-reference](https://github.com/codenet/vmm-reference)

Currently, the Block device implemented in this file only uses the Memory Mapped Input/Output (MMIO). A block device is a part of more general virtual io devices present in the `vmm-reference`. It uses many [rust-vmm](https://github.com/rust-vmm) crates and traits such as `vm-vritio` and `vm-device`. We will first see what is the structure of a block device, and then what all methods and traits this strcture implements.

## Structure

The `struct` for a Block device is present in [src/devices/src/virtio/block/device.rs](https://github.com/codenet/vmm-reference/blob/main/src/devices/src/virtio/block/device.rs)

```rs
pub struct Block<M: GuestAddressSpace> {
```
> Struct that represents a block device
```rs
    cfg: CommonConfig<M>,
```
> Holds configuration objects which are common to all current devices
```rs
    file_path: PathBuf,
```
> This is a file path that is used to create a [StdIoBackend](https://github.com/rust-vmm/vm-virtio/blob/main/crates/devices/virtio-blk/src/stdio_executor.rs) of `vm-virtio` crate in `rust-vmm`.
```rs
    read_only: bool,
```
> Flag that denotes if this block device is read only or not
```rs
    // We'll prob need to remember this for state save/restore unless we pass the info from
    // the outside.
    _root_device: bool,
}
```

## Implementations

To create a new `Block` object, it implements a public method,

```rs
// Create `Block` object, register it on the MMIO bus, and add any extra required info to
// the kernel cmdline from the environment.
pub fn new<B>(env: &mut Env<M, B>, args: &BlockArgs) -> Result<Arc<Mutex<Self>>>
```
The call chain that invokes this method is,

```
main.rs -> Vmm::try_from -> Vmm::add_block_device -> Block::new
```

The coniguration of the block device to be used in VM is passed as a command line argument. `main.rs` parses this argument and passes a configuration object to the `Block::new` via the call chain. The `Block::new` method then uses this configuration to create a new block device object.

An important trait implemented by `Block` is `VirtioDeviceActions`, which is present in [vm-vrtio](https://github.com/rust-vmm/vm-virtio/tree/main/crates/virtio-device) package of `rust-vmm`.

The description of this trait according to it's developer is,
> Helper trait that can be implemented for objects which represent virtio devices. Together with VirtioDeviceType, it enables an automatic VirtioDevice implementation for objects that also implement BorrowMut<VirtioConfig>

The methods in this trait are `activate` and `reset`. As their name suggests, these methods are used to activate and reset the device respectively.

Next three traits implemented by `Block` are,

```rs
impl<M: GuestAddressSpace + Clone + Send + 'static> Borrow<VirtioConfig<M>> for Block<M> {
    fn borrow(&self) -> &VirtioConfig<M> {
        &self.cfg.virtio
    }
}

impl<M: GuestAddressSpace + Clone + Send + 'static> BorrowMut<VirtioConfig<M>> for Block<M> {
    fn borrow_mut(&mut self) -> &mut VirtioConfig<M> {
        &mut self.cfg.virtio
    }
}

impl<M: GuestAddressSpace + Clone + Send + 'static> VirtioDeviceType for Block<M> {
    fn device_type(&self) -> u32 {
        BLOCK_DEVICE_ID
    }
}
```

> `VirtioDeviceType` is a trait implemented by virtio devices. Hence, also implemented by [Net](https://github.com/codenet/vmm-reference/blob/main/src/devices/src/virtio/net/device.rs). Because `Block` also implements `VirtioDeviceActions` trait, `Borrow` and `BorrowMut` traits are also implemeted by `Block`.

`Block` also implements `VirtioMmioDevice` trait, which is also present in [vm-vrtio](https://github.com/rust-vmm/vm-virtio/tree/main/crates/virtio-device) package of `rust-vmm`. `Block` uses default implementations of read and write operations provided by this trait. The description of `VirtioMmioDevice` according to it's developer is,
> A common interface for Virtio devices that use the MMIO transport, which also provides a default implementation of read and write operations from/to the device registers and configuration space.

The last trait implemented by `Block` is `MutDeviceMmio` of [vm-device](https://github.com/rust-vmm/vm-device/blob/main/src/lib.rs) trait in `rust-vmm`. This trait has two functions,

```rs
fn mmio_read(&mut self, _base: MmioAddress, offset: u64, data: &mut [u8]) {
    self.read(offset, data);
}
```
> Handles a read operation on the device. Takes arguments, `base`: base address on a MMIO bus, `offset`: base address' offset, and `data`: a buffer to store the read data.

```rs
fn mmio_write(&mut self, _base: MmioAddress, offset: u64, data: &[u8]) {
    self.write(offset, data);
}
```
> Handles the write operation to the device. Same arguments as read, except `data` now contains the data to be written on the device.