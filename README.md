# ONYX BOOX Poke5 ABL 启动模式逆向分析

## 一、结论

这份 Poke5 ABL 中可以确认的启动模式包括：正常启动、Recovery、ABL
Fastboot（bootloader）、Fastbootd、Qualcomm EDL、关机充电、闹钟启动、
FFBM 和 QMMI。

Poke5 最有价值的隐藏入口，是一套厂商新增的纯电源键 Recovery 手势：

1. 关机状态按住电源键开机。
2. ABL 开始识别电源键后，继续保持按下至少 3 秒，但必须在 10 秒前松开。
3. 松开后，再快速完成 5 次“按下并松开”电源键。
4. 每次等待按下和等待松开的时间都不能超过 2 秒。

成功后 ABL 直接设置 Recovery 标志。需要注意，3 秒和 10 秒是从 ABL
开始扫描按键后计算，不是从手指第一次按下电源键开始计算。因此实际操作时，
总按住时间会比 3 秒稍长。

### 各模式的实际进入方法

| 模式 | 推荐进入方法 | ABL 中确认的其他入口 |
|---|---|---|
| Recovery | `adb reboot recovery` | 电源键隐藏手势；启动时按住隐藏 VOL+ 输入；USB/充电上电时按住 VOL- 10 秒；reset reason `1`；`misc` 中的 `boot-recovery` |
| ABL Fastboot | `adb reboot bootloader` | 冷启动且无 USB/充电器时，持续按住 VOL- 10 秒；Power+VOL- 也满足条件；reset reason `2`；无可启动 slot 时自动进入 |
| Fastbootd | `adb reboot fastboot` | ABL Fastboot 中执行 `fastboot reboot fastboot`；通过 `misc` 的 `boot-fastboot` 走 Recovery/Fastbootd |
| Qualcomm EDL | 继续使用 EDL 线、测试点或现有进 9008 的方法最稳妥 | VOL+ 与 VOL- 同时按下会立即 dload；USB/充电上电时按住 VOL- 至少 5 秒、不到 10 秒时松开也会 dload |
| 关机充电 | 关机后插入充电器 | ABL 设置 `androidboot.mode=charger` |
| 闹钟启动 | 系统设置 RTC alarm 后关机 | reset reason `3`，ABL 设置 `androidboot.alarmboot=true` |
| FFBM | Bootloader 菜单或写入对应 `misc` cookie | 支持 `ffbm-00`、`ffbm-02` 等 cookie |
| QMMI | Bootloader 菜单 | 菜单中存在 `Boot to QMMI` 项 |

### 物理按键路径的精确判定

ABL 每轮等待固定为 100 ms，所以以下阈值是从机器码中精确还原的：

- 30 轮 = 3 秒
- 50 轮 = 5 秒
- 100 轮 = 10 秒
- 20 轮 = 2 秒

对于 VOL- 单独按下产生的 `0x02`，或者 Power+VOL- 产生的 `0x08`，ABL
根据上电来源采用两套逻辑。

正常冷启动，且没有充电器、没有 USB PON 原因：

- 连续按住满 10 秒：进入 ABL Fastboot。
- 不到 10 秒松开：继续正常启动。

由 USB/充电器触发的上电路径：

- 不到 5 秒松开：继续正常启动或关机充电。
- 按住至少 5 秒、不到 10 秒时松开：触发 dload，重启进入 EDL。
- 连续按住满 10 秒：进入 Recovery。

按键码映射方面：

| 按键码 | ABL 中的行为 | 判断 |
|---|---|---|
| `0x01` | 菜单向上；启动时直接请求 Recovery | VOL+ 单独按下 |
| `0x02` | 菜单向下；参与 Fastboot/Recovery/EDL 长按检测 | VOL- 单独按下 |
| `0x05` | ABL 启动模式入口未处理 | Power + VOL+ |
| `0x08` | 启动模式检测中等同于 `0x02` | Power + VOL- |
| `0x17` | 立即触发 dload | VOL+ + VOL- |
| `0x102` | 菜单确认；启动时进入电源键手势检测 | Power |

Poke5 外壳上没有常规音量键，但 XBL 的 `ButtonsDxe` 证明主板仍保留两路音量
输入：VOL+ 使用 TLMM GPIO 96，VOL- 使用 PMIC PON 第二路实时按键状态。
因此它们更可能对应未外露的测试点或板级输入，而不是 USB 键盘或软件虚拟键。
不拆机时，最可操作的物理入口仍是纯电源键 Recovery 手势。

### 软件重启入口

ABL 对 reset reason 的处理如下：

| Reset reason | 处理结果 |
|---|---|
| `0` | 正常启动 |
| `1` | Recovery |
| `2` | ABL Fastboot |
| `3` | 闹钟启动 |
| `4`、`5` | 修改 Verified Boot 模式 |
| `6` | 重置 Verified Boot 设备状态 |

ABL Fastboot 明确支持以下命令：

- `reboot-recovery`
- `reboot-fastboot`
- `reboot-bootloader`

