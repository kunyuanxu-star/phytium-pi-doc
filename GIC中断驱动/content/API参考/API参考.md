
# API参考

<cite>
**本文档中引用的文件**
- [lib.rs](file://gic-driver/src/lib.rs)
- [define.rs](file://gic-driver/src/define.rs)
- [version/mod.rs](file://gic-driver/src/version/mod.rs)
- [version/v2/mod.rs](file://gic-driver/src/version/v2/mod.rs)
- [version/v3/mod.rs](file://gic-driver/src/version/v3/mod.rs)
- [version/v2/gicd.rs](file://gic-driver/src/version/v2/gicd.rs)
- [version/v2/gicc.rs](file://gic-driver/src/version/v2/gicc.rs)
- [version/v3/gicd.rs](file://gic-driver/src/version/v3/gicd.rs)
- [version/v3/gicr.rs](file://gic-driver/src/version/v3/gicr.rs)
</cite>

## 目录
1. [VirtAddr结构体](#virtaddr结构体)
2. [IntId类型](#intid类型)
3. [GICv2 Gic结构体](#gicv2-gic结构体)
4. [GICv3 Gic结构体](#gicv3-gic结构体)
5. [CpuInterface接口](#cpuinterface接口)
6. [HypervisorInterface接口](#hypervisorinterface接口)
7. [安全边界与unsafe代码使用](#安全边界与unsafe代码使用)

## VirtAddr结构体

`VirtAddr`结构体为内存映射寄存器访问提供了虚拟地址的安全包装。它确保类型安全，同时允许高效的指针操作。

### new方法
创建一个新的`VirtAddr`实例。

**方法签名**
```rust
pub const fn new(val: usize) -> Self
```

**参数说明**
- `val`：作为`usize`值的虚拟地址

**返回值解释**
返回一个新的`VirtAddr`实例

**使用示例**
```rust
use arm_gic_driver::VirtAddr;
let addr = VirtAddr::new(0xF900_0000);
```

### as_ptr方法
获取指定类型的目标指针。

**方法签名**
```rust
pub const fn as_ptr<T>(&self) -> *mut T
```

**类型参数**
- `T`：目标指针类型

**返回值解释**
返回指向类型`T`的原始可变指针

**安全要求**
调用者必须确保：
- 地址对目标类型`T`有效
- 内存区域已正确映射且可访问
- 对于并发访问使用适当的同步机制

**使用示例**
```rust
use arm_gic_driver::VirtAddr;
let addr = VirtAddr::new(0xF900_0000);
let ptr: *mut u32 = addr.as_ptr();
```

**Section sources**
- [lib.rs](file://gic-driver/src/lib.rs#L25-L55)

## IntId类型

`IntId`结构体表示GIC的中断标识符（INTID），代表可用于GIC硬件的唯一中断ID。

### raw方法
从原始中断ID创建新的`IntId`。

**方法签名**
```rust
pub const unsafe fn raw(id: u32) -> Self
```

**参数说明**
- `id`：原始中断ID值

**安全要求**
调用者必须确保`id`根据GIC规范表示有效的中断ID。无效的ID在与GIC操作一起使用时可能导致未定义行为。

**使用示例**
```rust
use arm_gic_driver::IntId;
let intid = unsafe { IntId::raw(32) }; // SPI #0
```

### sgi方法
为软件生成的中断创建中断ID。

**方法签名**
```rust
pub const fn sgi(sgi: u32) -> Self
```

**参数说明**
- `sgi`：SGI编号（0-15）

**可能的错误类型**
如果`sgi`大于或等于16，则会panic。

**使用示例**
```rust
use arm_gic_driver::IntId;
let sgi1 = IntId::sgi(1);
assert_eq!(sgi1.to_u32(), 1);
assert!(sgi1.is_sgi());
```

### ppi方法
为私有外设中断创建中断ID。

**方法签名**
```rust
pub const fn ppi(ppi: u32) -> Self
```

**参数说明**
- `ppi`：PPI编号（0-15，映射到中断ID 16-31）

**可能的错误类型**
如果`ppi`大于或等于16，则会panic。

**使用示例**
```rust
use arm_gic_driver::IntId;
let ppi2 = IntId::ppi(2);
assert_eq!(ppi2.to_u32(), 18); // 16 + 2
assert!(ppi2.is_private());
```

### spi方法
为共享外设中断创建中断ID。

**方法签名**
```rust
pub const fn spi(spi: u32) -> Self
```

**参数说明**
- `spi`：SPI编号（0-987，映射到中断ID 32-1019）

**可能的错误类型**
如果`spi`会导致中断ID >= 1020，则会panic。

**使用示例**
```rust
use arm_gic_driver::IntId;
let spi42 = IntId::spi(42);
assert_eq!(spi42.to_u32(), 74); // 32 + 42
assert!(!spi42.is_private());
```

### is_sgi方法
检查此中断ID是否为软件生成的中断。

**方法签名**
```rust
pub fn is_sgi(&self) -> bool
```

**返回值解释**
如果这是SGI（ID 0-15），则返回`true`；否则返回`false`。

**使用示例**
```rust
use arm_gic_driver::IntId;
assert!(IntId::sgi(5).is_sgi());
assert!(!IntId::ppi(5).is_sgi());
```

### is_private方法
检查此中断ID是否为CPU核心私有。

**方法签名**
```rust
pub fn is_private(&self) -> bool
```

**返回值解释**
如果这是私有中断（SGI或PPI），则返回`true`；对于SPI返回`false`。

**使用示例**
```rust
use arm_gic_driver::IntId;
assert!(IntId::sgi(1).is_private());   // SGI
assert!(IntId::ppi(5).is_private());   // PPI
assert!(!IntId::spi(42).is_private()); // SPI
```

### to_u32方法
获取原始中断ID作为u32值。

**方法签名**
```rust
pub const fn to_u32(self) -> u32
```

**返回值解释**
返回GIC硬件使用的中断ID。

**使用示例**
```rust
use arm_gic_driver::IntId;
let spi = IntId::spi(10);
assert_eq!(spi.to_u32(), 42); // 32 + 10
```

### is_special方法
检查此中断ID是否在特殊范围内。

**方法签名**
```rust
pub fn is_special(&self) -> bool
```

**返回值解释**
如果这是特殊中断ID，则返回`true`；否则返回`false`。

**使用示例**
```rust
use arm_gic_driver::IntId;
let special = unsafe { IntId::raw(1023) };
assert!(special.is_special());

let normal = IntId::spi(10);
assert!(!normal.is_special());
```

**Section sources**
- [define.rs](file://gic-driver/src/define.rs#L120-L316)

## GICv2 Gic结构体

`Gic`结构体是GICv2驱动程序的主要接口，管理分发器（GICD）和CPU接口（GICC）的控制。

### new方法
创建一个新的GICv2驱动程序实例。

**方法签名**
```rust
pub const unsafe fn new(gicd: VirtAddr, gicc: VirtAddr, hyper: Option<HyperAddress>) -> Self
```

**参数说明**
- `gicd`：分发器寄存器基地址
- `gicc`：CPU接口寄存器基地址
- `hyper`：超管理器接口地址（可选）

**安全要求**
调用者必须确保提供的指针有效并指向正确的GICv2寄存器。

**使用示例**
```rust
use arm_gic_driver::{VirtAddr, v2::Gic, v2::HyperAddress};

let gicd_addr = VirtAddr::new(0x0800_0000);
let gicc_addr = VirtAddr::new(0x0801_0000);
let hyper_addr = Some(HyperAddress::new(
    VirtAddr::new(0x0802_0000),
    VirtAddr::new(0x0803_0000)
));

let gic = unsafe { Gic::new(gicd_addr, gicc_addr, hyper_addr) };
```

### init方法
根据GICv2规范初始化GIC。

**方法签名**
```rust
pub fn