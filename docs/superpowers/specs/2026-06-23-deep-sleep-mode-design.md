# 深度睡眠模式改动方案

> **日期**: 2026-06-23
> **目标板**: M5Stack Core S3 (StackChan-XiaoZhi)
> **状态**: 已批准, 待实现
> **改动范围**: 1 个文件 (`main/boards/m5stack-core-s3/m5stack_core_s3.cc`)

---

## 1. 需求

在现有 30 秒空闲休眠基础上, 增加**深度睡眠模式**:

- **进入条件**: 设备 Idle 状态连续 30 秒无交互 (沿用现有 `PowerSaveTimer` 触发条件)
- **睡眠时关闭**: 全部 LED 灯环、唤醒词检测、舵机空闲扫视、屏幕背光
- **唤醒方式**: **仅点击屏幕** (任意位置单击), 唤醒词不再可用
- **唤醒后**: 进入聆听状态, 可直接对话
- **硬约束**: 不影响其他任何功能; 睡眠/唤醒都要流畅

### 关键决策 (经澄清确认)

| 决策点 | 选择 | 理由 |
|---|---|---|
| 睡眠期 WiFi/服务器连接 | **全保持** | ESP32 屏幕休眠不会断 WiFi, 保持连接是"流畅唤醒"的前提 |
| 唤醒时 OTA 版本检查 | **不跑** | OTA 仅开机激活时跑一次, 唤醒只做聆听, 避免每次 1~3 秒 HTTP 延迟 |
| 唤醒手势 | **任意单击** | 最直观, 复用现有短按检测 |
| 唤醒词检测 | **睡眠时关闭** | 符合"关闭唤醒词"要求, 省 CPU/内存; 唤醒后恢复 |
| 防误触 | **需要** | 进入睡眠那一刻手还在屏上不应唤醒 |

---

## 2. 现状分析 (源码事实)

### 2.1 现有睡眠机制

`main/boards/common/power_save_timer.{h,cc}`:
- `PowerSaveTimer(cpu_max_freq, seconds_to_sleep=30, seconds_to_shutdown=-1)`
- 每秒 tick, 连续 `seconds_to_sleep` 秒满足 `Application::CanEnterSleepMode()` → 触发 `OnEnterSleepMode`
- `CanEnterSleepMode()` = `kDeviceStateIdle` && 无音频通道 && `audio_service_.IsIdle()`

`m5stack_core_s3.cc:1735` `InitializePowerSaveTimer()`:
- 构造时 `cpu_max_freq_ = -1`
- **关键**: `power_save_timer.cc` 里关唤醒词的代码被 `if (cpu_max_freq_ != -1)` 包住 → **当前唤醒词检测 24 小时常开**, 现有 `OnEnterSleepMode` 回调只关背光/LED/舵机, 不关唤醒词
- `OnExitSleepMode` 回调恢复背光/LED/舵机

### 2.2 现有触摸手势识别

`m5stack_core_s3.cc:1945` `PollTouchpad()`:
- 20ms 周期定时器调用
- 用一组 `static` 局部变量 (`was_touched`, `pending_single_release`, `touch_start_time` 等) 跟踪按下/抬起/滑动/双击/长按
- 短按 → `ToggleChatState()`; 长按 5 秒 (Idle) → 配网; 滑动/双击 → 手势消息池

### 2.3 现有聆听入口

`Application::ToggleChatState()` (事件驱动, 线程安全):
- `Idle` 状态下 → `HandleToggleChatEvent` → `ContinueOpenAudioChannel(GetDefaultListeningMode())`
- 内部 `SetPowerSaveLevel(PERFORMANCE)` 切性能模式降延迟
- 因 WiFi 保持, `OpenAudioChannel` 复用已建立的 WebSocket, 通常 <500ms

### 2.4 现有外设控制接口 (均已存在, 无需新增)

| 接口 | 位置 | 作用 |
|---|---|---|
| `Display::SetPowerSaveMode(bool)` | `display.h:42` | 停/启 LVGL 刷新 |
| `Backlight::RestoreBrightness()` | `backlight.h:15` | 恢复背光(有渐变) |
| `Backlight::SetBrightness(0)` | `backlight.h:16` | 熄灭背光 |
| `AudioService::EnableWakeWordDetection(bool)` | `audio_service.h:123` | 开关唤醒词 |
| `AudioService::IsWakeWordRunning()` | `audio_service.h:119` | 查询唤醒词是否在跑 |
| `StackChanServo::PauseScan()/ResumeScan()` | board 内 | 停/恢复舵机空闲扫视 |
| `Py32SetLedFrame(colors, 12)` | board 内 | 设置 12 颗 LED 颜色 |

---

## 3. 方案选型

### 方案 A (采用): board 内 `in_deep_sleep_` 标志 + PowerSaveTimer 回调扩展

- 不新增 DeviceState, 在 board 加 `volatile bool in_deep_sleep_` 标志
- 扩展现有 `OnEnterSleepMode`/`OnExitSleepMode` 回调: 加关/恢复唤醒词 + 置/清标志
- `PollTouchpad` 开头加短路守卫: 睡眠态只识别"抬起唤醒", 跳过所有手势识别
- 唤醒调 `power_save_timer_->WakeUp()` + `Application::ToggleChatState()`

