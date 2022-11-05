#Description of devices:
</br>

###Use Of Convoluted Bounds And Queues :

- Here in the method below, we are using a bound which is more convoluted so that it'll become easy for us to pass both references and smart pointers, such as mutex guards here.

- Along with this, we are also maintaining a pair of queue rx/tx.These queues are being  used to transfer packets bw VM and net devices. it is actually used along with MIMO between the guest and the devices.

- Then we'll need to explicitly set MAC address. So to make this happen We'll need a minimal config space to support setting an explicit MAC addr,on the guest interface at least. We use an empty one for now.

    



```
impl<M> Net<M>

    M: GuestAddressSpace + Clone + Send + 'static,
{
    pub fn new<B>(env: &mut Env<M, B>, args: &NetArgs) -> Result<Arc<Mutex<Self>>>
    
        
        B: DerefMut,
        B::Target: MmioManager<D = Arc<dyn DeviceMmio + Send + Sync>>,
    {
        let device_features = (1 << VIRTIO_F_VERSION_1)
            | (1 << VIRTIO_F_RING_EVENT_IDX)
            | (1 << VIRTIO_F_IN_ORDER)
            | (1 << VIRTIO_NET_F_CSUM)
            | (1 << VIRTIO_NET_F_GUEST_CSUM)
            | (1 << VIRTIO_NET_F_GUEST_TSO4)
            | (1 << VIRTIO_NET_F_GUEST_TSO6)
            | (1 << VIRTIO_NET_F_GUEST_UFO)
            | (1 << VIRTIO_NET_F_HOST_TSO4)
            | (1 << VIRTIO_NET_F_HOST_TSO6)
            | (1 << VIRTIO_NET_F_HOST_UFO);

      
        let queues = vec![
            Queue::new(env.mem.clone(), QUEUE_MAX_SIZE),
            Queue::new(env.mem.clone(), QUEUE_MAX_SIZE),
        ];

        
        let config_space = Vec::new();
        let virtio_cfg = VirtioConfig::new(device_features, queues, config_space);

        let common_cfg = CommonConfig::new(virtio_cfg, env).map_err(Error::Virtio)?;

        let net = Arc::new(Mutex::new(Net {
            cfg: common_cfg,
            tap_name: args.tap_name.clone(),
        }));

        env.register_mmio_device(net.clone())
            .map_err(Error::Virtio)?;

        Ok(net)
    }
}
```

.

</br>
</br>
</br>

###Set offload flags:
- Then we are using Set offload flags to match the relevent virtio features of the device .
- For now statically set in the constructor 


        
       
        tap.set_offload(
            bindings::TUN_F_CSUM
                | bindings::TUN_F_UFO
                | bindings::TUN_F_TSO4
                | bindings::TUN_F_TSO6,
        )
        .map_err(Error::Tap)?;
       
       
</br>
</br>
</br>


###Defining Layout Of Header :
- Here we are defining the layout of the header.
- The layout of the header is specified in the standard and is 12 bytes in size. We should define this somewhere.
       
        tap.set_vnet_hdr_size(VIRTIO_NET_HDR_SIZE as i32)
            .map_err(Error::Tap)?;

        let driver_notify = SingleFdSignalQueue {
            irqfd: self.cfg.irqfd.clone(),
            interrupt_status: self.cfg.virtio.interrupt_status.clone(),
        };
        


      