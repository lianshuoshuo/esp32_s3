# 官方 MCP 接入官方路径操作手册（v1）

## 1. 目标
这份手册只讲**官方代码已经支持**的 MCP 接入路径，目的是避免“接入方式混用”和“配置写对但链路不生效”。

适用目录：
- `services/xiaozhi-esp32-server/main/xiaozhi-server`
- `firmware/xiaozhi-esp32/main`

---

## 2. 三条官方路径（先看这个）

| 路径 | 典型用途 | 配置位置 | 是否依赖 MCP 接入点服务 |
|---|---|---|---|
| 设备端 MCP（固件工具） | 调用设备自身/板级工具 | 固件与设备握手（`features.mcp=true`） | 否 |
| 服务端 MCP（Server MCP） | 让服务端直接连本地/远端 MCP Server | `data/.mcp_server_settings.json` | 否 |
| MCP 接入点（Endpoint） | 把独立 MCP 客户端/工具桥接到小智智能体 | `data/.config.yaml` 的 `mcp_endpoint` | 是（`mcp-endpoint-server`） |

结论：  
**这三条链路可以并存**，最终统一汇总到 `UnifiedToolHandler` 的工具列表里。

---

## 3. 路径 A：设备端 MCP（固件工具）

### 3.1 官方链路
1. 设备连接服务端后发送 `hello`，声明 `features.mcp=true`。  
2. 服务端在 `hello` 处理里创建 `MCPClient` 并发送 `initialize`。  
3. 继续发送 `tools/list`，拉取设备端工具。  
4. 工具加入统一工具管理器，后续可被 LLM 调用。

### 3.2 关键代码入口
- `firmware/xiaozhi-esp32/main/protocols/websocket_protocol.cc:203`（固件 hello 中声明 MCP 特性）
- `services/xiaozhi-esp32-server/main/xiaozhi-server/core/handle/helloHandle.py:47`（识别 `features.mcp`）
- `services/xiaozhi-esp32-server/main/xiaozhi-server/core/providers/tools/device_mcp/mcp_handler.py:243`（发送 initialize）
- `services/xiaozhi-esp32-server/main/xiaozhi-server/core/providers/tools/device_mcp/mcp_handler.py:273`（发送 tools/list）
- `services/xiaozhi-esp32-server/main/xiaozhi-server/core/handle/textHandler/mcpMessageHandler.py:17`（处理 `type=mcp` 消息）

### 3.3 生效检查
- 设备端已接入并正常握手。
- 服务端日志可看到设备 MCP 初始化与工具列表加载相关日志（如“客户端设备支持的工具数量”）。

---

## 4. 路径 B：服务端 MCP（`data/.mcp_server_settings.json`）

### 4.1 适合场景
你想让 `xiaozhi-server` 直接连接 MCP Server（stdio / SSE / streamable-http），不经过 MCP 接入点。

### 4.2 配置步骤（官方）
1. 准备配置文件：  
   - 目标：`services/xiaozhi-esp32-server/main/xiaozhi-server/data/.mcp_server_settings.json`  
   - 可复制模板：`services/xiaozhi-esp32-server/main/xiaozhi-server/mcp_server_settings.json`
2. 在 `"mcpServers"` 内填写服务。支持三种模式：  
   - `command + args`（stdio）  
   - `url`（SSE，默认）  
   - `url + transport: "streamable-http"`
3. 重启 `xiaozhi-server`。

### 4.3 关键代码入口
- `services/xiaozhi-esp32-server/main/xiaozhi-server/core/providers/tools/server_mcp/mcp_manager.py:25`（读取 `data/.mcp_server_settings.json`）
- `services/xiaozhi-esp32-server/main/xiaozhi-server/core/providers/tools/server_mcp/mcp_client.py:169`（按 command/url 建连接）
- `services/xiaozhi-esp32-server/main/xiaozhi-server/core/providers/tools/server_mcp/mcp_client.py:243`（`list_tools` 并缓存工具）
- `services/xiaozhi-esp32-server/main/xiaozhi-server/core/providers/tools/server_mcp/mcp_executor.py:21`（初始化入口）

### 4.4 生效检查
- 启动日志出现“初始化服务端MCP客户端: xxx”。
- 出现“服务端MCP客户端已连接，可用工具: [...]”。
- 出现“当前支持的函数列表: [...]”，并包含你的 MCP 工具名。

### 4.5 常见坑
- 文件名必须是 `data/.mcp_server_settings.json`（不是别名）。
- JSON 末尾逗号会导致加载失败。
- `command: "npx"` 时依赖系统环境可找到 `npx`。
- 旧字段 `API_ACCESS_TOKEN` 仍可兼容，但官方代码已提示改成 `headers.Authorization`。

