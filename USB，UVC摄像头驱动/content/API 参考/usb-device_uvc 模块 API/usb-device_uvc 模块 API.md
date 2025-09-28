
# usb-device/uvc 模块 API

<cite>
**本文档引用的文件**
- [lib.rs](file://usb-device/uvc/src/lib.rs)
- [descriptors.rs](file://usb-device/uvc/src/descriptors.rs)
- [frame.rs](file://usb-device/uvc/src/frame.rs)
- [stream.rs](file://usb-device/uvc/src/stream.rs)
</cite>

## 目录
1. [简介](#简介)
2. [核心结构体与状态机](#核心结构体与状态机)
3. [视频格式与控制数据结构](#视频格式与控制数据结构)
4. [异步方法与描述符解析](#异步方法与描述符解析)
5. [控制传输协议细节](#控制传输协议细节)
6. [视频帧处理与序列化](#视频帧处理与序列化)

## 简介
`usb-device/uvc` 模块是基于 `crab-usb` 的 USB Video Class (UVC) 设备驱动库，专为支持 UVC 摄像头设备的功能暴露而设计。该模块提供了完整的设备检测、视频流控制、格式协商和图像参数调节能力。通过异步编程模型，实现了非阻塞的视频帧接收与处理，适用于高性能视频捕获场景。本 API 参考文档系统化地阐述了核心组件的设计与使用方式。

## 核心结构体与状态机

`UvcDevice` 是模块的核心结构体，封装了对 UVC 摄像头设备的全部操作接口。其构造函数 `new` 接收一个 `Device` 实例，并通过异步初始化完成对视频控制接口（Video Control Interface）和视频流接口（Video Streaming Interface）的探测与声明。在初始化过程中，系统会遍历设备配置描述符，识别出符合 UVC 规范的接口（类为 `Class::Video` 且子类分别为 1 和 2），并建立相应的接口句柄。

设备的状态由 `UvcDeviceState` 枚举精确管理，包含四种状态：`Unconfigured`（未配置）、`Configured`（已配置但未流式传输）、`Streaming`（正在流式传输）和 `Error(String)`（错误状态）。状态转换遵循严格的流程：设备创建后处于 `Configured` 状态；调用 `set_format` 方法成功设置视频格式后，仍保持 `Configured` 状态；当调用 `start_streaming` 方法并成功启动数据流后，状态跃迁至 `Streaming`；若在任何阶段发生错误，则进入 `Error` 状态。此状态机确保了设备操作的有序性和安全性。

`check` 方法是设备检测的关键入口，它接收一个 `DeviceInfo` 引用，通过检查接口描述符来判断设备是否为有效的 UVC 设备。该方法要求设备同时具备视频控制接口（subclass=1）和视频流接口（subclass=2）才能返回 `true`，从而保证了后续操作的可行性。

**Section sources**
- [lib.rs](file://usb-device/uvc/src/lib.rs#L100-L200)

## 视频格式与控制数据结构

`VideoFormat` 结构体定义了视频流的核心属性，包括宽度（`width`）、高度（`height`）、帧率（`frame_rate`）以及格式类型（`format_type`）。`format_type` 是一个枚举 `VideoFormatType`，支持 `Uncompressed`（未压缩）、`Mjpeg` 和 `H264` 三种主要格式。对于未压缩格式，`UncompressedFormat` 枚举进一步区分了 `Yuy2`、`Nv12`、`Rgb24` 和 `Rgb32` 等具体像素布局。

`StreamControl` 结构体直接映射到 UVC 规范中的 VS 控制请求载荷，用于在 `PROBE` 和 `COMMIT` 阶段与设备协商流参数。其字段语义如下：
- `hint`: 位掩码，指示哪些字段在 `GET_CUR` 响应中必须保持不变。
- `format_index` 和 `frame_index`: 分别指向 VS 接口描述符中的格式和帧配置索引。
- `frame_interval`: 以 100ns 为单位的帧间隔，决定了实际帧率。
- `max_video_frame_size`: 主机端估算的最大视频帧大小，用于带宽分配。
- `max_payload_transfer_size`: 单次传输的最大有效载荷大小。

这些字段共同构成了主机与设备之间进行视频流配置协商的数据契约。

**Section sources**
- [lib.rs](file://usb-device/uvc/src/lib.rs#L50-L100)
- [lib.rs](file://usb-device/uvc/src/lib.rs#L250-L300)

## 异步方法与描述符解析

`get_supported_formats` 是一个关键的异步方法，负责解析设备的 VS 接口描述符并生成可用的 `VideoFormat` 列表。其执行流程分为两步：首先尝试通过 `GET_DESCRIPTOR` 请求获取完整的配置描述符，然后调用 `parse_vs_interface_descriptors` 函数进行解析。该函数会遍历配置描述符中的所有字节，识别出 `CS_INTERFACE` 类型的描述符，并根据其子类型（如 `VS_FORMAT_MJPEG`, `VS_FORMAT_UNCOMPRESSED`）调用相应的解析器。

例如，当遇到 `VS_FORMAT_UNCOMPRESSED` 描述符时，系统会提取其中的 GUID 字段，并与预定义的常量（如 `format_guids::YUY2`）进行比对，以确定具体的像素格式。随后，对于每个关联的 `VS_FRAME_UNCOMPRESSED` 或 `VS_FRAME_MJPEG` 帧描述符，系统会解析其分辨率（`width`, `height`）和默认帧间隔（`default_frame_interval`），并利用 `DescriptorParser::interval_to_fps` 将其转换为人类可读的帧率值，最终构建出一个 `VideoFormat` 对象并加入结果列表。

**Section sources**
- [lib.rs](file://usb-device/uvc/src/lib.rs#L300-L450)
- [descriptors.rs](file://usb-device/uvc/src/descriptors.rs#L300-L400)

## 控制传输协议细节

`control_in` 控制传输调用在获取配置描述符时遵循标准的 USB 控制传输协议。具体而言，在 `get_full_configuration_descriptor` 方法中，系统首先构建一个 `ControlSetup` 结构体，其 `request_type` 为 `Standard`，`recipient` 为 `Device`，`request` 为 `GetDescriptor`，`value` 设置为 `(0x02 << 8)`（表示配置描述符类型），`index` 为 0。第一次短读取（9字节）用于获取配置描述符头部，从中提取出总长度字段。随后，系统分配一个相应大小的缓冲区，并发起第二次 `control_in` 调用以获取完整的配置描述符数据。这种分步读取的方式是处理变长描述符的标准实践。

**Section sources**
- [lib.rs](file://usb-device/uvc/src/lib.rs#L350-L380)

## 视频帧处理与序列化

`VideoFrame` 结构体代表一个完整的视频数据帧，包含 `data`（原始字节）、`timestamp`、`frame_number`、`format` 和 `end_of_frame` 标志。`end_of_frame` 标志在帧组装过程中起着至关重要的作用。`FrameParser` 组件负责从连续的等时传输包中重组出完整的 `VideoFrame`。每个传输包都以一个 UVC 载荷头开始，其中的 `EOF` 位（`End of Frame`）明确指示了当前包是否为一帧数据的最后一个包。`FrameParser` 会持续累积包数据，直到遇到一个 `EOF` 位被置位的包，此时它会将累积的缓冲区打包成一个 `FrameEvent` 并触发 `Some` 返回