镜像中没有发现 `reboot-edl`、`reboot-dload` 或同类 Fastboot 命令。因此不能
假定 `fastboot reboot edl` 在这台设备上可用。

### 启动模式优先级

1. 启动初期先读取 PMIC 上电原因、充电器状态和按键码。
2. 如果物理按键判定为 dload，ABL 立即携带 dload 参数重启，不再继续加载 Android。
3. 随后读取 reset reason，并合并 Recovery、Fastboot、Alarm 等软件请求。
4. 再读取 `misc`，处理 `boot-recovery`、`boot-fastboot` 和 FFBM cookie。
5. Fastboot 标志存在时跳过 Android 镜像加载，直接启动 ABL Fastboot 应用。
6. Recovery 标志存在时选择 Recovery 镜像启动。
7. 没有可启动 slot、boot 分区不可启动，或者 Android 镜像加载失败时，最终也会落入 ABL Fastboot。

## 二、详细逆向过程

### 1. 原始镜像

分析对象：

- `backups/poke5/edl-full-20260710/abl_a.bin`
- `abl_a.bin` 和 `abl_b.bin` 完全一致
- SHA-256：`d7bd84fe92ebdbcc056bbfac471a37ae676d043c161c6bcc70b6572f53222ae5`

原始分区镜像是 1 MiB 的 Qualcomm 签名 ELF32 ARM 容器，不是可以直接导入
反编译器的 AArch64 ABL 主程序。

### 2. 镜像结构拆解

ELF 中只有一个实际代码加载段：

- 文件偏移：`0x3000`
- 大小：`0x22000`
- 加载地址：`0x9fa00000`

该段从 `0x3010` 开始出现 `_FVH`，说明内部是 UEFI Firmware Volume。
Firmware Volume 中只有一个大型 GUID-defined section，其 GUID 为常见的 UEFI
LZMA 解压 GUID。LZMA 数据流起始于原镜像 `0x3078`。

解压后得到大小 `0x720c8` 的数据，内部结构是：

1. 一个 RAW section。
2. 一个 Firmware Volume Image section。
3. 名称为 `LinuxLoader` 的 FFS 文件。
4. 一个 PE32+ AArch64 EFI application。

真正用于逆向的 LinuxLoader：

- 在解压数据中的偏移：`0xb8`
- 大小：`0x72000`
- SHA-256：`cffb15122442a123c51aac2fbc7dfb6bcfba22de212eac089aabca1cfabac01e`

### 3. 关键日志字符串

解压后可以直接找到以下厂商日志：

```text
Pressed down Power key[%d].
BootIntoDload detected.
BootIntoFastboot detected.
GetBootIntoModeRecovery detect begin. Power key Down
GetBootIntoModeRecovery Power key Up
BootIntoMode detect seg[%d] Power key Down
BootIntoMode detect seg[%d] Power key Up
BootIntoRecovery detected.
KeyPress:%u, BootReason:%u
Fastboot=%d, Recovery:%d
```

这些字符串的交叉引用集中在 LinuxLoader 入口和两个按键检测函数中，说明这不是
通用字符串残留，而是当前 Poke5 ABL 启动路径实际调用的代码。

### 4. 总入口函数 `0x13d4`

LinuxLoader 主入口按以下顺序执行：

1. 初始化设备信息并枚举分区。
2. 检测 A/B slot。
3. 读取 PMIC PON reason 和充电器状态。
4. 读取一次 UEFI 扩展键盘输入。
5. 根据初始按键码进入不同物理检测函数。
6. 读取 reset reason。
7. 读取 `misc` 中的 bootloader message。
8. 根据 Fastboot/Recovery 标志选择镜像或启动 Fastboot 应用。

初始按键分支可以直接还原为：

```text
key == 0x01  -> VOL+，Recovery
key == 0x02  -> VOL-，长按模式检测
key == 0x05  -> Power+VOL+，没有专用启动模式
key == 0x08  -> Power+VOL-，长按模式检测
key == 0x17  -> VOL++VOL-，立即 dload
key == 0x102 -> Power，电源键 Recovery 手势检测
```

### 5. 长按模式函数 `0x1c00`

该函数接收 PMIC PON reason 的低字节，同时读取全局充电器状态。

冷启动分支条件为：

```text
charger_present == 0
并且 (pon_reason & 0x10) == 0
并且 pon_reason bit 7 已设置
```

日志中 bit 7 对应 `KPDPWR`，`0x10` 对应 USB 上电原因。因此这个分支表示
由电源键触发、没有 USB/充电器参与的冷启动。它要求 VOL- 单独按下，或者
Power+VOL- 组合连续存在 100 轮，然后设置 Fastboot 标志。

USB/充电分支同样最多检测 100 轮：

- 50 轮前松开：不设置模式。
- 50 到 99 轮之间松开：设置 dload 标志。
- 100 轮始终按下：设置 Recovery 标志。

### 6. 电源键 Recovery 函数 `0x1e10`

该函数首先最多检测 100 轮电源键是否松开。只有在已经按住超过 29 轮后松开，
才进入第二阶段。

第二阶段固定执行 5 组检测：

