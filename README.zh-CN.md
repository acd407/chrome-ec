# Redrix EC 固件

为 HP Elite Dragonfly Chromebook (redrix) 定制的 Chromium OS Embedded Controller 固件。

## 硬件平台

- **设备**: HP Elite Dragonfly Chromebook (redrix)
- **EC 芯片**: Nuvoton NPCX9M3F
- **EC Flash**: 512KB（内部 flash，独立于主 BIOS 的 32MB SPI flash）

## EC Flash 布局与双固件机制

```
Offset        Size    Content
0x00000      256KB    RO 区域（硬件写保护，出厂锁定）
0x40000      256KB    RW 区域（可自由写入，日常运行）
```

EC 启动流程：

1. NPCX boot ROM 读取 flash 头部，将 RO 镜像从 flash 拷贝到 SRAM
2. EC 从 RO 启动
3. AP 发命令 `reboot_ec RW`，RO 调用 `download_from_flash()` ROM API
4. NPCX 将 RW 镜像从 flash 拷贝到 SRAM 并跳转执行（sysjump）

若 RW 损坏或校验失败，EC 回退到 RO 运行。RO 是出厂安全网，RW 是主力运行固件。

## EC 问题与修复

此分支基于主线 chrome-ec，针对 redrix 机型在非 ChromeOS 系统上的使用做了以下关键修复：

### 键盘背光在 sysjump 后失效

#### 现象

刷写 RW 固件后（`ectool reboot_ec RW` 或系统重启），键盘背光灯不亮：

```
$ sudo ectool pwmsetkblight 100
Keyboard backlight set.
$ sudo ectool pwmgetkblight
Keyboard backlight disabled.
```

`pwmsetkblight` 返回成功但硬件没有响应。

#### 原理

键盘背光灯驱动结构体 `kblight` 是一个静态全局变量（`common/keyboard_backlight.c`）：

```c
static struct kblight_conf kblight;  // BSS 段，初始时 kblight.drv = NULL
```

驱动注册函数 `keyboard_backlight_init()` 绑定在 `HOOK_CHIPSET_STARTUP` 上：

```c
DECLARE_HOOK(HOOK_CHIPSET_STARTUP, keyboard_backlight_init, HOOK_PRIO_DEFAULT);
```

`HOOK_CHIPSET_STARTUP` 只在 chipset 从 S5（关机）→ S4 → S3 转换时触发——这是 AP 冷启动的电源序列。Sysjump 后 chipset 已在 S0（运行态），该 hook 不会再触发。

时序如下：

```
RO 固件（出厂）
  冷启动 → G3→S5→S4→S3
    → HOOK_CHIPSET_STARTUP 触发
    → kblight_register() → kblight.drv 指向 PWM 驱动 ✅

  AP 启动 → ectool reboot_ec RW
    → sysjump 到 RW
───────────────────────────────────────────────────
RW 固件（新刷入）
  sysjump, BSS 清零 → kblight.drv = NULL
  chipset 已在 S0
    → HOOK_CHIPSET_STARTUP 不触发 ❌
    → kblight.drv = NULL 永久

  AP 发 pwmsetkblight 100:
    → kblight_set(100) → 变量设置成功 ✔
    → kblight_enable(1) → deferred call 调度 ✔
    → kblight_enable_deferred 执行:
      → if (!kblight.drv) return;  ← NULL, 静默跳过
    → PWM 硬件从未被操作 ❌
```

`kblight_set()` 和 `kblight_enable()` 仅设置内存变量和调度延迟调用，不做硬件操作。真正的 PWM 操作在 deferred function 中，但它检查 `kblight.drv` 是否为 NULL，为 NULL 则静默返回。因此 `ectool pwmsetkblight 100` 返回成功，但硬件从未被操作。

#### 修复

在 `common/keyboard_backlight.c` 中添加 sysjump 后的重新注册路径。新增函数注册在 `HOOK_INIT` 上（sysjump 时也会触发），通过 `system_jumped_to_this_image()` 判断是否为 sysjump 场景，仅重新设置 `kblight.drv`，不复位 PWM 硬件（硬件寄存器在软跳转中保留原有值）。

- **冷启动**: `HOOK_CHIPSET_STARTUP` → 正常初始化；`HOOK_INIT` 虽也触发但 sysjump 检查不通过，直接返回。
- **Sysjump**: `HOOK_CHIPSET_STARTUP` 不触发；`HOOK_INIT` → 重新注册驱动。

