# CPU接口 (CPU Interface)

<cite>
**Referenced Files in This Document**   
- [gicc.rs](file://gic-driver/src/version/v2/gicc.rs)
- [mod.rs](file://gic-driver/src/version/v2/mod.rs)
</cite>

## 目录
1. [寄存器结构定义](#寄存器结构定义)
2. [初始化流程](#初始化流程)
3. [中断应答与结束](#中断应答与结束)
4. [优先级管理](#优先级管理)
5. [虚拟化支持](#虚拟化支持)
6. [多核同步](#多核同步)

## 寄存器结构定义

GICv2 CPU接口通过`CpuInterfaceReg`结构体定义了一组关键寄存器，这些寄存器用于控制和管理CPU级别的中断处理。该结构体位于`gic-driver/src/version/v2/gicc.rs`文件中，采用`register_structs!`宏进行声明，每个寄存器都有明确的内存偏移地址。

核心寄存器包括：
- **CTLR (CPU Interface Control Register)**：位于偏移`0x0000`，用于启用或禁用Group 0和Group 1中断，配置FIQ/IRQ行为以及EOI模式。
- **PMR (Priority Mask Register)**：位于偏移`0x0004`，用于设置优先级掩码，屏蔽低于特定优先级的中断。
- **IAR (Interrupt Acknowledge Register)**：位于偏移`0x000c`，用于读取当前最高优先级的待处理中断，执行中断应答操作。
- **EOIR (End of Interrupt Register)**：位于偏移`0x0010`，用于通知GIC中断处理已完成。
- **DIR (Deactivate Interrupt Register)**：位于偏移`0x1000`，在特定模式下用于显式停用中断。

这些寄存器的位域通过`register_bitfields!`宏进行详细定义，例如`IAR`寄存器包含`InterruptID`（中断ID）和`CPUID`（源CPU ID）两个字段，分别占用不同的位范围。

**Section sources**
- [gicc.rs](file://gic-driver/src/version/v2/gicc.rs#L1-L150)

## 初始化流程

`init_current_cpu()`函数负责初始化当前CPU的接口，确保其处于正确的运行状态。该函数在`gic-driver/src/version/v2/mod.rs`文件的`CpuInterface`结构体实现中定义。

初始化流程遵循严格的顺序和安全要求：
1. **禁用CPU接口**：首先将`CTLR`寄存器清零，确保在配置过程中不会意外触发中断处理。
2. **设置优先级掩码**：通过向`PMR`寄存器写入`0xFF`，允许所有优先级的中断通过，确保初始化后所有中断都处于可响应状态。
3. **启用CPU接口**：最后，通过设置`CTLR`寄存器中的`EnableGrp0`位来启用Group 0中断，完成初始化。

此顺序至关重要，因为它遵循了“先配置，后启用”的安全原则，防止在配置未完成时发生中断，从而避免系统处于不一致的状态。

**Section sources**
- [mod.rs](file://gic-driver/src/version/v2/mod.rs#L362-L380)

## 中断应答与结束

### 中断应答 (ack)

`ack()`方法用于从`IAR`寄存器中提取中断信息。该方法在`gic-driver/src/version/v2/mod.rs`文件中实现。

其工作流程如下：
1. 读取`IAR`寄存器的原始值。
2. 通过`From<u32> for Ack`的转换实现，解析出中断ID (`InterruptID`) 和源CPU ID (`CPUID`)。
3. 根据中断ID判断是否为SGI（软件生成中断），如果是，则返回包含`intid`和`cpu_id`的`Ack::SGI`枚举；否则返回`Ack::Other(intid)`。

此方法将底层的寄存器数据抽象为一个类型安全的`Ack`枚举，便于上层代码处理。

### 中断结束 (eoi) 与中断停用 (dir)

`eoi()`和`dir()`方法的行为取决于`CTLR`寄存器中`EOImodeNS`位的设置，这定义了两种不同的操作模式。

- **EOImodeNS = 0 (一步模式)**：
    - `eoi()`方法：向`EOIR`寄存器写入中断ID。此操作同时完成“降低运行优先级”和“停用中断”两个功能。
    - `dir()`方法：在此模式下，对`DIR`寄存器的访问是不可预测的（UNPREDICTABLE），不应被调用。

- **EOImodeNS = 1 (两步模式)**：
    - `eoi()`方法：向`EOIR`寄存器写入中断ID。此操作仅完成“降低运行优先级”功能。
    - `dir()`方法：向`DIR`寄存器写入中断ID。此操作专门用于“停用中断”，必须在`eoi()`之后调用。

这种模式分离允许更精细的中断管理，特别是在虚拟化环境中，可以将优先级管理和中断生命周期管理分开。

**Section sources**
- [mod.rs](file://gic-driver/src/version/v2/mod.rs#L391-L432)

## 优先级管理

### 设置优先级掩码

`set_priority_mask()`方法用于配置`PMR`寄存器，以过滤中断。其使用示例如下：

```rust
// 假设 cpu_if 是一个已初始化的 CpuInterface 实例
cpu_if.set_priority_mask(0x80); // 屏蔽优先级值大于等于 0x80 的所有中断
```

此调用会将`PMR`寄存器的`Priority`字段设置为`0x80`。GIC会将所有优先级数值大于或等于此掩码值的中断视为被屏蔽，从而阻止它们被发送到CPU。

### 查询最高优先级待处理中断

`get_highest_priority_pending()`方法用于获取当前最高优先级的待处理中断ID。其工作原理是：

1. 读取`HPPIR`（Highest Priority Pending Interrupt Register）寄存器的值。
2. 通过位操作`hppir & 0x3FF`提取低10位，这10位包含了中断ID。

此方法提供了一种快速查询机制，用于在中断处理程序中确定下一个要处理的中断，而无需遍历所有中断线。

**Section sources**
- [mod.rs](file://gic-driver/src/version/v2/mod.rs#L415-L425)

## 虚拟化支持

### TrapOp 机制

`TrapOp`结构体为虚拟化环境提供了一个非所有权的寄存器访问接口。它在`gic-driver/src/version/v2/mod.rs`中定义。

其核心思想是：
1. `TrapOp`持有一个指向`CpuInterfaceReg`的原始指针（`*mut CpuInterfaceReg`），而不是拥有`CpuInterface`的所有权。
2. 通过`const fn new()`创建，可以在`const`上下文中初始化。
3. 提供与`CpuInterface`相同的`ack()`、`eoi()`、`dir()`等方法，但内部通过`gicc()`方法安全地解引用指针来访问寄存器。

这使得`TrapOp`可以作为一个轻量级的、可共享的句柄，供虚拟机监控器（Hypervisor）或其他需要直接访问CPU接口寄存器但又不希望或不能拥有完整`CpuInterface`所有权的组件使用。

**Section sources**
- [mod.rs](file://gic-driver/src/version/v2/mod.rs#L434-L472)

## 多核同步

在多核系统中，对GIC CPU接口寄存器的访问必须考虑内存屏障（Memory Barrier）以确保操作的顺序性和可见性。

虽然在提供的代码片段中没有显式调用内存屏障指令（如`dsb`、`dmb`），但其必要性体现在：
- **写操作的顺序性**：当一个CPU写入`EOIR`或`DIR`寄存器以结束中断时，必须确保该写操作在其他CPU观察到中断状态变化之前完成。这通常需要在写操作后插入一个数据同步屏障（DSB）。
- **读操作的及时性**：当CPU读取`IAR`寄存器时，必须确保它能看到其他CPU写入的最新状态。这可能需要在读操作前插入一个数据内存屏障（DMB）。

在实际的底层实现中，这些屏障通常由硬件抽象层（HAL）或编译器隐式处理，或者在关键的寄存器访问宏中显式添加，以保证多核环境下的正确同步。