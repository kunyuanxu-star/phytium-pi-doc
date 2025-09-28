# 结构体API

<cite>
**Referenced Files in This Document**   
- [lib.rs](file://src/lib.rs)
- [chip/mod.rs](file://src/chip/mod.rs)
- [types/config/endpoint.rs](file://src/types/config/endpoint.rs)
- [types/config/pci_bridge.rs](file://src/types/config/pci_bridge.rs)
- [bar_alloc.rs](file://src/bar_alloc.rs)
- [types/mod.rs](file://src/types/mod.rs)
- [types/config/mod.rs](file://src/types/config/mod.rs)
</cite>

## 目录
1. [PcieGeneric结构体](#pciegeneric结构体)
2. [Endpoint结构体](#endpoint结构体)
3. [PciPciBridge结构体](#pcipci桥接器结构体)
4. [BusNumber辅助结构体](#busnumber辅助结构体)
5. [Deref/DerefMut实现](#deref-derefmut实现)

## PcieGeneric结构体

`PcieGeneric`结构体通过`pub use chip::PcieGeneric`语句在库根模块中公开导出，作为PCIe控制器的主要驱动实现。该结构体封装了MMIO（内存映射I/O）基地址，并实现了与硬件交互所需的核心接口。

其`new`方法接收一个`NonNull<u8>`类型的MMIO基地址参数，用于初始化`PcieGeneric`实例。该方法将传入的基地址存储在结构体内部，后续所有对PCI配置空间的读写操作都将基于此基地址进行偏移计算。`PcieGeneric`同时实现了`DriverGeneric`和`Interface` trait，提供了`open`、`close`、`read`和`write`等方法，这些方法构成了访问PCI设备配置空间的基础。

**Section sources**
- [lib.rs](file://src/lib.rs#L10)
- [chip/mod.rs](file://src/chip/mod.rs#L7-L52)

## Endpoint结构体

`Endpoint`结构体代表一个PCIe终端设备，是处理非桥接PCI设备的核心数据结构。它包含一个基础的`PciHeaderBase`字段和一个具体的`EndpointHeader`字段，用于解析和操作设备的配置头信息。

### 构造函数逻辑

`Endpoint::new`构造函数接收一个`PciHeaderBase`实例和一个可选的`SimpleBarAllocator`引用。函数首先使用`PciHeaderBase`中的原始头信息创建一个类型安全的`EndpointHeader`实例。如果提供了分配器，则立即调用`realloc_bar`方法，根据设备的BAR需求重新分配内存空间，确保设备能够正确访问其资源。

### BAR访问方法

`Endpoint`提供了`bar`和`bars`两个核心方法来访问设备的基地址寄存器（BAR）。`bars`方法返回一个`BarVec`枚举，其中包含了设备所有六个BAR的解析结果。`bar`方法则接受一个索引参数，从`bars`的结果中提取指定BAR的地址范围（`Range<usize>`），方便上层应用直接获取可用的内存或I/O区域。

### BAR重分配机制

`realloc_bar`方法是资源管理的关键。它首先禁用设备的I/O和内存访问命令位，然后利用传入的`SimpleBarAllocator`为每个需要内存空间的BAR分配新的物理地址。分配完成后，它会更新设备配置空间中的BAR寄存器，并重新启用内存访问命令位。这一过程确保了设备资源的动态、安全分配。

此外，`Endpoint`还提供了`set_interrupt_pin`和`set_interrupt_line`等方法，允许修改设备的中断配置。例如，可以通过`set_interrupt_pin(1)`将设备的中断引脚设置为INTA。

**Section sources**
- [types/config/endpoint.rs](file://src/types/config/endpoint.rs#L10-L237)
- [types/config/mod.rs](file://src/types/config/mod.rs#L6)

## PciPci桥接器结构体

`PciPciBridge`结构体用于表示和管理PCI到PCI的桥接设备，负责连接不同的PCI总线段。

### 总线编号更新

`update_bus_number`方法是其核心功能之一，它允许安全地修改桥接器的主总线号、次级总线号和下级总线号。该方法接收一个闭包作为参数，该闭包接收当前的`BusNumber`状态并返回一个新的`BusNumber`。方法内部直接读取配置空间偏移量0x18处的32位寄存器，使用`bit_field`库解析出三个8位的总线编号字段，将它们构造成`BusNumber`传递给闭包。闭包执行后，新值被写回同一寄存器。这种设计模式既保证了原子性，又避免了直接暴露底层寄存器细节。

**Section sources**
- [types/config/pci_bridge.rs](file://src/types/config/pci_bridge.rs#L9-L110)

## BusNumber辅助结构体

`BusNumber`是一个简单的辅助结构体，定义在`types/mod.rs`中，用于封装PCI桥接器的三个总线编号：主总线号（primary）、次级总线号（secondary）和下级总线号（subordinate）。它的设计意图是提供一个类型安全且易于使用的接口，以替代直接操作原始的32位寄存器。通过将其与`update_bus_number`方法结合，开发者可以以更直观的方式操作总线编号，例如：
```rust
bridge.update_bus_number(|mut bus| {
    bus.secondary = 1;
    bus.subordinate = 5;
    bus
});
```
这比手动进行位操作要清晰和安全得多。

**Section sources**
- [types/mod.rs](file://src/types/mod.rs#L11-L15)

## Deref/DerefMut实现

为了简化对基础字段的访问，`Endpoint`和`PciPciBridge`结构体都实现了`Deref`和`DerefMut` trait，使其能够透明地“解引用”到其内部的`PciHeaderBase`字段。

这意味着，任何可以直接在`PciHeaderBase`上调用的方法（如`address()`、`vendor_id()`、`device_id()`、`command()`等），都可以直接在`Endpoint`或`PciPciBridge`实例上调用，而无需显式访问`.base`字段。例如，`let addr = endpoint.address();` 这行代码实际上是通过`Deref`机制自动转换为`let addr = endpoint.base.address();`的。这极大地提升了API的流畅性和易用性，使代码更加简洁。

**Section sources**
- [types/config/endpoint.rs](file://src/types/config/endpoint.rs#L217-L231)
- [types/config/pci_bridge.rs](file://src/types/config/pci_bridge.rs#L104-L108)