# API参考

<cite>
**Referenced Files in This Document**   
- [lib.rs](file://igb/src/lib.rs)
- [ring/mod.rs](file://igb/src/ring/mod.rs)
- [ring/rx.rs](file://igb/src/ring/rx.rs)
- [ring/tx.rs](file://igb/src/ring/tx.rs)
- [descriptor.rs](file://igb/src/descriptor.rs)
- [err.rs](file://igb/src/err.rs)
</cite>

## 目录
1. [Igb结构体公共方法](#igb结构体公共方法)
2. [Request结构体设计](#request结构体设计)
3. [RxRing和TxRing流类型](#rxring和txring流类型)
4. [公开枚举与类型别名](#公开枚举与类型别名)
5. [错误类型DError](#错误类型derror)

## Igb结构体公共方法

`Igb` 结构体是Intel IGB以太网驱动的核心，提供了对网络接口的完整控制。以下为其所有公共方法的详细说明。

### new
```rust
pub fn new(iobase: NonNull<u8>) -> Result<Self, DError>
```
**参数**:  
- `iobase`: 设备内存映射I/O基地址的非空指针。

**返回值**:  
- `Result<Self, DError>`: 成功时返回初始化的`Igb`实例，失败时返回`DError`。

**前置条件**:  
- 提供的`iobase`必须指向有效的设备MMIO区域。
- 系统处于可以安全访问硬件寄存器的状态。

**副作用**:  
- 初始化MAC和PHY子系统。
- 为接收和发送环分配内部地址数组。

**Section sources**
- [lib.rs](file://igb/src/lib.rs#L60-L72)

### open
```rust
pub fn open(&mut self) -> Result<(), DError>
```
**参数**:  
- `&mut self`: 可变的`Igb`实例引用。

**返回值**:  
- `Result<(), DError>`: 成功时返回空元组，失败时返回`DError`。

**前置条件**:  
- `Igb`实例必须已通过`new`方法成功创建。

**副作用**:  
- 执行完整的硬件重置流程。
- 禁用并重新启用中断。
- 配置链路模式，启动PHY电源，等待自动协商完成。
- 启用MAC层的接收和发送功能。

**Section sources**
- [lib.rs](file://igb/src/lib.rs#L74-L119)

### new_ring
```rust
pub fn new_ring(&mut self) -> Result<(TxRing, RxRing), DError>
```
**参数**:  
- `&mut self`: 可变的`Igb`实例引用。

**返回值**:  
- `Result<(TxRing, RxRing), DError>`: 成功时返回一对新创建的发送和接收环，失败时返回`DError`。

**前置条件**:  
- `Igb`实例必须已通过`open`方法成功打开。

**副作用**:  
- 在指定的MMIO基址上为索引0创建新的发送和接收描述符环。
- 使用默认大小（256）初始化环。

**Section sources**
- [lib.rs](file://igb/src/lib.rs#L121-L128)

### handle_interrupt
```rust
pub unsafe fn handle_interrupt(&mut self)
```
**参数**:  
- `&mut self`: 可变的`Igb`实例引用。

**返回值**:  
- 无。

**前置条件**:  
- **此函数只能从中断处理程序中调用**。
- 调用者必须确保当前上下文是合法的中断上下文。

**副作用**:  
- 读取并确认中断状态。
- 记录中断消息用于调试。
- 处理队列相关的中断（目前仅检查队列索引）。

**安全边界**:  
这是一个`unsafe`函数，因为直接操作硬件寄存器存在风险。调用者必须保证：
1. 该函数在中断禁用或适当的同步机制下执行，以避免竞态条件。
2. 不会从普通用户代码中调用，只能由内核的中断向量跳转而来。

**Section sources**
- [lib.rs](file://igb/src/lib.rs#L170-L177)

### 其他公共方法
- `read_mac() -> MacAddr6`: 读取并返回设备的MAC地址。
- `check_vid_did(vid: u16, did: u16) -> bool`: 检查给定的厂商ID和设备ID是否被此驱动支持。
- `status() -> MacStatus`: 返回当前MAC控制器的状态。
- `enable_loopback()` / `disable_loopback()`: 启用/禁用MAC回环模式，用于测试。
- `irq_mode_legacy()`: 配置为传统中断模式。

**Section sources**
- [lib.rs](file://igb/src/lib.rs#L130-L170)

## Request结构体设计

`Request` 结构体封装了用于DMA传输的数据缓冲区，是驱动与上层协议栈交换数据的基本单元。

### 内部DVec的用途
`Request` 的核心是一个名为`buff`的`DVec<u8>`字段。`DVec`来自`dma_api`库，它代表一个可以在CPU和设备之间共享的、具有物理连续性保证的DMA缓冲区。其主要用途包括：
- **内存管理**: 自动管理底层`Vec<u8>`的生命周期。
- **DMA映射**: 在创建时将虚拟内存地址映射到物理总线地址，这是设备进行DMA操作所必需的。
- **方向控制**: 指定数据传输方向（`ToDevice` 或 `FromDevice`），以便DMA API进行正确的缓存一致性操作。

### bus_addr()方法的重要性
```rust
pub fn bus_addr(&self) -> u64
```
此方法返回`DVec`对应的物理总线地址。这个地址对于驱动至关重要，因为：
1. **硬件编程**: 发送和接收描述符中的地址字段必须填写物理总线地址，而不是虚拟地址。`bus_addr()`提供了这个关键信息。
2. **安全性**: 它抽象了复杂的DMA映射过程，使用者无需关心底层细节即可获得正确的地址。
3. **正确性**: 确保设备能准确地找到要读写的内存位置，避免因地址错误导致的数据损坏或系统崩溃。

**Section sources**
- [lib.rs](file://igb/src/lib.rs#L15-L42)

## RxRing和TxRing流类型

`RxRing`和`TxRing`是基于环形缓冲区的异步流，用于高效地处理网络数据包的接收和发送。

### 异步迭代行为
- **RxRing**: 实现了一个生产者-消费者模型。硬件作为生产者，将接收到的数据包写入环形缓冲区；驱动作为消费者，通过`next_pkt()`方法异步获取已完成的数据包。`next_pkt()`是非阻塞的，如果没有完成的数据包，它会立即返回`None`。
- **Tx