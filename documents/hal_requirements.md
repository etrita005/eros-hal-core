# HAL Layer 技术需求文档

## 1. 设计目标

HAL（Hardware Abstraction Layer）用于屏蔽具体硬件实现差异，为上层系统提供统一、稳定的设备访问与控制接口。

设计目标：

-   **硬件解耦**：隔离具体硬件驱动、总线协议、设备差异。
-   **模块化架构**：通过组件化 HAL Component 实现设备能力组合。
-   **控制与数据分离**：控制面与数据面解耦，提升实时性与系统稳定性。
-   **能力导向设计**：HAL 暴露设备能力（Capability），而不是业务逻辑。
-   **安全与异常管理**：统一处理急停、低电、设备异常等安全事件。
-   **可扩展性**：支持多种 Fieldbus 与设备类型扩展。

------------------------------------------------------------------------

# 2. HAL Layer 总体架构

HAL Layer 由以下核心模块组成：

-   HAL Component Manager
-   HAL Components（High-level / Low-level）
-   Fieldbus Gateway
-   System Safety Manager

系统结构逻辑如下：

                    +-----------------------+
                    |   Upper Layer System  |
                    | (Navigation / Logic)  |
                    +----------+------------+
                               |
                       Control Plane (gRPC/DBus)
                               |
                     +---------v---------+
                     | HAL Component     |
                     | Manager           |
                     +---------+---------+
                               |
            +------------------+------------------+
            |                                     |
    +-------v-------+                     +-------v-------+
    | High Level    |                     | High Level    |
    | HAL Component |                     | HAL Component |
    +-------+-------+                     +-------+-------+
            |                                     |
            | In-process call                     |
            |                                     |
    +-------v-------------------------------------v-------+
    |               Low Level HAL Components               |
    +-----------------------+------------------------------+
                            |
                    +-------v-------+
                    | Fieldbus      |
                    | Gateway       |
                    +-------+-------+
                            |
                        Hardware

------------------------------------------------------------------------

# 3. HAL Component Manager

HAL Component Manager 负责 HAL 组件生命周期管理与系统协调。

## 核心职责

### 1. HAL组件生命周期管理

-   根据设备树(Device Tree)创建 HAL Component
-   支持组件运行在：
    -   当前进程
    -   独立进程
-   维护 HAL Component 注册表
-   提供组件状态管理

组件状态示例：

-   Unconfigured
-   Initializing
-   Inactive
-   Activating
-   Active
-   Deactivating
-   RecoverableError
-   FatalError
-   EmergencyStop
-   Finalized

### 2. HAL组件组合

High Level HAL Component 通过组合方式调用多个 Low Level HAL Component。

组合比例：

    High Level HAL : Low Level HAL = 1 : N

例如 Lift HAL 可能组合：

-   Motor HAL
-   Encoder HAL
-   LimitSwitch HAL

---

## Low Level HAL Component（低层HAL组件）

### 核心职责
Low Level HAL Component **直接与硬件交互**，是硬件驱动的封装层。

### 设计原则
- **硬件相关**：针对具体硬件设备设计
- **接口标准化**：提供统一的硬件访问接口
- **不包含业务逻辑**：只负责底层硬件操作

### 主要功能
1. **设备驱动封装**：封装底层硬件驱动程序
2. **Fieldbus通讯**：处理现场总线（CAN、EtherCAT、RS485等）通信
3. **状态读取**：读取硬件设备的状态和传感器数据
4. **基础控制**：执行基础的硬件控制指令

### 示例组件
- Motor HAL（电机）
- Encoder HAL（编码器）
- GPIO HAL（通用输入输出）
- Camera HAL（相机）

---

## High Level HAL Component（高层HAL组件）

### 核心职责
High Level HAL Component **负责设备能力组合与控制逻辑**，通过组合多个Low Level HAL Component来实现更复杂的设备功能。

### 设计原则
- **不包含业务行为**：只负责设备能力组合，不涉及业务逻辑
- **能力导向**：暴露设备能力，而非业务语义

### 主要功能
1. **组件组合**：通过进程内调用组合多个Low Level HAL Component
2. **协调控制**：协调多个底层组件的工作时序
3. **内部控制器**：实现PID控制、限幅、安全保护等控制逻辑
4. **状态聚合**：聚合多个底层组件的状态，提供统一的设备状态

