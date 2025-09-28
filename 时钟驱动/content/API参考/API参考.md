<cite>
**本文档引用的文件**
- [lib.rs](file://src/lib.rs)
- [basic_usage.rs](file://examples/basic_usage.rs)
</cite>

# API参考

## 目录
1. [核心类型定义](#核心类型定义)
2. [初始化与配置函数](#初始化与配置函数)
3. [API模块函数](#api模块函数)
4. [with_clock!宏](#with_clock宏)

## 核心类型定义

### ClockController
时钟控制器结构体，用于直接操作硬件寄存器。

**字段**
- `regs`: 指向时钟控制器寄存器的非空指针（`NonNull<ClockRegs>`）

**生命周期**
- 通过`new()`方法创建实例
- 实例持有对底层硬件寄存器的独占访问权
- 生命周期与系统运行周期一致

**方法签名**

#### new
```rust
pub unsafe fn new(base: *mut u8) -> Self
```
- **是否为unsafe**: 是
- **参数约束**: `base`必须指向有效的时钟控制器寄存器地址空间
- **返回值**: 新的`ClockController`实例
- **错误码**: 无显式错误码，但传入无效地址会导致未定义行为

#### set_frequency
```rust
pub fn set_frequency(&mut self, freq: u32) -> bool
```
- **是否为unsafe**: 否
- **参数约束**: 
  - `freq`必须大于0且不超过50MHz
  - 计算出的分频系数不能超过255
- **返回值**: 成功设置返回`true`，否则返回`false`
- **错误码**:
  - `false`: 频率超出有效范围或超时未就绪

#### get_frequency
```rust
pub fn get_frequency(&self) -> u32
```
- **是否为unsafe**: 否
- **参数约束**: 无
- **返回值**: 当前时钟频率（Hz）
- **错误码**: 无

#### enable
```rust
pub fn enable(&mut self)
```
- **是否为unsafe**: 否
- **参数约束**: 无
- **返回值**: 无
- **错误码**: 无

#### disable
```rust
pub fn disable(&mut self)
```
- **是否为unsafe**: 否
- **参数约束**: 无
- **返回值**: 无
- **错误码**: 无

#### is_ready
```rust
pub fn is_ready(&self) -> bool
```
- **是否为unsafe**: 否
- **参数约束**: 无
- **返回值**: 时钟是否准备就绪
- **错误码**: 无

**最小可复现示例**
```rust
let mut controller = unsafe { ClockController::new(0x2800_0000 as *mut u8) };
controller.set_frequency(25_000_000);
```

**Section sources**
- [lib.rs](file://src/lib.rs#L47-L169)

### ClockConfig
时钟配置结构体，包含初始化所需的基本信息。

**字段**
- `instance_id`: 实例ID（`u32`）
- `base_address`: 寄存器基地址（`usize`）

**生命周期**
- 在调用`init_clock`前创建
- 被`ClockHandle`持有并用于初始化`ClockController`

**方法签名**

#### new
```rust
pub const fn new() -> Self
```
- **是否为unsafe**: 否
- **参数约束**: 无
- **返回值**: 默认配置的`ClockConfig`实例
- **错误码**: 无

**最小可复现示例**
```rust
let config = ClockConfig {
    instance_id: 0,
    base_address: 0x2800_0000,
};
```

**Section sources**
- [lib.rs](file://src/lib.rs#L172-L181)

### ClockHandle
时钟控制句柄，管理全局状态。

**字段**
- `config`: 时钟配置信息（`ClockConfig`）
- `is_ready`: 初始化状态标志（`u32`）

**生命周期**
- 作为全局静态变量存在
- 通过`Once`机制确保单次初始化

**方法签名**

#### new
```rust
pub const fn new() -> Self
```
- **是否为unsafe**: 否
- **参数约束**: 无
- **返回值**: 新的`ClockHandle`实例
- **错误码**: 无

**最小可复现示例**
```rust
static GLOBAL_CLOCK: Once<Mutex<ClockHandle>> = Once::new();
```

**Section sources**
- [lib.rs](file://src/lib.rs#L184-L195)

## 初始化与配置函数

### init_clock
初始化时钟控制器。

```rust
pub fn init_clock(config: ClockConfig) -> Result<(), &'static str>
```

- **是否为unsafe**: 否
- **参数约束**: 
  - `config`必须包含有效的实例ID和基地址
- **返回值**: 
  - `Ok(())`: 初始化成功
  - `Err("Clock already initialized")`: 时钟已初始化
- **错误码**:
  - `"Clock already initialized"`: 全局时钟实例已被初始化

**调用前提**
- 必须在系统初始化阶段调用
- 只能成功调用一次

**行为描述**
- 创建全局时钟句柄
- 检查是否已初始化
- 存储配置信息
- 设置就绪标志

**最小可复现示例**
```rust
let config = lookup_config(0).unwrap();
init_clock(config).unwrap();
```

**Section sources**
- [lib.rs](file://src/lib.rs#L198-L217)

### lookup_config
获取指定实例的时钟配置。

```rust
pub fn lookup_config(instance_id: u32) -> Option<ClockConfig>
```

- **是否为unsafe**: 否
- **参数约束**: 
  - `instance_id`必须为0（当前仅支持单实例）
- **返回值**: 
  - `Some(ClockConfig)`: 找到有效配置
  - `None`: 实例ID无效
- **错误码**: 无显式错误码，返回`None`表示查找失败

**调用前提**
- 无特殊前提条件
- 可在任何阶段调用

**行为描述**
- 验证实例ID有效性
- 返回预设的配置信息

**最小可复现示例**
```rust
let config = lookup_config(0).expect("Failed to get clock config");
```

**Section sources**
- [lib.rs](file://src/lib.rs#L220-L233)

## API模块函数

### set_frequency
通过全局接口设置时钟频率。

```rust
pub fn set_frequency(freq: u32) -> Result<bool, &'static str>
```

- **是否为unsafe**: 否
- **参数约束**: 
  - `freq`必须大于0且不超过50MHz
  - 分频系数不能超过255
- **返回值**: 
  - `Ok(true)`: 频率设置成功
  - `Ok(false)`: 参数无效导致设置失败
  - `Err("Clock not initialized")`: 时钟未初始化
- **错误码**:
  - `"Clock not initialized"`: 全局时钟未初始化
  - 返回`false`表示频率超出有效范围

**调用前提**
- 必须先调用`init_clock`完成初始化
- 必须在单线程上下文中调用

**行为描述**
- 通过`with_clock!`宏获取全局时钟访问权
- 调用`ClockController::set_frequency`
- 处理未初始化状态

**最小可复现示例**
```rust
match api::set_frequency(25_000_000) {
    Ok(true) => log::info!("Frequency set successfully"),
    Err(e) => log::error!("Error: {}", e),
    _ => {}
}
```

**Section sources**
- [lib.rs](file://src/lib.rs#L247-L253)

### get_frequency
获取当前时钟频率。

```rust
pub fn get_frequency() -> Result<u32, &'static str>
```

- **是否为unsafe**: 否
- **参数约束**: 无
- **返回值**: 
  - `Ok(u32)`: 当前频率值
  - `Err("Clock not initialized")`: 时钟未初始化
- **错误码**:
  - `"Clock not initialized"`: 全局时钟未初始化

**调用前提**
- 必须先调用`init_clock`完成初始化

**行为描述**
- 通过`with_clock!`宏获取全局时钟访问权
- 调用`ClockController::get_frequency`
- 返回实际频率值

**最小可复现示例**
```rust
match api::get_frequency() {
    Ok(freq) => log::info!("Current frequency: {} Hz", freq),
    Err(e) => log::error!("Error: {}", e),
}
```

**Section sources**
- [lib.rs](file://src/lib.rs#L255-L259)

### enable
使能时钟输出。

```rust
pub fn enable() -> Result<(), &'static str>
```

- **是否为unsafe**: 否
- **参数约束**: 无
- **返回值**: 
  - `Ok(())`: 成功使能
  - `Err("Clock not initialized")`: 时钟未初始化
- **错误码**:
  - `"Clock not initialized"`: 全局时钟未初始化

**调用前提**
- 必须先调用`init_clock`完成初始化

**行为描述**
- 通过`with_clock!`宏获取全局时钟访问权
- 调用`ClockController::enable`
- 确保时钟信号输出

**最小可复现示例**
```rust
match api::enable() {
    Ok(()) => log::info!("Clock enabled"),
    Err(e) => log::error!("Error: {}", e),
}
```

**Section sources**
- [lib.rs](file://src/lib.rs#L261-L267)

### disable
禁用时钟输出。

```rust
pub fn disable() -> Result<(), &'static str>
```

- **是否为unsafe**: 否
- **参数约束**: 无
- **返回值**: 
  - `Ok(())`: 成功禁用
  - `Err("Clock not initialized")`: 时钟未初始化
- **错误码**:
  - `"Clock not initialized"`: 全局时钟未初始化

**调用前提**
- 必须先调用`init_clock`完成初始化

**行为描述**
- 通过`with_clock!`宏获取全局时钟访问权
- 调用`ClockController::disable`
- 停止时钟信号输出

**最小可复现示例**
```rust
match api::disable() {
    Ok(()) => log::info!("Clock disabled"),
    Err(e) => log::error!("Error: {}", e),
}
```

**Section sources**
- [lib.rs](file://src/lib.rs#L269-L275)

### is_ready
检查时钟是否准备就绪。

```rust
pub fn is_ready() -> Result<bool, &'static str>
```

- **是否为unsafe**: 否
- **参数约束**: 无
- **返回值**: 
  - `Ok(bool)`: 时钟就绪状态
  - `Err("Clock not initialized")`: 时钟未初始化
- **错误码**:
  - `"Clock not initialized"`: 全局时钟未初始化

**调用前提**
- 必须先调用`init_clock`完成初始化

**行为描述**
- 通过`with_clock!`宏获取全局时钟访问权
- 调用`ClockController::is_ready`
- 返回硬件就绪状态

**最小可复现示例**
```rust
match api::is_ready() {
    Ok(true) => log::info!("Clock is ready"),
    Ok(false) => log::warn!("Clock is not ready"),
    Err(e) => log::error!("Error: {}", e),
}
```

**Section sources**
- [lib.rs](file://src/lib.rs#L277-L283)

## with_clock!宏

### 展开机制
该宏简化了对全局时钟实例的安全访问。

**展开过程**
1. 获取全局时钟实例引用
2. 检查实例是否存在
3. 获取互斥锁保护的句柄
4. 验证初始化状态
5. 创建临时`ClockController`实例
6. 执行传入的操作闭包
7. 自动释放锁和资源

**使用上下文限制**
- 只能在单线程环境中安全使用
- 不能在中断处理程序中使用（可能死锁）
- 操作闭包内不应有长时间阻塞
- 不能递归调用自身

**最小可复现示例**
```rust
with_clock!(|mut controller| {
    controller.set_frequency(25_000_000);
    controller.enable();
})
```

**Section sources**
- [lib.rs](file://src/lib.rs#L236-L245)