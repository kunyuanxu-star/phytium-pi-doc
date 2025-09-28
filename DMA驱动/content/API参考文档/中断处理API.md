# 中断处理API

<cite>
**Referenced Files in This Document**   
- [lib.rs](file://src/lib.rs)
- [reg.rs](file://src/reg.rs)
</cite>

## 目录
1. [简介](#简介)
2. [核心组件](#核心组件)
3. [中断处理流程](#中断处理流程)
4. [CompletedChannels结构体详解](#completedchannels结构体详解)
5. [集成与使用建议](#集成与使用建议)

## 简介
本文档详细描述了飞腾DDMA（Direct Memory Access）驱动中的中断处理机制。该机制通过`IrqHandler`结构体提供了一套安全、高效的中断处理接口，允许上层代码在中断服务例程（ISR）中快速获取DMA传输完成状态，并对多个通道的完成事件进行批量处理。文档重点阐述了中断句柄的创建方式、硬件状态读取与清除行为，以及如何解析和利用返回的状态信息。

## 核心组件

`IrqHandler`结构体是DDMA中断处理的核心，通过`DDMA::irq_handler()`方法创建。该方法返回一个包含控制器寄存器引用的安全句柄，其设计确保了线程安全性。

`IrqHandler`被标记为`Send`和`Sync`，表明其实例可以在多线程环境中安全地发送和共享。这一设计意图是为了适应现代操作系统中断框架的需求，允许多个CPU核心或任务安全地访问同一个中断处理实例，从而支持更复杂的并发场景。

**Section sources**
- [lib.rs](file://src/lib.rs#L260-L268)

## 中断处理流程

### 获取中断句柄
中断处理的第一步是从`DDMA`控制器实例获取`IrqHandler`。这通过调用`irq_handler()`方法实现：

```rust
let irq_handler = dma_controller.irq_handler();
```

此方法不消耗`self`，而是创建一个共享的、不可变的寄存器引用，保证了原始控制器实例的可用性。

### 处理中断请求
`handle_irq()`是处理中断的核心方法。当系统中断触发时，应立即调用此方法。

该方法执行以下关键操作：
1.  **读取状态寄存器**：从`DMA_STAT`寄存器读取当前所有通道的完成状态位。
2.  **转换为位掩码**：将`CHAL0_SEL`至`CHAL7_SEL`八个状态位逐一检查，并将已完成的通道编号记录到`CompletedChannels`结构体的内部`u8`位掩码中。
3.  **自动清除状态**：在返回结果前，向`DMA_STAT`寄存器写入全1值（`u32::MAX`），根据硬件规范，这会一次性清除所有已置位的通道完成标志。

这种“读取-转换-清除”的原子化操作确保了中断状态不会丢失，即使在高频率的DMA传输场景下也能可靠工作。

```mermaid
flowchart TD
A[调用 handle_irq()] --> B[读取 DMA_STAT 寄存器]
B --> C{检查 CHAL0_SEL}
C --> |是| D[设置 CompletedChannels 位0]
C --> |否| E{检查 CHAL1_SEL}
D --> F
E --> |是| G[设置 CompletedChannels 位1]
E --> |否| H{...}
G --> F
H --> I[检查所有8个通道]
I --> J[向 DMA_STAT 写入 0xFFFFFFFF]
J --> K[返回 CompletedChannels 实例]
```

**Diagram sources**
- [lib.rs](file://src/lib.rs#L270-L300)

**Section sources**
- [lib.rs](file://src/lib.rs#L270-L300)
- [reg.rs](file://src/reg.rs#L90-L120)

## CompletedChannels结构体详解

`CompletedChannels`是一个轻量级的结构体，用于封装从硬件读取的多个通道完成状态。

### 方法说明
-   **`bitmask() -> u8`**: 返回一个8位的位掩码，其中每一位对应一个DMA通道（位0对应通道0，位7对应通道7）。如果某位为1，则表示对应的通道已完成传输。此方法便于进行位运算或日志记录。
-   **`is_channel_completed(channel: u8) -> bool`**: 提供一种更直观的方式来查询特定通道是否完成。它接受一个通道编号（0-7），并返回布尔值。对于需要针对特定通道执行不同逻辑的应用程序来说，这是最推荐的查询方式。

### 在中断上下文中的高效处理示例
以下伪代码展示了如何在中断服务程序中高效地处理多个完成事件：

```rust
// 假设已在初始化时获取了 irq_handler
fn dma_isr() {
    let completed = irq_handler.handle_irq(); // 一行代码完成读取和清除
    
    // 高效处理每个可能完成的通道
    for channel_id in 0..8 {
        if completed.is_channel_completed(channel_id) {
            // 执行该通道的后续处理，如释放资源、通知用户等
            process_completed_transfer(channel_id);
        }
    }
}
```

这种方法避免了逐个轮询寄存器的开销，通过一次硬件访问和一次函数调用即可处理所有待决的完成事件，极大提高了中断处理效率。

**Section sources**
- [lib.rs](file://src/lib.rs#L269-L289)

## 集成与使用建议

### 调用时机要求
`handle_irq()`方法必须在中断服务例程（ISR）中尽快调用。由于该方法会清除`DMA_STAT`寄存器中的所有状态位，延迟调用可能导致新的中断到来时覆盖旧的状态，从而造成状态丢失。最佳实践是在ISR的入口处立即调用此方法，以确保及时捕获所有完成事件。

### 用户责任
该API仅负责**检测和清除**硬件中断状态。驱动本身**不包含**任何后续的资源管理或数据处理逻辑。用户有责任在确认通道完成后，自行实现以下逻辑：
- 释放为该次传输分配的内存缓冲区。
- 更新相关的数据结构或状态机。
- 通知上层应用程序传输已完成。
- 启动下一次传输（如果需要）。

这种设计分离了中断处理的底层硬件交互和上层业务逻辑，使得驱动更加通用和灵活。