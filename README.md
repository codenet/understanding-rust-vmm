# Understanding Rust VMM

We will try to understand `vmm-reference`, at least partially. Course project
will build features on top of `vmm-reference` so please try to understand it
well.

# vmm-reference

[Design](https://github.com/rust-vmm/vmm-reference/blob/main/docs/DESIGN.md).
vmm-reference project uses kvm-ioctls, kvm-bindings, vm-memory, vm-superio, etc.
vmm-reference is relatively simpler to understand since it does not support
advanced features like VM pausing / resuming, snapshotting, and migration. 

[Introduction to rust vmm crates](https://www.youtube.com/watch?v=QUH1moScriw).
Basically, vm-memory etc are generic interfaces which get implemented by the
actual hypervisors. This is so that we can get reusability. Another point is to
pick and choose. One can build a VMM without picking the block device Rust crate.

# Setting up your system
Clone [vmm-reference](https://github.com/codenet/vmm-reference) on your Linux
machine. Make sure that you have KVM on your machine. You can check it by
running the `simple.rs` from Lecture 6 and verify that it is indeed printing 4.

If you do not have a Linux machine, you can email course TAs to get access to
one. One good way of using a remote machine, while getting IDE features, like
jumping to a rust method is to use the [SSH
extension](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-ssh) in VSCode.

# Boot Linux VM
You can download a pre-built kernel image from [here](bzimage-hello-busybox) and
launch vmm-reference as follows:

```
dell@abhilash:~/Documents/courses/col732/vmm-reference$ ./target/debug/vmm-reference --kernel path=~/Documents/courses/col732/vmm-reference/resources/kernel/bzimage-hello-busybox
....
[    0.673230] zswap: default zpool zbud not available
[    0.673738] zswap: pool creation failed
[    0.674195] Key type ._fscrypt registered
[    0.674621] Key type .fscrypt registered
[    0.675339] Key type encrypted registered
[    0.676273] Freeing unused decrypted memory: 2040K
[    0.677088] Freeing unused kernel image memory: 2800K
[    0.682218] Write protecting the kernel read-only data: 14336k
[    0.683567] Freeing unused kernel image memory: 2012K
[    0.684193] Freeing unused kernel image memory: 272K
[    0.684714] Run /init as init process
                                                   
                 _                                 
  _ __ _   _ ___| |_    __   ___ __ ___  _ __ ___  
 | '__| | | / __| __|___\ \ / / '_ \ _ \| '_ \ _ \ 
 | |  | |_| \__ \ ||_____\ V /| | | | | | | | | | |
 |_|   \__,_|___/\__|     \_/ |_| |_| |_|_| |_| |_|
                                                   
                                                   
Hello, world, from the rust-vmm reference VMM!
/ # ls
bin      etc      init     mnt      sbin     usr
dev      home     linuxrc  proc     sys
/ # echo "HELLO"
HELLO
/ # 
```

You shall see messages like above. This means your VM is running!

# Understanding vmm-reference

You can now start understanding the code. Start here:
* [[vmm-reference-kvmvcpu]]
* [[vmm-reference-kvmvm]]

The notes are not exhaustive. They are only to get you started on how to explore
the code on your own.  In this journey, ask from and listen to the following
friends:
* [Intel® 64 and IA-32 Architectures Software Developer’s Manuals](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)
* [KVM API](https://www.kernel.org/doc/Documentation/virtual/kvm/api.txt)
* [Rust-vmm crate documentation](https://crates.io/teams/github:rust-vmm:gatekeepers)
* [Rust book](https://doc.rust-lang.org/book/)

### Miscellaneous

It might be easier to navigate this repo by cloning it locally, opening it in
VSCode, and installing [Foam
extension](https://marketplace.visualstudio.com/items?itemName=foam.foam-vscode)
in VSCode.

