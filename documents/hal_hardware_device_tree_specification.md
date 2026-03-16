# HAL Layer 硬件设备树规范

## 1. 概述

本文档定义了机器人操作系统硬件抽象层（HAL）的硬件设备树规范，用于标准化描述机器人系统的硬件拓扑、通信接口、设备参数，实现硬件信息与业务代码解耦，支持多硬件兼容、多总线设备、模块化生命周期管理。

### 1.1 设计目标

- 统一机器人硬件的描述标准，实现一套 HAL 代码适配多代硬件
- 支持单设备多总线接入的复杂场景，打破树状拓扑的限制
- 多语言友好，适配 C++/Python/Rust 等机器人常用开发语言
- 轻量化、易读易维护，适合嵌入式、工控、机器人场景

### 1.2 核心设计原则

- **Describe hardware, not policy or software semantics**：描述硬件是什么，而不是软件能做什么。设备树只陈述硬件的客观属性（厂商、型号、地址、电气参数），不定义软件策略（如控制算法、业务逻辑、功能特性）
- **扁平拓扑**：总线配置与设备定义完全解耦，独立描述
- **多总线兼容**：原生支持单设备同时接入多个通信总线
- **逻辑分组**：通过 subdevices 实现设备的逻辑从属与生命周期联动
- **强标识规则**：通过 vid/pid/hardware_version/instance_id 实现设备的唯一精准定位
- **稳定标识**：bus_id 提供总线的稳定标识，不受 bus_name 变更影响
- **分类清晰**：device_type 使用 `分类。类型` 格式（如 `actuator.dc_motor`），兼顾层次与简洁

### 1.3 文件基础规范

- **推荐文件扩展名**：`.hal.yaml`（配置文件）、`.md`（规范文档）
- **编码格式**：强制 UTF-8
- **缩进规则**：YAML 配置使用 2 空格缩进，禁止使用 Tab
- **字段命名**：统一使用蛇形命名法（snake_case）
- **版本规则**：使用主版本。次版本格式，重大变更升级主版本

## 2. 顶级字段定义

顶级字段为配置文件的根节点，定义机器人整机的基础信息、总线列表、设备列表。

| 字段名 | 数据类型 | 强制 / 可选 | 说明 | 示例值 |
|--------|----------|-------------|------|--------|
| schema_version | string | ✅ 强制 | 本规范的版本号，固定使用本文档版本 | `"2.2"` |
| robot_model | string | ✅ 强制 | 机器人产品型号，需全局唯一 | `"my_diff_drive_robot_x1"` |
| robot_vid | string | ✅ 强制 | 机器人整机厂商 ID（VID），建议与 USB/PCIe VID 规则对齐 | `"1234"` |
| robot_pid | string | ✅ 强制 | 机器人整机产品 ID（PID），区分同厂商不同型号 | `"5678"` |
| robot_instance_id | int | ⚪ 可选 | 机器人整机实例 ID，用于多机协同场景区分同型号设备 | `0` |
| buses | list[object] | ✅ 强制 | 系统全局总线列表，定义所有通信总线的基础配置 | `-` |
| devices | list[object] | ✅ 强制 | 系统设备列表，定义所有板卡、外设、执行器、传感器的完整信息 | `-` |

## 3. 总线层规范

总线层仅负责定义通信总线的基础配置，不包含挂载的设备信息，实现总线与设备的完全解耦。

### 3.1 总线通用字段

所有总线类型通用的必填字段：

| 字段名 | 数据类型 | 强制 / 可选 | 说明 | 示例值 |
|--------|----------|-------------|------|--------|
| bus_name | string | ✅ 强制 | 总线系统名称，可能因系统配置变化，仅供显示使用 | `"can0"` / `"eth0"` |
| bus_id | string | ✅ 强制 | 总线唯一标识符，不随名称变化而改变，设备通过该 ID 关联总线 | `"can_chassis"` / `"eth_main"` |
| bus_type | enum | ✅ 强制 | 总线类型，限定为预定义的枚举值 | 可选值：`can` / `ethernet` / `i2c` / `spi` / `usb` / `uart` / `ethercat` |
| bus_config | object | ✅ 强制 | 总线专属配置，不同总线类型对应不同的配置字段，详见 3.2 节 | `-` |

