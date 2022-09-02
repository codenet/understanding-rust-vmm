# [KvmVm](vmm-reference/src/vm-vcpu/src/vm.rs)

```rs
/// A KVM specific implementation of a Virtual Machine.
///
/// Provides abstractions for working with a VM. Once a generic Vm trait will be available,
/// this type will become one of the concrete implementations.
pub struct KvmVm<EH: ExitHandler + Send> {
    fd: Arc<VmFd>,
```
> Supports all "vm ioctl" in
> https://www.kernel.org/doc/Documentation/virtual/kvm/api.txt such as 
> KVM_SET_USER_MEMORY_REGION, KVM_SET_TSS_ADDR, and so on.

```rs
    config: VmConfig,
```
> `VmConfig` just remember VCpus: 
> ```rs
> pub struct VmConfig {
>     pub num_vcpus: u8,
>     pub vcpus_config: VcpuConfigList,
> }
> ```


```rs
    // Only one of `vcpus` or `vcpu_handles` can be active at a time.
    // To create the `vcpu_handles` the `vcpu` vector is drained.
    // A better abstraction should be used to represent this behavior.
    vcpus: Vec<KvmVcpu>,
    vcpu_handles: Vec<JoinHandle<()>>,
    exit_handler: EH,
```
> We start one thread for each Vcpu. So whenever we start a new Vcpu run in a
thread, we get a handle to that thread called `JoinHandle` in rust. After we stop 
Vcpu, we also send a termination method to the 
> ```rs
>    /// Run the `Vm` based on the passed `vcpu` configuration.
>    ///
>    /// Returns an error when the number of configured vcpus is not the same as the number
>    /// of created vcpus (using the `create_vcpu` function).
>    ///
>    /// # Arguments
>    ///
>    /// * `vcpu_run_addr`: address in guest memory where the vcpu run starts. This can be None
>    ///  when the IP is specified using the platform dependent registers.
>    pub fn run(&mut self, vcpu_run_addr: Option<GuestAddress>) -> Result<()> {
>        if self.vcpus.len() != self.config.num_vcpus as usize {
>            return Err(Error::RunVcpus(io::Error::from(ErrorKind::InvalidInput)));
>        }
>
>        KvmVcpu::setup_signal_handler().unwrap();
>
>        for (id, mut vcpu) in self.vcpus.drain(..).enumerate() {
>            let vcpu_exit_handler = self.exit_handler.clone();
>            let vcpu_handle = thread::Builder::new()
>                .name(format!("vcpu_{}", id))
>                .spawn(move || {
>                    // TODO: Check the result of both vcpu run & kick.
>                    let _ = vcpu.run(vcpu_run_addr).unwrap();
>                    let _ = vcpu_exit_handler.kick();
>                    vcpu.run_state.set_and_notify(VmRunState::Exiting);
>                })
>                .map_err(Error::RunVcpus)?;
>            self.vcpu_handles.push(vcpu_handle);
>        }
>
>        Ok(())
>    }
>   /// Shutdown a VM by signaling the running VCPUs.
>   pub fn shutdown(&mut self) {
>       self.vcpu_run_state.set_and_notify(VmRunState::Exiting);
>       self.vcpu_handles.drain(..).for_each(|handle| {
>           #[allow(clippy::identity_op)]
>           let _ = handle.kill(SIGRTMIN() + 0);
>           let _ = handle.join();
>       })
>   }
>   ```
>  Note that rust threads do not support killing. See [[vm-shutdown]] for more
details.  

> `exit_handler` implements a simple trait with one method called `kick`. `kick`
> just sets an AtomicBool called `keep_running` to `false` which lets `impl Vmm`
> to exit gracefully. `exit_handler` is kicked when `vcpu.run()` ends. `vcpu.run` 
> [is an infinite loop](vmm-reference/src/vm-vcpu/src/vcpu/mod.rs)
> which breaks out on seeing `Shutdown` or `Hlt` in the kvm `exit_reason` loop.
> 
> ```rs
> VcpuExit::Shutdown | VcpuExit::Hlt => {
>     println!("Guest shutdown: {:?}. Bye!", exit_reason);
>     if stdin().lock().set_canon_mode().is_err() {
>         eprintln!("Failed to set canon mode. Stdin will not echo.");
>     }
>     self.run_state.set_and_notify(VmRunState::Exiting);
>     break;
> }
> ```

