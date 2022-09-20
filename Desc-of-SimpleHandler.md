# SimpleHandler

**File Path** - vmm-reference/src/devices/src/virtio/net/simple_handler.rs  

SimpleHandler object maintains a pair of queue `RX/TX`. These queues would be used for packets transfer(communication) between VM and `net` device. These queues facilitate in MMIO between guest and devices.

SimpleHandler is used in the `QueueHandler` in the net device.

```rs
pub struct SimpleHandler<M: GuestAddressSpace, S: SignalUsedQueue> {
    pub driver_notify: S,
    pub rxq: Queue<M>,
    pub rxbuf_current: usize,
    pub rxbuf: [u8; MAX_BUFFER_SIZE],
    pub txq: Queue<M>,
    pub txbuf: [u8; MAX_BUFFER_SIZE],
    pub tap: Tap,
}
```
- `driver_notify` gets an instance of `SingleFdSignalQueue` which is used in notification to driver about the used queue events. 
- `rxq`(for receiving) and `txq`(for transmiting) are instances of rust's `Queue` which is present in the [virtio-queue](https://github.com/rust-vmm/vm-virtio/blob/main/crates/virtio-queue/README.md) interface. It consist of Descriptor tables, Available Ring, Used Ring. Descriptor contains information of length and address of buffer in the guest address space. 
- `tap` is an interface which provides virtual machines access to physical network. 

`iter` method of `Queue` consumes over all descriptor heads in queue to return a chain. `DescriptorChain` can be used to parse descriptors provided by the device, which represent input or output memory areas for device I/O.

## Methods
```rs
    pub fn process_rxq(&mut self) -> result::Result<(), Error>
```
This method invokes `process_tap` methods on self.
<hr>

```rs
    pub fn process_tap(&mut self) -> result::Result<(), Error>
```
This method reads frames from the tap into the `rxbuf` of simple_handler object. 
<hr>

```rs
    fn write_frame_to_guest(&mut self) -> result::Result<bool, Error>
```
This method is invoked in `process_tap` method. It iters over the `rxq` to check for descriptors for receving. If found any, frames from  `rxbuf` are written to the address space specified by descriptors in the `DescriptionChain`.
<hr>

```rs
    pub fn process_txq(&mut self) -> result::Result<(), Error>
```
For transmission, it gets the descriptors chain by iterating over the `txq`. Here the descriptors points the address space containing the frames that needs to be transmitted. It then calls `send_frame_from_chain`
<hr>

```rs
    fn send_frame_from_chain(
        &mut self,
        chain: &mut DescriptorChain<M::T>,
    ) -> result::Result<u32, Error>
```
It iterates over chain to read data from guest address space into `txbuf`. Using `txbuf` it further writes data into the tap device. 
<hr>



