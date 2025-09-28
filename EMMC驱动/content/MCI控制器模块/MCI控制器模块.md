# MCI控制器模块

<cite>
**Referenced Files in This Document**   
- [mci_hardware.rs](file://src/mci/mci_hardware.rs)
- [regs.rs](file://src/mci/regs.rs)
- [mci_intr.rs](file://src/mci/mci_intr.rs)
- [mod.rs](file://src/mci/mod.rs)
- [consts.rs](file://src/mci/consts.rs)
</cite>

## 目录
1. [MCI控制器模块](#mci控制器模块)
2. [硬件初始化](#硬件初始化)
3. [寄存器结构体定义](#寄存器结构体定义)
4. [中断处理机制](#中断处理机制)
5. [中断配置示例](#中断配置示例)

## 硬件初始化

MCI控制器模块的硬件初始化过程主要在`mci_hardware.rs`文件中实现，通过`MCI`结构体的私有方法`reset`完成。该过程涉及时钟使能、电源管理、寄存器重置等关键操作，确保控制器处于已知的稳定状态。

初始化流程首先配置FIFO（先进先出）缓冲区和卡阈值，为数据传输做好准备。随后，通过`clock_set(false)`关闭卡时钟，并调用`init_external_clk()`方法初始化外部时钟源。电源管理方面，`power_set(true)`用于开启卡的供电，而`voltage_1_8v_set(false)`则将工作电压设置为默认的3.3V。

寄存器重置是初始化的核心环节。`ctrl_reset`方法被用来复位控制器、FIFO和DMA（直接内存访问）模块。该方法接收一个`MCICtrl`位标志作为参数，指定需要重置的具体功能模块。重置操作通过向控制寄存器写入特定的复位位来触发，并通过轮询等待复位完成，确保硬件状态已成功恢复。

**Section sources**
- [mci_hardware.rs](file://src/mci/mci_hardware.rs#L150-L220)
- [mod.rs](file://src/mci/mod.rs#L550-L620)

## 寄存器结构体定义

`regs.rs`文件利用Rust的`bitflags`宏为MCI控制器的各个硬件寄存器定义了类型安全的结构体。这些结构体（如`MCICtrl`, `MCIIntMask`, `MCIDMACIntEn`等）将底层的32位寄存器映射为具有语义的Rust类型，极大地提高了代码的可读性和安全性。

每个寄存器结构体都定义了其对应的位字段（bit field），例如`MCICtrl`结构体中的`CONTROLLER_RESET`、`FIFO_RESET`和`DMA_RESET`。开发者可以通过结构体的常量来操作寄存器，而不是直接使用易错的十六进制数值。例如，要设置控制器复位，可以使用`MCICtrl::CONTROLLER_RESET`，这比直接写`1 << 0`更加清晰和安全。

这种设计模式不仅防止了位操作错误，还提供了编译时检查。如果尝试设置一个不存在的位标志，编译器会报错。此外，`bitflags`宏还自动生成了按位与（`&`）、或（`|`）、非（`!`）等操作符的实现，使得对多个标志的组合操作变得非常直观。

**Section sources**
- [regs.rs](file://src/mci/regs.rs#L10-L800)

## 中断处理机制

`mci_intr.rs`文件实现了MCI控制器的中断处理机制，该机制分为中断配置和中断服务例程（ISR）两大部分。

中断配置通过`interrupt_mask_get`和`interrupt_mask_set`两个公共方法实现。`interrupt_mask_get`用于查询当前的中断屏蔽状态，而`interrupt_mask_set`则用于启用或禁用特定的中断源。该机制区分了两种中断类型：通用中断（`GeneralIntr`）和DMA中断（`DmaIntr`），分别对应`MCIIntMask`和`MCIDMACIntEn`寄存器。

中断服务例程`fsdif_interrupt_handler`负责处理实际发生的中断。它首先读取`MCIRawInts`（原始中断状态）和`MCIDMACStatus`（DMA状态）寄存器以获取中断源。然后，根据中断掩码和状态位，通过一系列条件判断来分发不同的中断事件，例如命令完成、数据传输完成或错误发生。处理完中断后，必须通过向状态寄存器写入其当前值来清除中断标志，这是防止中断重复触发的关键步骤。

**Section sources**
- [mci_intr.rs](file://src/mci/mci_intr.rs#L10-L177)

## 中断配置示例

以下代码示例展示了如何使用`interrupt_mask_set`方法来配置MCI控制器的中断。该示例演示了如何启用命令完成（`CMD_BIT`）和数据传输完成（`DTO_BIT`）这两个关键中断。

```rust
// 假设 `mci` 是一个已初始化的 MCI 实例
let mci = /* ... */;

// 定义要设置的中断掩码，使用位或操作组合多个中断源
let mask_to_set = MCIIntMask::CMD_BIT.bits() | MCIIntMask::DTO_BIT.bits();

// 启用指定的中断
mci.interrupt_mask_set(MCIIntrType::GeneralIntr, mask_to_set, true);

// 此时，当命令执行完成或数据传输结束时，都会触发中断
```

在此示例中，`MCIIntrType::GeneralIntr`指定了操作的是通用中断寄存器。`mask_to_set`是通过获取`MCIIntMask`枚举中对应位标志的原始位值（使用`.bits()`方法）并进行按位或操作构建的。最后一个参数`true`表示启用这些中断。若要禁用中断，可将此参数设为`false`。

**Section sources**
- [mci_intr.rs](file://src/mci/mci_intr.rs#L20-L35)