### 示例组合
例如**Lift HAL**可能组合：
- Motor HAL（电机驱动）
- Encoder HAL（位置检测）
- LimitSwitch HAL（限位开关）

### 能力级目标接口示例
High Level HAL提供的是**能力级目标**，而非业务级目标：

```cpp
// 正确：能力级目标
LiftHAL.set_target_height(1.0m)      // 设置目标高度
MotorHAL.set_target_velocity(2.0rad/s)  // 设置目标速度
GimbalHAL.set_target_angle(30deg)    // 设置目标角度

// 错误：业务级目标（应在上层实现）
LiftHAL.go_to_position(3)                // 去3米高度
Robot.start_delivery()             // 开始配送
```

---

## 关键区别总结

| 维度 | High Level HAL Component | Low Level HAL Component |
|------|------------------------|-----------------------|
| **交互对象** | 上层系统 + 低层HAL | 硬件 + Fieldbus |
| **核心职责** | 设备能力组合与控制逻辑 | 硬件驱动封装与基础控制 |
| **业务逻辑** | 不包含（能力级） | 不包含 |
| **组合关系** | 1:N（一个高层组合多个低层） | 被组合者 |
| **示例** | LiftHAL、ChassisHAL | Motor HAL、Encoder HAL、GPIO HAL |

High Level HAL 只负责 **设备能力组合与控制逻辑**，而不是业务行为。

------------------------------------------------------------------------

# 3.1 HAL Component 状态机设计

HAL Component 采用分层状态机（Hierarchical State Machine, HSM）设计，分为三大顶层状态域。

## 3.1.1 状态层次结构

```
┌─────────────────────────────────────────────────────────────────┐
│                        Top Level State                            │
└─────────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│  Operational  │   │     Error     │   │   Finalized   │
│   (正常运行)   │   │   (异常状态)   │   │   (终止状态)   │
└───────┬───────┘   └───────┬───────┘   └───────────────┘
        │                     │
        │                     │
        ▼                     ▼
   子状态：             子状态：
┌───────────────┐   ┌───────────────────┐
│ Unconfigured  │   │  RecoverableError │
└───────┬───────┘   │   (可自动恢复)     │
        │           └─────────┬─────────┘
        ▼                     │
┌───────────────┐             ▼
│ Initializing  │   ┌───────────────────┐
│  (中间状态)    │   │    FatalError     │
└───────┬───────┘   │   (需手动修复)     │
        │           └─────────┬─────────┘
        ▼                     │
┌───────────────┐             ▼
│   Inactive    │   ┌───────────────────┐
└───────┬───────┘   │  EmergencyStop    │
        │           │   (急停状态)       │
        │           └───────────────────┘
        │
        ├──────────────────────────────────────────┐
        │                                          │
        ▼                                          ▼
┌───────────────┐                          ┌─────────────────┐
│  Activating   │                          │   OTA 子状态机   │
│  (中间状态)    │                          └────────┬────────┘
└───────┬───────┘                                   │
        │                                            ▼
        ▼                                    ┌─────────────────┐
┌───────────────┐                            │  OTAPreparing   │
│    Active     │                            │  (OTA准备中)     │
└───────┬───────┘                            └────────┬────────┘
        │                                             │
        ▼                                             ▼
┌───────────────┐                            ┌─────────────────┐
│ Deactivating  │                            │ OTADownloading  │
│  (中间状态)    │                            │  (固件下载中)     │
└───────┬───────┘                            └────────┬────────┘
        │                                             │
        ▼                                             ▼
┌───────────────┐                            ┌─────────────────┐
│   Inactive    │                            │  OTAValidating  │
└───────────────┘                            │  (固件校验中)     │
                                             └────────┬────────┘
                                                      │
                                                      ▼
                                             ┌─────────────────┐
                                             │  OTAUpdating    │
                                             │  (固件升级中)     │
                                             └────────┬────────┘
                                                      │
                                                      ▼
                                             ┌─────────────────┐
                                             │  OTARestarting  │
                                             │  (重启应用中)     │
                                             └────────┬────────┘
                                                      │
                                ┌─────────────────────┼─────────────────────┐
                                │                     │                     │
                                ▼                     ▼                     ▼
                       ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
                       │   OTASuccess    │  │   OTAFailed     │  │   (回退路径)     │
                       │   (OTA成功)      │  │   (OTA失败)      │  └─────────────────┘
                       └────────┬────────┘  └────────┬────────┘
                                │                     │
                                ▼                     ▼
                       ┌─────────────────┐  ┌─────────────────┐
                       │    Inactive     │  │    Inactive     │
                       └─────────────────┘  └─────────────────┘
```

