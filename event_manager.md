# Exit Handler & Event Manager

## Exit Handler
```rust
struct VmmExitHandler {
    exit_event: EventFd,
    keep_running: AtomicBool,
}
```

- The `Vmm Exit Handler` is used to exit the event manager loop.
- After `VM finish execution` it used kick method to notify event manager for an exit. `Kick` method will write some content to the *exit_event* fd to which will envoke the `callback` and it will set the *keep_running* 

```rust
impl MutEventSubscriber for VmmExitHandler {
    fn process(&mut self, events: Events, ops: &mut EventOps) {
        if events.event_set().contains(EventSet::IN) {
            self.keep_running.store(false, Ordering::Release);
        }
        if events.event_set().contains(EventSet::ERROR) {
            let _ = ops.remove(Events::new(&self.exit_event, EventSet::IN));
        }
    }
    fn init(&mut self, ops: &mut EventOps) {
        ops.add(Events::new(&self.exit_event, EventSet::IN))
            .expect("Cannot initialize exit handler.");
    }
}
```
- We are implementing `MutEventSubscriber` trait for VmmExitHandler
- This trait will allow interaction between the `Event Manager and different subscribers`. 
- All types which implement `MutEventSubscriber` need to describe two methods `init` and `process`. 

- `init(&mut self, ...)` method is used by subscriber to register the events it want to monitor (ex. IN event)

- `process(&mut self, ...)` is a `callback function `so whenever any event for which subscriber is listening get raised then `Event Manager` should know what it need to process.


## Event Manager

```rust
pub struct EventManager<T> {
    subscribers: Subscribers<T>,
    epoll_context: EpollWrapper,
    ...,
}
```

- `subscriber` : This maintains a HashMap<Subscriber_id, T> which is used to add a new subscriber and provide it a unique id.
- `epoll_context` : 
    ```rust
    struct EpollWrapper {
        epoll: Epoll,
        fd_dispatch: HashMap<RawFd, SubscriberId>,
        subscriber_watch_list: HashMap<SubscriberId, Vec<RawFd>>,
        ready_events: Vec<EpollEvent>,
    }
    
    ```
    1. `fd_dispatch` :  For a given file discriptor which subscriber id is associcated with that/
    2. `subscriber_watch_list` : For a given subscriber id it maps all the Fd




```rust 
struct WrappedExitHandler(Arc<Mutex<VmmExitHandler>>);
```

- Since this `exit handler` need to be shared between the `KvmVm` and `Event Manager` we need to make a wrapper to the exit handler 

```rust
impl WrappedExitHandler {
    fn new() -> Result<WrappedExitHandler> {
        ...
    }

    fn keep_running(&self) -> bool {
        self.0.lock().unwrap().keep_running.load(Ordering::Acquire)
    }
}
```

- Creating a object of the `WrappedExitHandler` will set *keep_running* to true and will also setup the exit_event. 

- Function `keep_running` return the bool value of the `keep_running : AtomicBool`.


```rust

impl ExitHandler for WrappedExitHandler {
    fn kick(&self) -> io::Result<()> {
        self.0.lock().unwrap().exit_event.write(1)
    }
}


```
- Also we will implement `ExitHandler trait` for `Wrapped Exit handler`. This trait will allow VM to signal that VCPUs have exited. Also this trait need clone as it will shared amongs vcpu's, thus they can call `kick()`.


```rust
    impl TryFrom<VMMConfig> for Vmm {
        ...
        let wrapped_exit_handler = WrappedExitHandler::new()?;
        ...
        let mut event_manager = EventManager::<Arc<Mutex<dyn MutEventSubscriber + Send>>>::new()
            
        event_manager.add_subscriber(wrapped_exit_handler.0.clone());
    }


```

- We are aksing the `event_manager` to add the subscriber which is of `VMMExitHandler` type.

```rust

fn add_subscriber(&mut self, subscriber: T) -> SubscriberId {
        let subscriber_id = self.subscribers.add(subscriber);
        self.subscribers
            .get_mut_unchecked(subscriber_id)
            .init(&mut self.epoll_context.ops_unchecked(subscriber_id));
        subscriber_id
    }

```

