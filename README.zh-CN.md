# Primus EC 固件

为 Google Primus（Chromebook，brya baseboard）定制的 Chromium OS Embedded
Controller 固件，目标是让它在**非 ChromeOS 系统**（自编 Linux）上正常工作。

## 硬件平台

- **设备**: Google Primus（brya baseboard，Alder Lake-P）
- **EC 芯片**: Nuvoton NPCX993F
- **USB-PD**: RT1715 TCPC（`CONFIG_USB_PD_TCPM_RT1715`）
- **Retimer**: Burnside Bridge（bb_retimer）

## 分支说明

`primus-rw` 基于 redrix 上游的 `0654d51ba4`。选这个基点是因为它**不含**
破坏性 commit `393bea15b8`（"brya: Enable CONFIG_SVDM_RSP_DFP_ONLY"）——
那个 commit 删掉了 brya baseboard 的整套 TBT/USB4 SVDM responder
（`baseboard/brya/usb_pd_policy.c` 减少 186 行），会让对端认为本机
"无 UFS modal / 无 USB4 / 无 Intel SVID"。

本分支相对基点有 5 个 commit：

| Commit | 说明 |
|---|---|
| `e1f9578dc4` | **primus 专属**：启用 EC 自主 USB4/TBT alt-mode 协商 |
| `cc288a3984` | 通用：为错误命令添加调试语句（cherry-pick 自 redrix-rw）|
| `054fd9c9a4` | 通用：npcx rom_chip 抑制 GCC `-Warray-bounds` |
| `ec64ab774e` | 通用：ecst 抑制 GCC 14 误报警告 |
| `6b9b2143ad` | 通用：keyboard_backlight 在 sysjump 后重注册驱动 |

## EC 问题与修复

### USB PD AP Mode Entry（USB4 / 雷电自动协商）

#### 问题

`CONFIG_USB_PD_REQUIRE_AP_MODE_ENTRY` 启用时，EC 不主动进入任何 alt mode
（DP/TBT/USB4），而是等待 AP 通过 host command（`ectool typeccontrol`）指定模式。
这是 ChromeOS 的标准行为——AP 端根据用户设置和显示器需求决定进入哪种模式。

在非 ChromeOS 系统上没有这个 AP 端组件，EC 就一直停在 USB3 模式，
无法自动协商 USB4/Thunderbolt。表现为：

- `ectool inventory` 出现 `42: Host-controlled Type-C mode entry`
- `boltctl` 看不到任何对端设备
- `/sys/class/typec/` 下没有 USB4/雷电相关条目

#### 修复

取消该配置项（`primus/board: enable autonomous USB4/TBT alt-mode entry`）。
之后 EC 不再等待 AP 指令，会自动进入端口双方都支持的最优模式
（USB4 > TBT > DP）。

同时补上 `CONFIG_USB_PD_DATA_RESET_MSG`——USB4 协商必需，redrix 一直有，
primus 一直缺。

启用后 `EC_FEATURE_TYPEC_REQUIRE_AP_MODE_ENTRY`（bit 42）消失。

#### 已知局限：host-to-host 还需内核补丁

以上修复让 EC 侧能自主协商，足以应付「主机 ↔ USB4/TBT 外设」（如显卡坞）。
但两台这样的机器**直连**（host ↔ host）时，PD 协商成功后链路依旧起不来：
`usb4_port*/link` 恒为 `none`。

原因是内核 `cros_ec_typec.c` 只在 `ap_driven_altmode`（即 EC 报告了
`EC_FEATURE_TYPEC_REQUIRE_AP_MODE_ENTRY`）时才注册 TBT alt mode。
EC 自主后该条件为假，`port_altmode[CROS_EC_ALTMODE_TBT]` 为 `NULL`，
`cros_typec_enable_tbt()` 于是把 SoC 侧 Type-C mux 打成 SAFE_MODE。