```rs
    vcpu_barrier: Arc<Barrier>,
    vcpu_run_state: Arc<VcpuRunState>,
}
```

> `Barrier` is passed to `KvmVcpu` so that it doesn't enter its `run` loop 
> until every `Vcpu` is ready. From `KvmVcpu.run`: 
> 
> ```rs
>     /// vCPU emulation loop.
>     ///
>     /// # Arguments
>     ///
>     /// * `instruction_pointer`: Represents the start address of the vcpu. This can be None
>     /// when the IP is specified using the platform dependent registers.
>     #[allow(clippy::if_same_then_else)]
>     pub fn run(&mut self, instruction_pointer: Option<GuestAddress>) -> Result<()> {
>         ...
>         self.run_barrier.wait();
>         'vcpu_run: loop {
>             let mut interrupted_by_signal = false;
>             match self.vcpu_fd.run() { ...
> ```
> 
> `vcpu_run_state` is also a shared object between the VM and the CPU. It
> captures the state of the CPU. If it is exiting / suspended / etc. VMM may 
> change run state to exiting in shutdown, CPU will block in its while loop 
> until the state is indeed exiting. 

Summary: So overall, KvmVm only has states related to managing multiple CPUs. 

## Methods 

When we look at the methods we learn that KvmVm manages both memory and IO even
when it itself doesn't have any states for those.

```rs
    // Helper function for creating a default VM.
    // This is needed as the same initializations needs to happen when creating
    // a fresh vm as well as a vm from a previously saved state.
    fn create_vm<M: GuestMemory>(
        kvm: &Kvm,
        config: VmConfig,
        exit_handler: EH,
        guest_memory: &M,
    ) -> Result<Self> {
        let vm_fd = Arc::new(kvm.create_vm().map_err(Error::CreateVm)?);
        let vcpu_run_state = Arc::new(VcpuRunState::default());

        let vm = KvmVm {
            vcpu_barrier: Arc::new(Barrier::new(config.num_vcpus as usize)),
            config,
            fd: vm_fd,
            vcpus: Vec::new(),
            vcpu_handles: Vec::new(),
            exit_handler,
            vcpu_run_state,
        };
        vm.configure_memory_regions(guest_memory, kvm)?;

        Ok(vm)
    }
```

`configure_memory_regions` is pretty straightforward. It calls
`set_user_memory_region` for each memory region. The point of having multiple
memory regions is to have the ability to set different flags in each slot. 
* If we mark `KVM_MEM_READONLY`, we will exit with `KVM_EXIT_MMIO` every time someone
tries to write to this memory. This is critical for handling memory mapped IO.
* If we set `KVM_MEM_LOG_DIRTY_PAGES` we can later do ioctl call with
`KVM_GET_DIRTY_LOG` to get a bitmap containing pages dirtied since the last call
to this ioctl. Bit 0 is the first page in the memory slot. This is critical to 
implement live #migration.

`create_vm` only creates an empty list of vcpus. These Vcpus are actually 
created in `new` after it calls `create_vm`.
```rs
    /// Create a new `KvmVm`.
    pub fn new<M: GuestMemory>(
        kvm: &Kvm,
        vm_config: VmConfig,
        guest_memory: &M,
        exit_handler: EH,
        bus: Arc<Mutex<IoManager>>,
    ) -> Result<Self> {
        let vcpus_config = vm_config.vcpus_config.clone();
        let mut vm = Self::create_vm(kvm, vm_config, exit_handler, guest_memory)?;

        #[cfg(target_arch = "x86_64")]
        MpTable::new(vm.config.num_vcpus)?.write(guest_memory)?;

        #[cfg(target_arch = "x86_64")]
        vm.setup_irq_controller()?;

        vm.create_vcpus(bus, vcpus_config, guest_memory)?;

        Ok(vm)
    }
```

Now that we understand memory and [CPU](vmm-reference-kvmvcpu.md).