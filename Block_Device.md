VirtIO Devices - These are the abstraction layer that enable VM to use the hardware like disk, network on the host.
The main task of VirtIO devices is to communicate the data between VM and host.

Three main data structures are maintained to facilitate this communication.
1) Available ring - Can be read by Host and Guest but only Guest can write to it. 
It is primarily used to communicate the requests to the host.
2) Descriptor ring holds the base, length, Read/Write access, next pointer (used for chaining of multiple descriptor pointers) to the guest memory.
3) Used ring - only device can write to it contains the decriptors which 
are to be communicated to the driver (guest).

High level operation of VirtIO devices

1) The driver sends notification to the device to seek service.
2) The device reads the available ring, processes, writes in the memory pointed by 
descriptor and pastes the descriptor of memory and the len in bytes it has written in the used Ring.
3) Device notifies the driver to read the processed data.

Reference : https://blogs.oracle.com/linux/post/introduction-to-virtio

----------------------------------------------------------------------------------------------------------
The Block device models the Hard Disk.
It uses MMIO transport in vmm-reference.


vmm-reference/src/devices/src/virtio/block/device.rs 
Contains the Block object definition and functions 
```rs
pub struct Block<M: GuestAddressSpace> {
    cfg: CommonConfig<M>,
    file_path: PathBuf,
    read_only: bool,
    // We'll prob need to remember this for state save/restore unless we pass the info from
    // the outside.
    _root_device: bool,
}
```
```rs
pub struct CommonConfig<M: GuestAddressSpace> {
    pub virtio: VirtioConfig<M>,
    pub mmio: MmioConfig,
    pub endpoint: RemoteEndpoint<Subscriber>,
    pub vm_fd: Arc<VmFd>,
    pub irqfd: Arc<EventFd>,
}
```
```rs
//taken from https://github.com/rust-vmm/vm-virtio
pub struct Queue {
    /// The maximum size in elements offered by the device.
    max_size: u16,
    /// Tail position of the available ring.
    next_avail: Wrapping<u16>,
    /// Head position of the used ring.
    next_used: Wrapping<u16>,
    /// VIRTIO_F_RING_EVENT_IDX negotiated.
    event_idx_enabled: bool,
    /// The number of descriptor chains placed in the used ring via `add_used`
    /// since the last time `needs_notification` was called on the associated queue.
    num_added: Wrapping<u16>,
    /// The queue size in elements the driver selected.
    size: u16,
    /// Indicates if the queue is finished with configuration.
    ready: bool,
    /// Guest physical address of the descriptor table.
    desc_table: GuestAddress,
    /// Guest physical address of the available ring.
    avail_ring: GuestAddress,
    /// Guest physical address of the used ring.
    used_ring: GuestAddress,
}
```

FUNCTIONS
1) create_block - creates and returns the Block object
the block contains configuration (Contains a Queue object), file_path, access permission, root

2) new - Creates a block object and registers it on the MMIO bus and returns Arc<Mutex<Block>>

3) has helper functions to borrow, borrow_mut the block. Borrow gives the reference of VirtioConfig.

4) VirtioDeviceType - returns BLOCK_DEVICE_ID

5) activate - activate method to notify the device to be ready to handle IO from the guest.
Creates the objects like QueueHandler, Disk, SignalQueue that will be used by the Device

6) reset - called by guest driver when it needs to reset and release all the emulated device resources.
Not implemented for now.





vmm-reference/src/devices/src/virtio/block/inorder_handler.rs
InOrderQueueHandler - Processes the requests from the guest.
```rs
pub struct InOrderQueueHandler<M: GuestAddressSpace, S: SignalUsedQueue> {
    pub driver_notify: S,
    pub queue: Queue<M>,
    pub disk: StdIoBackend<File>,
}
```
FUNCTIONS - 
process_chain(DescriptorChain) - It processes the requests in the Descriptor chain.
process_queue - 
1) It disables the notification from the guest
2) takes the chain referenced by the Available ring calls process_chain
3) Enables the notification to receive the next request




vmm-reference/src/devices/src/virtio/block/queue_handler.rs
QueueHandler - wrapper over InOrderQueueHandler
```rs
// This object simply combines the more generic `InOrderQueueHandler` with a concrete queue
// signalling implementation based on `EventFd`s, and then also implements `MutEventSubscriber`
// to interact with the event manager. `ioeventfd` is the `EventFd` connected to queue
// notifications coming from the driver.
pub(crate) struct QueueHandler<M: GuestAddressSpace> {
    pub inner: InOrderQueueHandler<M, SingleFdSignalQueue>,
    pub ioeventfd: EventFd,
}
```
FUNCTIONS - 
1) fn process(&mut self, events: Events, ops: &mut EventOps) - this function is called by event manager 
to read the notifications and process the data requests using the InOrderQueueHandler.

2) fn init(&mut self, ops: &mut EventOps)  - Adds the ioeventfd to the EventOps with flags IOEVENT_DATA, EventSet::IN,
so that the event manager can use the ioeventfd to notify the data events.
