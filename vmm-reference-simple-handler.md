# [Simple Handler](./../src/devices/src/virtio/net/simple_handler.rs)

`SimpleHandler` contains methods for handling Tx/Rx queues. This handler is generalized over the implementaton of queues (does not make any assumptions the way queues are implemented). 

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

__A brief overview of all the struct variables:__

1. `driver_notify:`
    
    A SignalUsedQueue object to model the operation of signaling the driver about used events. It should implement a method named `signal_used_queue` that takes a parameter `index` and notifies the driver about index. Index values used here in SimpleHandler are `RXQ_INDEX` (for receiving) and `TXQ_INDEX` (for transmitting).

2. `rxq:`  Recieving queue of `GuestAddressSpace`
3. `rxbuf_current:` Number of bytes in `rxbuf` 
4. `rxbuf:` Recieving buffer of maximum size `MAX_BUFFER_SIZE`
5. `txq:`  Transmit queue of `GuestAddressSpace`
6. `txbuf:` Transmit buffer of maximum size `MAX_BUFFER_SIZE`
7. `tap:` Object of the Tap struct that wraps the file descriptor for the tap device so methods can run ioctls on the interface

```rs
const MAX_BUFFER_SIZE: usize = 65562;
```
<table>
<caption>
Incomping Packet Size
</caption>
    <tr>
        <th colspan="2">
        &nbsp;
        vitio
        &nbsp;&nbsp;&nbsp;
        &nbsp;&nbsp;&nbsp;&nbsp;
        ethernet header + UDP/TCP Packet
        </th>
    </tr>
    <tr>
        <td style="outline: thin solid">
        </td>
        <td style="outline: thin solid">
          &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
          &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
          &nbsp;&nbsp;&nbsp;&nbsp;
        </td>
    </tr>
    <tr>
        <td colspan="2">
        12 bits
        &nbsp;&nbsp;&nbsp;&nbsp;&thinsp;
        &nbsp;&nbsp;&nbsp;&nbsp;&thinsp;
        &nbsp;&nbsp;&nbsp;&nbsp;&thinsp;
        &nbsp;&nbsp;&nbsp;&nbsp;&thinsp;
        &nbsp;&nbsp;&nbsp;&nbsp;&thinsp;
        65550 bits
    <br>
        <div align = "center">Total = 65562 bits</dev>
      </td>
    </tr>
</table>

__A brief overview of all the struct methods:__

1. `Constructor:`

```rs
pub fn new(driver_notify: S, rxq: Queue<M>, txq: Queue<M>, tap: Tap) -> Self
```
Creates a SimpleHandler with the provided driver_notify, rxq, txq and tap. Sets rxbuf_current and all entries of Tx and Rx buffers to 0.

2. `write_frame_to_guest:`

```rs
fn write_frame_to_guest(&mut self) -> result::Result<bool, Error> 
```
Writes elements in Rx buffer to Rx queue. Also performs multiple checks in between like:
* Returns Ok(false) if unable to create iterator of Rx queue
* Throws `GuestMemory` Error if unable to write
* Warns about Rx frame being too large if buffer size is smaller then sum of size of all elements in queue 

3. `process_tap:`
```rs
pub fn process_tap(&mut self) -> result::Result<(), Error>
```

While any one of write_frame_to_guest or enable_notification from Rx queue is true, it sets rxbuf_current to the value read by tap from the Rx buffer (only if rxbuf_current is 0). Finally, if the Rx queue requires notification, notifies driver_notify with the index RXQ_INDEX

4. `send_frame_from_chain:`
```rs
fn send_frame_from_chain(
        &mut self,
        chain: &mut DescriptorChain<M::T>,
    ) -> result::Result<u32, Error>
```
Reads elements of Tx queue to Tx buffer sequentially and finally writes Tx buffer to tap.

5. `process_txq:`
```rs
pub fn process_txq(&mut self) -> result::Result<(), Error> 
```
In a loop, it does the following:
* Disable notification of Tx queue
* Call send_frame_from_chain on Iterator of Tx queue
* Notifies driver_notify with index as `TXQ_INDEX` if Tx queue needs notification
* Returns Ok if notification of Tx queue is disabled
6. `process_rxq:`
```rs
pub fn process_rxq(&mut self) -> result::Result<(), Error>
```

Disables notification on Rx queue and calls process_tap

__USAGE:__

1. [QueueHandler](./../src/devices/src/virtio/net/queue_handler.rs)
```rs
pub struct QueueHandler<M: GuestAddressSpace> {
    pub inner: SimpleHandler<M, SingleFdSignalQueue>,
```
Used to call process methods on tap and queues (`process_tap`, `process_rxq`, `process_txq`) and get appropriate error if thrown while execution of those methods

2. [Net](./../src/devices/src/virtio/net/device.rs)
```rs
fn activate(&mut self) -> Result<()> {
..
let inner = SimpleHandler::new(driver_notify, rxq, txq, tap);
```

