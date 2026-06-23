# 🤖 StackChan-XiaoZhi

> M5Stack Core S3 Stack-chan 陪伴机器人固件 —— 触摸 · 体感 · 情绪灯 · 舵机

基于 [xiaozhi-esp32](https://github.com/78/xiaozhi-esp32)，专注**陪伴感**体验。源自 [mo-hantang/Stackchan-HtSz](https://github.com/mo-hantang/Stackchan-HtSz)。

<p align="center">
  <img src="https://img.shields.io/badge/platform-ESP32--S3-green?logo=espressif" alt="ESP32-S3">
  <img src="https://img.shields.io/badge/framework-ESP--IDF%20v5.5-blue?logo=espressif" alt="ESP-IDF v5.5">
  <img src="https://img.shields.io/badge/license-MIT-lightgrey" alt="License">
  <img src="https://img.shields.io/github/v/release/howecheung/StackChan-XiaoZhi" alt="Release">
</p>

---

## ✨ 功能

| 模块 | 硬件 | 说明 |
|---|---|---|
| 🎤 唤醒词 | Multinet6 | "你好小智"，灵敏度 Low / Medium / High |
| 👆 头顶触摸 | SI12T | 3 区电容触摸，摸头对话，7 条随机回应 |
| 📱 屏幕触摸 | FT6336 | 双击 / 滑动 / 长按 6 种手势，60 条动作文本 |
| 🕺 体感检测 | BMI270 | 摇晃 + 抱起互动，说话自动屏蔽，5 分钟冷却 |
| 💡 情绪灯环 | WS2812×12 | 21 种情绪颜色，PY32 IO Expander 控制 |
| 🎯 舵机追踪 | SCS 总线 | GC0308 人脸追踪 + 空闲扫视，对话暂停 |
| 😴 智能休眠 | — | 30 秒空闲浅休（关灯/停舵机/熄屏）→ 60 秒深度睡眠（黑屏静音），点击屏幕唤醒进聆听 |
| 🔌 重新配网 | — | Idle 长按屏幕 5 秒进入 192.168.4.1 配网（页面已美化，升级至 esp-wifi-connect v3.2.1，支持暗色模式） |
| ✂️ 文本打断 | — | 对话中插文本消息，不静默丢弃 |
| 🔧 MCP 工具 | — | LED / 音量 / 亮度 / 摄像头 / 系统信息 |

## 📡 全链路部署指南

从硬件固件到云端联网搜索的完整部署链路 → **[README_ALL.md](README_ALL.md)**

---

## 🚀 快速开始

从 [Releases](https://github.com/howecheung/StackChan-XiaoZhi/releases) 下载 `bootloader.bin`、`partition-table.bin`、`ota_data_initial.bin`、`xiaozhi.bin`，一行命令烧录：

```bash
pip install esptool
esptool.py --chip esp32s3 -p COM4 -b 460800 \
  --before default_reset --after hard_reset \
  write_flash --flash_mode dio --flash_size 16MB --flash_freq 80m \
  0x0       bootloader.bin \
  0x8000    partition-table.bin \
  0xd000    ota_data_initial.bin \
  0x410000  xiaozhi.bin
```

> `COM4` 换成实际串口号。首次烧录建议先 `esptool.py erase_flash` 清空旧配置。

---

## 🔨 源码编译

需要 **ESP-IDF v5.5.x**，在 "ESP-IDF 5.5 CMD" 中执行：

```bat
git clone https://github.com/howecheung/StackChan-XiaoZhi.git
cd StackChan-XiaoZhi

:: 首次编译
idf.py set-target esp32s3
idf.py fullclean
idf.py build flash

:: 后续增量编译
idf.py build flash
```

低配机器防 OOM：`idf.py build flash -- -j1`

---

## ⚙️ 配置

编辑 `sdkconfig.defaults`：

| 配置项 | 默认值 | 说明 |
|---|---|---|
| `CONFIG_CUSTOM_WAKE_WORD` | `ni hao xiao zhi` | 唤醒词拼音 |
| `CONFIG_CUSTOM_WAKE_WORD_DISPLAY` | `你好小智` | 中文显示 |
| `CONFIG_WAKE_WORD_SENSITIVITY` | `MEDIUM` | `LOW` / `MEDIUM` / `HIGH` |

### 🛠 自定义行为池

编辑 `main/boards/m5stack-core-s3/m5stack_core_s3.cc`：

| 搜索关键词 | 对应功能 |
|---|---|
| `HeadTouchPool` | 摸头回应文本 |
| `DoubleClickPool` / `UpSwipePool` … | 手势动作文本 |
| `ShakePool` / `LiftPool` | 体感动作（MotionMsg 格式） |
| `UpdateLedsFromEmotion` | 情绪灯颜色映射 |

---

## 📖 功能使用方法

### 🔌 配网页面预览（无需烧录）

修改配网页面后，可本地预览效果，无需编译烧录：

```bash
node scripts/wifi-preview.js
```

浏览器打开 `http://localhost:3000`，修改 `managed_components/78__esp-wifi-connect/assets/wifi_configuration.html` 后刷新即可。模拟了 5 个 WiFi 热点 + 3 个已保存网络，OTA 和睡眠模式选项可见。

---

## ⚠️ 已知问题

<details>
<summary>展开查看</summary>

1. **电池供电 + 灯全亮可能无法开机** —— 插 USB 再开机
2. **LLM 控灯偶尔卡住 / 口头说调灯实际不动** —— prompt 优化中
3. **烧录后需物理重启** —— 拔 USB 关机，等 30 秒再插电
4. **唤醒词灵敏度需自行调整** —— 生僻词选 High，常见词选 Medium
5. **自部署服务端注意安全**：`auth.enabled=true`、MAC 白名单、API Key 不暴露、端口不裸奔

</details>

---

## 📋 更新日志

<details>
<summary>展开查看</summary>

- **2026-06-23** · 深度睡眠模式：30 秒空闲进入深度睡眠（黑屏静音），点击屏幕唤醒进聆听模式；屏蔽体感 BMI270 摇晃/抱起和顶部触摸 SI12T 的睡眠误唤醒路径
- **2026-06-16** · **v0.0.2** 发布
- **2026-06-16** · 配网页面持续美化：🤖 机器人图标 + "StackChan 网络配置"品牌标识 + 日夜主题切换（自动/浅色/深色）+ 信号柱状条分色重绘 + 语言精简为中英文；新增 `self.face.expression` MCP 工具（19 种表情）+ `self.get_system_info` 改为通用工具；配网连接超时 60s → 10s + 连续 3 次失败自动清除失效凭据 + Idle 长按进配网时清空旧 WiFi；本地预览服务器 `node scripts/wifi-preview.js`
- **2026-06-16** · esp-wifi-connect v3.1.5 → v3.2.1：CSS 变量设计令牌 + 暗色模式 + 渐变背景 + 扫描动画 + 45 种语言；开启 OTA 配置和休眠模式可见性
- **2026-06-15** · 配网页面（192.168.4.1）美化：iOS 风格分段按钮 + 大圆角卡片 + 信号强度可视化 + StackChan 品牌标识
- **2026-06-11** · 修复休眠机制（恢复空闲检查，对话中不再误休眠，超时 60s → 30s）；Idle 长按屏幕 5 秒进入配网模式；说话时屏蔽 BMI270 体感避免舵机晃动误触 IMU
- **2026-06-10** · 编译修复 + 版本检查重试优化（`MAX_RETRY=1`）+ 默认 WebSocket 协议；触摸/SendUserText 长文本拦截修复（≤24 字节）；屏蔽 OTA 远程升级；体感池拆 `display+tag`、删早安问候、删 `upgrade_firmware` MCP 工具
- **2026-06-06** · 唤醒词灵敏度 Low/Medium/High 三档；文本打断优化；休眠背光彻底熄灭；SPIRAM 改回 QUAD

</details>

---

## 🙏 致谢

- [xiaozhi-esp32](https://github.com/78/xiaozhi-esp32) — 基础固件
- [xiaozhi-esp32-server](https://github.com/xinnan-tech/xiaozhi-esp32-server) — 服务端
- [M5Stack StackChan-BSP](https://github.com/m5stack/StackChan-BSP) — SI12T 参考
- [esp-wifi-connect](https://github.com/78/esp-wifi-connect) — WiFi 配网组件

## 📄 License

同上游 [xiaozhi-esp32](https://github.com/78/xiaozhi-esp32)。