1. 最多等待 20 轮，直到按键码变为 `0x102`。
2. 最多等待 20 轮，直到按键码变为 `0`。
3. 五组全部完成后设置 Recovery 标志。

因此可严格推导出“按住 3 到 10 秒，松开后再快速按 5 次”的手势。

### 7. 100 ms 时间单位的证据

两个按键检测函数在进入循环前都构造常量：

```text
0x000186a0 = 100000
```

循环中的辅助函数最终跳转到 UEFI Boot Services 表偏移 `0xf8`，该位置是
`Stall()`。参数为微秒，所以每次等待是 `100000 us = 100 ms`。

### 8. ABL 按键读取函数 `0x1dad0`

该函数打开 UEFI Extended Text Input Protocol，读取 `EFI_KEY_DATA`，并把
`EFI_INPUT_KEY.ScanCode` 写入调用者变量。读取成功后还会 reset 输入状态，避免
同一个按键事件被重复消费。

Bootloader 菜单处理函数进一步证明：

- `0x01` 用于向上移动。
- `0x02` 用于向下移动。
- `0x102` 用于确认。

### 9. XBL `ButtonsDxe` 的实体按键映射

为了继续确认 `0x08` 和 `0x17`，又拆解了 `xbl_a.bin` 中的嵌套 UEFI
Firmware Volume，并提取出 GUID 为
`5bd181db-0487-4f1a-ae73-820e165611b3` 的 `ButtonsDxe`。

`ButtonsDxe` 轮询顺序是：

1. Power：调用 PMIC PON `GetPonRtStatus(0, ...)`。
2. VOL+：读取 TLMM GPIO 96，低电平表示按下。
3. VOL-：调用 PMIC PON `GetPonRtStatus(1, ...)`。

按键组合函数 `0x33e4` 给出了明确映射：

```text
VOL+                  -> ScanCode 0x01
VOL-                  -> ScanCode 0x02
Power                 -> ScanCode 0x102
Power + VOL+          -> ScanCode 0x05
Power + VOL-          -> ScanCode 0x08
VOL+ + VOL-           -> ScanCode 0x17
```

因此 `0x08` 已经可以确定是 Power+VOL-，`0x17` 已经可以确定是双音量键，
不再属于未知 OEM 按键码。

### 10. dload 重启函数 `0x107f8`

物理检测设置 dload 标志后，主入口调用该函数并传入 `-1`。函数构造
`RESET_PARAM`，随后调用 UEFI ResetSystem。`-1` 分支还会写入专用 reset data，
用于让下一阶段进入 Qualcomm download/EDL，而不是普通重启。

### 11. `misc` 解析函数 `0x1e760`

该函数读取 bootloader message：

- 命令为 `boot-recovery` 时设置 Recovery。
- 设备支持对应能力时，`boot-fastboot` 也设置 Recovery 启动路径，由 Recovery
  进入 userspace fastbootd。
- recovery 参数中存在 `--flash_sn=` 时走单独的刷机序列号逻辑。
- FFBM cookie 由后续启动代码读取并传入 Android。

### 12. Fastboot 和异常回退

Fastboot 标志存在时，主入口不调用 Android 镜像加载函数，而是直接启动地址
`0x30e34` 的 Fastboot EFI application。

镜像验证和 slot 选择代码中还存在两条明确日志：

```text
No bootable slots found enter fastboot mode
Non Multi-slot: Unbootable entering fastboot mode
```

因此即使没有按键或软件请求，只要 A/B slot 全部不可启动，ABL 也会自动进入
bootloader Fastboot。这是救砖时非常重要的回退路径。

## 三、关键函数索引

| 地址 | 作用 |
|---|---|
| `0x13d4` | LinuxLoader 总入口和模式优先级 |
| `0x196c` | PMIC PON reason、充电器状态读取 |
| `0x1c00` | VOL- 或 Power+VOL- 长按 Fastboot、Recovery、dload 检测 |
| `0x1e10` | Poke5 电源键 Recovery 手势 |
| `0x2064` | reset reason 协议读取 |
| `0x1dad0` | UEFI ScanCode 读取 |
| `0x1e760` | `misc` 中 `boot-recovery`/`boot-fastboot` 解析 |
| `0x107f8` | ResetSystem 和 dload 参数构造 |
| `0x30e34` | ABL Fastboot EFI application |

## 四、仍需真机确认的点

ScanCode 映射已经由 XBL 定死，剩下的不确定性是 Poke5 主板上 VOL+ GPIO 96 和
PMIC VOL-/RESIN 输入具体引到了哪里。设备外壳没有音量键，可能存在以下情况：

1. 主板保留了工厂测试点，可以直接短接模拟 VOL+ 或 VOL-。
2. 某个排线座或未装配按键焊盘保留了这两路信号。
3. 硬件设计未将其中一路实际引出，驱动只是沿用 Qualcomm 通用配置。

下一步若要把隐藏 Fastboot/EDL 入口变成可执行的操作，需要查看 Poke5 主板照片、
原理图或用万用表追 GPIO 96 与 PMIC PON index 1（通常为 RESIN）输入。纯软件
逆向已经无法确定焊盘位置。
