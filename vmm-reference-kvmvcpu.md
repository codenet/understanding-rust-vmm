# [KvmVcpu](/home/dell/Documents/courses/col732/vmm-reference/src/vm-vcpu/src/vcpu/mod.rs)

`KvmVcpu` contains methods for interacting with the VCPU. It has reference to `VcpuFd`.

```rs
pub struct KvmVcpu {
    /// KVM file descriptor for a vCPU.
    pub(crate) vcpu_fd: VcpuFd,
    /// Device manager for bus accesses.
    device_mgr: Arc<Mutex<IoManager>>,
    config: VcpuConfig,
    run_barrier: Arc<Barrier>,
    pub(crate) run_state: Arc<VcpuRunState>,
}
```

> [`VcpuFd`](/home/dell/.cargo/registry/src/github.com-1ecc6299db9ec823/kvm-ioctls-0.11.0/src/ioctls/vcpu.rs)
lives in `kvm-ioctls` and has methods to get and set various states about CPU
such as `get_regs`, `get_sregs` and so on. We need not interact with this
directly and can just talk to `KvmVcpu`.

KvmVcpu also has a `IoManager`, a `Barrier`, and a `VcpuRunState`.

> `VcpuRunState` is just a synchronization wrapper over
[`VmRunState`](/home/dell/Documents/courses/col732/vmm-reference/src/vm-vcpu/src/vm.rs).
One can wait on `VmRunState` using a conditional variable and get notified when
the `VmRunState` is changed. `VmRunState` is just enum of `Running`, `Suspending`,
`Exiting`.

> `Barrier` is shared across `KvmVcpu` so that it doesn't enter its `run` loop 
> until every `Vcpu` is ready.

`KvmVcpu` has methods to save its state and to restore it from a state. CPU
state is encapsulated in `VcpuState`. 

### pub struct VcpuState {
```rust
    pub regs: kvm_regs,
    pub sregs: kvm_sregs,
```
[[x86-registers]]

```rust
    pub cpuid: CpuId,
```
```rust
    pub msrs: Msrs,
    pub debug_regs: kvm_debugregs,
```
[[x86-debugregs]]

```rust
    pub lapic: kvm_lapic_state,
```
[[x86-lapic]]
```rust
    pub mp_state: kvm_mp_state,
```
[[x86-mpstate]]
```rust
    pub vcpu_events: kvm_vcpu_events,
```
```rust
    pub xcrs: kvm_xcrs,
```
```rust
    pub xsave: kvm_xsave,
```
```rust
    pub config: VcpuConfig,
}
```



