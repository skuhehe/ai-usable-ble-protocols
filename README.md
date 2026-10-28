# 格锐特调控制器与蚂蚁保护板 BLE 协议（AI 可直接使用）

> 用途：给其他 AI、固件或主机程序作为可执行实现依据。协议、字段与限制按设备分开；不得把控制器帧套用到 BMS。
> 整理日期：2026-10-29。来源是本仓库源码和已有证据，不表示本次连接了设备或做过真机验证。

## 0. 给实现者的指令

1. 先识别设备类别和品牌，再使用对应 GATT、请求帧与解析器。`gret` 是格锐特调控制器；`ant` 是蚂蚁 BMS。
2. BLE 通知可能被任意切片或合并；使用有界流式缓冲器，不假设一次通知就是一帧。
3. 保留原始帧、时间戳、数据来源和字段有效位。字段缺失、过期或未经设备证实，必须表示为 unknown/null，不得用 0 或演示值填充。
4. 格锐特帧校验规则尚未由现有资料确认。不要自行补造 checksum、尾字节或认证握手规则。
5. 蚂蚁协议按本仓库实现为只读集成。不可发送 MOS、均衡、蓝牙开关、参数写入或认证帧；本项目没有五级权限密码。
6. 报告证据等级：ANT 解析有在线真机数据依据；格锐特有来源代码和合成向量依据，硬件兼容与数值语义尚待真机确认。

## 1. 设备与协议速览

| 项目 | 格锐特调控制器 (`gret`) | 蚂蚁保护板 (`ant`) |
|---|---|---|
| 类别 | 电机控制器 | BMS 保护板 |
| GATT 服务 | `0000FFF0-0000-1000-8000-00805F9B34FB` | `0000FFE0-0000-1000-8000-00805F9B34FB` |
| Notify | `0000FFF1-0000-1000-8000-00805F9B34FB` | `0000FFE1-0000-1000-8000-00805F9B34FB` |
| Write | `0000FFF2-0000-1000-8000-00805F9B34FB` | `0000FFE1-0000-1000-8000-00805F9B34FB`（与通知同特征） |
| 主机请求 | 周期发送单字节 `96` | 每 1000 ms 发送实时读取帧 `7E A1 01 00 00 BE 18 55 AA 55` |
| 响应 | 固定 25 字节，起始 `EC 9F` | 功能类型 `0x11` 为实时数据；`0x12` 为设置回读 |
| 当前字段 | 电压、原始电流、源 RPM、MOS 温度、电机温度 | 总压、电流、SOC/SOH、串数与单体、温度、MOS、容量、状态/告警等 |
| 写操作 | 本规范仅提供读取，不做调参 | 禁止写入，仅读取 |

`FFF0`/`FFE0` 等是 16 位 UUID 的 Bluetooth 基础 UUID 展开形式。设备品牌识别不能只看 UUID，因为多个设备共用服务族。

## 2. 格锐特调控制器（`gret`）

### 2.1 收发

- 连接同一服务 `FFF0` 下的 Notify `FFF1` 和 Write `FFF2`。
- Notify 订阅成功后周期写入单字节 `96`（十六进制）。仓库候选调度间隔 100 ms；这是来源适配器参数，不是已实测设备刷新率。
- 响应解析器从字节流中查找 `EC 9F`，需要完整 25 字节后解码。当前解析器未要求校验和或固定帧尾。
- 若目标硬件未返回有效 `EC 9F` 帧，应保留原始数据并报告“不兼容/未知”，不要尝试其他控制器协议或写配置命令。

### 2.2 25 字节帧字段

偏移从 0 起，整数为大端（代码明确读取大端的字段）：

| 偏移 | 长度 | 解码 | 输出/限制 |
|---|---:|---|---|
| 0..1 | 2 | 固定起始 `EC 9F` | 不匹配时滑动丢弃一个字节再找帧头 |
| 3 | 1 | 无符号温度原值减 40 | MOSFET 温度，℃ |
| 4..5 | 2 | `u16_be / 100` | 电压，V；仅接受 1..250 V |
| 6..7 | 2 | `i16_be / 10` | 原始电流，A；电流方向未知，不能据此算有符号功率 |
| 8..9 | 2 | `u16_be / pole_pairs` | RPM；`pole_pairs = byte[14]`，若 0 则代码回退为 1。缺失/默认极对数时 RPM 不能称为可信机械车速 |
| 14 | 1 | 无符号极对数 | 0 时按 1 处理（源码兼容行为，不代表设备真实极对数为 1） |
| 15 | 1 | 无符号温度原值减 40 | 电机温度，℃ |
| 其余 | — | 未映射 | 保留原始字节，不推断含义 |

