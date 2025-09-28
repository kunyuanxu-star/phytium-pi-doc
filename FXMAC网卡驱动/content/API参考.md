# API参考

<cite>
**本文档中引用的文件**
- [fxmac.rs](file://src/fxmac.rs)
- [fxmac_const.rs](file://src/fxmac_const.rs)
- [mii_const.rs](file://src/mii_const.rs)
- [fxmac_dma.rs](file://src/fxmac_dma.rs)
- [fxmac_phy.rs](file://src/fxmac_phy.rs)
- [lib.rs](file://src/lib.rs)
</cite>

## 目录
1. [导出函数](#导出函数)
2. [结构体](#结构体)
3. [常量](#常量)

## 导出函数

### xmac_init
初始化FXMAC网卡设备。

**函数签名**
```rust
pub fn xmac_init(hwaddr: &[u8; 6]) -> &'static mut FXmac
```

**参数说明**
- `hwaddr`: 网络接口的MAC地址，长度为6字节的数组。

**返回值**
- 返回指向已初始化的`FXmac`实例的静态可变引用。

**使用示例**
```rust
let hwaddr: [u8; 6] = [0x55, 0x44, 0x33, 0x22, 0x11, 0x00];
let fxmac_device: &'static mut FXmac = xmac_init(&hwaddr);
```

**Section sources**
- [fxmac.rs](file://src/fxmac.rs#L106-L289)

### FXmacStart
启动以太网控制器。

**函数签名**
```rust
pub fn FXmacStart(instance_p: &mut FXmac)
```

**参数说明**
- `instance_p`: 指向`FXmac`实例的可变引用。

**功能描述**
- 启用发送器（如果设置了`FXMAC_TRANSMITTER_ENABLE_OPTION`）
- 启用接收器（如果设置了`FXMAC_RECEIVER_ENABLE_OPTION`）
- 启动SG DMA发送和接收通道，并启用设备中断

**调用上下文**
在完成PHY初始化、DMA初始化和中断设置后调用此函数来启动网卡。

**Section sources**
- [fxmac.rs](file://src/fxmac.rs#L293-L347)

### FXmacStop
优雅地停止以太网MAC。

**函数签名**
```rust
pub fn FXmacStop(instance_p: &mut FXmac)
```

**参数说明**
- `instance_p`: 指向`FXmac`实例的可变引用。

**功能描述**
- 禁用所有来自该设备的中断
- 停止DMA通道
- 禁用发送器和接收器

**使用场景**
当需要关闭网络接口或重新配置时调用。

**Section sources**
- [fxmac.rs](file://src/fxmac.rs#L349-L380)

### FXmacSetQueuePtr
设置传输/接收缓冲区队列的起始地址。

**函数签名**
```rust
pub fn FXmacSetQueuePtr(queue_p: u64, queue_num: u8, direction: u32)
```

**参数说明**
- `queue_p`: 队列基地址（物理地址）
- `queue_num`: 缓冲区队列索引（0-FXMAC_QUEUE_MAX_NUM）
- `direction`: 方向标识符（FXMAC_SEND表示发送，FXMAC_RECV表示接收）

**注意事项**
必须在开始数据传输之前设置缓冲区队列地址，因此应在调用`FXmacStart()`之前调用此函数。

**Section sources**
- [fxmac.rs](file://src/fxmac.rs#L609-L676)

### FXmacSgsend
发送数据包。

**函数签名**
```rust
pub fn FXmacSgsend(instance_p: &mut FXmac, p: Vec<Vec<u8>>) -> u32
```

**参数说明**
- `instance_p`: 指向`FXmac`实例的可变引用
- `p`: 要发送的数据包向量，每个元素是一个字节数组

**返回值**
- 成功时返回0，失败时返回错误码

**处理流程**
1. 分配BD（Buffer Descriptor）
2. 将数据复制到DMA缓冲区
3. 刷新缓存
4. 提交到硬件进行发送

**调用上下文**
```rust
let mut tx_vec = Vec::new();
tx_vec.push(packet.to_vec());
FXmacSgsend(fxmac_device, tx_vec);
```

**Section sources**
- [fxmac_dma.rs](file://src/fxmac_dma.rs#L757-L875)

### FXmacRecvHandler
处理接收到的数据包。

**函数签名**
```rust
pub fn FXmacRecvHandler(instance_p: &mut FXmac) -> Option<Vec<Vec<u8>>>
```

**参数说明**
- `instance_p`: 指向`FXmac`实例的可变引用

**返回值**
- 包含接收到的数据包的`Option<Vec<Vec<u8>>>`，如果没有数据包则返回`None`

**处理流程**
1. 从硬件获取已完成的BD
2. 从DMA缓冲区读取数据
3. 使缓存无效
4. 组装并返回数据包

**使用示例**
```rust
let recv_packets = FXmacRecvHandler(fxmac_device);
```

**Section sources**
- [fxmac_dma.rs](file://src/fxmac_dma.rs#L876-L1022)

### FXmacPhyInit
初始化PHY设备。

**函数签名**
```rust
pub fn FXmacPhyInit(instance_p: &mut FXmac, reset_flag: u32) -> u32
```

**参数说明**
- `instance_p`: 指向`FXmac`实例的可变引用
- `reset_flag`: 重置标志（XMAC_PHY_RESET_ENABLE或XMAC_PHY_RESET_DISABLE）

**返回值**
- 成功时返回0，失败时返回错误码

**功能描述**
根据配置设置PHY的速度：
- 如果启用了自动协商，则执行自动协商过程
- 如果禁用了自动协商，则手动设置指定速度和双工模式

**Section sources**
- [fxmac_phy.rs](file://src/fxmac_phy.rs#L100-L178)

### FXmacPhyWrite
向指定的PHY寄存器写入数据。

**函数签名**
```rust
pub fn FXmacPhyWrite(instance_p: &mut FXmac, phy_address: u32, register_num: u32, phy_data: u16) -> u32
```

**参数说明**
- `instance_p`: 指向`FXmac`实例的可变引用
- `phy_address`: PHY地址（0-31）
- `register_num`: 寄存器编号
- `phy_data`: 要写入的数据

**返回值**
- 成功时返回0，失败时返回错误码

**应用场景**
用于配置PHY芯片的各种参数，如控制寄存器、自协商寄存器等。

**Section sources**
- [fxmac_phy.rs](file://src/fxmac_phy.rs#L77-L100)

### FXmacPhyRead
从指定的PHY寄存器读取数据。

**函数签名**
```rust
pub fn FXmacPhyRead(instance_p: &mut FXmac, phy_address: u32, register_num: u32, phydat_aptr: &mut u16) -> u32
```

**参数说明**
- `instance_p`: 指向`FXmac`实例的可变引用
- `phy_address`: PHY地址（0-31）
- `register_num`: 寄存器编号
- `phydat_aptr`: 指向存储读取数据的u16变量的可变引用

**返回值**
- 成功时返回0，失败时返回错误码

**使用示例**
```rust
let mut status: u16 = 0;
let result = FXmacPhyRead(instance_p, phy_addr, PHY_STATUS_REG_OFFSET, &mut status);
```

**Section sources**
- [fxmac_phy.rs](file://src/fxmac_phy.rs#L48-L75)

## 结构体

### FXmac
FXMAC设备实例的主要结构体。

**字段说明**
- `config`: `FXmacConfig`类型的配置结构体
- `is_ready`: 设备是否已初始化并准备就绪（FT_COMPONENT_IS_READY表示就绪）
- `is_started`: 设备是否已启动
- `link_status`: 链路状态（FXMAC_LINKUP/FXMAC_LINKDOWN/FXMAC_NEGOTIATING）
- `options`: 当前选项掩码
- `mask`: 中断掩码
- `caps`: 功能能力掩码
- `lwipport`: LwIP端口相关数据
- `tx_bd_queue`: 发送BD队列
- `rx_bd_queue`: 接收BD队列
- `moudle_id`: 模块识别号
- `max_mtu_size`: 最大MTU大小
- `max_frame_size`: 最大帧大小
- `phy_address`: PHY地址
- `rxbuf_mask`: 接收缓冲区掩码

**Section sources**
- [fxmac.rs](file://src/fxmac.rs#L32-L56)

### FXmacConfig
FXMAC设备的配置结构体。

**字段说明**
- `instance_id`: 设备ID
- `base_address`: 基地址
- `extral_mode_base`: 扩展模式基地址
- `extral_loopback_base`: 扩展环回基地址
- `interface`: PHY接口类型（`FXmacPhyInterface`枚举）
- `speed`: 速度（10/100/1000 Mbps）
- `duplex`: 双工模式（1表示全双工，0表示半双工）
- `auto_neg`: 是否启用自动协商
- `pclk_hz`: 外设时钟频率
- `max_queue_num`: XMAC控制器队列数量
- `tx_queue_id`: 发送队列ID（0-FXMAC_QUEUE_MAX_NUM）
- `rx_queue_id`: 接收队列ID（0-FXMAC_QUEUE_MAX_NUM）
- `hotplug_irq_num`: 热插拔中断号
- `dma_brust_length`: DMA突发长度
- `network_default_config`: 网络默认配置
- `queue_irq_num`: 队列中断号数组
- `caps`: 用于配置尾指针功能的能力位
- `mac`: MAC地址数组[6]

**Section sources**
- [fxmac.rs](file://src/fxmac.rs#L58-L87)

### FXmacBdRing
DMA缓冲区描述符环形队列结构体。

**字段说明**
- `phys_base_addr`: 物理基地址
- `base_bd_addr`: BD基地址
- `high_bd_addr`: BD高地址
- `length`: 队列总长度
- `run_state`: 运行状态
- `separation`: 相邻BD之间的间隔
- `free_head`: 空闲BD头指针
- `pre_head`: 预处理BD头指针
- `hw_head`: 硬件处理头指针
- `hw_tail`: 硬件处理尾指针
- `post_head`: 后处理BD头指针
- `bda_restart`: 重启BD地址
- `hw_cnt`: 硬件处理计数
- `pre_cnt`: 预处理计数
- `free_cnt`: 空闲计数
- `post_cnt`: 后处理计数
- `all_cnt`: 总计数

**Section sources**
- [fxmac_dma.rs](file://src/fxmac_dma.rs#L101-L144)

### FXmacQueue
FXMAC队列结构体。

**字段说明**
- `queue_id`: 队列ID
- `bdring`: 关联的BD环形队列

**Section sources**
- [fxmac.rs](file://src/fxmac.rs#L79-L85)

### FXmacLwipPort
LwIP端口相关数据结构体。

**字段说明**
- `buffer`: `FXmacNetifBuffer`类型的缓冲区
- `feature`: 功能特性掩码
- `hwaddr`: 硬件MAC地址[6]
- `recv_flg`: 接收中断触发次数计数器

**Section sources**
- [fxmac_dma.rs](file://src/fxmac_dma.rs#L176-L183)

## 常量

### fxmac_const.rs中的关键常量

#### 中断处理程序标识符
这些常量用作`FXmacSetHandler()`的参数：

- `FXMAC_HANDLER_DMASEND`: 发送中断 (值: 1)
- `FXMAC_HANDLER_DMARECV`: 接收中断 (值: 2)
- `FXMAC_HANDLER_ERROR`: 异常中断 (值: 3)
- `FXMAC_HANDLER_LINKCHANGE`: 连接状态变化 (值: 4)
- `FXMAC_HANDLER_RESTART`: 发送描述符队列异常 (值: 5)

#### 链路状态常量
- `FXMAC_LINKDOWN`: 链路断开 (值: 0)
- `FXMAC_LINKUP`: 链路连接 (值: 1)
- `FXMAC_NEGOTIATING`: 正在协商 (值: 2)

#### 速度常量
- `FXMAC_SPEED_10`: 10 Mbps
- `FXMAC_SPEED_100`: 100 Mbps
- `FXMAC_SPEED_1000`: 1000 Mbps
- `FXMAC_SPEED_2500`: 2500 Mbps
- `FXMAC_SPEED_5000`: 5000 Mbps
- `FXMAC_SPEED_10000`: 10000 Mbps

#### 功能能力掩码
- `FXMAC_CAPS_ISR_CLEAR_ON_WRITE`: 中断状态参数在读取后需要写入才能清除
- `FXMAC_CAPS_TAILPTR`: 使用尾指针功能

#### 默认选项
`FXMAC_DEFAULT_OPTIONS`包含以下选项的组合：
- `FXMAC_FLOW_CONTROL_OPTION`: 启用流控
- `FXMAC_FCS_INSERT_OPTION`: 生成FCS字段
- `FXMAC_FCS_STRIP_OPTION`: 剥离FCS和PAD
- `FXMAC_BROADCAST_OPTION`: 允许广播地址接收
- `FXMAC_LENTYPE_ERR_OPTION`: 启用长度/类型错误检查
- `FXMAC_TRANSMITTER_ENABLE_OPTION`: 启用发送器
- `FXMAC_RECEIVER_ENABLE_OPTION`: 启用接收器
- `FXMAC_RX_CHKSUM_ENABLE_OPTION`: 启用RX校验和卸载
- `FXMAC_TX_CHKSUM_ENABLE_OPTION`: 启用TX校验和卸载

**Section sources**
- [fxmac_const.rs](file://src/fxmac_const.rs#L639-L667)
- [fxmac.rs](file://src/fxmac.rs#L9-L27)

### mii_const.rs中的关键常量

#### MII寄存器偏移量
- `MII_BMCR`: 基本模式控制寄存器 (0x00)
- `MII_BMSR`: 基本模式状态寄存器 (0x01)
- `MII_PHYSID1`: PHY ID 1 (0x02)
- `MII_PHYSID2`: PHY ID 2 (0x03)
- `MII_ADVERTISE`: 自协商广告控制寄存器 (0x04)
- `MII_LPA`: 链路伙伴能力寄存器 (0x05)

#### BMCR（基本模式控制寄存器）位定义
- `BMCR_FULLDPLX`: 全双工模式 (0x0100)
- `BMCR_ANENABLE`: 启用自动协商 (0x1000)
- `BMCR_SPEED100`: 选择100Mbps (0x2000)
- `BMCR_RESET`: 复位到默认状态 (0x8000)

#### BMSR（基本模式状态寄存器）位定义
- `BMSR_LSTATUS`: 链路状态 (0x0004)
- `BMSR_ANEGCOMPLETE`: 自动协商完成 (0x0020)
- `BMSR_10HALF`: 支持10Mbps半双工 (0x0800)
- `BMSR_10FULL`: 支持10Mbps全双工 (0x1000)
- `BMSR_100HALF`: 支持100Mbps半双工 (0x2000)
- `BMSR_100FULL`: 支持100Mbps全双工 (0x4000)

#### PHY地址相关常量
- `PHY_FIXED_ID`: 固定PHY ID (0xa5a55a5a)
- `PHY_NCSI_ID`: NCSI PHY ID (0xbeefcafe)
- `PHY_GMII2RGMII_ID`: GMII到RGMII转换器ID (0x5a5a5a5a)
- `PHY_MAX_ADDR`: 最大PHY地址 (32)

**Section sources**
- [mii_const.rs](file://src/mii_const.rs#L0-L274)