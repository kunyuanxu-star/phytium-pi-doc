# API参考

<cite>
**Referenced Files in This Document**   
- [src/lib.rs](file://src/lib.rs)
</cite>

## 目录
1. [简介](#简介)
2. [核心控制器方法](#核心控制器方法)
3. [外设ID枚举](#外设id枚举)
4. [配置与句柄结构体](#配置与句柄结构体)
5. [初始化与查找函数](#初始化与查找函数)
6. [with_cru宏机制](#with_cru宏机制)
7. [API模块导出](#api模块导出)

## 简介
本文档详细描述了Phytium Pi平台时钟与复位单元（CRU）驱动的公共API。该驱动提供对硬件复位功能的安全访问，支持系统级和外设级复位操作。所有接口设计遵循Rust的内存安全原则，同时通过适当的unsafe边界封装底层寄存器操作。

**Section sources**
- [src/lib.rs](file://src/lib.rs#L1-L50)

## 核心控制器方法

### system_reset
触发系统复位并等待完成。

- **参数**: 无
- **返回值语义**: 复位成功返回`true`，超时未完成返回`false`
- **成功条件**: 成功写入复位寄存器且在500个周期内检测到复位完成标志
- **失败条件**: 超时未检测到复位完成状态
- **副作用**: 触发整个系统的硬件复位动作
- **使用示例**:
```rust
let result = controller.system_reset();
if result {
    log::info!("System reset completed");
}
```

**Section sources**
- [src/lib.rs](file://src/lib.rs#L89-L100)

### reset_peripheral
复位指定的单个外设。

- **参数**: `periph_id` - 外设标识符（PeripheralId类型）
- **返回值语义**: 操作成功返回`true`，失败返回`false`
- **成功条件**: 成功写入对应外设的复位掩码并在超时前完成
- **失败条件**: 写入失败或等待超时
- **副作用**: 触发指定外设的硬件复位
- **使用示例**:
```rust
let result = controller.reset_peripheral(PeripheralId::Gpio0);
if result {
    log::info!("GPIO0 reset completed");
}
```

**Section sources**
- [src/lib.rs](file://src/lib.rs#L103-L117)

### reset_peripherals
批量复位多个外设。

- **参数**: `periph_ids` - 外设ID切片引用
- **返回值语义**: 所有操作成功返回`true`，否则返回`false`
- **成功条件**: 输入非空，成功写入组合掩码并在超时前完成
- **失败条件**: 输入为空、写入失败或等待超时
- **副作用**: 同时触发多个外设的硬件复位
- **使用示例**:
```rust
let peripherals = &[PeripheralId::Gpio0, PeripheralId::Gpio1];
let result = controller.reset_peripherals(peripherals);
```

**Section sources**
- [src/lib.rs](file://src/lib.rs#L120-L138)

### is_reset_done
查询当前复位状态。

- **参数**: 无
- **返回值语义**: 复位完成返回`true`，正在进行中返回`false`
- **成功条件**: 读取到复位状态寄存器的完成标志位为1
- **失败条件**: 无（此方法仅查询状态）
- **副作用**: 无
- **使用示例**:
```rust
if controller.is_reset_done() {
    log::info!("Reset operation finished");
}
```

**Section sources**
- [src/lib.rs](file://src/lib.rs#L141-L148)

### wait_for_reset_done
阻塞等待复位操作完成。

- **参数**: 无
- **返回值语义**: 在超时前完成返回`true`，超时返回`false`
- **成功条件**: 在500次轮询周期内检测到完成标志
- **失败条件**: 超过500次轮询仍未完成
- **副作用**: 执行忙等待循环，消耗CPU周期
- **使用示例**:
```rust
// 通常由其他复位方法内部调用
let success = controller.wait_for_reset_done();
```

**Section sources**
- [src/lib.rs](file://src/lib.rs#L151-L164)

## 外设ID枚举

### PeripheralId
表示可复位的外设类型。

#### 枚举成员
- `Gpio0`: GPIO0外设
- `Gpio1`: GPIO1外设  
- `Gpio2`: GPIO2外设
- `Gpio3`: GPIO3外设
- `Gpio4`: GPIO4外设
- `Gpio5`: GPIO5外设

### mask方法
生成对应的位掩码用于寄存器操作。

- **行为**: 将枚举值转换为32位掩码，每个外设对应一个独立的位
- **返回值**: `u32`类型的位掩码
- **使用示例**:
```rust
let mask = PeripheralId::Gpio2.mask(); // 返回 0b100 (即 4)
```

**Section sources**
- [src/lib.rs](file://src/lib.rs#L48-L64)

## 配置与句柄结构体

### CruConfig
CRU控制器的配置信息。

#### 字段用途
- `instance_id`: 实例标识符，用于区分不同CRU实例
- `base_address`: 寄存器基地址的物理内存地址

**Section sources**
- [src/lib.rs](file://src/lib.rs#L167-L173)

### CruHandle
全局CRU状态句柄。

#### 字段用途
- `config`: 存储当前CRU配置
- `is_ready`: 就绪状态标志，0x11111111表示已初始化

**Section sources**
- [src/lib.rs](file://src/lib.rs#L176-L185)

## 初始化与查找函数

### init_cru
初始化CRU控制器。

- **前置条件**: 必须在全局初始化阶段调用，且只能成功调用一次
- **行为逻辑**: 设置全局句柄的配置信息和就绪状态
- **错误码含义**: 
  - `Ok(())`: 初始化成功
  - `"CRU already initialized"`: 已经初始化过
- **使用示例**:
```rust
let config = CruConfig {
    instance_id: 0,
    base_address: 0x2800_0000,
};
let result = init_cru(config);
```

**Section sources**
- [src/lib.rs](file://src/lib.rs#L188-L208)

### lookup_config
根据实例ID查找配置。

- **前置条件**: 实例ID必须小于1
- **行为逻辑**: 返回预定义的配置信息
- **错误码含义**: 
  - `Some(config)`: 找到有效配置
  - `None`: 实例ID无效
- **使用示例**:
```rust
if let Some(config) = lookup_config(0) {
    // 使用配置信息
}
```

**Section sources**
- [src/lib.rs](file://src/lib.rs#L211-L222)

## with_cru宏机制

### 展开机制
`with_cru`宏提供了一种安全访问全局CRU资源的方式。

#### 执行流程
1. 获取全局锁：通过`GLOBAL_CRU.get()`获取全局实例
2. 检查就绪状态：验证`is_ready`标志是否为0x11111111
3. 构造临时实例：使用安全的`unsafe`块创建`CruController`
4. 执行闭包：将控制器传递给用户提供的操作闭包
5. 错误传播：任何检查失败都返回`Err("CRU not initialized")`

#### 安全考虑
- **Unsafe操作**: 仅在确认初始化后进行指针转换
- **资源竞争风险**: 通过`Mutex`确保线程安全
- **使用示例**:
```rust
let result = with_cru!(|mut ctrl| {
    ctrl.reset_peripheral(PeripheralId::Gpio0)
});
```

**Section sources**
- [src/lib.rs](file://src/lib.rs#L225-L248)

## API模块导出

### 重新导出的函数
`api`模块重新导出了以下带错误处理的便捷函数：

- `system_reset() -> Result<bool, &'static str>`
- `reset_peripheral(periph_id: PeripheralId) -> Result<bool, &'static str>`
- `reset_peripherals(periph_ids: &[PeripheralId]) -> Result<bool, &'static str>`
- `is_reset_done() -> Result<bool, &'static str>`

这些函数均基于`with_cru`宏实现，自动处理初始化状态检查和错误传播。

**Section sources**
- [src/lib.rs](file://src/lib.rs#L251-L270)