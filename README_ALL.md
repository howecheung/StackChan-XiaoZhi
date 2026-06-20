# StackChan 全链路运行指南

本文档详细介绍从硬件固件到云端服务的 **StackChan 语音助手全链路部署流程**。

---

## 一、整体架构

全链路数据流：

```
ESP32 设备 → WiFi → WebSocket → 后端服务 → MCP 服务 → 智谱AI 搜索
  (固件)              (xiaozhi.me / 自建)   (webvox-mcp)   (联网查询)
```

各组件协作流程：

- ESP32 硬件搭载小智固件，采集语音并上报至后端
- 后端服务接收语音请求，进行 ASR/TTS 处理和意图识别
- MCP 服务注册联网查询工具，后端通过 WebSocket 管道调用
- 智谱 AI Web Search API 执行联网检索并返回结构化结果

---

## 二、硬件准备

推荐使用以下任一 ESP32 开发板：

- **ESP32-S3-DevKitC**（推荐，性能较好）
- **ESP32-WROOM-32D**（通用型，性价比高）
- **M5Stack CoreS3 / AtomS3**（带屏幕，体验更好）

配件需求：I2S 麦克风模块（INMP441 或 ES7210）、扬声器或耳机接口、LED 指示灯（可选）。

接线方式请参考各固件仓库的 README 说明。

---

## 三、固件烧录

### 方案 A：StackChan-XiaoZhi 固件

> 仓库地址：https://github.com/howecheung/StackChan-XiaoZhi

**Step 1 — 克隆仓库并配置环境：**

```bash
git clone https://github.com/howecheung/StackChan-XiaoZhi.git
cd StackChan-XiaoZhi
```

**Step 2 — 安装 ESP-IDF 开发框架**（推荐 VSCode + ESP-IDF 插件），通过 ESP-IDF 工具链编译。

**Step 3 — 在 menuconfig 中配置** WiFi SSID/密码，设置 WebSocket 后端地址（官方或自建）。

**Step 4 — 编译并烧录：**

```bash
idf.py build flash monitor
```

烧录完成后通过串口监视器确认设备启动和 WiFi 连接状态。

> 也可以从 [Releases](https://github.com/howecheung/StackChan-XiaoZhi/releases) 直接下载 bin 文件用 esptool 烧录，无需编译。

### 方案 B：基础小智 ESP32 固件

> 仓库地址：https://github.com/78/xiaozhi-esp32

**Step 1 — 克隆并初始化子模块：**

```bash
git clone --recursive https://github.com/78/xiaozhi-esp32.git
```

**Step 2 — 按照 README** 配置 WiFi、语言、GPIO 引脚等参数。

**Step 3 — 编译烧录**（同上，使用 ESP-IDF 工具链）。

**Step 4 — 开发板首次启动后**，通过串口日志确认连接状态。

---

## 四、后端服务

### 方案 A：小智官方云服务

访问小智 AI 控制台：https://xiaozhi.me/

在控制台中操作：

- 注册或登录账号
- 添加设备，获取设备 ID 和 MCP 接入点 token
- MCP 接入点格式：`wss://api.xiaozhi.me/mcp/?token=<你的token>`

将获取到的 token 填入固件配置或 MCP 服务配置中。

### 方案 B：自建服务端

> 仓库地址：https://github.com/xinnan-tech/xiaozhi-esp32-server

部署步骤：

- 服务器要求：Linux (Ubuntu 20.04+) 或 Docker，Python 3.10+
- 克隆仓库后按 README 配置 `config.yaml` 和 ASR/TTS/LLM 参数
- 启动服务后获取 WebSocket 地址，格式：`ws://<服务器IP>:<端口>`
- 将固件中的后端地址改为自建服务地址

自建方案支持自定义唤醒词、本地 ASR/TTS、私有知识库等高级功能。

---

## 五、MCP 联网搜索服务

> 仓库地址：https://github.com/howecheung/webvox-mcp

WebVox-MCP 为语音助手提供实时联网检索能力，需要智谱 AI API Key（免费申请）。

智谱 AI Key 申请：https://open.bigmodel.cn/

### Docker 部署（NAS / 服务器推荐）

完整教程见项目 README，以下为核心步骤：

```bash
mkdir ~/webvox-mcp && cd ~/webvox-mcp

# 下载项目文件
curl -O https://raw.githubusercontent.com/howecheung/webvox-mcp/master/Dockerfile
curl -O https://raw.githubusercontent.com/howecheung/webvox-mcp/master/docker-compose.yaml
curl -O https://raw.githubusercontent.com/howecheung/webvox-mcp/master/docker-entrypoint.sh
curl -O https://raw.githubusercontent.com/howecheung/webvox-mcp/master/mcp_pipe.py
curl -O https://raw.githubusercontent.com/howecheung/webvox-mcp/master/联网查询.py
curl -O https://raw.githubusercontent.com/howecheung/webvox-mcp/master/config_manager.py
curl -O https://raw.githubusercontent.com/howecheung/webvox-mcp/master/requirements.txt

# 配置环境变量
cat > .env << EOF
MCP_ENDPOINT=wss://api.xiaozhi.me/mcp/?token=你的token
ZHIPU_API_KEY=你的智谱API密钥
EOF

# 构建并启动
docker compose up -d
docker logs -f webvox-mcp
```

### Windows 部署

从 GitHub Releases 下载 `webvox-mcp.exe` 或源码 ZIP 包：

https://github.com/howecheung/webvox-mcp/releases

- `webvox-mcp.exe` — 绿色单文件，双击运行 GUI 配置面板
- `webvox-mcp_0.0.1.zip` — 源码压缩包，需 Python 环境

### 源码运行

```bash
git clone https://github.com/howecheung/webvox-mcp.git
cd webvox-mcp
pip install -r requirements.txt
python 启动_main.py
```

---

## 六、全链路联调

按以下顺序逐一确认各环节状态：

1. **确认 ESP32** 正常启动并连接 WiFi，串口无报错
2. **确认固件连接后端** — 串口日志显示 WebSocket 连接成功，或在 xiaozhi.me 控制台看到设备在线
3. **确认 MCP 服务** 启动且 WebSocket 连接成功：
   ```bash
   docker logs -f webvox-mcp
   # 看到 "Successfully connected to WebSocket server" 即成功
   ```
4. **语音测试** — 对小智说「查询最新新闻」或「金价多少」触发联网搜索
5. **后台确认** — 在小智控制台查看 MCP 工具调用日志，确认搜索请求被正确处理

---

## 七、参考链接

| 项目 | 地址 |
|------|------|
| StackChan-XiaoZhi 固件 | https://github.com/howecheung/StackChan-XiaoZhi |
| 基础小智 ESP32 固件 | https://github.com/78/xiaozhi-esp32 |
| 小智官方云服务 | https://xiaozhi.me/ |
| 自建后端服务 | https://github.com/xinnan-tech/xiaozhi-esp32-server |
| WebVox-MCP 搜索服务 | https://github.com/howecheung/webvox-mcp |
| 智谱 AI 开放平台 | https://open.bigmodel.cn/ |
| 小智 MCP 服务接入指南 | https://my.feishu.cn/docx/JKFXd8bLYo6YZtxz9ORcbnA8nbe |