注意：源码把偏移 3、15 解为减 40 后温度，但没有温度传感器位置证据；电流符号也未知。输出字段名应保留 `raw_current`、`source_rpm`、`mosfet_temperature_c`、`motor_temperature_c`。

### 2.3 合成向量（只验证解码算术）

`EC 9F 00 58 1C 20 00 96 17 70 00 00 00 00 05 5A` 后补 9 个任意字节至 25 字节时，当前仓库解析器得到：72.00 V、15.0 A 原始电流、1200 RPM、48℃ MOSFET、50℃ 电机。此为合成测试向量，不是真机抓包；该帧格式不含已核实 checksum。

## 3. 蚂蚁保护板（ANT BMS）

### 3.1 GATT 与通用帧

- 广播识别提示：名称以 `ANT` 开头或包含“蚂蚁”；名称仅作提示，仍应确认服务与有效响应。
- 服务 `FFE0`；通知/写入 `FFE1`。
- 多字节整数字段均按小端，除非具体字段说明另有定义。
- 基础帧：`7E A1 <function> <address/value LE16> <N> <payload N bytes> <CRC-r> <CRC-t> <tail>`。
- 帧长 = `10 + N`。请求/普通实时响应通常尾字节为 `AA 55`；`0x12` 参数回读及 `0x43` 认证应答的尾部可能是 `FF <status>`。校验覆盖 `A1` 至载荷末尾；CRC 两字节顺序按实现为 `[r,t]`。
- CRC 实际对应标准 CRC16/MODBUS（多项式 `0xA001`、初值 `0xFFFF`、小端输出）；仓库用两张表实现。复用仓库实现优于自行重写。
- 主机每秒发送一次实时请求；当前资料描述请求分片为 20 字节、间隔约 18 ms（实时请求本身为 10 字节）。

### 3.2 请求/应答代码

| 功能 | 主机请求或应答结构 | 用途 |
|---|---|---|
| `0x01` | `7E A1 01 00 00 BE 18 55 AA 55` | 读实时数据；正常响应类型为 `0x11` |
| `0x11` | `7E A1 11 00 00 <N> <payload> <CRC> AA 55` | 实时数据 |
| `0x02` | `7E A1 02 <addr LE16> <N> <CRC> AA 55` | 读设置寄存器；通常读取 2 字节，容量地址 `00A2/00A6/00AA` 读 4 字节 |
| `0x12` | 响应 payload：地址 LE16、长度 N、N 字节值 LE | 设置寄存器读回；值乘对应 register scale 得工程量 |

本规范仅建议 `0x01` 实时读取与 `0x02` 设置只读查询。不要发 `0x23` 认证或 `0x51` 写入帧。

### 3.3 实时响应 `0x11` 字段解析

载荷从帧偏移 6 开始。下面顺序表采用仓库 `antbms_protocol.py::parse_realtime` 实现；所有字节偏移均相对于 payload 首字节：

| payload 偏移/顺序 | 类型 | 输出与换算 |
|---|---|---|
| 0 | `u8` | 当前权限/操作权限原值 `sys_op_auth`；不要据此绕过写权限 |
| 1 | `u8` | 系统状态码 `system_state` |
| 2 | `u8` | 温度点数量 M |
| 3 | `u8` | 电芯串数 N |
| 4..7 | `u32 LE` | 保护位图 1 |
| 8..11 | `u32 LE` | 保护位图 2 |
| 12..15 | `u32 LE` | 告警位图 1 |
| 16..19 | `u32 LE` | 告警位图 2 |
| 20..27 | 8 bytes | 保留 |
| 28..`28+2N-1` | N × `u16 LE` | 单体电压 V = mV / 1000；按载荷长度保护读取。现有源保留逻辑应复用并核查其高位状态位边界，勿自行更改位语义 |
| 随后 M × 2 bytes | M × `i16 LE` | 温度原始值，具体显示换算以对应设备/现有解析模型为准 |
| 接着 2 bytes | `i16 LE` | MOS 温度原始值 |
| 接着 2 bytes | `i16 LE` | 均衡温度原始值 |
| 接着 2 bytes | `u16 LE / 100` | 总电压 V |
| 接着 2 bytes | `i16 LE / 10` | 电流 A |
| 接着 2 bytes | `u16 LE` | SOC 百分比字段 |
| 接着 2 bytes | `u16 LE` | SOH 百分比字段 |
| 接着 1 byte | `u8` | 放电 MOS 状态码 |
| 接着 1 byte | `u8` | 充电 MOS 状态码 |
| 接着 2 bytes | `u16 LE` | 均衡状态码 |
| 接着 4 bytes | `u32 LE / 1e6` | 满容量 Ah |
| 接着 4 bytes | `u32 LE / 1e6` | 剩余容量 Ah |
| 接着 4 bytes | `u32 LE / 1e3` | 循环容量 Ah |
| 接着 4 bytes | `i32 LE` | 功率原始值；按解析器语义处理 |
| 接着 4 bytes | `u32 LE` | uptime 原始值 |
| 接着 4 bytes | `u32 LE` | 均衡位图 |
| 接着 2 bytes | `u16 LE / 1000` | 最大单体 V |
| 接着 2 bytes | `u16 LE` | 最大单体位置 |
| 接着 2 bytes | `u16 LE / 1000` | 最小单体 V |
| 接着 2 bytes | `u16 LE` | 最小单体位置 |
| 更长扩展帧尾部 | `u32 LE / 1e3` × 2 | 累计放电 Ah、累计充电 Ah；只有 `capacity_totals_available=true` 时有效 |

