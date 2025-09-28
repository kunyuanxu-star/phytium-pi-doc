# API操作函数

<cite>
**Referenced Files in This Document **   
- [lib.rs](file://src/lib.rs)
- [basic_usage.rs](file://examples/basic_usage.rs)
</cite>

## Table of Contents
1. [API函数详解](#api函数详解)
2. [运行时异常与恢复策略](#运行时异常与恢复策略)

## API函数详解

### `set_frequency(frequency: u32)`

该函数用于设置系统时钟频率。

**接口说明**
- **参数范围**: 频率值必须在 196,088 Hz (约196kHz) 到 50,000,000 Hz (50MHz) 之间。输入值为0或超过50MHz将导致调用失败。
- **返回类型**: `Result<bool, &'static str>`，成功时返回`Ok(true)`表示频率已成功设置，`Ok(false)`表示硬件操作失败，错误时返回`Err("Clock not initialized")`。
- **副作用**: 触发硬件重配置，包括修改分频系数寄存器(CLK_DIV)、使能时钟控制寄存器(CLK_CON)，并等待时钟稳定。
- **线程安全性保证**: 通过`with_clock!`宏实现，使用`Once<Mutex<ClockHandle>>`确保全局实例的线程安全访问。
- **宏使用**: 内部使用`with_clock!`宏访问全局时钟实例。
- **阻塞行为**: 可能阻塞最多500次轮询等待，每次轮询包含1000次`spin_loop()`，总延迟取决于平台时钟速度。
- **生效条件与延迟**: 频率调整在写入分频寄存器后开始，但需等待硬件反馈就绪状态(READY位)才视为完全生效。存在最大约500个轮询周期的延迟。
- **调用示例**:
```rust
match api::set_frequency(25_000_000) {
    Ok(true) => log::info!("Frequency set successfully"),
    Ok(false) => log::error!("Failed to set frequency"),
    Err(e) => log::error!("Error: {}", e),
}
```
- **安全标记**: 函数本身不是`unsafe`，但底层`ClockController::new`是`unsafe`，由`with_clock!`宏内部处理。

**Section sources**
- [lib.rs](file://src/lib.rs#L247-L251)

### `get_frequency()`

该函数用于获取当前系统时钟频率。

**接口说明**
- **参数范围**: 无参数。
- **返回类型**: `Result<u32, &'static str>`，成功时返回当前频率(Hz)，错误时返回`Err("Clock not initialized")`。
- **副作用**: 读取硬件寄存器(CLK_DIV)，无其他副作用。
- **线程安全性保证**: 通过`with_clock!`宏和`Mutex`锁机制保证线程安全。
- **宏使用**: 内部使用`with_clock!`宏访问全局时钟实例。
- **阻塞行为**: 不阻塞，直接读取寄存器值。
- **调用示例**:
```rust
match api::get_frequency() {
    Ok(freq) => log::info!("Current frequency: {} Hz", freq),
    Err(e) => log::error!("Failed to get frequency: {}", e),
}
```
- **安全标记**: 不是`unsafe`函数。

**Section sources**
- [lib.rs](file://src/lib.rs#L253-L256)

### `enable()`

该函数用于使能时钟输出。

**接口说明**
- **参数范围**: 无参数。
- **返回类型**: `Result<(), &'static str>`，成功时返回`Ok(())`，错误时返回`Err("Clock not initialized")`。
- **副作用**: 修改时钟控制寄存器(CLK_CON)的ENABLE位，触发硬件时钟输出。
- **线程安全性保证**: 通过`with_clock!`宏和`Mutex`保证多线程环境下的安全访问。
- **宏使用**: 内部使用`with_clock!`宏访问全局时钟实例。
- **阻塞行为**: 不阻塞，立即返回。
- **调用示例**:
```rust
match api::enable() {
    Ok(()) => log::info!("Clock enabled"),
    Err(e) => log::error!("Failed to enable clock: {}", e),
}
```
- **安全标记**: 不是`unsafe`函数。

**Section sources**
- [lib.rs](file://src/lib.rs#L258-L263)

### `disable()`

该函数用于禁用时钟输出。

**接口说明**
- **参数范围**: 无参数。
- **返回类型**: `Result<(), &'static str>`，成功时返回`Ok(())`，错误时返回`Err("Clock not initialized")`。
- **副作用**: 清除时钟控制寄存器(CLK_CON)的ENABLE位，停止硬件时钟输出。
- **线程安全性保证**: 通过`with_clock!`宏和`Mutex`机制确保线程安全。
- **宏使用**: 内部使用`with_clock!`宏访问全局时钟实例。
- **阻塞行为**: 不阻塞，立即返回。
- **调用示例**:
```rust
match api::disable() {
    Ok(()) => log::info!("Clock disabled"),
    Err(e) => log::error!("Failed to disable clock: {}", e),
}
```
- **安全标记**: 不是`unsafe`函数。

**Section sources**
- [lib.rs](file://src/lib.rs#L265-L270)

### `is_ready()`

该函数用于检查时钟是否处于稳定工作状态。

**接口说明**
- **参数范围**: 无参数。
- **返回类型**: `Result<bool, &'static str>`，成功时返回`Ok(bool)`，其中`true`表示时钟已准备好，`false`表示未准备好；错误时返回`Err("Clock not initialized")`。
- **副作用**: 读取时钟状态寄存器(CLK_STATUS)，无其他副作用。
- **线程安全性保证**: 通过`with_clock!`宏和`Mutex`提供线程安全保证。
- **宏使用**: 内部使用`with_clock!`宏访问全局时钟实例。
- **阻塞行为**: 不阻塞，直接读取寄存器状态。
- **‘就绪’状态判定标准**: 硬件通过检测时钟信号的稳定性来设置CLK_STATUS寄存器的READY位(偏移0，1位宽)。当硬件确认时钟信号已稳定且无抖动时，将此位置为1。
- **调用示例**:
```rust
match api::is_ready() {
    Ok(true) => log::info!("Clock is ready"),
    Ok(false) => log::warn!("Clock is not ready"),
    Err(e) => log::error!("Failed to check clock status: {}", e),
}
```
- **安全标记**: 不是`unsafe`函数。

**Section sources**
- [lib.rs](file://src/lib.rs#L272-L274)

## 运行时异常与恢复策略

### 可能的运行时异常情况

1. **控制器未初始化**
   - **表现**: 所有API函数返回`Err("Clock not initialized")`
   - **原因**: `init_clock()`未被调用或调用失败
   - **诊断**: 检查`GLOBAL_CLOCK`是否已通过`call_once`初始化

2. **寄存器访问失败**
   - **表现**: `set_frequency`返回`Ok(false)`
   - **原因**: 
     - 频率超出有效范围(0或>50MHz)
     - 分频系数计算结果大于255
     - 硬件超时(500次轮询内未收到就绪信号)
   - **诊断**: 检查输入频率值和硬件连接状态

3. **配置查找失败**
   - **表现**: `lookup_config()`返回`None`
   - **原因**: 请求的实例ID不为0（目前仅支持ID=0）

### 错误恢复策略

1. **初始化失败恢复**
   ```rust
   // 尝试重新初始化
   let config = lookup_config(0).expect("No valid configuration");
   match init_clock(config) {
       Ok(()) => log::info!("Clock reinitialized"),
       Err("Clock already initialized") => {
           // 已初始化，可继续操作
           log::info!("Clock already running");
       }
       Err(e) => panic!("Fatal: Cannot initialize clock: {}", e),
   }
   ```

2. **频率设置失败恢复**
   ```rust
   fn safe_set_frequency(target_freq: u32) -> Result<u32, &'static str> {
       const MAX_FREQ: u32 = 50_000_000;
       const MIN_FREQ: u32 = 196_088; // SYS_CLK_HZ / 255
       
       if target_freq == 0 || target_freq > MAX_FREQ {
           return Err("Invalid frequency range");
       }
       
       // 调整到最近的有效频率
       let adjusted_freq = if target_freq < MIN_FREQ { 
           MIN_FREQ 
       } else { 
           target_freq 
       };
       
       match api::set_frequency(adjusted_freq) {
           Ok(true) => Ok(adjusted_freq),
           _ => Err("Hardware operation failed"),
       }
   }
   ```

3. **通用错误处理模式**
   ```rust
   // 在应用中统一处理时钟操作
   macro_rules! clock_operation {
       ($op:expr) => {
           match $op {
               Ok(result) => result,
               Err("Clock not initialized") => {
                   log::error!("Clock subsystem not available");
                   // 触发系统级错误处理
                   return Err(SystemError::ClockUnavailable);
               }
               _ => {
                   log::error!("Clock operation failed");
                   // 可选：尝试复位时钟控制器
                   api::disable();
                   // 短暂延时后重试
                   delay_ms(10);
                   api::enable();
                   return Err(SystemError::ClockOperationFailed);
               }
           }
       };
   }
   ```

4. **监控与健康检查**
   ```rust
   fn clock_health_check() -> ClockHealth {
       if let Ok(ready) = api::is_ready() {
           if ready {
               if let Ok(freq) = api::get_frequency() {
                   ClockHealth::Stable(freq)
               } else {
                   ClockHealth::Unstable("Cannot read frequency")
               }
           } else {
               ClockHealth::Initializing
           }
       } else {
           ClockHealth::NotInitialized
       }
   }
   ```

**Section sources**
- [lib.rs](file://src/lib.rs#L167-L211)
- [basic_usage.rs](file://examples/basic_usage.rs#L35-L64)