# PIO模式

<cite>
**Referenced Files in This Document**   
- [mci_pio.rs](file://src/mci/mci_pio.rs)
- [mci_data.rs](file://src/mci/mci_data.rs)
- [consts.rs](file://src/mci/consts.rs)
- [regs.rs](file://src/mci/regs.rs)
</cite>

## 目录
1. [PIO模式概述](#pio模式概述)
2. [核心数据结构](#核心数据结构)
3. [PIO写入操作分析](#pio写入操作分析)
4. [PIO读取操作分析](#pio读取操作分析)
5. [寄存器与常量定义](#寄存器与常量定义)
6. [PIO模式使用示例](#pio模式使用示例)
7. [PIO与DMA模式对比](#pio与dma模式对比)
8. [使用限制与建议](#使用限制与建议)

## PIO模式概述

PIO（Programmed I/O）模式是一种通过CPU直接控制外设进行数据传输的机制。在本驱动中，PIO模式用于MCI（MultiMedia Card Interface）控制器与存储卡之间的数据交换。该模式通过直接读写`MCIDataReg`寄存器与硬件FIFO（First In, First Out）缓冲区进行交互，实现对存储卡的数据读写操作。

与DMA（Direct Memory Access）模式不同，PIO模式不依赖专用的DMA控制器，而是由CPU主动发起每一次数据传输。这种模式适用于小数据量或对延迟要求极高的场景，因为它避免了DMA配置的开销。然而，其缺点是会占用大量CPU资源，尤其是在进行大数据块传输时。

本驱动中的PIO模式主要由`mci_pio.rs`文件中的`pio_write_data`和`pio_read_data`两个核心函数实现，它们分别处理数据的写入和读取流程。

**Section sources**
- [mci_pio.rs](file://src/mci/mci_pio.rs#L1-L50)

## 核心数据结构

### MCIData结构体

`MCIData`结构体是PIO模式下数据缓冲区管理的核心。它封装了传输所需的所有数据信息，包括实际的数据缓冲区、DMA地址、块大小、块计数和总数据长度。

该结构体通过`buf()`和`buf_mut()`方法提供对内部`Vec<u32>`缓冲区的安全访问。`buf()`方法返回一个不可变引用，用于读取操作；`buf_mut()`方法返回一个可变引用，用于写入操作。这种设计遵循了Rust的所有权和借用规则，确保了内存安全。

当缓冲区未初始化时，`buf()`和`buf_mut()`方法会返回`None`，调用者需要检查此返回值以避免空指针异常。`MCIData`还提供了`buf_take()`方法，用于获取并清空缓冲区的所有权，这在数据传输完成后释放资源时非常有用。

**Section sources**
- [mci_data.rs](file://src/mci/mci_data.rs#L1-L72)

## PIO写入操作分析

### pio_write_data函数

`pio_write_data`函数负责将数据从内存写入MCI控制器的FIFO缓冲区。其工作流程如下：

1.  **参数准备**：函数接收一个指向`MCIData`结构体的不可变引用，该结构体包含了待写入的数据。
2.  **寄存器获取**：通过`self.config.reg()`获取MCI控制器的寄存器操作接口。
3.  **写入次数计算**：根据`data.datalen()`返回的字节长度，计算需要进行的32位写入次数。计算公式为`datalen / 4`，因为每次写入操作处理一个32位（4字节）的`u32`值。
4.  **缓冲区访问**：调用`data.buf()`安全地获取数据缓冲区的引用。如果缓冲区未初始化，函数会立即返回`MCIError::NotInit`错误。
5.  **FIFO写入**：首先向`MCICmd`寄存器写入`DAT_WRITE`命令，通知硬件即将进行数据写入。然后，在一个循环中，将缓冲区中的每个`u32`值通过`write_reg`方法写入`MCIDataReg`寄存器，从而填充硬件FIFO。
6.  **返回结果**：所有数据写入完成后，返回`Ok(())`表示操作成功。

该函数的关键在于将字节长度精确地转换为32位写入次数，确保了数据不会被截断或越界。

**Section sources**
- [mci_pio.rs](file://src/mci/mci_pio.rs#L6-L23)

## PIO读取操作分析

### pio_read_data函数

`pio_read_data`函数负责从MCI控制器的FIFO缓冲区读取数据到内存。其工作流程如下：

1.  **参数准备**：函数接收一个指向`MCIData`结构体的可变引用，用于存储读取到的数据。
2.  **寄存器获取**：同样通过`self.config.reg()`获取寄存器操作接口。
3.  **数据长度与读取次数**：获取待读取的数据长度`datalen`，并计算32位读取次数`rd_times = datalen / 4`。
4.  **缓冲区访问与清空**：调用`data.buf_mut()`获取可变的缓冲区引用。如果缓冲区未初始化，返回`MCIError::NotInit`。随后调用`buf.clear()`清空缓冲区，为新数据做准备。
5.  **FIFO容量检查**：在读取前，函数会检查`datalen`是否超过了`MCI_MAX_FIFO_CNT`（定义为0x800字节）的限制。如果超过，会记录错误日志并返回`MCIError::NotSupport`，因为PIO模式不支持超大容量的单次传输。
6.  **FIFO读取**：在一个循环中，通过`read_reg::<MCIDataReg>()`从`MCIDataReg`寄存器读取32位数据，并将其`bits()`（原始位值）推入缓冲区。
7.  **返回结果**：所有数据读取完成后，返回`Ok(())`。

该函数的读取流程与写入类似，但增加了对FIFO容量的严格检查，这是PIO模式的一个重要限制。

**Section sources**
- [mci_pio.rs](file://src/mci/mci_pio.rs#L25-L49)

## 寄存器与常量定义

### MCIDataReg寄存器

`MCIDataReg`是定义在`regs.rs`文件中的一个`bitflags`宏，它代表了MCI控制器的数据寄存器（偏移地址`FSDIF_DATA_OFFSET`）。该寄存器直接映射到硬件FIFO，CPU通过读写此寄存器来与FIFO交互。每次读或写操作都会触发FIFO的出队或入队。

### MCI_MAX_FIFO_CNT常量

`MCI_MAX_FIFO_CNT`是在`consts.rs`文件中定义的一个常量，其值为`0x800`（2048字节）。它代表了PIO模式下允许传输的最大数据量。这个限制源于硬件FIFO的物理大小和PIO模式的设计，旨在防止缓冲区溢出和系统不稳定。

**Section sources**
- [regs.rs](file://src/mci/regs.rs#L799-L825)
- [consts.rs](file://src/mci/consts.rs#L169-L170)

## PIO模式使用示例

以下是一个使用PIO模式进行数据传输的简化代码示例：

```rust
// 1. 创建并初始化MCIData结构体
let mut data = MCIData::new();
let mut buffer = vec![0u32; 512]; // 2KB缓冲区
data.buf_set(Some(buffer));
data.datalen_set(2048); // 设置数据长度

// 2. 执行PIO写入
match mci_controller.pio_write_data(&data) {
    Ok(()) => info!("PIO write successful"),
    Err(e) => error!("PIO write failed: {:?}", e),
}

// 3. 执行PIO读取
match mci_controller.pio_read_data(&mut data) {
    Ok(()) => info!("PIO read successful"),
    Err(e) => error!("PIO read failed: {:?}", e),
}
```

此示例展示了如何准备数据、调用PIO函数以及处理返回结果。在实际应用中，还需要在调用PIO函数前配置好MCI控制器的命令和时钟等参数。

## PIO与DMA模式对比

| 特性 | PIO模式 | DMA模式 |
| :--- | :--- | :--- |
| **CPU占用率** | 高 | 低 |
| **系统吞吐量** | 较低 | 高 |
| **延迟** | 低（小数据量时） | 稍高（有配置开销） |
| **适用场景** | 小数据量、低延迟、简单操作 | 大数据块、高吞吐量传输 |
| **实现复杂度** | 简单 | 复杂（需配置描述符、处理中断） |
| **资源占用** | 占用CPU周期 | 占用DMA通道 |

**Section sources**
- [consts.rs](file://src/mci/consts.rs#L30-L36)

## 使用限制与建议

PIO模式的主要局限性在于其不支持大数据块传输。由于`MCI_MAX_FIFO_CNT`的限制，任何超过2048字节的传输请求都会被拒绝。此外，长时间的PIO传输会持续占用CPU，导致系统性能下降。

**使用建议**：
1.  **优先使用DMA**：对于大于1KB的数据传输，应优先考虑使用DMA模式，以释放CPU资源。
2.  **PIO用于控制命令**：PIO模式非常适合用于发送短小的控制命令或读取状态信息。
3.  **小数据量传输**：当传输的数据量很小（如几百字节）且对延迟敏感时，PIO模式是理想选择。
4.  **错误处理**：在调用`pio_write_data`和`pio_read_data`时，必须检查返回的`MCIResult`，以妥善处理`NotInit`和`NotSupport`等错误。

通过合理选择传输模式，可以在性能和资源消耗之间取得最佳平衡。