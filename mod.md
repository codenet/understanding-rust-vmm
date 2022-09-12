## Description of [src/vmm/src/config/mod.rs](https://github.com/codenet/vmm-reference/blob/main/src/vmm/src/config/mod.rs)
The VMMConfig struct implemented in this file is used in [main.rs](https://github.com/codenet/vmm-reference/blob/main/src/main.rs) file. If all the configurations are valid, then VM starts running.
This code uses CfgArgParser defined in [arg_parser.rs](https://github.com/codenet/vmm-reference/blob/main/src/vmm/src/config/arg_parser.rs). The CggArgParser is defined as below. It has a HashMap which stores 'parameter name' as key and the 'value of parameter' as value. The 'value_of' method defined in 'arg_parser.rs' file is used to get the value for a given parameter name.
```rs
pub(super) struct CfgArgParser {
    args: HashMap<String, String>,
}
```

These are the possible errors while converting the *Config objects. There can be parsing errors while configuring Kernel, Memory, Vcpus, Network, Block
```rs
pub enum ConversionError {
    /// Failed to parse the string representation for the kernel.
    ParseKernel(String),
    /// Failed to parse the string representation for guest memory.
    ParseMemory(String),
    /// Failed to parse the string representation for the vCPUs.
    ParseVcpus(String),
    /// Failed to parse the string representation for the network.
    ParseNet(String),
    /// Failed to parse the string representation for the block.
    ParseBlock(String),
}
```

The MemoryConfig struct store size of Guest Memory (which is of type u32) in MiB. By default, the size is 256MiB
```rs
pub struct MemoryConfig {
    /// Guest memory size in MiB.
    pub size_mib: u32,
}

impl Default for MemoryConfig {
    fn default() -> Self {
        MemoryConfig { size_mib: 256u32 }
    }
}
```

The string argument for Memory configuration is parsed using CfgArgParser. Then, it call 'value_of' method of arg_parser which parameter name 'size_mib' as argument. The converted value is saved in size_mb. It will raise a ConversionError if there is an error while parsing.
```rs
impl TryFrom<&str> for MemoryConfig {
    type Error = ConversionError;

    fn try_from(mem_cfg_str: &str) -> result::Result<Self, Self::Error> {
        // Supported options: `size=<u32>`
        let mut arg_parser = CfgArgParser::new(mem_cfg_str);

        let size_mib = arg_parser
            .value_of("size_mib")
            .map_err(ConversionError::new_memory)?
            .unwrap_or(256);
        arg_parser
            .all_consumed()
            .map_err(ConversionError::new_memory)?;
        Ok(MemoryConfig { size_mib })
    }
}
```
The VcpuConfig struct stores the number of Virtual CPUs (which is of type u8). By default, num = 1 
```rs
pub struct VcpuConfig {
    /// Number of vCPUs.
    pub num: u8,
}

impl Default for VcpuConfig {
    fn default() -> Self {
        VcpuConfig { num: 1u8 }
    }
}
```

Similar to the case of MemoryConfig, this uses CfgArgParser to parse. Here, argument to 'value_of' method is "num" 
```rs
impl TryFrom<&str> for VcpuConfig {
    type Error = ConversionError;

    fn try_from(vcpu_cfg_str: &str) -> result::Result<Self, Self::Error> {
        // Supported options: `num=<u8>`
        let mut arg_parser = CfgArgParser::new(vcpu_cfg_str);
        let num = arg_parser
            .value_of("num")
            .map_err(ConversionError::new_vcpus)?
            .unwrap_or_else(|| num::NonZeroU8::new(1).unwrap());
        arg_parser
            .all_consumed()
            .map_err(ConversionError::new_vcpus)?;
        Ok(VcpuConfig { num: num.into() })
    }
}
```
The KernelConfig struct store cmdline (Kernel Command line), path (path to the kernel image) and load_addr (the address where kernel should be loaded). In try_from method of this struct, it parses the string using CfgArgParser. Then gets these three attributes by passing 'cmdline', 'path', 'kernel_load_addr' as parameters respectively to 'value_of' method.
```rs
pub struct KernelConfig {
    /// Kernel command line.
    pub cmdline: Cmdline,
    /// Path to the kernel image.
    pub path: PathBuf,
    /// Address where the kernel is loaded.
    pub load_addr: u64,
}
```

The NetConfig struct store tap_name, which is the name of tap_device. In try_from method of this struct, it parses the string using CfgArgParser. Then gets these three attributes by passing 'tap' as parameter to 'value_of' method.
```rs
pub struct NetConfig {
    /// Name of tap device.
    pub tap_name: String,
}
```

The BlockConfig struct stores pub path, which is the path to block device. Then gets these three attributes by passing 'path' as parameter to 'value_of' method in 'try_from' method.
```rs
pub struct BlockConfig {
    /// Path to the block device backend.
    pub path: PathBuf,
}
```

The VMMConfig struct stores the configurations of VMM. It stores the Memory, vCPU, kernel, Network device, Block device configurations.
```rs
pub struct VMMConfig {
    /// Guest memory configuration.
    pub memory_config: MemoryConfig,
    /// vCPU configuration.
    pub vcpu_config: VcpuConfig,
    /// Guest kernel configuration.
    pub kernel_config: KernelConfig,
    /// Network device configuration.
    pub net_config: Option<NetConfig>,
    /// Block device configuration.
    pub block_config: Option<BlockConfig>,
}
```