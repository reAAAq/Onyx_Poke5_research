# ONYX BOOX Poke5 Android 刷新策略分析

分析日期：2026-07-14  
设备固件：Android 11，`2024-06-05_12-04_3.5.1_19ed122a7`

## 结论

Poke5 的刷新策略是一个分层状态机，不是简单地把 Android 每一帧都送到墨水屏：

1. SurfaceFlinger 以 40 Hz 合成，但 EPDC 只处理实际 damage 或被 Onyx 强制指定的刷新。40 Hz 是合成节拍，不是墨水屏固定刷新率。
2. 普通静态界面使用自动波形：内部模式 `5`，SDM 日志表现为 `waveform_mode=255, update_mode=0, flags=0`。刷新区域由 Surface damage 决定，可为局部，也可因图层变化扩大到全屏。
3. 当前设备的全局滚动策略是 `A2`：`scrollingRefreshMode=2`。触摸滚动开始时进入瞬态 `A2 quality`，内部编码 `2308/0x904`；实测底层为 `waveform=4, flags=0x2000`。
4. 当前设置为 `gcAfterScrolling=true`。滚动停止约 1 秒后，系统退出 A2 瞬态模式并做一次全屏 GC16：实测 `waveform=2, update_mode=1, flags=0x1000`，用于清除滚动残影。
5. `sys.onyx.idledelay=5000` 不是滚动结束后的回刷延迟。它属于 OnyxPowerManager，表示最后一次用户活动约 5 秒后把系统电源模式从 run 切到 Qualcomm `freeze` idle。滚动回刷使用另一个参数 `Settings.Global.scroll_refresh_delay`，当前未设置，源码默认值为 1000 ms。
6. 应用刷新策略由 `oec_service` 统一管理。每个应用有 EAC 配置，核心参数包括 `updateMode`、`turbo`、`gcInterval`、`animationDuration`、`antiFlicker`、`useGCForNewSurface` 和显示增强参数。
7. 当前设备默认应用模式为 Normal，`gcInterval=20`，即按 20 次操作做周期性全刷；滚动结束全刷是另一条独立策略，不受这个计数替代。

## 当前运行时配置

通过 `IOECService` Binder 读取到：

| 参数 | 当前值 | 含义 |
|---|---:|---|
| EAC 总开关 | `true` | OEC/EAC 策略启用 |
| 应用作用域刷新模式 | `0` | 当前前台为 Normal |
| 滚动刷新模式 | `2` | 滚动期间使用 A2 |
| 滚动结束全刷 | `true` | 退出滚动模式时执行 GC |
| 全刷频率 | `20` | 按操作次数累计，每 20 次全刷 |
| 动画过滤 | `20 ms` | 只对 Normal 模式有效 |
| Turbo | `0` | A2 加速等级关闭 |
| Anti-flicker | `0` | 抗闪烁附加策略关闭 |
| 全局对比度 | `30` | 默认显示增强参数 |
| Mono level | `10` | 灰阶/单色处理参数 |
| Dither threshold | `128` | 默认抖动阈值 |
| Enhance | `true` | 显示增强启用 |
| BW mode | `0` | 非强制黑白模式 |
| 新 Surface 强制 GC | `false` | 新图层不自动执行 GC |
| EAC 触摸事件延迟 | `1500 ms` | 无障碍/触摸兼容策略参数 |

设备默认应用刷新配置如下：

```text
updateMode=0
turbo=0
gcInterval=20
animationDuration=20
animationType=null
useGCForNewSurface=false
supportRegal=false
antiFlicker=0
```

## 用户模式与内部编码

OEC 对外使用以下应用模式编号：

| UI/配置模式 | OEC 值 | 核心策略 |
|---|---:|---|
| Normal | `0` | 自动波形，优先显示质量，适合文字阅读 |
| DU | `1` | 黑白快速刷新，轻微残影和锯齿，适合快速翻页 |
| A2 | `2` | 动画快速刷新，可叠加 Turbo 0-6，适合滚动 |
| Regal | `3` | Regal 波形，针对局部刷新残影补偿 |
| X | `4` | 极快动画模式，细节损失和残影最大，适合网页或视频 |

当前设备默认配置的 `supportRegal=false`，Regal 许可白名单只有 `com.tencent.weread.eink`，所以该模式并非对所有应用开放。

Poke5 的 SDM SDK 把 `UpdateMode` 映射为传给 SurfaceFlinger 的位编码：

| SDK UpdateMode | 内部值 | 编码解释 |
|---|---:|---|
| None/default | `5 / 0x5` | AUTO |
| DU / GU_FAST | `1 / 0x1` | DU |
| DU_QUALITY | `2305 / 0x901` | DU + dither + Y1 |
| DU4 | `2312 / 0x908` | DU4 + dither + Y1 |
| GU | `2 / 0x2` | GU/GC16 waveform 编号 2 的局部形式 |
| GC | `98 / 0x62` | GC16 + FULL + WAIT |
| GCC | `107 / 0x6b` | GCC16 + FULL + WAIT |
| DEEP_GC | `108 / 0x6c` | DEEP_GC16 + FULL + WAIT |
| ANIMATION | `4 / 0x4` | 动画波形 |
| ANIMATION_QUALITY | `2308 / 0x904` | 动画 + dither + Y1，当前滚动使用 |
| ANIMATION_MONO | `33554436 / 0x02000004` | 动画 + mono |
| ANIMATION_X | `16777220 / 0x01000004` | 动画 + X dither |
| GC4 | `3 / 0x3` | GC4 |
| REGAL | `6 / 0x6` | Regal |
| REGAL_D | `4102 / 0x1006` | Regal-D |
| HAND_WRITING_REPAINT | `524290 / 0x80002` | 手写 GU repaint |

