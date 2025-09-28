
# CPU接口控制

<cite>
**本文档中引用的文件**  
- [icc.rs](file://gic-driver/src/sys_reg/icc.rs)
- [gicr.rs](file://gic-driver/src/version/v3/gicr.rs)
- [gicd.rs](file://gic-driver/src/version/v3/gicd.rs)
- [mod.rs](file://gic-driver/src/version/v3/mod.rs)
</cite>

## 目录
1. [引言](#引言)
2. [CPU接口结构与系统寄存器ICC](#cpu接口结构与系统寄存器icc)
3. [中断确认与结束操作](#中断确认与结束操作)
4. [CPU接口初始化流程](#cpu接口初始化流程)
5. [与重分发器的协同工作](#与重分发器的协同工作)
6. [SGI与PPI生命周期管理](#sgi与ppi生命周期管理)
7. [中断优先级掩码与抢占阈值配置](#中断优先级掩码与抢占阈值配置)
8. [代码示例：中断处理中的ack与eoi调用](#代码示例中断处理中的ack与eoi调用)

## 引言

本文档全面解析GICv3架构下CPU接口的控制机制，重点阐述CpuInterface结构体如何通过系统寄存器ICC与CPU核心进行交互，处理中断的确认（ack）和结束（EOI）操作。文档详细描述了CPU接口的初始化流程、与重分发器（Redistributor）的协同工作机制，以及对SGI（软件生成中断）和PPI（私有外设中断）的全生命周期管理。同时，结合代码逻辑说明了如何通过CPU接口配置中断优先级掩码和抢占阈值，为理解GICv3中断控制器的CPU侧操作提供完整的理论与实践指导。

## CPU接口结构与系统寄存器ICC

CpuInterface结构体是GICv3中CPU接口的核心抽象，它通过访问ARM定义的ICC（Interrupt Controller CPU interface）系统寄存器来实现对当前CPU中断处理环境的控制。该结构体主要包含指向重分发器（RedistributorV3）的指针和当前安全状态（SecurityState），其主要功能是桥接软件逻辑与底层硬件寄存器。

ICC系统寄存器是一组专用于CPU接口的协处理器寄存器，它们不通过内存映射访问，而是通过特殊的系统指令（如MRS/MSR）进行读写。CpuInterface通过操作这些寄存器来完成中断的接收、处理和完成。关键的ICC寄存器包括：
- **ICC_SRE_ELx**: 系统寄存器使能寄存器，用于启用对其他ICC寄存器的访问。
- **ICC_IGRPEN0/1_ELx**: 中断组使能寄存器，用于启用Group 0和Group 1的中断。
- **ICC_CTLR_ELx**: 控制寄存器，用于配置二进制点、EOI模式等行为。
- **ICC_PMR_EL1**: 优先级掩码寄存器，用于屏蔽低于特定优先级的中断。
- **ICC_IAR0/1_EL1**: 中断确认寄存器，用于读取当前最高优先级的待处理中断。
- **ICC_EOIR0/1_EL1**: 中断结束寄存器，用于通知GIC中断处理已完成。

**Section sources**
- [mod.rs](file://gic-driver/src/version/v3/mod.rs#L800-L1150)
- [icc.rs](file://gic-driver/src/sys_reg/icc.rs#L0-L252)

## 中断确认与结束操作

中断的确认（Acknowledge）和结束（End of Interrupt, EOI）是CPU接口处理中断的核心流程。当一个中断被路由到CPU并触发异常时，CPU通过读取ICC_IAR0_EL1或ICC_IAR1_EL1寄存器来“确认”该中断。此操作会从GIC的待处理中断队列中移除该中断，并将其状态标记为“活跃（Active）”，同时返回中断的ID（INTID）。

在中断服务程序（ISR）执行完毕后，CPU必须通过向ICC_EOIR0_EL1或ICC_EOIR1_EL1寄存器写入相同的INTID来“结束”该中断。这会通知GIC中断处理已完成，GIC可以将该中断从“活跃”状态清除，并根据需要重新激活更高优先级的待处理中断。

CpuInterface结构体提供了`ack0`、`ack1`、`eoi0`和`eoi1`等方法来封装这些底层寄存器操作。`ack0`和`ack1`分别用于Group 0和Group 1的中断，它们读取IAR寄存器并返回一个IntId对象。`eoi0`和`eoi1`则将IntId对象的值写入相应的EOIR寄存器。

**Section sources**
- [mod.rs](file://gic-driver/src/version/v3/mod.rs#L800-L1150)
- [icc.rs](file://gic-driver/src/sys_reg/icc.rs#L0-L252)

## CPU接口初始化流程

CPU接口的初始化是通过调用`init_current_cpu`方法完成的，该方法遵循GICv3架构规范，确保当前CPU的中断处理环境被正确配置。初始化流程包含以下几个关键步骤：

1.  **唤醒重分发器**：首先调用`self.rd().lpi.wake()`方法，通过清除WAKER寄存器的ProcessorSleep位来唤醒与当前CPU关联的重分发器，并等待ChildrenAsleep位被清除，确保重分发器已准备好处理中断。

2.  **初始化SGI/PPI寄存器**：调用`self.rd().sgi.init_sgi_ppi(self.security_state)`方法，将重分发器中SGI和PPI相关的寄存器（如使能、挂起、优先级寄存器）初始化为已知状态。这包括清除所有挂起和活跃的中断，禁用所有中断，并根据当前安全状态设置中断组（Group 0或Group 1）。

3.  **使能系统寄存器访问**：根据当前的异常级别（EL），写入ICC_SRE_EL1或ICC_SRE_EL2寄存器，设置SRE、DFB和DIB位，以启用对ICC系统寄存器的访问。

4.  **配置优先级掩码**：通过写入ICC_PMR_EL1寄存器，将优先级掩码设置为0xFF，允许所有优先级的中断通过，确保CPU可以接收任何中断。

5.  **使能中断组**：根据安全状态，配置ICC_IGRPEN0_EL1和ICC_IGRPEN1_EL1寄存器来启用相应的中断组。例如，在单安全状态下，会同时启用Group 0和Group 1。

6.  **配置EOI模式**：可选地配置ICC_CTLR_EL1寄存器的EOIMODE位，以确定EOI操作是单步还是两步（结合ICC_DIR_EL1使用）。

**Section sources**
- [mod.rs](file://gic-driver/src/version/v3/mod.rs#L800-L1150)

## 与重分发器的协同工作

CPU接口与重分发器（GICR）紧密协作，共同管理CPU核心的私有中断。重分发器是GICv3架构中的一个关键组件，它位于每个CPU核心附近，负责处理SGI和PPI。CpuInterface结构体通过其`rd`字段直接持有指向当前CPU对应重分发器的指针。

这种协同工作体现在：
- **初始化**：CPU接口的初始化流程首先唤醒并初始化其关联的重分发器。
- **中断控制**：当需要使能、禁用、设置优先级或查询SGI/PPI的状态时，CpuInterface会调用重分发器中SGI结构体的相应方法（如`set_enable_interrupt`、`set_priority`等）。
- **中断路由**：虽然SGI和PPI是私有的，但重分发器的配置决定了它们的行为。CPU接口通过操作重分发器的寄存器来管理这些中断。

重分发器的存在使得GICv3能够支持更复杂的拓扑结构和更高效的中断处理，而CPU接口则是软件访问这些功能的统一入口。

**Section sources**
- [mod.rs](file://gic-driver/src/version/v3/mod.rs#L800-L1150)
- [gicr.rs](file://gic-driver/src/version/v3/gicr.rs#L0-L552)

## SGI与PPI生命周期管理

CpuInterface提供了完整的API来管理SGI和PPI的生命周期。这些中断的生命周期包括挂起（Pending）、活跃（Active）和非活跃（Inactive）等状态。

- **挂起（Pending）**：通过`set_pending`方法可以将一个SGI或PPI手动设置为挂起状态，这会触发一个中断。`is_pending`方法可以查询中断的挂起状态。
- **活跃（Active）**：当一个挂起的中断被CPU确认（ack）后，它会自动进入活跃状态。`set_active`和`is_active`方法允许软件直接查询和修改中断的活跃状态，这在虚拟化等场景中非常有用。
- **使能/禁用**：`set_irq_enable`和`is_irq_enable`方法用于控制中断是否被使能。只有使能的中断才能被传递给CPU。
- **优先级与配置**：`set_priority`和`get_priority`用于设置和查询中断的优先级。`set_cfg`和`get_cfg`用于配置中断是边沿触发还是电平触发。

所有这些操作最终都通过CpuInterface委托给其关联的重分发器（RedistributorV3）中的SGI结构体来完成。

**Section sources**
- [mod.rs](file://gic-driver/src/version/v3/mod.rs#L800-L1150)
- [gicr.rs](file://gic-driver/src/version/v3/gicr.rs#L0-L552)

## 中断优先级掩码与抢占阈值配置

中断优先级掩码（Priority Mask）是CPU接口中一个关键的配置，它决定了CPU可以接收的中断的最低优先级。优先级数值越小，优先级越高。通过配置ICC_PMR_EL1寄存器，