**优点**: 改动最小 (1 文件), 零改动状态机/application.cc/其他 board, 完全满足"不影响其他功能"
**缺点**: 深度睡眠态不是 DeviceState (对本需求足够)

### 方案 B (否决): 新增 `kDeviceStateSleep` 状态

要改 `device_state_machine.cc` 转换表 (影响所有 board) + 所有 `switch(state)` 分支, 风险面大, 违背硬约束。

### 方案 C (否决): 独立 `DeepSleepManager` 类

与 `PowerSaveTimer` 职责重叠, 两套定时器打架。

---

## 4. 详细设计

### 4.1 睡眠态定义

board 私有成员 (不进 DeviceState, 设备对外仍是 `kDeviceStateIdle`):

```cpp
volatile bool in_deep_sleep_ = false;    // 是否处于深度睡眠
int64_t deep_sleep_enter_us_ = 0;        // 进入睡眠时刻(用于触摸避让)
```

### 4.2 进入睡眠回调 (扩展现有 `OnEnterSleepMode`)

```
现有(保留):
  display->SetPowerSaveMode(true)
  backlight->SetBrightness(0)
  servo_.PauseScan()
  Py32SetLedFrame(off, 12)
新增:
  if audio_service.IsWakeWordRunning():
      audio_service.EnableWakeWordDetection(false)
  in_deep_sleep_ = true
  deep_sleep_enter_us_ = esp_timer_get_time()
```

### 4.3 退出睡眠回调 (扩展现有 `OnExitSleepMode`, 顺序调整)

```
新增(最先):
  in_deep_sleep_ = false
现有(顺序明确化):
  display->SetPowerSaveMode(false)   // 先恢复显示驱动
  backlight->RestoreBrightness()     // 再点亮背光(避免花屏)
  servo_.ResumeScan()
  Py32SetLedFrame(neutral, 12)
新增(最后):
  audio_service.EnableWakeWordDetection(true)
```

**顺序论证**:
- 先清 `in_deep_sleep_`: 恢复过程中任何触摸走正常分支, 不进睡眠守卫
- 先恢复显示驱动再亮背光: 避免花屏一瞬间
- 唤醒词恢复放最后: 它最不敏感, 且唤醒靠点击不靠它

### 4.4 唤醒手势识别与防误触 (`PollTouchpad` 开头短路守卫)

新增独立 static 变量 (不污染现有手势状态):
```cpp
static bool sleep_touch_armed = false;     // 等"下一次新按下"
static bool sleep_press_seen = false;      // 已捕获睡眠期按下, 等抬起
static int64_t sleep_press_down_us = 0;
```

逻辑 (插在 `ft6336_->UpdateTouchPoint()` 之后, 现有手势逻辑之前):

```
if in_deep_sleep_:
    now_us = esp_timer_get_time()
    if num > 0 and not armed:
        if (now_us - deep_sleep_enter_us_) < 800000:   # 800ms 避让
            armed = true; press_seen = false           # 残留触摸, 丢弃
        else:
            press_seen = true; press_down_us = now_us  # 唤醒候选
        return
    if num > 0 and armed:
        press_seen = true; press_down_us = now_us      # 新按下
        return
    if num == 0 and press_seen:
        press_seen = false; armed = false              # 抬起 → 唤醒
        WakeUpFromDeepSleep()
        return
    return
```

**三道防线防误触**:
1. **时间避让** (800ms): 进睡眠后 800ms 内的按下视为"睡着时手还在屏上", 丢弃并 arm
2. **必须抬起才唤醒**: 按下后 num 不归 0 不唤醒 (防物体长期压住)
3. **armed 锁**: 残留触摸进 armed 后, 必须 num 先归 0 再重新按下才认 (防长按压住持续触发)

**边界情况**:

| 情况 | 行为 |
|---|---|
| 进睡眠时无手在屏 | armed=false, 下次按下→抬起即唤醒 |
| 进睡眠时手按着没松 | 800ms 内丢弃→armed, 等松手再按 |
| 睡着后物体长期压住 | num 一直 >0, press_seen 一直 true 但永不抬起, **不唤醒** |
| 睡着后单次轻点 | 按下→抬起, 立即唤醒 |
| 唤醒瞬间还在拖动 | 已恢复正常手势识别, 按现有滑动逻辑处理 |

**为何用独立 static 变量**: 现有 `was_touched` 跨睡眠/唤醒语义混乱; 独立变量完全隔离睡眠态触摸状态, 不污染现有手势状态机。

### 4.5 唤醒动作 (`WakeUpFromDeepSleep` 新增私有方法)

```cpp
void WakeUpFromDeepSleep() {
    power_save_timer_->WakeUp();                  // 触发 OnExitSleepMode 恢复外设 + 复位 tick
    Application::GetInstance().ToggleChatState(); // 进入聆听(事件驱动, 线程安全)
}
```