修复提交：`keyboard_backlight: re-register driver after sysjump`

### MKBP Host Event 事件传递（侧边音量按键无效）

#### 现象

侧边按键（音量+/音量-/电源）已在 Linux 输入子系统中注册：

```
$ evtest /dev/input/event7
Input device name: "cros_ec_buttons"
  Event code 114 (KEY_VOLUMEDOWN)
  Event code 115 (KEY_VOLUMEUP)
  Event code 116 (KEY_POWER)
```

但按下按键时 `evtest` 不产生任何事件。`ectool` 可正常通信（`ectool version`, `ectool flashread` 等均正常）。

#### 原理

Chrome EC 有两条通知 AP 的硬件路径：

| 路径 | 机制 | 硬件信号 |
|------|------|---------|
| **GPIO 中断** | EC 拉低 `GPIO_EC_PCH_INT_ODL` | EC→PCH 的专用 GPIO 引脚 |
| **Host event + SCI** | EC 通过 eSPI 虚拟线触发 SCI | eSPI virtual wire (SERIRQ) |

两条路径完全独立，由不同的 AP 硬件模块处理。

GPIO 中断路径在自定义 Linux（非 ChromeOS）上不通，原因是 ACPI GPIO 中断映射配置不完整，AP 的 GPIO 控制器驱动未正确注册该中断。结果是内核的 `cros_ec` 驱动永远不会调用 `EC_CMD_GET_NEXT_EVENT` 来消费 MKBP 事件。EC 启动日志可以确认这一点：

```
[4.097459 already in S0]              ← power_chipset_init 确认 AP 在 S0
[4.151074 mkbp switches: 1]           ← MKBP 事件已产生
[6.152177 MKBP: The AP is failing to respond despite being powered on.]
          ↑ EC 明确知道 AP 处于 S0 运行状态，但 AP 仍然没有消费 MKBP 事件
```

（键盘仍可正常使用，因为键盘走 8042 协议（eSPI PS/2 兼容），不依赖 MKBP 中断。）

#### 修复

不使用 GPIO 中断路径，而是启用 **host event + SCI** 路径作为备选。SCI（System Control Interrupt）通过 eSPI 虚拟线传递，是独立于 GPIO 总线的另一套硬件通路。需要两个改动：

**Fix 1: `common/mkbp_event.c` — S0 下也发送 host event**

原代码只在 AP 处于 suspend 状态时才设置 `EC_HOST_EVENT_MKBP`，因为 ChromeOS 下 S0 时该事件不在 SCI mask 中。修复后始终设置，在 S0 下也尝试通过 SCI 通知 AP。

**Fix 2: `board/redrix/board.c` — 设置 SCI mask**

EC 的 NPCX 芯片会检查 SCI mask 来决定是否真的触发 SCI 脉冲。默认 SCI mask 为 0（BSS 初始化），需要显式加入 `EC_HOST_EVENT_MKBP`。此外 SCI mask 会在每次 S3→S0 转换时被清零，因此还需通过 `HOOK_CHIPSET_RESUME` 在唤醒后重新设置。

修复后的事件通路：

```
物理按键按下
    → EC GPIO 中断 → mkbp_fifo_add()
    → activate_mkbp_with_events()
    → host_set_single_event(EC_HOST_EVENT_MKBP)  ← 新增：S0 下也走此路
    → NPCX eSPI SCI 虚拟线
    → AP PCH 接收 SCI → ACPI SCI handler
    → cros_ec 驱动 → EC_CMD_GET_NEXT_EVENT
    → cros_ec_buttons → input subsystem → evtest ✅
```

修复提交：
- `redrix/board: enable SCI delivery for MKBP host events`
- `mkbp_event: send host event in S0 as fallback notification`

### USB PD AP Mode Entry（USB4 / 雷电自动协商）

#### 问题

`CONFIG_USB_PD_REQUIRE_AP_MODE_ENTRY` 启用时，EC 不主动进入任何 alt mode（DP/TBT/USB4），而是等待 AP 通过 host command（`ectool typeccontrol`）来指定要进入的模式。这是 ChromeOS 的标准行为——AP 端根据用户设置和显示器需求来决定是否进入 DP 或 USB4。

在非 ChromeOS 系统上，没有 AP 端来下发这些 host command，EC 会一直停在 USB3 模式，无法自动协商 USB4/Thunderbolt。

#### 修复

