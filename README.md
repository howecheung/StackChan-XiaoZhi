# Stackchan-XiaoZhi

M5Stack Core S3 Stack-chan 固件 —— 基于 [xiaozhi-esp32](https://github.com/78/xiaozhi-esp32)，增加触摸、体感、情绪灯、舵机等交互，侧重**陪伴感**体验。

> 源自 [mo-hantang/Stackchan-HtSz](https://github.com/mo-hantang/Stackchan-HtSz)

## 快速开始

从 [Releases](https://github.com/howecheung/Stackchan-XiaoZhi/releases) 下载 4 个 `.bin` 文件，一行命令烧录：

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

> `COM4` 换成你的串口号。首次烧录建议先 `esptool.py erase_flash` 清空旧配置。

## 功能

| 模块 | 说明 |
|------|------|
| **唤醒词** | Multinet6 "你好小智"，灵敏度 Medium（可调） |
| **头顶触摸** (SI12T) | 3 区电容触摸，摸头触发对话，7 条随机回应 |
| **屏幕触摸** (FT6336) | 双击/滑动/长按 6 种手势，60 条动作文本 |
| **体感检测** (BMI270) | 摇晃 + 抱起触发互动；说话时自动屏蔽，5 分钟冷却 |
| **情绪灯环** (WS2812×12) | 21 种情绪颜色映射，PY32 IO Expander 控制 |
| **舵机** (SCS 总线) | GC0308 人脸追踪、空闲扫视、对话时暂停 |
| **休眠** | 30 秒空闲自动关 LED / 停舵机 / 熄屏，唤醒词恢复 |
| **重新配网** | Idle 状态长按屏幕 5 秒进入 192.168.4.1 配网 |
| **文本打断** | 对话中可插文本消息，不被静默丢弃 |
| **MCP 工具** | LED 控制、音量、亮度、摄像头拍照、系统信息 |

## 源码编译

需要 **ESP-IDF v5.5.x**，在"ESP-IDF 5.5 CMD"中执行：

```bat
git clone https://github.com/howecheung/Stackchan-XiaoZhi.git
cd Stackchan-XiaoZhi

:: 首次编译（每个新克隆只跑一次）
idf.py set-target esp32s3
idf.py fullclean
idf.py build flash

:: 后续增量编译
idf.py build flash
```

低配机器防 OOM：`idf.py build flash -- -j1`

## 配置

修改 `sdkconfig.defaults`：

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `CONFIG_CUSTOM_WAKE_WORD` | `ni hao xiao zhi` | 唤醒词拼音 |
| `CONFIG_CUSTOM_WAKE_WORD_DISPLAY` | `你好小智` | 唤醒词中文显示 |
| `CONFIG_WAKE_WORD_SENSITIVITY` | `MEDIUM` | 灵敏度：`LOW` / `MEDIUM` / `HIGH` |

## 已知问题

1. **电池供电 + 灯全亮可能无法开机** —— 插 USB 再开机
2. **LLM 控灯偶尔卡住 / 口头说调灯实际不动** —— prompt 优化中
3. **烧录后需物理重启** —— 拔 USB 关机，等 30 秒再插电
4. **唤醒词灵敏度需自行调整** —— 生僻词选 High，常见词选 Medium
5. **自部署服务端注意安全**：auth.enabled=true、MAC 白名单、API Key 不暴露、端口不裸奔

## 更新日志

### 2026-06-11
- 修复休眠机制：恢复空闲检查，对话中不再误休眠；超时 60s → 30s
- Idle 长按屏幕 5 秒进入配网模式
- 说话时屏蔽 BMI270 体感，避免舵机晃动误触 IMU

### 2026-06-10
- 编译修复 + 版本检查重试优化（MAX_RETRY=1）+ 默认 WebSocket 协议
- 触摸/SendUserText 长文本拦截修复（≤24 字节）
- 屏蔽 OTA 远程升级，仅支持 `idf.py flash` 手动升级
- 体感池拆 display+tag、删早安问候、删 upgrade_firmware MCP 工具

### 2026-06-06
- 唤醒词灵敏度 Low/Medium/High 三档
- 文本打断优化 + 休眠背光彻底熄灭 + SPIRAM 改回 QUAD

## 自定义

编辑 `main/boards/m5stack-core-s3/m5stack_core_s3.cc`：

- 触摸回应文本：搜索 `HeadTouchPool`
- 手势动作池：搜索 `DoubleClickPool`、`UpSwipePool`
- 体感动作：搜索 `ShakePool`、`LiftPool`（MotionMsg display+tag 格式）
- 情绪灯颜色：搜索 `UpdateLedsFromEmotion`

## 致谢

- [xiaozhi-esp32](https://github.com/78/xiaozhi-esp32) — 基础固件
- [xiaozhi-esp32-server](https://github.com/xinnan-tech/xiaozhi-esp32-server) — 服务端
- [M5Stack StackChan-BSP](https://github.com/m5stack/StackChan-BSP) — SI12T 参考

## License

同上游 [xiaozhi-esp32](https://github.com/78/xiaozhi-esp32)。