解决办法是内核侧小补丁（无条件注册 + `mode_selection = true`，
不影响 USB4/DP 路径），已打包为 DKMS：

<https://github.com/acd407/cros-ec-typec-dkms>

> 注：不要被 `CONFIG_USB_PD_REQUIRE_AP_MODE_ENTRY` 这个名字误导。
> 内核的 `cros_ec_typec` 驱动**并不是**那个缺失的 AP——它有
> `EC_CMD_TYPEC_CONTROL` 的实现，但在 EC 自主模式下会陷入
> `typec_altmode_enter()` 的 `-EPERM` 死锁（EC 等内核先 enter、
> 内核等 EC 先报 active），ChromeOS 靠 typecd 守护进程打破这个环。
> 恢复该选项还会弄坏 USB4 显卡坞的自动协商（内核从不自动发 `Enter_USB`）。
> **保持 EC 自主 + 打内核补丁**才是正确方向。

### 键盘背光在 sysjump 后失效

EC 从 RO sysjump 到 RW 时会清零 BSS 段，`kblight.drv` 变为 NULL。
而键盘背光驱动的注册函数绑定在 `HOOK_CHIPSET_STARTUP` 上，该 hook 只在
AP 冷启动（S5→S4→S3）时触发，sysjump 后 chipset 已在 S0，不会触发。

修复方案是在 `HOOK_INIT` 中添加 sysjump 后的重新注册路径
（`6b9b2143ad`，与 redrix 同源）。

### GCC 14+ 编译警告

现代 GCC（14 及以上）对以下位置报 `-Werror` 级别的误报，已在
`ec64ab774e`、`054fd9c9a4` 中抑制：

- `util/ecst.c`：有意为之的写法被误判
- `chip/npcx/rom_chip.c`：ROM API 表越过数组边界（实际合法）

## 编译

需要 arm-none-eabi 交叉编译工具链。

```bash
make CROSS_COMPILE=arm-none-eabi- BOARD=primus
```

编译产物：

| 文件 | 大小 | 说明 |
|---|---|---|
| `build/primus/ec.bin` | 512KB | RO + RW 全量镜像 |
| `build/primus/RO/ec.RO.flat` | ~252KB | 仅 RO 固件 |
| `build/primus/RW/ec.RW.bin` | ~243KB | 仅 RW 固件（刷写用这个） |

> 编译结束时可能出现 `CHECK_ALLOWED ... env: "vpython3": 没有那个文件或目录`，
> 这是无关告警，固件已正常产出。

## 刷写 EC 固件

### 安全刷写流程（仅刷 RW，不动 RO）

```bash
# 1. 备份当前 EC 全量固件
sudo ectool flashread 0 524288 ec_backup.bin

# 2. 擦除 RW 区域
sudo ectool flasherase 0x40000 262144

# 3. 刷入 RW 固件
sudo ectool flashwrite 0x40000 build/primus/RW/ec.RW.bin

# 4. 回读校验
SZ=$(stat -c%s build/primus/RW/ec.RW.bin)
sudo ectool flashread 0x40000 $SZ check.bin
sha256sum build/primus/RW/ec.RW.bin check.bin   # 必须一致

# 5. 重启 EC 到 RW
sudo ectool reboot_ec cold
```

### 验证刷写结果

```bash
# 确认固件已运行
sudo ectool version
# Firmware copy: RW      ← 关键行
# RW version: primus_v0.0.xxxx-xxxxxxxxxx
```

## 常见问题

### 刷完后 EC 回退到 RO

RW 镜像校验失败会导致回退。检查 `ectool version` 的 `Firmware copy` 字段——
若显示 `RO` 说明 RW 未通过校验，需要重新刷写。

### RO 区域保护

RO 区域有硬件写保护，本流程不动它。**不要**尝试 `flasherase 0`。

## 许可

EC 固件代码版权归 ChromiumOS Authors 所有，遵循其原始许可证（见 `LICENSE`）。
本分支的修改同样遵循该许可。