取消该配置项 (`redrix/board: undef CONFIG_USB_PD_REQUIRE_AP_MODE_ENTRY`)。之后 EC 不再等待 AP 指令，会自动进入端口双方都支持的最优模式（USB4 > TBT > DP）。

## 编译

需要 arm-none-eabi 交叉编译工具链。

```bash
make CROSS_COMPILE=arm-none-eabi- BOARD=redrix
```

编译产物：

| 文件 | 大小 | 说明 |
|---|---|---|
| `build/redrix/ec.bin` | 512KB | RO + RW 全量镜像 |
| `build/redrix/RO/ec.RO.flat` | ~252KB | 仅 RO 固件 |
| `build/redrix/RW/ec.RW.bin` | ~243KB | 仅 RW 固件（刷写用这个） |

### 添加自定义版本标记

编辑 `common/version.c` 中的 `build_info` 字符串，加入 `LOCAL_BUILD` 等自定义标记：

```c
const char build_info[] =
    VERSION " " CROS_FWID32 " " DATE " " BUILDER " LOCAL_BUILD";
```

重建后 `ectool version` 的 Build info 行会显示该标记，方便区分自定义固件与官方固件。

## 刷写 EC 固件

### 查看 Flash 信息

```bash
sudo ectool flashinfo
```

典型输出：
```
FlashSize 524288
WriteSize 1
EraseSize 65536
ProtectSize 65536
WriteIdealSize 240
Flags 0x0
```

Flash 总大小 512KB (524288 bytes)，其中 RO 占低 256KB，RW 占高 256KB。我们刷写的是 RW 区域。

### 安全刷写流程（仅刷 RW，不动 RO）

```bash
# 1. 备份当前 EC 全量固件
sudo ectool flashread 0 524288 ec_backup.bin

# 2. 擦除 RW 区域
sudo ectool flasherase 0x40000 262144

# 3. 刷入 RW 固件
sudo ectool flashwrite 0x40000 build/redrix/RW/ec.RW.bin

# 4. 重启 EC 到 RW（或全系统重启）
sudo ectool reboot_ec RW
```

### 验证刷写结果

```bash
# 回读验证
sudo ectool flashread 0x40000 262144 rw_readback.bin
sha256sum build/redrix/RW/ec.RW.bin rw_readback.bin

# 确认固件已运行
sudo ectool version
# Firmware copy: RW    ← 关键行
# Build info: ... LOCAL_BUILD  ← 自定义标记
```

## 禁止 EC 使用 BIOS 里面的 ECRW 固件

如果使用 [mrchromebox.tech](https://mrchromebox.tech/) 的 coreboot/edk2 固件替换了原厂 Chrome OS 固件（大家应该都是这样的），需要在 BIOS 中关闭 EC Software Sync，防止启动时 AP 固件自动覆盖 EC RW 分区。

### 操作步骤

1. 重启设备，按 `ESC` 进入 BIOS 设置界面
2. 找到 **EC Software Sync** 选项，关闭（Disable）它
3. 保存并退出

![BIOS EC Software Sync 设置](docs/images/BIOS_EC_Software_Sync.jpg)

如果不关闭此选项，每次开机时 AP 固件会将其内嵌的 EC RW 镜像写入 EC，覆盖你刷写的自定义固件。

## 常见问题

### 刷完后 EC 回退到 RO

- 尝试全系统重启（`sudo reboot`），让 EC 经历完整电源序列
- 确保 RW 区域已先擦除再写入（`flasherase` + `flashwrite`）
- 若在非 ChromeOS SDK 环境中编译，fwid 显示 `CROS_FWID_MISSING` 是正常现象

### RO 区域保护

RO 区域有硬件写保护（由主板 GPIO 电平控制）。即使 flashwrite 从 offset 0 开始写，RO 区域也会被硬件拒绝，不会真正损坏。通常不需要修改 RO。

## 版本字符串

`ectool version` 输出示例：

```
RO version:    redrix_v2.0.26378-d5dba7885d
RO cros fwid:  redrix_14505.831.0
RW version:    redrix_v2.0.27860-0f0590ceec
RW cros fwid:  redrix_16238.2+tbt5
Firmware copy: RW
Build info:    ... DATE BUILDER [LOCAL_BUILD]
```

版本号由 `util/getversion.sh` 生成：`git describe` 找到最近的 tag，commit 数为从 tag 到 HEAD 的提交数，有未提交修改时追加 `+`（dirty 标记）。