字段可用性由帧长度决定。短帧缺少的字段必须返回 null/无效位；不能将结构体默认零解释成设备实测零。完整告警/保护名称查 `reference/ant_official_bitnames.json`、`reference/ant_official_app_tables.json` 和项目说明；官方与社区版本映射存在边界，保留原始位图。

### 3.4 设置寄存器只读

寄存器索引：`reference/antbms_params.json`（项目的56项地址、缩放、单位和标签表）。一般按 `0x02 + address LE16 + N` 请求，响应 `0x12`。常见宽度为2字节，容量寄存器 `0x00A2`、`0x00A6`、`0x00AA` 为4字节。参数名和缩放必须从 JSON 对应寄存器项读取，不要按中文标签自行猜地址。

### 3.5 蚂蚁写入保护

该项目明确为纯只读。ANT 的写寄存器、MOS、均衡、蜂鸣器/蓝牙开关等都可能改变保护行为；当前没有五级权限密码。任何 AI/代理不得因为看到了帧格式、权限字段或历史代码而发送写入帧。若请求涉及写入，应停止在生成/解释静态帧层面，不连接设备、不执行下发。

## 4. 推荐给其他 AI 的集成输出模型

```json
{
  "brand_id": "ant | gret",
  "device_class": "bms | controller",
  "connected": true,
  "sampled_at_ms": 0,
  "raw_frame_hex": "...",
  "values": {
    "voltage_v": null,
    "raw_current_a": null,
    "current_sign": "unknown",
    "soc_percent": null,
    "rpm_source": null,
    "mosfet_temperature_c": null,
    "motor_temperature_c": null,
    "cell_voltages_v": null,
    "protect_flags": null,
    "warning_flags": null
  },
  "valid_fields": [],
  "evidence": "source-derived | synthetic-vector | hardware-observed",
  "hardware_verified": false
}
```

字段不适用于当前品牌时省略或置 null，并从 `valid_fields` 排除。数据超时后将 `connected` 与 last sample freshness 分开表示；BLE 连接状态不等于新鲜遥测。

## 5. 本仓库可复用实现与证据入口

- ANT Python：`../antbms_protocol.py`（帧构造、校验、切帧、实时/设置响应解析）；索引 `../ANT-资料索引.md`；寄存器表 `../reference/antbms_params.json`。
- ANT ESP32：`../esp32/ble_bridge/protocols.h` 的 `namespace ant`；GATT 连接/发现由 `ble_bridge.ino` 管理。
- 格锐特 ESP32：`../esp32/ble_bridge/mbus_controller_adapters.h` 的 `BRAND_CONTROLLER_GRET`、`requestAtForBrand()`、`parseGret()`。
- 格锐特协议能力矩阵：`multibrand-protocol-matrix.md`；来源审计：`habe-device-support-audit.md` 与 `habe-bms-controller-analysis.md`。
- 格锐特合成帧断言：`../tests/instrument/test_mbus_controller_adapters.cpp`。该向量只检验当前解码逻辑，不是硬件证据。

路径相对于本文件所在目录 `docs/`。移植时优先引用这些源码并保留协议版本/来源，不要将本文件当成厂商官方规格书。