---

## 5. 路径 C：MCP 接入点（`mcp-endpoint-server` + `mcp_pipe`）

### 5.1 适合场景
你有独立运行的 MCP 工具进程（如 Python MCP Server），希望通过官方“接入点”动态挂到智能体。

### 5.2 官方操作顺序
1. 先部署 `mcp-endpoint-server`（官方独立仓库）。  
2. 从日志拿到两个地址：  
   - 管理检测地址（`health?key=...`）  
   - MCP WebSocket 地址（`.../mcp/?token=...`）  
3. 如果是 Docker 部署，地址里的容器 IP 要替换为宿主机局域网 IP。  
4. 在 `xiaozhi-server` 配置 `mcp_endpoint` 为 **WebSocket MCP 地址**。  
5. 启动你的 MCP 客户端桥接（例如 `python mcp_pipe.py calculator.py`）。

### 5.3 `mcp_endpoint` 格式规范（官方代码硬校验）
校验函数：`services/xiaozhi-esp32-server/main/xiaozhi-server/core/utils/util.py:576`

必须满足：
1. 以 `ws` 开头。  
2. 包含 `/mcp/`。  
3. 不能包含 `key` 或 `call`。  

通过后，`app.py` 会把配置里的 `/mcp/` 自动替换成 `/call/` 用于内部调用：  
- `services/xiaozhi-esp32-server/main/xiaozhi-server/app.py:97`

所以你应该填写：
- `ws://<ip>:8004/mcp_endpoint/mcp/?token=xxx`（正确）

不应填写：
- `http://.../health?key=...`（错误，管理检测地址）
- `ws://.../call/...`（错误，会被校验拦截）

### 5.4 关键代码入口
- `services/xiaozhi-esp32-server/main/xiaozhi-server/core/providers/tools/unified_tool_handler.py:76`（初始化 MCP 接入点）
- `services/xiaozhi-esp32-server/main/xiaozhi-server/core/providers/tools/mcp_endpoint/mcp_endpoint_handler.py:13`（连接接入点）
- `services/xiaozhi-esp32-server/main/xiaozhi-server/core/providers/tools/mcp_endpoint/mcp_endpoint_handler.py:224`（initialize）
- `services/xiaozhi-esp32-server/main/xiaozhi-server/core/providers/tools/mcp_endpoint/mcp_endpoint_handler.py:250`（tools/list）
- `services/xiaozhi-esp32-server/main/xiaozhi-server/core/providers/tools/mcp_endpoint/mcp_endpoint_handler.py:273`（tools/call）

### 5.5 生效检查
- 服务端日志出现“正在初始化MCP接入点: ...”。
- 出现“MCP接入点连接成功”。
- 出现“MCP接入点支持的工具数量: N”。
- 最终“当前支持的函数列表”中包含接入点工具。

---

## 6. 你当前项目推荐路线（本地自建）

建议按这个顺序，稳定且不容易走偏：

1. **先跑通路径 B（服务端 MCP）**  
   原因：不依赖额外接入点服务，排错半径最小。
2. **再加路径 C（MCP 接入点）**  
   用于接入“独立运行”的 MCP 工具进程（如新闻、百科、电脑控制等）。
3. **设备端路径 A 保持开启**  
   设备工具（音量/亮度/板级能力）继续可用，不冲突。

---

## 7. 快速排错表

### 7.1 `mcp接入点不符合规范`
优先检查：
- 是否 `ws://` 或 `wss://`
- 是否包含 `/mcp/`
- 是否误填了 `health?key=...`
- 是否误填了 `/call/`

### 7.2 工具没出现在“当前支持的函数列表”
按顺序查：
1. 接入路径是否真的初始化成功（看日志）。  
2. MCP 服务是否返回了 `tools/list`。  
3. 工具名/参数是否满足 MCP 规范（命名清晰、schema 合法）。  
4. `tool_manager.refresh_tools()` 是否被触发（路径 A/B/C 代码中都有触发点）。

### 7.3 容器里地址能通、宿主机不通
这是典型 Docker 宿主机 IP 替换问题：  
日志中的 `172.x.x.x` 通常是容器内网地址，外部访问要换成宿主机局域网 IP。

---

## 8. 参考文档（官方仓库内）
- `services/xiaozhi-esp32-server/docs/mcp-endpoint-enable.md`
- `services/xiaozhi-esp32-server/docs/mcp-endpoint-integration.md`
- `services/xiaozhi-esp32-server/docs/mcp-get-device-info.md`
- `services/xiaozhi-esp32-server/main/xiaozhi-server/mcp_server_settings.json`