## 3.1.2 状态定义

### 顶层状态

| 状态 | 说明 |
|------|------|
| **Operational** | 正常运行域，包含所有正常工作状态 |
| **Error** | 异常状态域，包含所有错误状态 |
| **Finalized** | 终止状态，组件已关闭，不可再使用 |

### Operational 子状态

| 状态 | 类型 | 说明 | 硬件状态 | 场景示例 |
|------|------|------|---------|---------|
| **Unconfigured** | 原子 | 组件已创建，但未加载配置，未初始化硬件 | 未初始化 | 系统刚启动，还没读取配置文件 |
| **Initializing** | 中间 | 配置加载与硬件初始化进行中 | 初始化中 | 正在打开CAN总线、读取固件版本 |
| **Inactive** | 原子 | 已配置，硬件已初始化，但未使能输出 | 初始化完成，无动力 | 安全调试、参数配置、硬件诊断、OTA升级 |
| **Activating** | 中间 | 硬件激活进行中 | 激活中 | 正在使能电机、启动PID控制器 |
| **Active** | 原子 | 正常运行状态 | 完全使能 | 接收控制指令、发布传感器数据 |
| **Deactivating** | 中间 | 硬件停用进行中 | 停用中 | 正在停止电机、关闭传感器 |

### OTA 子状态（Operational 域的子状态机）

OTA 子状态机只能从 **Inactive** 状态进入，用于管理固件空中升级流程。

| 状态 | 类型 | 说明 | 硬件状态 | 场景示例 |
|------|------|------|---------|---------|
| **OTAPreparing** | 中间 | OTA 准备阶段：进入 OTA 模式、备份配置、准备升级环境 | 初始化完成，无动力 | 关闭无关服务、挂载升级分区 |
| **OTADownloading** | 中间 | 固件文件下载进行中 | 初始化完成，无动力 | 通过总线传输下载固件 |
| **OTAValidating** | 中间 | 固件校验：CRC 校验、签名验证、版本兼容性检查 | 初始化完成，无动力 | 验证固件完整性与合法性 |
| **OTAUpdating** | 中间 | 固件刷写/升级进行中 | 初始化完成，无动力 | 写入 Flash、更新 Bootloader 配置 |
| **OTARestarting** | 中间 | 设备重启并验证新固件 | 重启中 | 重启设备、新固件自检 |
| **OTASuccess** | 原子 | OTA 成功完成 | 初始化完成，无动力 | 新固件运行正常、升级完成 |
| **OTAFailed** | 原子 | OTA 失败，可尝试恢复或回退 | 初始化完成，无动力 | 下载失败、校验失败、刷写失败 |

### Error 子状态

| 状态 | 说明 | 触发源 | 恢复方式 | 场景示例 |
|------|------|--------|---------|---------|
| **RecoverableError** | 可自动恢复的异常 | 内部检测 | 自动恢复 | 通信暂时中断、数据CRC错误 |
| **FatalError** | 致命错误，需手动干预 | 内部检测 | 手动重置 + 修复 | 硬件故障、配置错误、总线完全断开 |
| **EmergencyStop** | 急停状态，最高优先级 | 外部触发 | 手动重置 | 按下急停按钮、上层系统急停指令 |

### EmergencyStop 与 FatalError 的区别

| 维度 | EmergencyStop（急停） | FatalError（致命错误） |
|------|---------------------|---------------------|
| **触发源** | 外部（急停按钮、上层系统、安全控制器） | 内部（硬件故障、严重通信错误） |
| **安全含义** | 主动安全措施 | 被动错误检测 |
| **恢复流程** | 确认安全 → 重置 → 重新配置 | 可能需要维修、更换硬件、重新校准 |
| **状态持久性** | 保持直到显式重置 | 保持直到问题修复 |

## 3.1.3 状态迁移表

