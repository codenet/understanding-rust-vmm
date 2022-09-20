# Description of queue-handler

### Brief about Virtio:  
It is virtualized driver for network and disk device. It improves the performance of guests by lowering guest I/O latency and
raising throughput to levels that are almost comparable to those of bare metal in KVM.

### About virtio block queue-handler:
- The virtio-block device presents a block device to the virtual machine.
- This object is used to filter the events with given condition of a block device, in case of exceptional events we need to remove the event from the QueueHandler.
- It has two components:
- inner - an Inorder Queue handler object which is used to handle requests.
- ioeventfd - a file descriptor with an underlying count within the kernel.
```rs
pub(crate) struct QueueHandler<M: GuestAddressSpace> {
    pub inner: InOrderQueueHandler<M, SingleFdSignalQueue>,
    pub ioeventfd: EventFd,
}
```

- In this process of its implementation, we match the condition of events and remove the unmatched ones from the EventOps of EventManager as follows:
  - filtering the events:
    1. events.event\_set() == EventSet::IN
    2. events.data() == IOEVENT_DATA
    3. self.ioeventfd.read().is\_err()
    4. inorderhander process\_queue() is giving error
  - Then, we remove the other event.  
- In this init of its implementation, we initializing op as EventOps by adding event of a file descriptor, associated data and EVENTSET::IN 
```rs
impl<M: GuestAddressSpace> MutEventSubscriber for QueueHandler<M> {      
    fn process(&mut self, events: Events, ops: &mut EventOps) {
    ...
    }
    fn init(&mut self, ops: &mut EventOps) {
    ...
    }
```
### About virtio net queue-handler:
- The virtio-net device use to access network adapters to the virtual machine.
- ![image](https://user-images.githubusercontent.com/70102514/189861275-e2c79fbb-8bd9-4131-9961-6fc9a81c3dfd.png)

- This object is used to filter the events with given condition of a network adapter, in case of exceptional events we need to remove the event from the QueueHandler.
- It has three components:
- inner - an Inorder Queue handler object which is used to handle requests.
- rx_ioevent - a file descriptor with an underlying count within the kernel.
- tx_ioevent - a file descriptor with an underlying count within the kernel.
```rs
pub struct QueueHandler<M: GuestAddressSpace> {
    pub inner: SimpleHandler<M, SingleFdSignalQueue>,
    pub rx_ioevent: EventFd,
    pub tx_ioevent: EventFd,
}
```

- In this handle_error of its implementation, we remove all events in ops of EventOps.  

```rs
impl<M: GuestAddressSpace> QueueHandler<M> {
    fn handle_error<S: AsRef<str>>(&self, s: S, ops: &mut EventOps) {
    ...
    }
}
```
- In this process of each of MutEventSubscriber of its implementation, we match the condition of events and remove the unmatched ones from the EventOps of EventManager as follows:
  - filtering the events:
    1. events.event\_set() == EventSet::IN
    2. no error in inorder handler processing request respectively:
      - process tap: inorder queue.process_tap()
      - process rx: inorder queue.process_rxq()
      - process tx: inorder queue.process_txq()
    3. no error in respective event read:
      - rx: rx_ioevent.read()
      - tx: tx_ioevent.read()
  - forward for exceptional events to call handle_error to remove them.
- In this init of its implementation, we initializing op as EventOps by adding event of a file descriptor(tap of inorder handler, rx_ioevent, tx_ioevent), associated data (TAPFD_DATA, RX_IOEVENT_DATA, TX_IOEVENT_DATA) and EVENTSET::IN (in case of tap process EDGE_TRIGGERED is also included)
```rs
impl<M: GuestAddressSpace> MutEventSubscriber for QueueHandler<M> {  
    fn process(&mut self, events: Events, ops: &mut EventOps) {
    ...
    }
    fn init(&mut self, ops: &mut EventOps) {
    ...
    }
```

