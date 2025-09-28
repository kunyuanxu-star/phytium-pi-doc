# usb-if 模块 API

<cite>
**Referenced Files in This Document**   
- [lib.rs](file://usb-if/src/lib.rs)
- [descriptor/mod.rs](file://usb-if/src/descriptor/mod.rs)
- [host/mod.rs](file://usb-if/src/host/mod.rs)
- [transfer/mod.rs](file://usb-if/src/transfer/mod.rs)
- [err.rs](file://usb-if/src/err.rs)
- [descriptor/lang_id.rs](file://usb-if/src/descriptor/lang_id.rs)
- [transfer/sync.rs](file://usb-if/src/transfer/sync.rs)
- [transfer/wait.rs](file://usb-if/src/transfer/wait.rs)
</cite>

## 目录
1. [简介](#简介)
2. [模块概览](#模块概览)
3. [描述符模块 (descriptor)](#描述符模块-descriptor)
4. [主机模块 (host)](#主机模块-host)
5. [传输模块 (transfer)](#传输模块-transfer)
6. [错误处理 (err)](#错误处理-err)
7. [并发与内存安全](#并发与内存安全)

## 简介

`usb-if` 模块为 USB 设备和主机控制器提供了一套抽象的公共 API 接口。该文档旨在全面记录其暴露的所有模块、trait 和数据类型，重点涵盖描述符解析、主机控制器抽象以及同步与异步传输机制。API 设计遵循 Rust 的异步编程模型，并通过精心设计的 trait 体系实现了对不同后端（如 libusb, xhci）的统一访问。

**Section sources**
- [lib.rs](file://usb-if/src/lib.rs#L1-L8)

## 模块概览

`usb-if` 模块由四个核心部分组成：`descriptor`、`host`、`transfer` 和 `err`。`lib.rs` 文件作为根模块，重新导出了这些子模块中的所有公共项，构成了整个库的公共接口。这种结构化的设计使得功能划分清晰，便于开发者按需使用。

```mermaid
graph TB
subgraph "usb-if"
lib[lib.rs]
descriptor[descriptor/mod.rs]
host[host/mod.rs]
transfer[transfer/mod.rs]
err[err.rs]
end
lib --> descriptor
lib --> host
lib --> transfer
### lib --> err
```

**Diagram sources **
- [lib.rs](file://usb-if/src/lib.rs#L1-L8)

**Section sources**
- [lib.rs](file://usb-if/src/lib.rs#L1-L8)

## 描述符模块 (descriptor)

### 语言 ID (lang_id)

`descriptor::lang_id` 模块定义了 `LanguageId` 枚举，用于表示 USB 字符串描述符中使用的语言标识符（LANGID）。该枚举将常见的 ISO 639-1 语言代码与国家/地区代码组合映射为具体的 `u16` 值，例如 `EnglishUnitedStates` 对应 `0x0409`。这极大地简化了在获取字符串描述符时指定所需语言的过程。

```rust
let lang_id = LanguageId::ChinesePRC; // 0x0804
```

**Section sources**
- [descriptor/lang_id.rs](file://usb-if/src/descriptor/lang_id.rs#L1-L305)

### 描述符解析结构

`descriptor` 模块提供了对标准 USB 描述符进行解析和表示的核心数据结构。它包含 `DeviceDescriptor`、`ConfigurationDescriptor`、`InterfaceDescriptor` 和 `EndpointDescriptor` 等结构体，它们分别对应设备、配置、接口和端点的描述信息。

- **`DeviceDescriptor`**: 包含设备的基本信息，如厂商 ID (`vendor_id`)、产品 ID (`product_id`)、USB 协议版本 (`usb_version`) 以及指向制造商、产品名称等字符串描述符的索引。
- **`ConfigurationDescriptor`**: 表示一个设备配置，包含该配置下的接口数量 (`num_interfaces`)、最大功耗 (`max_power`) 以及一个 `interfaces` 列表。
- **`InterfaceDescriptor`**: 描述一个接口，包含接口号 (`interface_number`)、类代码 (`class`)、子类代码 (`subclass`) 和协议代码 (`protocol`)，并包含一个 `endpoints` 列表。
- **`EndpointDescriptor`**: 完整地描述了一个端点，包括地址 (`address`)、最大包大小 (`max_packet_size`)、传输类型 (`transfer_type`) 和方向 (`direction`)。

这些结构体通常通过 `parse` 方法从原始字节流中创建，内部依赖于 `parser` 子模块进行实际的二进制解析。

**Section sources**
- [descriptor/mod.rs](file://usb-if/src/descriptor/mod.rs#L1-L242)

## 主机模块 (host)

### 主机控制器抽象 (Controller)

`host` 模块的核心是 `Controller` trait，它抽象了对底层 USB 主机控制器的访问。任何实现了此 trait 的类型都可以被用作驱动程序来管理连接的 USB 设备。

- **`init(&mut self)`**: 初始化主机控制器，返回一个 `LocalBoxFuture`，成功时无返回值，失败时返回 `USBError`。
- **`device_list(&self)`**: 获取当前连接到控制器的所有设备列表，返回一个包含 `DeviceInfo` 对象的向量。
- **`handle_event(&mut self)`**: 在中断上下文中调用，用于处理来自硬件的事件。

**Section sources**
- [host/mod.rs](file://usb-if/src/host/mod.rs#L1-L123)

### 设备与接口操作

`DeviceInfo` trait 提供了打开设备和查询其描述符的能力。`Device` trait 则代表一个已打开的设备，允许设置配置、声明接口等操作。`Interface` trait 进一步抽象了对特定接口的操作，如切换备用设置。

- **`claim_interface(&mut self, interface: u8, alternate: u8)`**: 声明一个接口并选择其备用设置，成功时返回一个 `Interface` 对象。
- **`control_in/out`**: 执行控制传输，这是主机与设备通信的基础方式。

**Section sources**
- [host/mod.rs](file://usb-if/src/host/mod.rs#L1-L123)

## 传输模块 (transfer)

### 同步与等待机制

`transfer` 模块定义了数据传输相关的基础类型和同步原语。`sync` 子模块提供了一个基于 `spin::RwLock` 的 `RwLock<T>`，用于在无标准库环境下实现读写锁，确保对共享资源的并发访问安全。

`wait` 子模块是异步传输的核心。`WaitMap<K, T>` 是一个线程安全的映射，用于跟踪正在进行的传输请求。每个请求通过一个唯一的键（如传输 ID）进行标识。`Waiter<'a, T>` 是一个 `Future`，消费者可以 `await` 它以等待特定传输的结果。

- **`preper_id(&self, id: &K)`**: 尝试为一个给定的 ID 准备一次等待操作，检查队列是否已满。
- **`wait_for_result(&self, id: K, on_ready: Option<CallbackOnReady>)`**: 返回一个 `Waiter` Future，当与 `id` 关联的传输完成时，该 Future 会就绪。
- **`unsafe set_result(&self, id: K, result: T)`**: 由传输完成的回调函数调用，用于设置结果并唤醒等待的 `Waiter`。

**Section sources**
- [transfer/sync.rs](file://usb-if/src/transfer/sync.rs#L1-L72)
- [transfer/wait.rs](file://usb-if/src/transfer/wait.rs#L1-L179)

### 传输参数

该模块还定义了传输过程中使用的各种枚举：
- **`Direction`**: 数据传输方向，`Out` (主机到设备) 或 `In` (设备到主机)。
- **`RequestType`**: 控制传输的请求类型，如 `Standard`, `Class`, `Vendor`。
- **`Recipient`**: 控制传输的接收者，如 `Device`, `Interface`, `Endpoint`。
- **`Request`**: 标准控制请求，如 `GetDescriptor`, `SetAddress`。

**Section sources**
- [transfer/mod.rs](file://usb-if/src/transfer/mod.rs#L1-L111)

## 错误处理 (err)

### TransferError 枚举

`err` 模块定义了 `TransferError` 枚举，用于表示在执行 USB 传输过程中可能发生的错误。

| 错误码 | 含义 | 恢复策略 |
| :--- | :--- | :--- |
| `Stall` | 端点停滞 | 发送清除特征请求 (`CLEAR_FEATURE`) 来重置端点。 |
| `RequestQueueFull` | 请求队列已满 | 等待一段时间后重试，或检查是否有未完成的旧请求。 |
| `Timeout` | 传输超时 | 检查设备连接状态，确认设备是否响应，然后重试。 |
| `Cancelled` | 传输被取消 | 通常由用户主动取消，无需特殊恢复。 |
| `Other(String)` | 其他未知错误 | 记录错误信息，根据具体情况进行诊断。 |

**Section sources**
- [err.rs](file://usb-if/src/err.rs#L1-L66)

## 并发与内存安全

### Send/Sync 边界条件

为了支持异步环境下的并发操作，`usb-if` 模块中的关键 trait 都带有 `Send + 'static` 的生命周期约束。这意味着