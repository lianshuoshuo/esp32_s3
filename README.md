# esp32_s3

这是一个面向本地部署的小智相关单仓库快照，整理了固件、服务端、MQTT 网关、MCP 接入点、声纹服务，以及本地运行时编排文件。

仓库目标不是替代各 upstream 的完整发布流程，而是把当前这套可复现的部署结构放到一个仓库里，方便统一管理和二次修改。

## 目录结构

- `deploy/xiaozhi-esp32`
  小智 ESP32 固件源码，当前用于 `otto-robot` 等板型编译。
- `deploy/xiaozhi-esp32-server`
  小智服务端与智控台源码。
- `deploy/xiaozhi-mqtt-gateway`
  MQTT 网关源码，默认通过 PM2 启动。
- `deploy/mcp-endpoint-server`
  MCP 接入点服务。
- `deploy/voiceprint-api`
  声纹服务。
- `deploy/runtime/xiaozhi-server`
  小智服务端运行时 `docker-compose.yml`。
- `deploy/runtime/xiaozhi-manager`
  智控台、MySQL、Redis 运行时 `docker-compose.yml`。

## 本仓库未上传的内容

以下内容被有意排除，不在仓库中：

- 运行时敏感配置
  例如 `deploy/runtime/xiaozhi-server/data/.config.yaml`、`deploy/xiaozhi-mqtt-gateway/.env`
- 模型和大资源
  例如 `model.pt`、音乐文件、测试资源、生成器静态模型
- 编译产物和运行时数据
  例如 `build/`、`releases/`、数据库目录、日志目录、上传目录

如果你要在新机器复现部署，需要自行补齐这些文件。

## 快速开始

### 1. 创建 Docker 网络

```bash
docker network create xiaozhi-local
```

### 2. 启动智控台

```bash
cd deploy/runtime/xiaozhi-manager
docker compose up -d
```

默认端口：

- `8002`: 智控台

### 3. 启动小智服务端

先自行准备：

- `deploy/runtime/xiaozhi-server/data/.config.yaml`
- `deploy/runtime/xiaozhi-server/models/SenseVoiceSmall/model.pt`

然后启动：

```bash
cd deploy/runtime/xiaozhi-server
docker compose up -d
```

默认端口：

- `8000`: WebSocket
- `8003`: OTA / 视觉分析

### 4. 启动 MCP 接入点

```bash
cd deploy/mcp-endpoint-server
docker compose up -d
```

默认端口：

- `8004`: MCP endpoint

### 5. 启动声纹服务

```bash
cd deploy/voiceprint-api
docker compose up -d --build
```

默认端口：

- `8005`: Voiceprint API

### 6. 启动 MQTT 网关

先自行准备：

- `deploy/xiaozhi-mqtt-gateway/.env`
- `deploy/xiaozhi-mqtt-gateway/config/mqtt.json`

然后启动：

```bash
cd deploy/xiaozhi-mqtt-gateway
npm install
pm2 start ecosystem.config.js --only xz-mqtt
```

默认端口：

- `1883`: MQTT
- `8884`: UDP
- `8007`: 网关管理 API

## 固件

固件源码在 `deploy/xiaozhi-esp32`。

常见流程：

```bash
cd deploy/xiaozhi-esp32
idf.py set-target esp32s3
idf.py build
idf.py -p /dev/tty.usbmodemXXXX flash
```

如果要针对 `otto-robot` 出包，可直接看仓库内脚本：

- `deploy/xiaozhi-esp32/scripts/release.py`

## 上游来源

这套单仓库内容主要来自以下上游项目：

- `xinnan-tech/xiaozhi-esp32-server`
- `78/xiaozhi-esp32`
- `xinnan-tech/xiaozhi-mqtt-gateway`
- `xinnan-tech/mcp-endpoint-server`
- `xinnan-tech/voiceprint-api`

如果你要继续升级某一模块，建议先对照对应 upstream 的最新变更。
