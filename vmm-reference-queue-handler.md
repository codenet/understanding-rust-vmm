# [Queue Handler](https://github.com/codenet/vmm-reference/blob/main/src/devices/src/virtio/net/queue_handler.rs)

`Queue Handler` handles the Receiving and Transmitting Queues and it does so by using the Simple Handler.

[Simple Handler](https://github.com/codenet/vmm-reference/blob/main/src/devices/src/virtio/net/simple_handler.rs) struct contains methods (`send_frame`, `write_frame`...) for handling these queues. 

``` rs
pub struct QueueHandler<M: GuestAddressSpace> {
    pub inner: SimpleHandler<M, SingleFdSignalQueue>,
    pub rx_ioevent: EventFd,
    pub tx_ioevent: EventFd,
}
```

1. `inner`: Instance of `SimpleHandler<M, SingleFdSignalQueue>` which will be used for calling processing functions of SimpleHandler like `process_rxq`, `process_txq` and `process_tap`.

2. `EventFd`: Wrapper class around linux `eventfd` that creates an "eventfd object" which can be used as an event wait/notify mechanism by user-space applications, and by the kernel to notify user-space applications of events.

3. `rx_ioevent`: Instance of EventFd that is used to check for any error in reading of Rxq

4. `tx_ioevent`: Instance of EventFd that is used to check for any error in reading of Txq

<br>

`Constants` used for event matching

```rs
const TAPFD_DATA: u32 = 0;
const RX_IOEVENT_DATA: u32 = 1;
const TX_IOEVENT_DATA: u32 = 2;
```

1. `TAPFD_DATA`: Signifies the event of processing tap (`process_tap`)

2. `RX_IOEVENT_DATA`: Signifies the event of processing Rx queue (`process_rxq`)

3. `TX_IOEVENT_DATA`: Signifies the event of processing Tx queue (`process_txq`)

<br>

### A brief overview of all the struct methods:

1. `handle_error`: 
```rs
fn handle_error<S: AsRef<str>>(&self, s: S, ops: &mut EventOps)
```
Helper method that receives an error message to be logged and the `ops` handle which is used to unregister all events.

2. `process`:
```rs
fn process(&mut self, events: Events, ops: &mut EventOps)
```
This method performs event matching with the available events, if the event code is not registered in the event set, then it throws an error `Unexpected event_set`.
Available event codes:
* `TAPFD_DATA`: Calls `SimpleHandler.process_tap`
* `RX_IOEVENT_DATA`: First checks for error in the rx_ioevent read function, then calls `SimpleHandler.process_rxq`
* `TX_IOEVENT_DATA`: First checks for error in the tx_ioevent read function, then calls `SimpleHandler.process_txq`

3. `init`:
```rs
fn init(&mut self, ops: &mut EventOps)
```
Initializes the ops object by registering the supported events

For eg. Adding TAPFD_DATA event
```rs
ops.add(Events::with_data(
            &self.inner.tap,
            TAPFD_DATA,
            EventSet::IN | EventSet::EDGE_TRIGGERED,
        ))
        .expect("Unable to add tapfd");
```


## Usage

1. virtio/block/device.rs
```rs
let handler = Arc::new(Mutex::new(QueueHandler {
            inner,
            ioeventfd: ioevents.remove(0),
        }));
```


2. virtio/net/device.rs
```rs
let handler = Arc::new(Mutex::new(QueueHandler {
            inner,
            rx_ioevent: ioevents.remove(0),
            tx_ioevent: ioevents.remove(0),
        }));
```