| 当前状态 | 事件 | 中间状态 | 目标状态 |
|---------|------|---------|---------|
| Unconfigured | 配置加载 | Initializing | Inactive / Error |
| Inactive | 激活 | Activating | Active / Error |
| Active | 停用 | Deactivating | Inactive |
| Inactive | 清理 | - | Unconfigured |
| Any | 关闭 | - | Finalized |
| RecoverableError | 自动恢复 | - | Active |
| FatalError | 手动重置 | - | Unconfigured |
| EmergencyStop | 手动重置 | - | Unconfigured |
| Any | 急停触发 | - | EmergencyStop |
| Any | 异常发生 | - | (相应Error子状态) |
| Inactive | 启动 OTA | OTAPreparing | (OTA 子状态) |
| OTAPreparing | 准备完成 | OTADownloading | - |
| OTADownloading | 下载完成 | OTAValidating | - |
| OTAValidating | 校验通过 | OTAUpdating | - |
| OTAUpdating | 刷写完成 | OTARestarting | - |
| OTARestarting | 重启验证成功 | - | OTASuccess |
| OTASuccess | 确认完成 | - | Inactive |
| (任何 OTA 中间状态) | OTA 失败 | - | OTAFailed |
| OTAFailed | 重试 OTA | OTAPreparing | (OTA 子状态) |
| OTAFailed | 放弃 OTA | - | Inactive |

### OTA 状态迁移说明

1. **进入 OTA**：只能从 `Inactive` 状态启动 OTA，确保设备处于安全状态
2. **OTA 流程**：OTAPreparing → OTADownloading → OTAValidating → OTAUpdating → OTARestarting
3. **成功路径**：OTARestarting → OTASuccess → Inactive
4. **失败处理**：任何 OTA 中间状态失败都进入 OTAFailed，可选择重试或放弃
5. **安全限制**：OTA 过程中拒绝激活请求，禁止进入 Active 状态

## 3.1.4 中间状态的作用

### Initializing（配置中）
- 区分"开始配置"和"配置完成"
- 支持取消配置操作
- 可显示配置进度
- 处理异步初始化场景

### Activating（激活中）
- 处理硬件使能的固有延迟
- 等待设备就绪
- 实现平滑的状态过渡

### Deactivating（停用中）
- 实现安全停止（先减速再断电）
- 保存运行状态
- 等待设备完全停止

------------------------------------------------------------------------

# 4. 控制面（Control Plane）

控制面用于设备管理、配置及离散控制。

推荐通信方式：

-   **gRPC**
-   **DBus**（适用于单机系统）

控制面特点：

-   低频
-   请求/响应模式
-   用于设备管理和离散动作

------------------------------------------------------------------------

## 4.1 离散控制（Discrete Control）

离散控制是瞬时动作控制。

特点：

-   操作立即执行
-   不涉及连续控制流

示例：

-   打开 LED
-   关闭电磁阀
-   Reset 编码器
-   使能电机

接口示例：

    LED.on()
    Solenoid.open()
    Encoder.reset()
    Motor.enable()

执行流程：

    Upper System
          |
    gRPC/DBus call
          |
    HAL Component Manager
          |
    Target HAL Component
          |
    Low Level HAL

------------------------------------------------------------------------

# 5. 数据面（Data Plane）

数据面用于实时数据流与控制数据。

推荐通信方式：

-   **FastDDS**
-   **Eclipse eCAL**

特点：

-   高频数据
-   发布/订阅模式
-   低延迟

数据面包含两类数据。

------------------------------------------------------------------------

## 5.1 传感器数据发布

HAL 向上层发布设备状态或传感器数据。

示例：

-   Encoder position
-   Motor current
-   IMU
-   Battery status

数据格式使用 **IDL** 定义统一结构。

示例：

    EncoderState
    MotorState
    BatteryState
    LiftState

------------------------------------------------------------------------

## 5.2 连续控制（Continuous Control）

连续控制用于实时控制设备。

特点：

-   高频控制
-   上层持续发送控制指令

示例：

-   电机速度控制
-   机器人底盘控制
-   舵机角度控制

控制流程：

    Upper Controller
            |
    FastDDS Topic
            |
    HAL Component
            |
    Internal Controller
            |
    Low Level HAL

示例控制数据：

    velocity_command
    torque_command
    position_command

HAL 内部控制器负责：

-   PID 控制
-   限幅
-   安全保护

------------------------------------------------------------------------

# 6. Target Control（目标控制 / 目标设定）

**Target Control 并不是业务级目标，而是设备能力级目标设定。**

HAL 不应该实现业务行为，例如：

错误示例（不推荐）：

    Lift.go_to_position(3)
    Robot.start_delivery()
    Dock.auto_dock()

这些属于 **上层业务逻辑**。

HAL 应提供 **能力级目标接口**：

