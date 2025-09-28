# API参考

<cite>
**本文档中引用的文件**  
- [lib.rs](file://src/lib.rs)
- [mci/mod.rs](file://src/mci/mod.rs)
- [mci/mci_cmd.rs](file://src/mci/mci_cmd.rs)
- [mci/mci_cmddata.rs](file://src/mci/mci_cmddata.rs)
- [mci/mci_config.rs](file://src/mci/mci_config.rs)
- [mci/mci_data.rs](file://src/mci/mci_data.rs)
- [mci_host/mod.rs](file://src/mci_host/mod.rs)
- [mci_host/mci_host_config.rs](file://src/mci_host/mci_host_config.rs)
- [mci_host/mci_host_transfer.rs](file://src/mci_host/mci_host_transfer.rs)
- [mci_host/sd/mod.rs](file://src/mci_host/sd/mod.rs)
</cite>

## 目录
1. [简介](#简介)
2. [顶层结构与函数](#顶层结构与函数)
3. [MCI控制器API](#mci控制器api)
4. [主机控制API](#主机控制api)
5. [高级用户API与内部细节](#高级用户api与内部细节)

## 简介
本API参考文档旨在全面覆盖`phytium-mci`驱动库中暴露的所有公共接口。文档重点描述了`lib.rs`中定义的顶层结构体和函数、`mci/mod.rs`中的MCI控制器API以及`mci_host/mod.rs`中的主机控制API。所有接口的参数、返回值、可能的错误以及调用示例均基于代码注释进行说明，确保开发者能够清晰地区分哪些API是为高级用户设计的，哪些是内部实现细节。

## 顶层结构与函数

`lib.rs`文件是整个驱动库的入口，它定义了核心的`Kernel` trait和一些公共的工具函数。

- **`Kernel` trait**：此trait定义了驱动与底层操作系统或内核交互所需的基本功能，包括睡眠、内存映射、缓存刷新和无效化。任何使用此驱动的系统都必须为其实现该trait。
  - **`sleep(duration: Duration)`**：使系统休眠指定的持续时间。
  - **`mmap(virt_addr: NonNull<u8>) -> u64`**：将虚拟地址映射到物理地址。
  - **`flush(addr: NonNull<u8>, size: usize)`**：刷新指定地址范围的缓存。
  - **`invalidate(addr: core::ptr::NonNull<u8>, size: usize)`**：使指定地址范围的缓存失效。

- **`set_impl!` 宏**：这是一个关键的宏，用于将`Kernel` trait的实现“注入”到驱动中。通过使用此宏，用户可以将其实现的`Kernel` trait绑定到驱动内部的`_phytium_mci_sleep`、`_phytium_mci_map`等外部函数上，从而完成驱动与系统底层功能的链接。

- **`MCI` 和 `SdCard` 结构体**：虽然`MCI`结构体在`mci/mod.rs`中定义，但其`new`方法是创建MCI控制器实例的顶层入口。`SdCard`结构体在`mci_host/sd/mod.rs`中定义，其`new`方法是创建SD卡实例的顶层入口，它封装了底层MCI主机和配置的初始化。

**Section sources**
- [lib.rs](file://src/lib.rs#L1-L84)
- [mci/mod.rs](file://src/mci/mod.rs#L1-L708)
- [mci_host/sd/mod.rs](file://src/mci_host/sd/mod.rs#L1-L799)

## MCI控制器API

`mci/mod.rs`模块提供了对MCI（MultiMedia Card Interface）硬件控制器的直接操作API。这些API是驱动的核心，负责发送命令、传输数据和管理控制器状态。

- **`config_init(&mut self, config: &MCIConfig) -> MCIResult`**：使用给定的配置初始化MCI控制器实例。如果控制器已初始化，则会发出警告。此方法会重置控制器、开启电源和时钟，并设置中断。
  - *参数*：`config` - 一个`MCIConfig`引用，包含控制器的配置信息。
  - *返回值*：`MCIResult`，表示操作是否成功。
  - *可能的错误*：`MCIError::NotInit`（设备未初始化）。
  - *示例*：`mci_instance.config_init(&config)?;`

- **`config_deinit(&mut self) -> MCIResult`**：反初始化MCI控制器实例。此方法会关闭中断、清除状态、关闭电源和时钟，并将控制器恢复到默认状态。
  - *返回值*：`MCIResult`。
  - *示例*：`mci_instance.config_deinit()?;`

- **`set_idma_list(&mut self, desc: &PoolBuffer, desc_num: u32) -> MCIResult`**：为SDIF控制器设置DMA描述符列表。此方法仅在DMA传输模式下有效。
  - *参数*：`desc` - 指向DMA描述符缓冲区的引用；`desc_num` - 描述符数量。
  - *可能的错误*：`MCIError::NotInit`（设备未初始化），`MCIError::InvalidState`（非DMA模式）。
  - *示例*：`mci_instance.set_idma_list(&desc_buffer, 1)?;`

- **`clk_freq_set(&mut self, clk_hz: u32) -> MCIResult`**：设置卡的时钟频率。此方法会根据目标频率选择合适的时序配置，更新时钟源和分频器，并重新配置时钟。
  - *参数*：`clk_hz` - 目标时钟频率（Hz）。
  - *示例*：`mci_instance.clk_freq_set(25_000_000)?; // 设置为25MHz`

- **`dma_transfer(&mut self, cmd_data: &mut MCICmdData) -> MCIResult`**：在DMA模式下启动命令和数据传输。此方法会检查控制器状态，重置FIFO和DMA，然后发送命令和数据。
  - *参数*：`cmd_data` - 一个可变的`MCICmdData`引用，包含要发送的命令和数据信息。
  - *可能的错误*：`MCIError::NoCard`（未检测到卡）。
  - *示例*：`mci_instance.dma_transfer(&mut cmd_data)?;`

- **`poll_wait_dma_end(&mut self, cmd_data: &mut MCICmdData) -> MCIResult`**：通过轮询等待DMA传输完成。此方法会持续检查中断状态，直到命令完成或数据超时。
  - *参数*：`cmd_data` - 一个可变的`MCICmdData`引用。
  - *可能的错误*：`MCIError::CmdTimeout`（命令超时）。

- **`pio_transfer(&self, cmd_data: &mut MCICmdData) -> MCIResult`** 和 **`poll_wait_pio_end(&mut self, cmd_data: &mut MCICmdData) -> MCIResult`**：这两个方法分别用于在PIO（Programmed I/O）模式下启动传输和等待传输完成。它们的功能与DMA模式下的对应方法类似，但直接通过CPU读写FIFO进行数据传输。

**Section sources**
- [mci/mod.rs](file://src/mci/mod.rs#L1-L708)
- [mci/mci_cmd.rs](file://src/mci/mci_cmd.rs#L1-L176)
- [mci/mci_cmddata.rs](file://src/mci/mci_cmddata.rs#L1-L91)

## 主机控制API

`mci_host/mod.rs`及其子模块提供了更高层次的抽象，用于管理SD卡或eMMC设备。这些API构建在MCI控制器API之上，实现了SD协议的命令序列。

- **`MCIHostConfig::new() -> Self`**：创建一个新的`MCIHostConfig`实例。该配置会根据编译时的feature（如`dma`、`pio`、`irq`、`poll`）自动设置相应的参数。
  - *示例*：`let config = MCIHostConfig::new();`

- **`MCIHost::new(dev: Box<SDIFDev>, config: MCIHostConfig) -> Self`**：使用给定的设备和配置创建一个新的`MCIHost`实例。这是初始化主机控制器的关键步骤。
  - *参数*：`dev` - 一个`SDIFDev`的Box；`config` - `MCIHostConfig`实例。
  - *示例*：`let host = MCIHost::new(Box::new(sdif_device), config);`

- **`SdCard::new(addr: NonNull<u8>, iopad: IoPad) -> Self`**：创建一个新的`SdCard`实例。这是使用该驱动的最顶层入口，它会自动初始化底层的`MCIHost`和`MCICardBase`。
  - *参数*：`addr` - 寄存器基地址；`iopad` - I/O引脚配置。
  - *示例*：`let sd_card = SdCard::new(reg_addr, iopad);`

- **`SdCard::init(&mut self, addr: NonNull<u8>) -> MCIHostStatus`**：初始化SD卡。此方法会执行完整的卡识别和初始化流程，包括发送`CMD0`、`CMD8`、`ACMD41`等命令，以使卡进入数据传输模式。
  - *参数*：`addr` - 寄存器基地址。
  - *返回值*：`MCIHostStatus`，表示初始化是否成功。
  - *可能的错误*：`MCIHostError::CardDetectFailed`（卡检测失败），`MCIHostError::CardInitFailed`（卡初始化失败）。
  - *示例*：`sd_card.init(reg_addr)?;`

- **`card_init_proc(&mut self) -> MCIHostStatus`**：`init`方法的内部实现，包含了完整的卡初始化流程。对于高级用户，了解此流程有助于调试。
  - *流程*：复位变量 -> 设置总线宽度为1位 -> 设置时钟为400KHz -> 探测总线电压 -> 发送`ALL_SEND_CID`（CMD2）-> 发送`SEND_RCA`（ACMD3）-> 发送`SEND_CSD`（CMD9）-> 选择卡（CMD7）-> 设置更高时钟 -> 发送`SEND_SCR`（ACMD51）-> 设置4位总线宽度（如果支持）-> 读取卡状态 -> 设置块大小 -> 选择总线时序。

**Section sources**
- [mci_host/mod.rs](file://src/mci_host/mod.rs#L1-L198)
- [mci_host/mci_host_config.rs](file://src/mci_host/mci_host_config.rs#L1-L83)
- [mci_host/sd/mod.rs](file://src/mci_host/sd/mod.rs#L1-L799)

## 高级用户API与内部细节

本节旨在帮助开发者区分公共API和内部实现。

- **供高级用户使用的API**：
  - `SdCard::new` 和 `SdCard::init`：这是应用层开发者最常使用的接口，提供了开箱即用的SD卡初始化功能。
  - `MCIHostConfig::new`：用于创建主机配置，允许用户通过编译feature来定制行为。
  - `MCI`结构体的`dma_transfer`和`pio_transfer`：对于需要直接控制数据传输的高级应用，这些方法提供了直接的访问通道。

- **内部实现细节（不建议直接调用）**：
  - `MCI`结构体的`private_cmd_send`和`private_cmd11_send`：这些是私有方法，用于发送底层命令，被`cmd_transfer`等公共方法所使用。
  - `MCI`结构体的`reset`、`poll_wait_busy_card`等方法：这些是控制器级别的内部操作，通常由`config_init`等高层方法调用。
  - `SdCard`结构体的`card_init_proc`、`bus_voltage_prob`等方法：这些是SD卡初始化流程的内部步骤，虽然逻辑清晰，但直接调用它们可能会破坏状态机。
  - 所有标记为`pub(crate)`的函数和结构体：这些是模块内部的公共接口，仅供同一crate内的其他模块使用，不应在外部代码中直接引用。

**Section sources**
- [mci/mod.rs](file://src/mci/mod.rs#L1-L708)
- [mci_host/sd/mod.rs](file://src/mci_host/sd/mod.rs#L1-L799)