### 3.2 各总线专属配置定义

针对不同总线的通信特性，定义专属的配置字段：

#### 3.2.1 CAN 总线

```yaml
bus_config:
  bitrate: 1000000        # 强制，总线波特率，单位 bps
  sample_point: 0.875     # 可选，采样点位置，默认 0.875
  fd_enable: false        # 可选，是否启用 CAN FD，默认 false
  fd_bitrate: 5000000     # 可选，CAN FD 数据段波特率，fd_enable=true 时必填
```

#### 3.2.2 Ethernet 总线

```yaml
bus_config:
  ip_address: "192.168.1.10"  # 强制，本机网口 IP 地址
  subnet_mask: "255.255.255.0" # 强制，子网掩码
  gateway: "192.168.1.1"      # 可选，网关地址
  mtu: 1500                   # 可选，MTU 值，默认 1500
```

#### 3.2.3 I2C 总线

```yaml
bus_config:
  clock_speed_hz: 400000  # 强制，总线时钟频率，单位 Hz，常用 100000/400000
```

#### 3.2.4 SPI 总线

```yaml
bus_config:
  clock_speed_hz: 10000000  # 强制，总线时钟频率，单位 Hz
  mode: 0                   # 强制，SPI 模式，可选值 0/1/2/3
  lsb_first: false          # 可选，是否低位在前，默认 false
```

#### 3.2.5 UART 串口总线

```yaml
bus_config:
  baudrate: 115200      # 强制，波特率
  data_bits: 8          # 可选，数据位，默认 8
  stop_bits: 1          # 可选，停止位，默认 1
  parity: "none"        # 可选，校验位，可选值 none/odd/even，默认 none
  flow_control: "none"  # 可选，流控，可选值 none/rts_cts，默认 none
```

#### 3.2.6 USB 总线

```yaml
bus_config:
  host_controller: "xchi"  # 可选，主机控制器标识，用于多 USB 控制器区分
```

## 4. 设备层规范

设备层是硬件描述的核心，用于定义板卡、传感器、执行器、控制器等所有硬件实体，支持逻辑子设备嵌套、多总线关联。

### 4.1 设备通用必填字段

所有设备（包括顶级设备和子设备）必须包含的核心字段：

| 字段名 | 数据类型 | 强制 / 可选 | 说明 | 示例值 |
|--------|----------|-------------|------|--------|
| name | string | ✅ 强制 | 设备唯一名称，全局不可重复 | `"motion_controller"` |
| device_type | string | ✅ 强制 | 设备类型，使用 `分类。类型` 格式 | `sensor.imu` / `sensor.lidar` / `actuator.dc_motor` / `actuator.servo` / `controller.motion_controller` / `hmi.tablet` / `board.sensor_hub` |
| role | string | ⚪ 可选 | 设备功能角色，用于上层业务区分同类型设备的用途 | `"chassis_imu"` / `"main_lidar"` |
| vid | string | ✅ 强制 | 当前设备的实际厂商 ID，与硬件出厂标识对齐 | `"1234"` / `"invensense"` |
| pid | string | ✅ 强制 | 当前设备的实际产品 ID，与硬件出厂标识对齐 | `"abcd"` / `"mpu6050"` |
| hardware_version | string | ✅ 强制 | 设备硬件版本号，与硬件出厂标识对齐 | `"rev2"` / `"rev2.1"` / `"v1.0"` |
| instance_id | int | ✅ 强制 | 设备实例 ID，同 vid+pid+hardware_version 的设备通过该字段区分，从 0 开始递增 | `0` / `1` |
| buses | list[object] | ✅ 强制 | 设备关联的通信总线列表，支持单设备同时接入多个总线 | `-` |
| params | object | ⚪ 可选 | 设备参数，如量程、采样率、减速比等设备固有参数 | `-` |
| lifecycle | object | ⚪ 可选 | 设备生命周期管理配置，用于 HAL 层的启动 / 停止 / 故障管理 | `-` |
| subdevices | list[object] | ⚪ 可选 | 逻辑从属子设备列表，子设备字段规则与顶级设备完全一致 | `-` |