- First get the new `subscriber_id` and also added it to the subscribers.
- Then on the subscriber we call the `init`() method to which we are passing `self.epoll_context.ops_unchecked(subscriber_id)`

```rust

pub(crate) fn ops_unchecked(&mut self, subscriber_id: SubscriberId) -> EventOps {
        EventOps::new(self, subscriber_id)

```

- So this is returning a `EventOps` type of object

```rust
pub struct EventOps<'a> {
    epoll_wrapper: &'a mut EpollWrapper,
    subscriber_id: SubscriberId,
}
```

- So `EventOps` binds together `subscriber_id and epoll_wrapper` and return back this object of EventOps type to `init()` method, which then utilize it to add the events. 


```rust
>vmm/src/lib.rs

impl Vmm {
    pub fn run(&mut self) -> Result<()> {
        ...
        loop {
            match self.event_mgr.run() {
                Ok(_) => (),
                Err(e) => ...
            }
            if !self.exit_handler.keep_running() {
                break;
            }
        }
        ... 
    }
```

- Inside `VMM run` we have a loop where `event_mgr.run()` is called.

```rust

impl<S: MutEventSubscriber> EventManager<S> {
    ...
    pub fn run(&mut self) -> Result<usize> {
        self.run_with_timeout(-1)
    }

    pub fn run_with_timeout(&mut self, milliseconds: i32) -> Result<usize> {
        let event_count = self.epoll_context.poll(milliseconds)?;
        self.dispatch_events(event_count);
        ...
    }

    fn dispatch_events(&mut self, event_count: usize) {
        let default_event: EpollEvent = EpollEvent::default();

        for ev_index in 0..event_count {
            let event = self.epoll_context.ready_events[ev_index];
            let fd = event.fd();

            ...

            if let Some(subscriber_id) = self.epoll_context.subscriber_id(fd) {
                self.subscribers.get_mut_unchecked(subscriber_id).process(
                    Events::with_inner(event),
                   &mut self.epoll_context.ops_unchecked(subscriber_id),
                );
            } 
            else {
                ...
            }
        }
    }
}
```

- `dispatch_events(&mut self, event_cound)` - This function check the epoll_context.ready_events list. For each of the events it first get the file descriptor and after that it gets the `subscriber_id` against the file descriptor. Then it pass subscriber_id to the get_mut_unchecked which returns it the object of VMMExitHandler type. Notice `process(`) is called which we have registered as callback while implementing the Event Manager trait. 


```rust
> vm-vcpu/src/vm.rs

impl <EH: 'static + ExitHandler + Send> KvmVm<EH> {
    ...
    pub fn run(&mut self, vcpu_run_addr: Option<GuestAddress>) -> Result<()> {
            ...

            for (id, mut vcpu) in self.vcpus.drain(..).enumerate() {
                let vcpu_exit_handler = self.exit_handler.clone();
                let vcpu_handle = thread::Builder::new()
                    .name(format!("vcpu_{}", id))
                    .spawn(move || {
                        let _ = vcpu.run(vcpu_run_addr).unwrap();
                        let _ = vcpu_exit_handler.kick();
                        vcpu.run_state.set_and_notify(VmRunState::Exiting);
                    })
                self.vcpu_handles.push(vcpu_handle);
            }

            Ok(())
        }

        ...

    }
}   


```

- Once `Vcpu finish it's execution` the vcpu_exit_handler calls kick()

- `kick()` will write to the exit_event. This will raise the event which wrapper_exit_handler has registered to the `Event Manager`. Event Manager loop is continuously checking for events when it get this event it will envoke the process which is the `callback` for the event. In this case `keep_running` is set to `false`.

```rust
> vmm/src/lib.rs

impl Vmm {
    pub fn run(&mut self) -> Result<()> {
        ...
        loop {
            match self.event_mgr.run() {
                Ok(_) => (),
                Err(e) => ...
            }
            if !self.exit_handler.keep_running() {
                break;
            }
        }
        self.vm.shutdown();
    }
```

- Above loop will break once keep_running is set to false and `vm.shutdown()` will be called.