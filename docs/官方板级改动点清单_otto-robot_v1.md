# 官方板级改动点清单（otto-robot，v1）

## 1. 目标
这份清单用于回答第 4 步：`boards/otto-robot` 哪些可以改、哪些建议不要动，并给出你当前硬件（ESP32-S3 N16R8 + ST7789 240x240 + INMP441 + MAX98357A）的落地建议。

---

## 2. 当前代码现状（先确认）
- 舵机总开关当前是关闭状态：`OTTO_ENABLE_SERVOS=0`  
  位置：`/Users/lss/Desktop/AI_MCP/firmware/xiaozhi-esp32/main/boards/otto-robot/config.h:9`
- `otto_controller/otto_movements/oscillator` 全部被 `#if OTTO_ENABLE_SERVOS` 包裹，关闭后不会参与运行逻辑  
  位置：  
  `/Users/lss/Desktop/AI_MCP/firmware/xiaozhi-esp32/main/boards/otto-robot/otto_controller.cc:7`  
  `/Users/lss/Desktop/AI_MCP/firmware/xiaozhi-esp32/main/boards/otto-robot/otto_movements.cc:3`  
  `/Users/lss/Desktop/AI_MCP/firmware/xiaozhi-esp32/main/boards/otto-robot/oscillator.cc:11`
- 这意味着：你“去掉舵机控制”在当前仓库状态下本质上已经完成（编译期开关已关）。

---

## 3. 改动分级清单（A/B/C）

## 3.1 A 级（建议改，风险低）
| 文件 | 建议改动 | 原因 |
|---|---|---|
| `/Users/lss/Desktop/AI_MCP/firmware/xiaozhi-esp32/main/boards/otto-robot/config.h` | 维护引脚映射（显示、麦克风、功放）、显示参数、硬件版本策略 | 这是板级适配的主入口，改这里最稳 |
| `/Users/lss/Desktop/AI_MCP/firmware/xiaozhi-esp32/main/boards/otto-robot/otto_emoji_display.cc` | 调整状态栏、表情、UI 展示行为 | 只影响显示层，不影响协议主链路 |
| `/Users/lss/Desktop/AI_MCP/firmware/xiaozhi-esp32/main/boards/otto-robot/power_manager.h` | 校准电量 ADC 阈值 | 只影响电量显示准确性 |
| `/Users/lss/Desktop/AI_MCP/firmware/xiaozhi-esp32/main/boards/otto-robot/websocket_control_server.cc` | 如果不用小程序控制可关掉或加鉴权 | 属于板级附加能力，不是主语音链路必需 |

## 3.2 B 级（可改，但要自测）
| 文件 | 建议改动 | 风险点 |
|---|---|---|
| `/Users/lss/Desktop/AI_MCP/firmware/xiaozhi-esp32/main/boards/otto-robot/otto_robot.cc` | 硬件自动检测策略、启动顺序、网络启动后附加服务 | 容易影响开机成功率、摄像头判定、显示初始化 |
| `/Users/lss/Desktop/AI_MCP/firmware/xiaozhi-esp32/main/boards/otto-robot/config.json` | 精简 sdkconfig_append（比如不需要摄像头时） | 改错会导致功能缺失或构建差异 |

## 3.3 C 级（当前阶段不建议动）
| 文件 | 不建议原因 |
|---|---|
| `/Users/lss/Desktop/AI_MCP/firmware/xiaozhi-esp32/main/protocols/protocol.*` | 是官方协议抽象主干，改动会影响所有板型 |
| `/Users/lss/Desktop/AI_MCP/firmware/xiaozhi-esp32/main/protocols/websocket_protocol.*` | 影响 WebSocket 全链路收发与握手 |
| `/Users/lss/Desktop/AI_MCP/firmware/xiaozhi-esp32/main/protocols/mqtt_protocol.*` | 影响 MQTT+UDP 加解密和时序 |
| `/Users/lss/Desktop/AI_MCP/firmware/xiaozhi-esp32/main/mcp_server.*` | 影响全局 MCP 协议兼容性（initialize/tools/list/tools/call） |

