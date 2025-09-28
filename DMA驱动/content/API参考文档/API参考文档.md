# API参考文档

<cite>
**Referenced Files in This Document**  
- [lib.rs](file://src/lib.rs)
- [chan.rs](file://src/chan.rs)
- [reg.rs](file://src/reg.rs)
</cite>

## 目录
1. [DDMA控制器](#ddma控制器)
2. [Channel通道](#channel通道)
3. [DmaDirection枚举](#dmadirection枚举)
4. [DmaChannelConfig配置结构体](#dmachannelconfig配置结构体)
5. [IrqHandler中断处理](#irqhandler中断处理)

## DDMA控制器

`DDMA` 结构体是飞腾（Phytium）DDMA驱动的核心控制器，负责管理整个DMA控制器的生命周期和通道分配。它通过内存映射寄存器与硬件交互。

### new方法
创建一个新的DDMA实例。
- **参数**: `base_addr: NonNull<u8>` - 指向DMA控制器寄存器基地址的非空指针。
- **行为语义**: 初始化DDMA结构体，将提供的基地址转换为指向`DdmaRegister`结构体的指针。
- **返回值**: 返回一个新创建的`DDMA`实例。
- **示例调用**: 
  ```rust
  let base_addr = NonNull::new(0x0002_8003_000 as *mut u8).unwrap();
  let ddma = DDMA::new(base_addr);
  ```
- **错误条件**: 无直接错误返回，但传入空指针会导致未定义行为。

**Section sources**
- [lib.rs](file://src/lib.rs#L52-L57)

### reset方法
重置DMA控制器及其所有通道。
- **参数**: 无。
- **行为语义**: 
  1. 禁用DMA控制器。
  2. 禁用全局中断。
  3. 对每个已绑定的通道，解除绑定、屏蔽中断、清除完成状态，并重置其配置。
  4. 执行软件复位（先置位再清零复位位）。
  5. 屏蔽所有中断。
- **返回值**: 无。
- **资源管理规则**: 此操作会释放所有通道的绑定状态，使其可被重新分配。
- **示例调用**: `ddma.reset();`

**Section sources**
- [lib.rs](file://src/lib.rs#L65-L98)

### enable方法
启用DMA控制器。
- **参数**: 无。
- **行为语义**: 
  1. 清除全局中断屏蔽位（启用全局中断）。
  2. 置位DMA使能位。
- **返回值**: 无。
- **使用前提**: 通常在完成控制器和通道的初始化后调用。
- **示例调用**: `ddma.enable();`

**Section sources**
- [lib.rs](file://src/lib.rs#L100-L103)

### disable方法
禁用DMA控制器。
- **参数**: 无。
- **行为语义**: 清除DMA使能位，停止控制器工作。
- **返回值**: 无。
- **使用场景**: 在修改通道配置前，需要先禁用控制器以确保操作安全。
- **示例调用**: `ddma.disable();`

**Section sources**
- [lib.rs](file://src/lib.rs#L105-L109)

### new_channel方法
创建并初始化一个新的DMA通道。
- **参数**: 
  - `n: u8` - 通道号 (0-7)。
  - `config: ChannelConfig` - 通道配置。
- **行为语义**: 
  1. 断言检查通道号和外设ID的有效性。
  2. 检查指定通道是否已被占用（通过`is_channel_bind`）。
  3. 如果通道被占用，返回`None`。
  4. 禁用DMA控制器。
  5. 计算该通道寄存器的内存地址。
  6. 调用`Channel::new`创建通道对象。
  7. 配置通道选择、绑定状态和中断屏蔽。
- **返回值**: 成功时返回`Some(Channel)`，失败时（如通道已占用或配置无效）返回`None`。
- **可能的错误条件**: 
  - 通道号超出范围 (0-7)，触发断言。
  - 外设ID超出范围 (0-31)，触发断言。
  - 通道已被占用，返回`None`。
- **资源管理规则**: 成功调用此方法会将通道标记为“已绑定”，防止其他代码重复分配。
- **示例调用**: 
  ```rust
  let config = ChannelConfig {
      slave_id: peripheral_ids::UART0_TX,
      direction: DmaDirection::MemoryToDevice,
      timeout_count: 1000,
      blk_size: 1024,
      dev_addr: uart_tx_register_addr,
      irq: true,
  };
  let channel = ddma.new_channel(0, config).expect("Failed to create channel");
  ```

**Section sources**
- [lib.rs](file://src/lib.rs#L111-L154)

## Channel通道

`Channel` 结构体代表一个独立的DMA传输通道，封装了特定通道的寄存器访问和数据缓冲区。

### active方法
激活通道，开始数据传输。
- **参数**: 无。
- **行为语义**: 置位通道控制寄存器中的`CHALX_EN`位，启动DMA传输。
- **返回值**: 无。
- **使用方式**: 在配置好通道并启用DMA控制器后调用。
- **示例调用**: `channel.active();`

**Section sources**
- [chan.rs](file://src/chan.rs#L88-L91)

### deactive方法
停用通道，停止数据传输。
- **参数**: 无。
- **行为语义**: 清除通道控制寄存器中的`CHALX_EN`位，停止DMA传输。
- **返回值**: 无。
- **使用方式**: 用于暂停或终止正在进行的传输。
- **示例调用**: `channel.deactive();`

**Section sources**
- [chan.rs](file://src/chan.rs#L93-L96)

## DmaDirection枚举

表示DMA传输的方向。

- **MemoryToDevice**: 内存到设备（TX），数据从系统内存传输到外设。
- **DeviceToMemory**: 设备到内存（RX），数据从外设传输到系统内存。

**Section sources**
- [lib.rs](file://src/lib.rs#L18-L25)

## DmaChannelConfig配置结构体

此结构体用于配置DMA通道的基本参数。注意：在`chan.rs`中实际使用的配置结构体名为`ChannelConfig`，但`lib.rs`中定义了同名的`DmaChannelConfig`，可能存在冗余或版本差异。

- **channel**: 通道号 (0-7)。
- **peripheral_id**: 外设从机ID (0-31)，用于选择DMA请求源。
- **direction**: 传输方向 (`DmaDirection`)。
- **timeout_enable**: 是否启用超时机制。
- **timeout_count**: 超时计数值。

**Section sources**
- [lib.rs](file://src/lib.rs#L27-L37)

## IrqHandler中断处理

`IrqHandler` 结构体用于处理来自DDMA控制器的硬件中断。

### handle_irq方法
处理DMA中断并返回已完成的通道信息。
- **参数**: 无。
- **行为语义**: 
  1. 读取DMA状态寄存器(`DMA_STAT`)。
  2. 检查每个通道的状态位，如果某通道的传输完成位被置起，则将其记录在`CompletedChannels`位掩码中。
  3. 向状态寄存器写入全1，清除所有通道的完成状态（写1清零）。
- **返回值**: `CompletedChannels` - 一个包含已完成通道位掩码的结构体。
- **使用方式**: 在中断服务程序(ISR)中调用此方法来确定哪些通道完成了传输，并进行相应的数据处理或回调。
- **示例调用**: 
  ```rust
  fn dma_isr() {
      let handler = ddma.irq_handler();
      let completed = handler.handle_irq();
      for chan_id in 0..8 {
          if completed.is_channel_completed(chan_id) {
              // 处理通道 chan_id 的完成事件
              process_transfer_completion(chan_id);
          }
      }
  }
  ```

**Section sources**
- [lib.rs](file://src/lib.rs#L217-L254)