示例：

    Lift.set_target_height(1.0m)
    Motor.set_target_velocity(2.0rad/s)
    Gimbal.set_target_angle(30deg)

特点：

-   表示设备物理状态目标
-   不包含业务语义
-   可以由 HAL 内部控制器实现

调用流程：

    Upper Controller
            |
    Control Plane / Data Plane
            |
    HAL Component
            |
    Internal Controller
            |
    Low Level HAL

例如 Lift：

    set_target_height(1.0m)

HAL 内部执行：

-   位置控制
-   限位保护
-   运动停止判断

但 **不包含业务语义（如楼层、任务等）**。

------------------------------------------------------------------------

# 7. Low Level HAL Component

Low Level HAL Component 直接与硬件交互。

设计原则：

-   硬件相关
-   接口标准化
-   不包含业务逻辑

职责：

-   设备驱动封装
-   Fieldbus 通讯
-   状态读取
-   基础控制

示例组件：

-   Motor HAL
-   Encoder HAL
-   GPIO HAL
-   Camera HAL

------------------------------------------------------------------------

# 8. Fieldbus Gateway

Fieldbus Gateway 提供统一总线访问接口。

支持总线示例：

-   CAN
-   UAVCAN
-   EtherCAT
-   RS485

核心功能包括：

### 1. 设备管理

-   节点发现
-   节点注册
-   心跳检测
-   在线状态管理

### 2. 数据读写

统一接口：

    read(node_id)
    write(node_id, data)

### 3. 总线调度

-   带宽管理
-   消息优先级
-   仲裁策略

### 4. 时钟同步

用于多设备协同：

-   时间同步
-   同步监控

### 5. 异常监控

监控：

-   总线错误
-   节点失联
-   数据异常

支持自动恢复机制。

------------------------------------------------------------------------

# 9. System Safety Manager

System Safety Manager 负责系统级安全管理。

建议与 **HAL Component Manager
运行在同一进程**，以减少安全事件响应延迟。

职责包括：

-   急停处理
-   低电保护
-   故障联动
-   安全状态广播

------------------------------------------------------------------------

## 9.1 急停处理

急停来源：

-   物理急停按钮
-   上层系统急停指令
-   安全控制器

处理流程：

    Emergency Stop
           |
    System Safety Manager
           |
    Broadcast Stop Event
           |
    All HAL Components → EmergencyStop 状态

HAL Component 必须支持急停处理：

-   立即停止运动
-   关闭动力输出
-   进入 EmergencyStop 状态（Error 域的子状态）

### 从急停恢复

从 EmergencyStop 状态恢复的流程：

1.  用户确认现场安全
2.  执行手动重置
3.  组件回到 Unconfigured 状态
4.  重新执行配置 → 激活流程

EmergencyStop 状态与其他 Error 子状态的区别详见 3.1.2 章节。

------------------------------------------------------------------------

## 9.2 低电保护

Battery HAL 上报电量状态。

策略示例：

  电量    系统策略
  ------- ----------------
  \<20%   限制功率
  \<10%   禁止高功率设备
  \<5%    系统保护模式

Safety Manager 负责广播系统状态。

------------------------------------------------------------------------

# 10. HAL 异常上报

HAL Component 必须支持异常上报。

异常类型包括：

-   Hardware Fault
-   Communication Fault
-   Over Current
-   Over Temperature

异常可以通过：

-   Control Plane
-   Data Plane

进行上报。

------------------------------------------------------------------------

# 11. 架构设计原则

HAL Layer 遵循以下原则：

### 1. 能力导向（Capability Oriented）

HAL 暴露设备能力，例如：

-   速度控制
-   位置控制
-   IO 控制

而不是业务行为。

### 2. 控制与数据分离

Control Plane：

-   设备管理
-   离散控制

Data Plane：

-   实时数据
-   连续控制

### 3. 组件化

设备能力通过 HAL Component 封装。

### 4. 硬件解耦

Low Level HAL 隔离硬件差异。

### 5. 安全优先

Safety Manager 统一管理系统安全。

------------------------------------------------------------------------

# 12. 总结

HAL Layer 提供统一硬件抽象能力，并通过：

-   HAL Component Manager
-   High / Low Level HAL Components
-   Fieldbus Gateway
-   System Safety Manager

构建完整硬件抽象体系。

该架构重点强调：

-   **能力导向 HAL 设计**
-   **控制与数据分离**
-   **系统级安全管理**
-   **良好的扩展能力**