### 4.2 多总线关联字段（buses）

用于定义设备接入的总线信息，支持单设备同时接入多个总线，每个总线对应独立的地址与专属参数。

| 字段名 | 数据类型 | 强制 / 可选 | 说明 | 示例值 |
|--------|----------|-------------|------|--------|
| bus_id | string | ✅ 强制 | 关联的总线 ID，必须与顶级 buses 中定义的 bus_id 完全一致 | `"can_chassis"` |
| device_address | object | ✅ 强制 | 设备在该总线上的专属地址，不同总线类型对应不同字段，详见下表 | `-` |
| params | object | ⚪ 可选 | 设备在该总线上的专属配置参数，如过滤器、协议类型等 | `-` |

#### 各总线 device_address 定义

| 总线类型 | 地址字段 | 说明 | 示例 |
|----------|----------|------|------|
| CAN | node_id | 设备在 CAN 总线上的节点 ID，11 位 / 29 位 | `node_id: 0x0B` |
| Ethernet | ip_address / mac_address | 设备的 IP 地址与 MAC 地址 | `ip_address: "192.168.1.11"` |
| I2C | address | 设备的 I2C 从机地址，7 位 / 10 位 | `address: 0x68` |
| SPI | chip_select | 设备的片选通道号 | `chip_select: 0` |
| UART | 无 | 设备独占串口，无需额外地址字段 | `-` |
| USB | vid / pid / serial | 设备的 USB VID/PID，可选序列号用于多设备区分 | `vid: "10c4"`, `pid: "ea60"` |

### 4.3 生命周期配置字段（lifecycle）

用于 HAL 层统一管理设备的启动、停止、故障处理，可选配置。

| 字段名 | 数据类型 | 强制 / 可选 | 说明 | 示例值 |
|--------|----------|-------------|------|--------|
| auto_start | bool | ⚪ 可选 | 是否随系统自动启动该设备，默认 false | `true` |
| restart_policy | enum | ⚪ 可选 | 设备故障后的重启策略，可选值：`never` / `on_failure` / `always`，默认 `never` | `on_failure` |

### 4.4 逻辑子设备字段（subdevices）

用于实现设备的逻辑分组与从属关系，不代表物理总线拓扑，核心作用：

- **生命周期联动**：父设备启动完成后子设备才能启动，父设备停止前先停止子设备
- **配置继承**：子设备可继承父设备的通用参数、总线配置
- **拓扑可视化**：清晰反映板卡与板载设备的物理从属关系

> **注意**：子设备的字段规则与顶级设备完全一致，必须包含 `name`/`device_type`/`vid`/`pid`/`hardware_version`/`instance_id`/`buses` 等必填字段。

## 5. 完整规范示例

