# 公共API参考

<cite>
**Referenced Files in This Document **  
- [lib.rs](file://src/lib.rs)
- [root.rs](file://src/root.rs)
- [chip/mod.rs](file://src/chip/mod.rs)
- [types/config/mod.rs](file://src/types/config/mod.rs)
- [types/config/endpoint.rs](file://src/types/config/endpoint.rs)
- [types/config/pci_bridge.rs](file://src/types/config/pci_bridge.rs)
</cite>

## 目录
1. [核心枚举函数](#核心枚举函数)
2. [公开结构体](#公开结构体)
3. [配置空间枚举](#配置空间枚举)
4. [通用访问接口](#通用访问接口)
5. [客户端使用示例](#客户端使用示例)

## 核心枚举函数

`enumerate_by_controller` 是 PCIe 设备发现的核心入口点，提供惰性迭代器模式遍历指定控制器下的所有 PCI 设备。

该函数接受两个参数：
- `controller`: 对 `PcieController` 的可变引用，代表一个 PCIe 控制器实例。此引用的生命周期 `'a` 确保了在迭代期间控制器不会被移动或销毁。
- `range`: 一个可选的总线范围（`Option<Range<usize>>`），用于限定设备探测的总线号区间。若为 `None`，则默认扫描从0到0xFF的所有总线。

函数返回一个实现了 `Iterator<Item = Endpoint>` trait 的迭代器。该迭代器采用惰性求值策略，在每次调用 `next()` 时才进行实际的硬件探测和设备解析。迭代器会自动跳过无效设备（如VID为0xFFFF的设备）和不支持的桥接类型（如CardBusBridge），确保最终产出的都是有效的、可操作的 `Endpoint` 实例。

**Section sources**
- [root.rs](file://src/root.rs#L10-L192)
- [lib.rs](file://src/lib.rs#L21)

## 公开结构体

### PcieGeneric

`PcieGeneric` 结构体是 PCIe 控制器的通用实现，封装了对特定硬件平台的内存映射I/O（MMIO）访问逻辑。

其主要方法为：
- `new(mmio_base: NonNull<u8>) -> Self`: 构造函数，接收一个指向控制器MMIO基地址的非空指针，并返回一个新的 `PcieGeneric` 实例。

该结构体实现了 `rdif_pcie::Interface` trait，提供了底层的PCI配置空间读写能力，是构建 `PcieController` 的基础组件。

**Section sources**
- [chip/mod.rs](file://src/chip/mod.rs#L7-L51)

### Endpoint

`Endpoint` 结构体代表一个PCIe终端设备，提供了丰富的设备信息查询和资源管理功能。

关键方法包括：
- `bar(&self, index: usize) -> Option<Range<usize>>`: 查询指定索引（0-5）的基址寄存器（BAR）所映射的内存或I/O地址范围。
- `bars(&self) -> BarVec`: 解析并返回该设备所有BAR的完整信息向量。
- `realloc_bar(&mut self, allocator: &mut SimpleBarAllocator) -> Result<(), pci_types::BarWriteError>`: 使用提供的分配器重新分配该设备的BAR地址，通常在系统初始化阶段用于避免地址冲突。

**Section sources**
- [types/config/endpoint.rs](file://src/types/config/endpoint.rs#L10-L237)

### PciPciBridge

`PciPciBridge` 结构体代表一个PCI-to-PCI桥接器，用于连接不同的PCI总线段。

其核心配置方法为：
- `update_bus_number<F>(&mut self, f: F) where F: FnOnce(BusNumber) -> BusNumber`: 更新桥接器的总线号配置。它接收一个闭包，允许用户修改当前的主总线（Primary）、次级总线（Secondary）和下级总线（Subordinate）号，并将更改写入硬件。

**Section sources**
- [types/config/pci_bridge.rs](file://src/types/config/pci_bridge.rs#L26-L110)

## 配置空间枚举

`PciConfigSpace` 是一个枚举类型，用于表示通过PCI配置空间读取到的不同类型的设备头。

其四个变体及其使用场景如下：
- `PciPciBridge(PciPciBridge)`: 表示一个标准的PCI-to-PCI桥接器。当需要管理多级总线拓扑时使用。
- `Endpoint(Endpoint)`: 表示一个终端设备（如网卡、显卡）。这是最常见的设备类型，用于与具体的功能设备交互。
- `CardBusBridge(CardBusBridge)`: 表示一个CardBus桥接器。目前代码中未实现处理逻辑，属于保留变体。
- `Unknown(Unknown)`: 表示一个未知类型的设备头。用于处理无法识别的头部格式，保证枚举的完备性。

**Section sources**
- [types/config/mod.rs](file://src/types/config/mod.rs#L15-L28)

## 通用访问接口

`PciHeaderBase` 结构体为所有PCI设备提供了统一的、基础的配置空间访问方法。

文档化的通用访问方法包括：
- `vendor_id(&self) -> u16`: 返回设备的厂商ID（VID）。
- `device_id(&self) -> u16`: 返回设备的产品ID（DID）。
- `command(&self) -> CommandRegister`: 读取设备的命令寄存器，用于控制设备的I/O和内存响应等行为。
- `status(&self) -> StatusRegister`: 读取设备的状态寄存器，获取设备的运行状态和错误信息。

这些方法构成了与任何PCI设备通信的基础，无论其具体类型如何。

**Section sources**
- [types/config/mod.rs](file://src/types/config/mod.rs#L50-L131)

## 客户端使用示例

以下是一个安全组合这些API构建设备探测工具的示例流程：

1.  创建 `PcieGeneric` 实例，并将其包装成 `PcieController`。
2.  调用 `enumerate_by_controller(controller, Some(0..0x10))` 来启动一个仅扫描前16条总线的迭代器。
3.  在迭代循环中，利用 `match` 语句处理 `PciConfigSpace` 的不同变体。
4.  对于 `Endpoint` 类型的设备，可以安全地调用其 `display` 方法打印设备信息，或调用 `realloc_bar` 进行资源重分配。
5.  对于 `PciPciBridge` 类型的设备，可以调用 `update_bus_number` 来动态调整其总线号配置。

整个过程由Rust的借用检查器和生命周期系统保障安全，防止了悬垂指针和数据竞争。

**Section sources**
- [lib.rs](file://src/lib.rs)
- [root.rs](file://src/root.rs)
- [types/config/mod.rs](file://src/types/config/mod.rs)