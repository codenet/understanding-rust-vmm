Rust threads do not support killing whereas pthreads do. We first overload
killing in `vmm-sys-utils`. 

> ```rs
> /// Trait for threads that can be signalled via `pthread_kill`.
> ///
> /// Note that this is only useful for signals between `SIGRTMIN()` and
> /// `SIGRTMAX()` because these are guaranteed to not be used by the C
> /// runtime.
> ///
> /// # Safety
> ///
> /// This is marked unsafe because the implementation of this trait must
> /// guarantee that the returned `pthread_t` is valid and has a lifetime at
> /// least that of the trait object.
> pub unsafe trait Killable {
>     /// Cast this killable thread as `pthread_t`.
>     fn pthread_handle(&self) -> pthread_t;
> 
>     /// Send a signal to this killable thread.
>     ///
>     /// # Arguments
>     ///
>     /// * `num`: specify the signal
>     fn kill(&self, num: c_int) -> errno::Result<()> {
>         validate_signal_num(num)?;
> 
>         // Safe because we ensure we are using a valid pthread handle,
>         // a valid signal number, and check the return result.
>         let ret = unsafe { pthread_kill(self.pthread_handle(), num) };
>         if ret < 0 {
>             return errno::errno_result();
>         }
>         Ok(())
>     }
> }
> 
> // Safe because we fulfill our contract of returning a genuine pthread handle.
> unsafe impl<T> Killable for JoinHandle<T> {
>     fn pthread_handle(&self) -> pthread_t {
>         // JoinHandleExt::as_pthread_t gives c_ulong, convert it to the
>         // type that the libc crate expects
>         assert_eq!(mem::size_of::<pthread_t>(), mem::size_of::<usize>());
>         self.as_pthread_t() as usize as pthread_t
>     }
> }
> ```

Next, we setup the kill handler in 
* `KvmVcpu.run` 
  * `KvmVcpu::setup_signal_handler().unwrap();`

```rs
   pub(crate) fn setup_signal_handler() -> Result<()> {
        extern "C" fn handle_signal(_: c_int, _: *mut siginfo_t, _: *mut c_void) {
            KvmVcpu::set_local_immediate_exit(1);
        }
        #[allow(clippy::identity_op)]
        register_signal_handler(SIGRTMIN() + 0, handle_signal)
            .map_err(Error::RegisterSignalHandler)?;
        Ok(())
    }
```

This calls set_local_immediate_exit on receiving a termination signal: 
```rs
fn set_local_immediate_exit(value: u8) {
    Self::TLS_VCPU_PTR.with(|v| {
        if let Some(vcpu) = *v.borrow() {
            // The block below modifies a mmaped memory region (`kvm_run` struct) which is valid
            // as long as the `VMM` is still in scope. This function is called in response to
            // SIGRTMIN(), while the vCPU threads are still active. Their termination are
            // strictly bound to the lifespan of the `VMM` and it precedes the `VMM` dropping.
            unsafe {
                let vcpu_ref = &*vcpu;
                vcpu_ref.vcpu_fd.set_kvm_immediate_exit(value);
            };
        }
    });
}
```