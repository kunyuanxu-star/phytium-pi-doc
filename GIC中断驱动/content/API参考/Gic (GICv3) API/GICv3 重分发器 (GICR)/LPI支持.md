<cite>
**Referenced Files in This Document**  
- [gicr.rs](file://gic-driver/src/version/v3/gicr.rs)
- [gicd.rs](file://gic-driver/src/version/v3/gicd.rs)
- [mod.rs](file://gic-driver/src/version/v3/mod.rs)
- [define.rs](file://gic-driver/src/define.rs)
- [lib.rs](file://gic-driver/src/lib.rs)
</cite>

# LPI支持

## Table of Contents
1. [RD_base寄存器帧结构](#rd_base寄存器帧结构)
2. [LPI操作机制](#lpi操作机制)
3. [PROPBASER和PENDBASER配置](#propbaser和pendbaser配置)
4. [TYPER寄存器功能](#typer寄存器功能)
5. [WAKER寄存器与电源管理](#waker寄存器与电源管理)

## RD_base寄存器帧结构

GICv3重分发器中的RD_base寄存器帧负责控制LPI（局部性外设中断）功能和重分发器的整体行为。该寄存器帧包含一系列用于LPI管理的寄存器，通过`LPI`结构体在代码中定义。

RD_base寄存器帧主要包含以下关键寄存器：
- `CTLR`：控制寄存器，用于启用/禁用LPI支持
- `TYPER`：类型寄存器，提供重分发器的特性信息
- `WAKER`：唤醒寄存器，用于电源管理
- `PROPBASER`：属性基地址寄存器，指向LPI配置表
- `PENDBASER`：挂起表基地址寄存器，指向LPI挂起状态表
- `SETLPIR`：设置LPI挂起寄存器
- `CLRLPIR`：清除LPI挂起寄存器
- `INVLPIR`：无效化LPI寄存器
- `INVALLR`：无效化所有LPI寄存器
- `SYNCR`：同步寄存器

这些寄存器共同构成了RD_base寄存器帧，为LPI功能提供了完整的控制接口。

**Section sources**
- [gicr.rs](file://gic-driver/src/version/v3/gicr.rs#L100-L150)

## LPI操作机制

LPI操作机制通过RD_base寄存器帧中的多个寄存器实现，提供了对LPI的全面控制功能。

### LPI使能/禁用

LPI支持通过`CTLR`寄存器的`EnableLPIs`位进行控制。当该位设置为1时，LPI功能被启用；当该位清除为0时，LPI功能被禁用。在禁用LPI功能后，需要等待`RWP`（Register Write Pending）位变为0，以确保寄存器写入操作已完成。

```rust
pub fn enable_lpi(&self) {
    self.CTLR.modify(RCtrl::EnableLPIs::SET);
}

pub fn disable_lpi(&self) {
    self.CTLR.modify(RCtrl::EnableLPIs::CLEAR);
    while self.CTLR.is_set(RCtrl::RWP) {
        spin_loop();
    }
}
```

### LPI挂起/清除

LPI的挂起状态通过`SETLPIR`和`CLRLPIR`寄存器进行控制。向`SETLPIR`寄存器写入LPI的中断ID可将该LPI设置为挂起状态，而向`CLRLPIR`寄存器写入中断ID可清除该LPI的挂起状态。

```rust
pub fn set_lpi_pending(&self, intid: u32) {
    self.SETLPIR.set(intid as u64);
}

pub fn clear_lpi_pending(&self, intid: u32) {
    self.CLRLPIR.set(intid as u64);
}
```

### LPI无效化

LPI的无效化操作通过`INVLPIR`和`INVALLR`寄存器实现。`INVLPIR`寄存器用于无效化单个LPI，而`INVALLR`寄存器用于无效化所有LPI。无效化操作会清除LPI的配置和挂起状态。

```rust
pub fn invalidate_lpi(&self, intid: u32) {
    self.INVLPIR.set(intid as u64);
}

pub fn invalidate_all_lpi(&self) {
    self.INVALLR.set(0);
}
```

**Section sources**
- [gicr.rs](file://gic-driver/src/version/v3/gicr.rs#L250-L350)

## PROPBASER和PENDBASER配置

`PROPBASER`（属性基地址寄存器）和`PENDBASER`（挂起表基地址寄存器）是LPI功能的关键配置寄存器，用于设置LPI相关数据结构的物理地址和缓存属性。

### PROPBASER配置

`PROPBASER`寄存器用于配置LPI属性表的基地址和属性。该寄存器包含以下关键字段：

- `PhysicalAddress`：40位物理地址，指向LPI属性表的基地址
- `IDbits`：5位，表示LPI ID的位数
- `InnerCache`：3位，设置内部缓存属性
- `OuterCache`：3位，设置外部缓存属性
- `Type`：2位，表示表类型

缓存属性支持以下配置：
- `NonCacheable`：非缓存
- `WaWb`：写分配/写回

```rust
register_bitfields! [
    u64,
    PROPBASER [
        IDbits OFFSET(0) NUMBITS(5) [],
        InnerCache OFFSET(7) NUMBITS(3) [
            NonCacheable = 0b001,
            WaWb = 0b111,
        ],
        Type OFFSET(10) NUMBITS(2) [],
        OuterCache OFFSET(56) NUMBITS(3) [
            NonCacheable = 0b001,
            WaWb = 0b111,
        ],
        PhysicalAddress OFFSET(12) NUMBITS(40) [],
    ],
];
```

### PENDBASER配置

`PENDBASER`寄存器用于配置LPI挂起表的基地址和属性。该寄存器包含以下关键字段：

- `PhysicalAddress`：36位物理地址，指向LPI挂起表的基地址
- `InnerCache`：3位，设置内部缓存属性
- `OuterCache`：3位，设置外部缓存属性
- `PTZ`：1位，表示挂起表大小

```rust
register_bitfields! [
    u64,
    PENDBASER [
        InnerCache OFFSET(7) NUMBITS(3) [
            NonCacheable = 0b001,
            WaWb = 0b111,
        ],
        OuterCache OFFSET(56) NUMBITS(3) [
            NonCacheable = 0b001,
            WaWb = 0b111,
        ],
        PTZ OFFSET(62) NUMBITS(1) [],
        PhysicalAddress OFFSET(16) NUMBITS(36) [],
    ],
];
```

表大小通过`PTZ`位确定，当`PTZ`为0时，表大小为8KB；当`PTZ`为1时，表大小为64KB。

**Section sources**
- [gicr.rs](file://gic-driver/src/version/v3/gicr.rs#L150-L200)

## TYPER寄存器功能

`TYPER`寄存器提供重分发器的类型信息和功能支持情况，其中包含LPI支持的关键位。

### PLPIS和VLPIS位

`TYPER`寄存器中的`PLPIS`（Physical LPI Support）和`VLPIS`（Virtual LPI Support）位用于指示重分发器对LPI的支持能力：

- `PLPIS`（位0）：指示GIC实现是否支持物理LPI
- `VLPIS`（位1）：指示重分发器是否支持虚拟LPI

通过读取这些位的值，可以确定当前GIC实现对LPI的支持情况。

```rust
register_bitfields! [
    u64,
    pub TYPER [
        PLPIS OFFSET(0) NUMBITS(1) [],
        VLPIS OFFSET(1) NUMBITS(1) [],
        Dirty OFFSET(2) NUMBITS(1) [],
        Last OFFSET(4) NUMBITS(1) [],
        DirectLPI OFFSET(3) NUMBITS(1) [],
        CommonLPIAff OFFSET(24) NUMBITS(2) [],
        ProcessorNumber OFFSET(8) NUMBITS(16) [],
        Affinity OFFSET(32) NUMBITS(32) [],
    ],
];
```

代码中提供了相应的辅助方法来检查这些功能的支持情况：

```rust
pub fn supports_physical_lpi(&self) -> bool {
    self.TYPER.is_set(TYPER::PLPIS)
}

pub fn supports_virtual_lpi(&self) -> bool {
    self.TYPER.is_set(TYPER::VLPIS)
}
```

**Section sources**
- [gicr.rs](file://gic-driver/src/version/v3/gicr.rs#L400-L450)

## WAKER寄存器与电源管理

`WAKER`寄存器用于重分发器的电源管理功能，控制处理器的睡眠状态和子系统睡眠状态。

### 电源管理功能

`WAKER`寄存器包含以下关键位：

- `ProcessorSleep`（位1）：处理器睡眠位，设置该位可使处理器进入睡眠状态
- `ChildrenAsleep`（位2）：子系统睡眠位，指示子系统是否处于睡眠状态

通过操作这些位，可以实现处理器的电源管理功能。当需要唤醒重分发器时，需要清除`ProcessorSleep`位，并等待`ChildrenAsleep`位变为0，表示子系统已完全唤醒。

```rust
pub fn wake(&self) -> Result<(), &'static str> {
    self.WAKER.write(WAKER::ProcessorSleep::CLEAR);

    while self.WAKER.is_set(WAKER::ChildrenAsleep) {
        spin_loop();
    }

    self.wait_for_rwp()
}
```

### RWP同步机制

在执行寄存器写入操作后，需要等待`RWP`（Register Write Pending）位变为0，以确保寄存器写入操作已完成。这是一种重要的同步机制，确保寄存器操作的原子性和完整性。

```rust
pub fn wait_for_rwp(&self) -> Result<(), &'static str> {
    const MAX_RETRIES: u32 = 1000;
    let mut retries = 0;

    while self.CTLR.is_set(RCtrl::RWP) {
        if retries > MAX_RETRIES {
            return Err("Timeout waiting for register write to complete");
        }
        core::hint::spin_loop();
        retries += 1;
    }
    Ok(())
}
```

该机制通过轮询`RWP`位的状态，最多重试1000次，如果超时则返回错误。这种实现确保了寄存器操作的可靠性和稳定性。

**Section sources**
- [gicr.rs](file://gic-driver/src/version/v3/gicr.rs#L200-L250)