注意：Java 层的 waveform 编号与 composer 最终日志不总是一一原样输出。比如普通 AUTO 内部值是 `5`，到 SDM 日志会显示 `waveform_mode=255`；composer 会再次解析位标志并生成最终 EPDC update。

## 滚动切换时序

在 Android 设置页执行一次 500 ms 上滑，得到以下时序：

```text
16:39:10.344  input swipe 开始
16:39:10.397  SWEpdcManager clearScreen(type=0)
16:39:10.397  setForceWaveform(mode=2308 / 0x904)
16:39:10.426  SDM waveform=4 update_mode=0 full-screen flags=0x2000
      ...     滚动期间按约 40 Hz 合成并提交 A2 全屏更新
16:39:11.705  最后一批 A2 更新
16:39:12.000  clearScreen(type=101), setForceWaveform(-1)
16:39:12.031  SDM waveform=2 update_mode=1 full-screen flags=0x1000
16:39:15.770  PowerManager 进入 idle/freeze
```

对应源码逻辑：

```text
滚动开始
  -> ScrollHelper.enterScrollRefreshModeImpl()
  -> getScrollingRefreshMode() == 2
  -> EACUtils.toEpdMode(2) == 2308
  -> ViewUpdateHelper.applyTransientUpdate(2308)

滚动停止
  -> 延迟 Settings.Global.scroll_refresh_delay
  -> 当前值不存在，因此采用 1000 ms
  -> clearTransientUpdate(gcAfterScrolling=true)
  -> 全屏 GC16
```

这里的 A2 更新覆盖全屏，是因为滚动时整个内容图层持续移动，damage 很快扩展为全屏；普通按钮、状态栏等小变化仍可提交局部矩形。

## 5 秒 Idle 的真实作用

`sys.onyx.idledelay=5000` 由 `android.onyx.pm.OnyxPowerManager` 读取：

```text
IDLE_DELAY_PROPERTY = "sys.onyx.idledelay"
idleDelayMS = SystemProperties.getInt(..., 3000)
```

Poke5 当前覆盖为 5000 ms。最后一次用户活动后，PowerManagerService 调用：

```text
switchToIdleState()
  -> setupStandbyAlarm()
  -> 写 Qualcomm kernel power mode "freeze"
  -> currentPowerMode = idle
```

下一次触摸时再切回 `off`，这里的 `off` 是 Onyx 对 Qualcomm run 状态使用的字符串，不是关机。

因此这 5 秒主要服务于功耗控制和 wakelock 策略。它可能间接影响合成活跃度，但不是 EPD 的“停止滚动后延时全刷”。实测全刷发生在停止滚动约 1 秒后，而进入 idle 发生在触摸后约 5 秒。

## 分层调用链

```text
App / View / WebView
  -> framework EInkHelper
  -> oec_service (IOECService)
  -> EAC 应用配置和设备默认配置
  -> ViewUpdateHelper
  -> SurfaceFlinger 私有 Binder transaction
  -> SWEpdcManager / Onyx SurfaceFlinger 扩展
  -> vendor.qti.hardware.display.composer-service
  -> onyx_epdc_update_to_display
  -> CommitEpdc
  -> kernel onyx_epdc_fb
  -> waveform/LUT
  -> E-Ink panel
```

策略信息还通过 `/dev/onyx/epdc-listener` 传递，消息类型包括：

- EPDC update
- Activity resume/pause/finish/top-resumed
- Input event
- Native Surface update
- 分屏和屏幕笔记状态

因此 OEC 能根据当前应用、Activity、View 类型、输入状态和滚动状态动态切换模式，而不是只读取一个全局刷新值。

## 普通刷新与强制刷新

普通界面更新：

```text
waveform_mode=255
update_mode=0
flags=0
Rect=<Surface damage>
```

特点：

- 波形由底层自动选择。
- 默认是 partial update。
- 区域来自图层 damage，可局部，也可全屏。
- 周期性 GC 由 `gcInterval` 计数补偿。

手动全刷或滚动结束清残影：

```text
waveform_mode=2
update_mode=1
flags=0x1000
Rect[0 0 1448 1072]
```

特点：

- GC16 全屏刷新。
- `update_mode=1` 对应 full update。
- 显式覆盖自动波形。

滚动 A2：

```text
waveform_mode=4
update_mode=0
flags=0x2000
Rect[0 0 1448 1072]
```

特点：

- 动画波形，优先响应速度。
- 使用 dither/Y1 质量版本，而不是最原始的 `0x4`。
- 停止后由 GC16 恢复显示质量。

## 推理依据

本报告同时使用了四类证据：

1. 运行时 Binder 数据：直接调用 `oec_service` 的 getter，读取当前配置和完整 JSON。
2. 受控动态日志：设置页滚动前清空 logcat，执行一次固定时长 swipe，继续观察 7 秒。
3. framework 反编译：`framework.jar` 中的 `EInkHelper`、`ScrollHelper`、`EACUtils`、`ViewUpdateHelper`、`OnyxPowerManager`。
4. 系统 APK 反编译：`kcb-release.apk` 中的 `SDMDevice`、`EACManagerV2`、刷新模式 UI 和中文资源。

动态验证过的映射包括 Normal/AUTO、滚动 A2 quality 和退出滚动后的 GC16。其余 SDK 模式编码来自同版本固件的静态反编译，尚未逐个在屏幕上强制切换验证，避免修改用户当前应用配置。