---

## 4. 与你硬件直接相关的关键点

## 4.1 显示与音频引脚（无摄像头版本）
你的板型对应无摄像头配置（`NON_CAMERA_VERSION_CONFIG`）：
- 显示：`MOSI=10`、`CLK=9`、`DC=46`、`RST=11`、`CS=12`、背光 `3`
- 麦克风（INMP441）：`WS=4`、`SCK=5`、`DIN=6`
- 功放（MAX98357A）：`DOUT=7`、`BCLK=15`、`LRCK=16`

位置：`/Users/lss/Desktop/AI_MCP/firmware/xiaozhi-esp32/main/boards/otto-robot/config.h:109`

## 4.2 GPIO12 的代码角色
- 在无摄像头配置里：`GPIO12` 被用作 `display_cs_pin`
  - 位置：`/Users/lss/Desktop/AI_MCP/firmware/xiaozhi-esp32/main/boards/otto-robot/config.h:142`
- 在摄像头引脚定义里：`GPIO12` 也被定义为 `CAMERA_D0`
  - 位置：`/Users/lss/Desktop/AI_MCP/firmware/xiaozhi-esp32/main/boards/otto-robot/config.h:155`

结论：`GPIO12` 的用途取决于硬件分支（摄像头版/无摄像头版），不能同时按两种用途使用。

## 4.3 建议固定硬件版本，避免误判
当前默认是自动检测：
- `OTTO_HARDWARE_VERSION OTTO_VERSION_AUTO`
  - 位置：`/Users/lss/Desktop/AI_MCP/firmware/xiaozhi-esp32/main/boards/otto-robot/config.h:17`
- 自动检测逻辑在：
  - `/Users/lss/Desktop/AI_MCP/firmware/xiaozhi-esp32/main/boards/otto-robot/otto_robot.cc:43`

对你这种固定无摄像头硬件，更稳妥做法是固定到 `OTTO_VERSION_NO_CAMERA`，减少误判分支带来的初始化偏差。

---

## 5. 面向“AI宠物（无舵机）”的最小改动方案
1. 保持 `OTTO_ENABLE_SERVOS=0`（当前已是 0）。  
2. 固定硬件版本到无摄像头（建议改 `OTTO_HARDWARE_VERSION`）。  
3. 保留 `config.h` 里当前 ST7789/INMP441/MAX98357A 引脚映射。  
4. 如果不使用微信小程序控制，关闭 `WebSocketControlServer` 启动路径：  
   - 启动点：`/Users/lss/Desktop/AI_MCP/firmware/xiaozhi-esp32/main/boards/otto-robot/otto_robot.cc:242`  
   - 服务实现：`/Users/lss/Desktop/AI_MCP/firmware/xiaozhi-esp32/main/boards/otto-robot/websocket_control_server.cc:87`
5. 仅做 UI/情绪表达优化，优先改 `otto_emoji_display.cc`，不要下沉到协议层。

---

## 6. 为什么这样分层最稳
- 应用层始终通过 `Protocol` 抽象发消息（listen/abort/mcp），板级不需要动协议主干。  
  位置：`/Users/lss/Desktop/AI_MCP/firmware/xiaozhi-esp32/main/protocols/protocol.h:44`
- MCP 通用工具与用户工具在应用初始化时统一注册，不依赖 otto 舵机模块。  
  位置：`/Users/lss/Desktop/AI_MCP/firmware/xiaozhi-esp32/main/application.cc:98`
- 因此“无舵机 AI 宠物”应优先走“板级裁剪 + UI 定制”，而不是改协议核心。

---

## 7. 快速自检清单（改完必测）
1. 上电后显示是否稳定点亮（240x240，方向/颜色正常）。  
2. 唤醒词和麦克风采集是否正常（INMP441）。  
3. TTS 播放是否正常（MAX98357A）。  
4. 设备能否正常建立会话、收发 `hello/listen/stt/tts`。  
5. 若关闭本地 WebSocket 控制服务，语音主链路是否不受影响。