# Serial console module
The device manager is shared between the VMM, and Vcpus. The VMM creates an instance of the device manager of type `IOManager` and shares it with the VM which then shares it with the Vcpus through the object of `KvmVcpu` struct

```rust
pub struct Vmm {
    ...
    device_mgr: Arc<Mutex<IoManager>>,
    ...
}
pub struct KvmVcpu {
    ...
    device_mgr: Arc<Mutex<IoManager>>,
    ...
}

```

## Creation of Device Manager
```rust
let device_mgr = Arc::new(Mutex::new(IoManager::new()));
```
In `try_from` method implemented for Vmm, the device manager object is being created of `Arc<Mutex>` type over `IoManager` so that it can be shared between multiple threads.s

```rust
fn try_from(config: VMMConfig) -> Result<Self> {
        ...
        let device_mgr = Arc::new(Mutex::new(IoManager::new()));
        ...
        let mut vmm = Vmm {
            ...
            device_mgr,
            ...
        };
        vmm.add_serial_console()?;
        ...
        Ok(vmm)
    }
```

```rust
pub struct IoManager {
    pio_bus: PioBus<Arc<dyn DevicePio + Send + Sync>>,
    mmio_bus: MmioBus<Arc<dyn DeviceMmio + Send + Sync>>,
}
```
The IoManager struct contains `pio_bus` and `mmio_bus` out of which `pio_bus` is being used by the serial console.
## Registering serial console device with the device manager

```rust

impl TryFrom<VMMConfig> for Vmm {
    ...
    let interrupt_evt = EventFdTrigger::new(libc::EFD_NONBLOCK).map_err(Error::IO)?;
    let serial = Arc::new(Mutex::new(SerialWrapper(Serial::new(
    interrupt_evt.try_clone().map_err(Error::IO)?, stdout()))));
    ...
}
```

