# Description of inoder-handler

### Brief about Virtio:  
virtIO is a virtualization standard for network and disk device drivers. It was chosen to be the main platform for IO virtualization in KVM.

### About virtio block:
- The virtio-blk device presents a block device to the virtual machine.
- Each virtio-blk device appears as a disk inside the guest.
- Virtio-blk is an abstraction layer over devices in a paravirtualized hypervisor.
- It mainly provides two abstractions:
  - Request: Contains block request parsing abstraction.
  - Executor: Contains a block request execution abstraction that is based on std::io::Read and std::io::Write.

### Inorder-handler: 

- This object is used to process the queue of a block device without making any assumptions about the notification mechanism.
- For the time being, it uses a special backend called "StdIoBackend", but in the future it will implement the more generic and flexible one.
- This queuehandler takes the request and processes it. Return the requests in the same order in which they were received.
```rs
pub struct InOrderQueueHandler<M: GuestAddressSpace, S: SignalUsedQueue> {
    pub driver_notify: S,
    pub queue: Queue<M>,
    pub disk: StdIoBackend<File>,
}
```

- In this process_queue implementation, we follow various steps:
  - First, we disable the notification event from guest drivers.  
  - Then, we process the available chain of requests through the process_chain method.  
  - If notification is enabled, then we process the next set of requests in the chain.  

```
    pub fn process_queue(&mut self) -> result::Result<(), Error> {
        
        //basically telling the way of cosuming descriptor from the queue.
        loop {
            self.queue.disable_notification()?;

            while let Some(chain) = self.queue.iter()?.next() {
                self.process_chain(chain)?;
            }

            if !self.queue.enable_notification()? {
                break;
            }
        }

        Ok(())
    }
```


- In this process_chain implementation, we take each request from the chain of requests, assuming there is no request parse error.
- Then, process each request and write it into the memory buffer.
```
fn process_chain(&mut self, mut chain: DescriptorChain<M::T>) -> result::Result<(), Error> {
        let used_len = match Request::parse(&mut chain) {
            Ok(request) => self.disk.process_request(chain.memory(), &request)?,
            Err(e) => {
                warn!("block request parse error: {:?}", e);
                0
            }
        };
```

- After processing each request, it is used to update the used ring through the chain descriptor and the total length of the descriptor chain that was used (written to). It is also sent the notification if needed, based on request.
self.queue.add_used(chain.head_index(), used_len)?;

        if self.queue.needs_notification()? {
            self.driver_notify.signal_used_queue(0);
        }
```