```yaml
schema_version: "2.2"
robot_model: "my_diff_drive_robot_x1"
robot_vid: "1234"
robot_pid: "5678"
robot_instance_id: 0

buses:
  - bus_name: "can0"
    bus_id: "can_chassis"
    bus_type: "can"
    bus_config:
      bitrate: 1000000
      sample_point: 0.875

  - bus_name: "eth0"
    bus_id: "eth_main"
    bus_type: "ethernet"
    bus_config:
      ip_address: "192.168.1.10"
      subnet_mask: "255.255.255.0"
      mtu: 1500

  - bus_name: "i2c_1"
    bus_id: "i2c_sensor_hub"
    bus_type: "i2c"
    bus_config:
      clock_speed_hz: 400000

  - bus_name: "usb1"
    bus_id: "usb_main"
    bus_type: "usb"
    bus_config:
      host_controller: "xchi"

  - bus_name: "ttyUSB0"
    bus_id: "uart_debug"
    bus_type: "uart"
    bus_config:
      baudrate: 115200
      data_bits: 8
      parity: "none"

devices:
  - name: "motion_controller"
    device_type: "controller.motion_controller"
    role: "chassis_motion_control"
    vid: "1234"
    pid: "abcd"
    hardware_version: "rev2"
    instance_id: 0
    buses:
      - bus_id: "can_chassis"
        device_address:
          node_id: 0x0B
        params:
          can:
            tx_filters: ["0x100-0x1FF"]
            rx_filters: ["0x200-0x2FF"]
      - bus_id: "eth_main"
        device_address:
          ip_address: "192.168.1.11"
          mac_address: "00:11:22:33:44:55"
        params:
          ethernet:
            protocol: "udp"
            port: 5000
    params:
      loop_rate_hz: 1000
    lifecycle:
      auto_start: true
      restart_policy: on_failure
    subdevices:
      - name: "left_wheel_motor"
        device_type: "actuator.dc_motor"
        role: "left_wheel"
        vid: "1234"
        pid: "dc_motor_v1"
        hardware_version: "v1"
        instance_id: 0
        buses:
          - bus_id: "can_chassis"
            device_address:
              node_id: 0x01
        params:
          motor:
            reduction_ratio: 50
            max_rpm: 100
            encoder_ppr: 1000
      - name: "right_wheel_motor"
        device_type: "actuator.dc_motor"
        role: "right_wheel"
        vid: "1234"
        pid: "dc_motor_v1"
        hardware_version: "v1"
        instance_id: 1
        buses:
          - bus_id: "can_chassis"
            device_address:
              node_id: 0x02
        params:
          motor:
            reduction_ratio: 50
            max_rpm: 100
            encoder_ppr: 1000
      - name: "lift_actuator"
        device_type: "actuator.servo"
        role: "payload_lift"
        vid: "1234"
        pid: "linear_servo_v1"
        hardware_version: "v1"
        instance_id: 0
        buses:
          - bus_id: "can_chassis"
            device_address:
              node_id: 0x03
        params:
          servo:
            min_position_mm: 0
            max_position_mm: 100
            max_speed_mm_s: 50

  - name: "sensor_hub"
    device_type: "board.sensor_hub"
    role: "internal_sensor_collection"
    vid: "1234"
    pid: "cdef"
    hardware_version: "rev_c"
    instance_id: 0
    buses:
      - bus_id: "can_chassis"
        device_address:
          node_id: 0x0A
    lifecycle:
      auto_start: true
      restart_policy: on_failure
    subdevices:
      - name: "chassis_imu"
        device_type: "sensor.imu"
        role: "chassis_imu"
        vid: "invensense"
        pid: "mpu6050"
        hardware_version: "any"
        instance_id: 0
        buses:
          - bus_id: "i2c_sensor_hub"
            device_address:
              address: 0x68
        params:
          sensor:
            sample_rate_hz: 200
            accel_range_g: 2
            gyro_range_dps: 250
      - name: "front_sonar"
        device_type: "sensor.ultrasonic_sonar"
        role: "front_obstacle_sensor"
        vid: "hc-sr04"
        pid: "generic"
        hardware_version: "any"
        instance_id: 0
        buses:
          - bus_id: "i2c_sensor_hub"
            device_address:
              address: 0x20
        params:
          sensor:
            max_range_m: 3
            sample_rate_hz: 10

  - name: "tablet_hmi"
    device_type: "hmi.tablet"
    role: "user_interface"
    vid: "raspberrypi"
    pid: "pi4"
    hardware_version: "any"
    instance_id: 0
    buses:
      - bus_id: "eth_main"
        device_address:
          ip_address: "192.168.1.12"
          mac_address: "AA:BB:CC:DD:EE:FF"
    lifecycle:
      auto_start: true
      restart_policy: on_failure

  - name: "main_lidar"
    device_type: "sensor.lidar"
    role: "main_lidar"
    vid: "10c4"
    pid: "ea60"
    hardware_version: "any"
    instance_id: 0
    buses:
      - bus_id: "usb_main"
        device_address:
          serial: "0"
    params:
      sensor:
        sample_rate_hz: 10
        max_range_m: 20

  - name: "rgbd_camera"
    device_type: "sensor.rgbd_camera"
    role: "front_camera"
    vid: "8086"
    pid: "0b3a"
    hardware_version: "any"
    instance_id: 0
    buses:
      - bus_id: "usb_main"
        device_address:
          vid: "8086"
          pid: "0b3a"
    params:
      sensor:
        resolution: "640x480"
        fps: 30
```