An interrupt event object is being created with [`EFD_NONBLOCK`](https://man7.org/linux/man-pages/man2/eventfd.2.html) flag which enables non blocking read/write interrupt events. The serial object is being created of type `SerialWrapper` which is a tuple struct containing single field of type `Serial` which contains an input buffer, output file descriptor along with interrupt event type and UART(Universal asynchronous receiver/transmitter) to manage the bus.

```rust
self.vm.register_irqfd(&interrupt_evt, 4)?;
```
For VM, an interrupt event is registered for `IRQ number 4`, so that it can listen to interrupts arising for read/write([IRQ setup](https://tldp.org/HOWTO/Serial-HOWTO-8.html)).

```rust
impl TryFrom<VMMConfig> for Vmm {
    ...
    let range = PioRange::new(PioAddress(0x3f8), 0x8).unwrap();
    self.device_mgr
        .lock()
        .unwrap()
        .register_pio(range, serial.clone())
        .unwrap();
    ...
}
```
The serial console object created is then registered to the device manager using `register_pio` method for the specific range of address `0x3f8..0x3f8 + 8`, so that it can store the mapping of address range to device and whenever some request for the addrress arrives, it can check for the concerned device.
```rust
impl<T> PioManager for T
where
    T: BusManager<PioAddress>,
    T::D: DevicePio,
{

    ...
    fn register_pio(&mut self, range: PioRange, device: Self::D) -> Result<(), bus::Error> {
        self.bus_mut().register(range, device)
    }
    ...
}

impl<A: BusAddress, D> Bus<A, D> {
    pub fn register(&mut self, range: BusRange<A>, device: D) -> Result<(), Error> {
        for r in self.devices.keys() {
            if range.overlaps(r) {
                return Err(Error::DeviceOverlap);
            }
        }
        self.devices.insert(range, device);
        Ok(())
    }
}
```

```rust
self.event_mgr.add_subscriber(serial);
```
The serial console is then added as a `subscriber` to the event manager initialized in the same `try_from` method.
```rust
fn add_subscriber(&mut self, subscriber: T) -> SubscriberId {
    let subscriber_id = self.subscribers.add(subscriber);
    self.subscribers.
    get_mut_unchecked(subscriber_id)
    .init(&mut self.epoll_context.ops_unchecked(subscriber_id));
    subscriber_id
}

fn init(&mut self, ops: &mut EventOps) {
    ops.add(Events::new(&stdin(), EventSet::IN))
        .expect("Failed to register serial input event");
}
```

`add_subscriber` method then adds the serial console as the subscriber and provides an `EventOps` object to the `init` method through which it can add the type of event for which it wants the event manager to listen on specified file descriptor, (in this case `stdin`). The event manager would keep on polling the special file pointed by the stdin file descriptor and as soon as it gets to know an `input` event is executed on the file pointed by the file descriptor, it invokes the process method.

```rust
fn process(&mut self, events: Events, ops: &mut EventOps) {
    let mut out = [0u8; 32];
    match stdin().read(&mut out) {
        Err(e) => {
            eprintln!("Error while reading stdin: {:?}", e);
        }
        Ok(count) => {
            let event_set = events.event_set();
            let unregister_condition =
                event_set.contains(EventSet::ERROR) | event_set.contains(EventSet::HANG_UP);
            if count > 0 {
                if self.0.enqueue_raw_bytes(&out[..count]).is_err() {
                    eprintln!("Failed to send bytes to the guest via serial input");
                }
            } else if unregister_condition {
                ops.remove(events)
                    .expect("Failed to unregister serial input");
            }
        }
    }
}
```
The `process` method reads if any data is available on the stdin and invokes `enqueue_raw_bytes` implemented in the `Serial` struct sending the data read from the stdin. The enqueue_raw_bytes method then stores the data into the buffer which will be demanded by the VM afterwards. In case of any `error` or `hang up call`, the event is de-registered.

The `device manager` object is shared with the Vcpu threads as well. In `Vcpu.run` method, the `vcpu_fd.run` method is invoked which exits on any sensitive instruction. In case of I/O input, it will exit and match with the case for the same.
```rust
VcpuExit::IoIn(addr, data) => {
    if (0x3f8..(0x3f8 + 8)).contains(&addr) {
        if self
            .device_mgr
            .lock()
            .unwrap()
            .pio_read(PioAddress(addr), data)
            .is_err()
        {
            debug!("Failed to read from serial port");
        }
    }
}
```
Here, it is checking for the same address range which was specified while registering the serial console with the device manager. Thus, if any input is demanded from the stdin, it will invoke pio_read method which is implemented for any type which implements the trait BusManager trait of type `PioAddress`. Thus, as the `IoManager` type implements the `BusManager` trait, the method will be invoked which will in turn invoke the `pio_read` method defined in the `SerialWrapper` which was used to create the serial device as it implements the trait MutDevicePio which provides the methods `pio_read` and `pio_write`. 


```rust
fn pio_read(&mut self, _base: PioAddress, offset: PioAddressOffset, data: &mut [u8]) {
    match offset.try_into() {
        Ok(offset) => self.bus_read(offset, data),
        Err(_) => debug!("Invalid serial console read offset."),
    }
}
fn bus_read(&mut self, offset: u8, data: &mut [u8]) {
    if data.len() != 1 {
        debug!("Serial console invalid data length on read: {}", data.len());
        return;
    }
    data[0] = self.0.read(offset);
}
```
`pio_read` method invokes `bus_read` method which in turn invokes `read` method for the serial object which reads from the `input_buffer` where it wrote the data through process method.


```rust
pub fn read(&mut self, offset: u8) -> u8 {
...
    DATA_OFFSET => {
        self.del_interrupt(IIR_RDA_BIT);
        let byte = self.in_buffer.pop_front().unwrap_or_default();
        if self.in_buffer.is_empty() {
            self.clear_lsr_rda_bit();
            self.events.in_buffer_empty();
        }
        self.events.buffer_read();
        byte
    }
...
```
The `byte` returned by the read method is then stored in the buffer passed passed as an argument passed by the `Vcpu_fd.run`.