**顺序论证**:
- `WakeUp()` 内部调 `on_exit_sleep_mode_` (恢复外设 + 清 `in_deep_sleep_`) 并复位 tick → "恢复外设"和"重置 30 秒计时"原子
- `ToggleChatState` 用 `GetDefaultListeningMode()` (自动停止), 内部切 PERFORMANCE 降延迟; 走已建立 WebSocket, <1 秒进聆听

### 4.6 唤醒后时序 (用户体感)

| 时刻 | 现象 |
|---|---|
| t=0 | 手指抬起 |
| t=0~50ms | 背光渐亮、舵机恢复扫视、LED 亮起 |
| t=50~200ms | 屏幕显示 avatar、状态栏"连接中" |
| t=200ms~1s | 打开音频通道 (WebSocket 复用, 通常 <500ms) |
| t≈1s | 进入聆听, avatar 嘴部动效, 可说话 |

**流畅性来源**: §1 决策"WiFi+连接全保持", `OpenAudioChannel` 无重连开销。

### 4.7 对话结束后 (无需额外代码)

对话走完正常流程回 `kDeviceStateIdle` → `PowerSaveTimer` 重新 30 秒计时 → 再次深度睡眠。完全复用现有周期。

### 4.8 失败兜底

- `ToggleChatState` 协议未就绪 (`protocol_==null`): 现有 `HandleToggleChatEvent` `ESP_LOGE` 并 return, 设备停 Idle, 30 秒后再睡。不卡死。
- 唤醒时 WiFi 恰断: 现有 `HandleNetworkDisconnectedEvent` 关音频通道, `ToggleChatState` 安全返回。不崩溃。

---

## 5. 改动文件清单

| 文件 | 改动 | 行数 |
|---|---|---|
| `main/boards/m5stack-core-s3/m5stack_core_s3.cc` | 修改 (核心) | +约 60 行 |
| `main/boards/common/power_save_timer.{h,cc}` | **零改动** | 0 |
| `main/application.{h,cc}` | **零改动** | 0 |
| `main/device_state*.{h,cc}` | **零改动** | 0 |
| 其他 board | **零改动** | 0 |
| `sdkconfig.defaults` | **零改动** | 0 |

**发现**: `PowerSaveTimer` 的回调机制完全够用, 关唤醒词/置标志/恢复全在 board 回调里完成, `PowerSaveTimer` 一行不用改, 把改动面缩到 1 文件。

---

## 6. 逐改动点说明 (代码级)

### 改动 1.1 — 新增私有成员

`M5StackCoreS3Board` private 区 (`servo_ok_` / `low_batt_warned_` 附近):
```cpp
volatile bool in_deep_sleep_ = false;
int64_t deep_sleep_enter_us_ = 0;
```

### 改动 1.2 — 扩展 `InitializePowerSaveTimer()`

`OnEnterSleepMode` 末尾加关唤醒词 + 置标志;
`OnExitSleepMode` 开头加清标志, 末尾加恢复唤醒词, 中间顺序加注释。

### 改动 1.3 — `PollTouchpad()` 开头插短路守卫

`ft6336_->UpdateTouchPoint()` 之后, 现有 `pending_single_release` 双击窗口判定之前, 插入睡眠态触摸分支。

### 改动 1.4 — 新增 `WakeUpFromDeepSleep()` 私有方法

放在 `PollTouchpad` 上方。

---

## 7. 不影响其他功能 — 逐项论证

| 顾虑 | 论证 |
|---|---|
| 状态机 / 其他 board | 0 改动, `in_deep_sleep_` 是 board 内 bool, DeviceState 仍 Idle |
| 现有手势识别 | 睡眠守卫用独立 static 变量, 不碰 `was_touched` 等; 唤醒后原样运行 |
| 现有 `OnEnterSleepMode` | 关背光/LED/舵机逻辑全保留, 仅新增关唤醒词+置标志 |
| 现有 `OnExitSleepMode` | 恢复逻辑全保留, 顺序微调+注释, 新增唤醒词恢复 |
| 唤醒词配置 | 不碰 sdkconfig, 唤醒词仅睡眠态关, 唤醒后照常 |
| 体感 BMI270 / SI12T | 不碰; 睡眠态触发走 WakeWordInvoke (bonus: 体感也能唤醒) |
| 配网 (Idle 长按 5 秒) | 不碰; 睡眠态长_press 走 armed 逻辑不误触配网 |
| 低电量告警 | 不碰, `batt_timer` 照常 |
| OTA / 激活 | 0 改动 |

---

## 8. 验证清单

1. `idf.py build` 通过 (无新依赖/头文件)
2. 空闲 30 秒 → 屏/LED/舵机/唤醒词全关
3. 睡眠态单击屏 → 1 秒内进入聆听可对话
4. 睡着时手按着不松 → 不唤醒
5. 唤醒后正常对话、手势、体感、配网全部正常
6. 对话结束 30 秒后再睡, 循环正常
7. 进睡眠那一刻手在屏上 → 不立即误唤醒 (等松手再按)
