# API参考文档

<cite>
**Referenced Files in This Document**  
- [lib.rs](file://src/lib.rs)
</cite>

## 目录
1. [简介](#简介)
2. [PwmController结构体方法](#pwmcontroller结构体方法)
3. [PwmConfig配置结构体](#pwmconfig配置结构体)
4. [with_pwm!宏](#with_pwm宏)
5. [api模块导出函数](#api模块导出函数)

## 简介
本技术参考文档详细描述了Phytium Pi平台PWM驱动的所有公共API。文档涵盖了`PwmController`结构体的完整方法集、`PwmConfig`配置结构体的默认实现、`with_pwm!`宏的运行时安全机制，以及`api`模块中所有导出函数的详细说明。每个API都明确标注了其异步特性、可重入性、线程安全性保障机制，并提供了简明的调用示例。

## PwmController结构体方法

### new方法
创建新的PWM控制器实例。

**安全特性及调用前提**：  
该方法被标记为`unsafe`，因为调用者必须确保传入的`base`指针指向有效的PWM寄存器地址。如果提供无效地址，可能导致未定义行为。

**调用示例**：
```rust
let controller = unsafe { PwmController::new(0x2804a000 as *mut u8) };
```

**Section sources**
- [lib.rs](file://src/lib.rs#L67-L73)

### init方法
初始化PWM控制器。

**参数含义**：
- `period`: PWM周期计数值
- `divider`: 时钟分频系数（默认使用1 = 2分频）

**内部操作步骤**：
1. 执行软件复位
2. 等待复位完成
3. 配置分频系数并禁用PWM
4. 设置指定周期
5. 配置为比较模式
6. 设置默认占空比为50%
7. 使能PWM输出

**调用示例**：
```rust
controller.init(10000, 1);
```

**Section sources**
- [lib.rs](file://src/lib.rs#L75-L119)

### set_duty_cycle方法
设置PWM占空比。

**输入范围验证**：  
占空比百分比必须在1-100范围内。若输入值为0或大于100，将返回错误。

**错误类型**：  
返回`Err("Duty cycle must be between 1-100")`表示输入值超出有效范围。

**调用示例**：
```rust
match controller.set_duty_cycle(75) {
    Ok(()) => println!("Duty cycle set successfully"),
    Err(e) => println!("Error: {}", e),
}
```

**Section sources**
- [lib.rs](file://src/lib.rs#L121-L135)

### get_duty_cycle方法
获取当前占空比。

**返回值计算逻辑**：  
从寄存器读取周期和占空比计数值，由于寄存器值是反向的，计算公式为：`100 - (duty_count * 100) / period`。若周期为0，则返回0以避免除零错误。

**调用示例**：
```rust
let duty = controller.get_duty_cycle();
println!("Current duty cycle: {}%", duty);
```

**Section sources**
- [lib.rs](file://src/lib.rs#L137-L153)

### set_period方法
设置PWM周期。

**保持占空比的实现**：  
首先通过`get_duty_cycle()`获取当前占空比百分比，然后设置新周期，最后调用`set_duty_cycle()`恢复原有的占空比比例，从而保证占空比不变。

**调用示例**：
```rust
controller.set_period(5000);
```

**Section sources**
- [lib.rs](file://src/lib.rs#L155-L167)

### get_period方法
获取当前周期。

**调用示例**：
```rust
let period = controller.get_period();
println!("Current period: {}", period);
```

**Section sources**
- [lib.rs](file://src/lib.rs#L169-L173)

### enable方法
使能PWM输出。

**调用示例**：
```rust
controller.enable();
```

**Section sources**
- [lib.rs](file://src/lib.rs#L175-L178)

### disable方法
禁用PWM输出。

**调用示例**：
```rust
controller.disable();
```

**Section sources**
- [lib.rs](file://src/lib.rs#L180-L183)

### is_enabled方法
检查PWM是否使能。

**调用示例**：
```rust
if controller.is_enabled() {
    println!("PWM is enabled");
} else {
    println!("PWM is disabled");
}
```

**Section sources**
- [lib.rs](file://src/lib.rs#L185-L188)

## PwmConfig配置结构体

### 结构体定义
`PwmConfig`结构体包含以下字段：
- `base_address`: 基地址
- `default_period`: 默认周期
- `default_divider`: 默认分频系数

### Default实现中的默认参数值
```rust
impl Default for PwmConfig {
    fn default() -> Self {
        Self {
            base_address: 0x2804a000,
            default_period: 10000,
            default_divider: 1,
        }
    }
}
```

**Section sources**
- [lib.rs](file://src/lib.rs#L190-L201)

## with_pwm!宏

### 语法结构
```rust
with_pwm!(|controller| operation(controller))
```

### 展开逻辑
宏展开后执行以下逻辑：
1. 获取全局PWM单例实例
2. 检查实例是否存在
3. 锁定互斥锁获取可变引用
4. 检查PWM控制器是否已初始化
5. 在闭包中执行指定操作
6. 自动释放锁并返回结果

### 运行时安全访问全局实例
通过`Once<Mutex<Option<PwmController>>>`实现：
- `Once`确保初始化只执行一次
- `Mutex`提供线程安全的可变访问
- `Option`允许检查是否已初始化
- 宏封装了所有安全检查，防止未初始化访问

**Section sources**
- [lib.rs](file://src/lib.rs#L203-L228)

## api模块导出函数

### set_duty_cycle函数
**对应底层方法**：`PwmController::set_duty_cycle`  
**异步**：否  
**可重入**：是（通过Mutex保护）  
**线程安全性**：通过全局Mutex实现  
**调用示例**：
```rust
match api::set_duty_cycle(50) {
    Ok(()) => println!("Success"),
    Err(e) => println!("Error: {}", e),
}
```

**Section sources**
- [lib.rs](file://src/lib.rs#L230-L234)

### get_duty_cycle函数
**对应底层方法**：`PwmController::get_duty_cycle`  
**异步**：否  
**可重入**：是（通过Mutex保护）  
**线程安全性**：通过全局Mutex实现  
**调用示例**：
```rust
match api::get_duty_cycle() {
    Ok(duty) => println!("Duty cycle: {}%", duty),
    Err(e) => println!("Error: {}", e),
}
```

**Section sources**
- [lib.rs](file://src/lib.rs#L236-L240)

### set_period函数
**对应底层方法**：`PwmController::set_period`  
**异步**：否  
**可重入**：是（通过Mutex保护）  
**线程安全性**：通过全局Mutex实现  
**调用示例**：
```rust
match api::set_period(8000) {
    Ok(()) => println!("Period set successfully"),
    Err(e) => println!("Error: {}", e),
}
```

**Section sources**
- [lib.rs](file://src/lib.rs#L242-L249)

### get_period函数
**对应底层方法**：`PwmController::get_period`  
**异步**：否  
**可重入**：是（通过Mutex保护）  
**线程安全性**：通过全局Mutex实现  
**调用示例**：
```rust
match api::get_period() {
    Ok(period) => println!("Period: {}", period),
    Err(e) => println!("Error: {}", e),
}
```

**Section sources**
- [lib.rs](file://src/lib.rs#L251-L255)

### enable函数
**对应底层方法**：`PwmController::enable`  
**异步**：否  
**可重入**：是（通过Mutex保护）  
**线程安全性**：通过全局Mutex实现  
**调用示例**：
```rust
match api::enable() {
    Ok(()) => println!("PWM enabled"),
    Err(e) => println!("Error: {}", e),
}
```

**Section sources**
- [lib.rs](file://src/lib.rs#L257-L263)

### disable函数
**对应底层方法**：`PwmController::disable`  
**异步**：否  
**可重入**：是（通过Mutex保护）  
**线程安全性**：通过全局Mutex实现  
**调用示例**：
```rust
match api::disable() {
    Ok(()) => println!("PWM disabled"),
    Err(e) => println!("Error: {}", e),
}
```

**Section sources**
- [lib.rs](file://src/lib.rs#L265-L271)

### is_enabled函数
**对应底层方法**：`PwmController::is_enabled`  
**异步**：否  
**可重入**：是（通过Mutex保护）  
**线程安全性**：通过全局Mutex实现  
**调用示例**：
```rust
match api::is_enabled() {
    Ok(enabled) => println!("Enabled: {}", enabled),
    Err(e) => println!("Error: {}", e),
}
```