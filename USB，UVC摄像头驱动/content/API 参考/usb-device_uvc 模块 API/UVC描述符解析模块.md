# UVC描述符解析模块

<cite>
**Referenced Files in This Document**
- [descriptors.rs](file://usb-device/uvc/src/descriptors.rs)
- [parser.rs](file://usb-if/src/descriptor/parser.rs)
</cite>

## 目录
1. [简介](#简介)
2. [核心组件](#核心组件)
3. [VideoControl接口描述符](#videcontrol接口描述符)
4. [VideoStreaming接口描述符](#videostreaming接口描述符)
5. [帧描述符结构](#帧描述符结构)
6. [描述符解析器](#描述符解析器)
7. [格式GUID常量](#格式guid常量)

## 简介
UVC（USB Video Class）描述符解析模块负责解析USB视频设备的标准描述符，以获取摄像头的功能、格式和配置信息。该模块实现了对UVC标准中定义的各类描述符的完整支持，包括VideoControl和VideoStreaming接口的描述符。

**Section sources**
- [descriptors.rs](file://usb-device/uvc/src/descriptors.rs#L1-L50)

## 核心组件
本模块的核心是`DescriptorParser`工具类，它提供了对各种UVC描述符的解析方法。同时，模块定义了多个结构体来表示不同类型的描述符，如`VcHeaderDescriptor`、`InputTerminalDescriptor`等。

```mermaid
classDiagram
class DescriptorParser {
+parse_vc_header(data[] u8) Result~VcHeaderDescriptor~
+parse_input_terminal(data[] u8) Result~InputTerminalDescriptor~
+parse_processing_unit(data[] u8) Result~ProcessingUnitDescriptor~
+parse_vs_input_header(data[] u8) Result~VsInputHeaderDescriptor~
+parse_uncompressed_format(data[] u8) Result~UncompressedFormatDescriptor~
+parse_mjpeg_format(data[] u8) Result~MjpegFormatDescriptor~
+parse_frame_descriptor(data[] u8) Result~FrameDescriptor~
+interval_to_fps(interval u32) u32
+fps_to_interval(fps u32) u32
}
class VcHeaderDescriptor {
+length usize
+bcd_uvc u16
+total_length u16
+clock_frequency u32
+in_collection u8
}
class InputTerminalDescriptor {
<<enumeration>>
+Camera
+Generic
}
class ProcessingUnitDescriptor {
+length usize
+unit_id u8
+source_id u8
+max_multiplier u16
+controls Vec~u8~
}
class VsInputHeaderDescriptor {
+length usize
+num_formats u8
+total_length u16
+endpoint_address u8
+info u8
+terminal_link u8
+still_capture_method u8
+trigger_support u8
+trigger_usage u8
+format_controls Vec~u8~
}
class UncompressedFormatDescriptor {
+length usize
+format_index u8
+num_frame_descriptors u8
+guid [u8; 16]
+bits_per_pixel u8
+default_frame_index u8
+aspect_ratio_x u8
+aspect_ratio_y u8
+interlace_flags u8
+copy_protect u8
}
class MjpegFormatDescriptor {
+length usize
+format_index u8
+num_frame_descriptors u8
+flags u8
+default_frame_index u8
+aspect_ratio_x u8
+aspect_ratio_y u8
+interlace_flags u8
+copy_protect u8
}
class FrameDescriptor {
+length usize
+frame_index u8
+capabilities u8
+width u16
+height u16
+min_bit_rate u32
+max_bit_rate u32
+max_video_frame_buffer_size u32
+default_frame_interval u32
+frame_interval_type u8
+frame_intervals Vec~u32~
}
DescriptorParser --> VcHeaderDescriptor : "parses"
DescriptorParser --> InputTerminalDescriptor : "parses"
DescriptorParser --> ProcessingUnitDescriptor : "parses"
DescriptorParser --> VsInputHeaderDescriptor : "parses"
DescriptorParser --> UncompressedFormatDescriptor : "parses"
DescriptorParser --> MjpegFormatDescriptor : "parses"
DescriptorParser --> FrameDescriptor : "parses"
```

**Diagram sources**
- [descriptors.rs](file://usb-device/uvc/src/descriptors.rs#L574-L659)

**Section sources**
- [descriptors.rs](file://usb-device/uvc/src/descriptors.rs#L574-L659)

## VideoControl接口描述符
VideoControl接口描述符用于描述视频控制功能，包括头描述符、输入终端和处理单元等。

### VcHeaderDescriptor
`VcHeaderDescriptor`表示VideoControl接口的头描述符，包含UVC版本、总长度、时钟频率等基本信息。

**Section sources**
- [descriptors.rs](file://usb-device/uvc/src/descriptors.rs#L574-L580)

### InputTerminalDescriptor
`InputTerminalDescriptor`表示输入终端描述符，分为摄像头终端和其他通用终端两种类型。摄像头终端包含焦距等特定信息。

**Section sources**
- [descriptors.rs](file://usb-device/uvc/src/descriptors.rs#L582-L604)

### ProcessingUnitDescriptor
`ProcessingUnitDescriptor`表示处理单元描述符，用于描述图像处理功能，如亮度、对比度调节等。

**Section sources**
- [descriptors.rs](file://usb-device/uvc/src/descriptors.rs#L605-L614)

## VideoStreaming接口描述符
VideoStreaming接口描述符用于描述视频流的格式和传输特性。

### VsInputHeaderDescriptor
`VsInputHeaderDescriptor`表示VideoStreaming输入头描述符，包含格式数量、端点地址、终端链接等信息。

**Section sources**
- [descriptors.rs](file://usb-device/uvc/src/descriptors.rs#L615-L629)

### UncompressedFormatDescriptor/MjpegFormatDescriptor
`UncompressedFormatDescriptor`和`MjpegFormatDescriptor`分别表示未压缩格式和MJPEG格式的描述符，定义了视频流的编码格式和参数。

**Section sources**
- [descriptors.rs](file://usb-device/uvc/src/descriptors.rs#L630-L644)

## 帧描述符结构
`FrameDescriptor`结构体表示视频帧的详细参数，包括分辨率、比特率范围、缓冲区大小和帧率等关键信息。

```mermaid
classDiagram
class FrameDescriptor {
+length usize
+frame_index u8
+capabilities u8
+width u16
+height u16
+min_bit_rate u32
+max_bit_rate u32
+max_video_frame_buffer_size u32
+default_frame_interval u32
+frame_interval_type u8
+frame_intervals Vec~u32~
}
FrameDescriptor : "width x height" resolution
FrameDescriptor : "min_bit_rate - max_bit_rate" bitrate_range
FrameDescriptor : "max_video_frame_buffer_size" buffer_size
FrameDescriptor : "default_frame_interval" default_interval
FrameDescriptor : "frame_intervals" interval_list
```

**Diagram sources**
- [descriptors.rs](file://usb-device/uvc/src/descriptors.rs#L659-L675)

**Section sources**
- [descriptors.rs](file://usb-device/uvc/src/descriptors.rs#L659-L675)

## 描述符解析器
`DescriptorParser`工具类提供了核心的解析方法，用于将原始字节数据转换为相应的描述符结构。

### 解析方法调用流程
```mermaid
sequenceDiagram
participant Client as "客户端"
participant Parser as "DescriptorParser"
participant Data as "原始描述符数据"
Client->>Parser : parse_frame_descriptor(data)
Parser->>Data : 验证数据长度(>=26)
alt 数据有效
Parser->>Parser : 解析基本字段
Parser->>Parser : 解析帧间隔数据
Parser-->>Client : 返回FrameDescriptor
else 数据无效
Parser-->>Client : 返回错误
end
Client->>Parser : parse_uncompressed_format(data)
Parser->>Data : 验证数据长度(>=27)
alt 数据有效
Parser->>Parser : 解析GUID等字段
Parser-->>Client : 返回UncompressedFormatDescriptor
else 数据无效
Parser-->>Client : 返回错误
end
```

**Diagram sources**
- [descriptors.rs](file://usb-device/uvc/src/descriptors.rs#L400-L450)

**Section sources**
- [descriptors.rs](file://usb-device/uvc/src/descriptors.rs#L300-L500)

## 格式GUID常量
`format_guids`模块定义了YUY2、NV12等视频格式的GUID常量，用于识别视频流的编码类型。

```mermaid
classDiagram
class format_guids {
+YUY2 [u8; 16]
+NV12 [u8; 16]
+RGB24 [u8; 16]
+UYVY [u8; 16]
+BGR24 [u8; 16]
}
format_guids : "YUY2" YUY2_format
format_guids : "NV12" NV12_format
format_guids : "RGB24" RGB24_format
format_guids : "UYVY" UYVY_format
format_guids : "BGR24" BGR24_format
```

**Diagram sources**
- [descriptors.rs](file://usb-device/uvc/src/descriptors.rs#L160-L180)

**Section sources**
- [descriptors.rs](file://usb-device/uvc/src/descriptors.rs#L160-L180)