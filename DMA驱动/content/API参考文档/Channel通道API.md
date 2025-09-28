# Channel通道API

<cite>
**Referenced Files in This Document**  
- [chan.rs](file://src/chan.rs)
- [reg.rs](file://src/reg.rs)
- [lib.rs](file://src/lib.rs)
</cite>

## Table of Contents
1. [Channel结构体初始化流程](#channel结构体初始化流程)
2. [ChannelConfig配置参数详解](#channelconfig配置参数详解)
3. [通道激活与去激活操作](#通道激活与去激活操作)
4. [缓冲区安全访问模式](#缓冲区安全访问模式)
5. [寄存器调试与运行状态查询](#寄存器调试与运行状态查询)

## Channel结构体初始化流程

`Channel::new()`方法在创建通道实例时执行完整的初始化流程。该过程首先根据`ChannelConfig`中的`blk_size`参数分配`DVec<u8>`缓冲区，并确保其具有128字节的DDR地址对齐要求，以满足DMA硬件访问的物理内存对齐约束。

初始化过程中会进行严格的参数验证：检查分配的缓冲区总线地址是否为4字节对齐，若不满足则返回`None`；同时验证传输块大小`blk_size`必须大于等于4字节且为4的整数倍，这是为了保证数据传输的原子性和总线协议兼容性。

当检测到通道控制寄存器（CTL）的`CHALX_EN`位已被置位时，表明通道处于活动状态，此时会调用私有`reset()`方法执行软复位操作。初始化流程最后将配置信息写入通道专用寄存器组：包括DDR内存的高低地址、外设设备地址、传输大小以及根据方向配置`CHALX_MODE`位。

**Section sources**
- [chan.rs](file://src/chan.rs#L0-L95)

## ChannelConfig配置参数详解

`ChannelConfig`结构体定义了通道工作的核心参数：

- **slave_id**: 外设从设备ID，取值范围为0-31，用于在`DDMA`控制器中选择对应的DMA请求源信号，需与硬件设计中的外设编号匹配。
- **direction**: 传输方向枚举，`MemoryToDevice`表示内存到设备（TX模式），`DeviceToMemory`表示设备到内存（RX模式），直接影响`CHALX_MODE`控制位的设置。
- **blk_size**: 单次传输的数据块大小，单位为字节，必须满足≥4且为4的倍数的约束条件。
- **dev_addr**: 外设端的数据地址，对于内存到设备传输是目标地址，对于设备到内存传输是源地址。
- **irq**: 中断使能标志，当设置为`true`时启用该通道的传输完成中断，由`DDMA`控制器在`new_channel`时配置中断掩码寄存器。

这些配置参数共同决定了DMA通道的工作模式和数据流方向。

**Section sources**
- [chan.rs](file://src/chan.rs#L0-L56)
- [lib.rs](file://src/lib.rs#L104-L136)

## 通道激活与去激活操作

`active()`方法通过设置通道控制寄存器（CTL）的`CHALX_EN`位来启动数据传输。在调用此方法前，按照C参考实现惯例，应在控制器层面清除可能存在的挂起中断，以避免误触发。

`clear_and_active()`方法提供了原子化的操作优势：它接收一个可变的`DDMA`引用，首先调用`clear_transfer_complete()`清除指定通道的传输完成状态位，然后立即激活通道。这种组合操作确保了在多通道环境中状态清理与启动的时序正确性，防止因遗留的中断状态导致逻辑错误。

`deactive()`方法通过清除`CHALX_EN`位来关闭通道。虽然寄存器文档未明确要求自旋等待，但初始化流程中的`reset()`方法展示了在禁用通道后通过`spin_loop()`轮询等待`CHALX_EN`位实际被清零的实践，这体现了对硬件状态机转换延迟的谨慎处理。

**Section sources**
- [chan.rs](file://src/chan.rs#L97-L147)
- [lib.rs](file://src/lib.rs#L155-L168)

## 缓冲区安全访问模式

`buff()`和`buff_mut()`方法分别提供对内部`DVec<u8>`缓冲区的不可变和可变引用。`DVec`类型封装了物理连续的DMA安全内存，其`bus_addr()`方法返回可用于DMA引擎直接访问的总线地址。

在`no_std`环境下，这种设计实现了零拷贝数据交换的核心价值：CPU可以直接在`buff_mut()`返回的缓冲区中准备待发送数据，或从接收到的数据中读取，而无需额外的内存复制开销。`DVec`确保了内存的物理连续性和适当的对齐，使得外设DMA控制器能够高效地进行数据传输，极大提升了嵌入式系统的I/O性能。

**Section sources**
- [chan.rs](file://src/chan.rs#L111-L117)

## 寄存器调试与运行状态查询

`debug_registers()`方法输出通道所有关键寄存器的当前值，包括DDR地址、设备地址、传输大小、当前传输指针以及控制和状态寄存器。这对于诊断DMA传输故障（如地址配置错误、传输卡死）非常有用，开发者可以通过日志观察到精确的硬件状态。

`is_running()`查询方法直接映射到通道控制寄存器（CTL）的`CHALX_EN`位，通过`is_set()`操作读取该位的状态来判断通道是否处于激活运行状态。这是一种轻量级的非阻塞查询方式，可用于在应用逻辑中快速检查通道的活跃性，而不影响正在进行的DMA操作。

**Section sources**
- [chan.rs](file://src/chan.rs#L119-L147)