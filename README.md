# zmk-config-soflewireless

[English](./README.en.md) · **简体中文**

Sofle 分体键盘的 ZMK 固件配置，已接入 [DYA Studio](https://studio.dya.cormoran.works/)。

> 本分支（`dya`）在 `main` 的基础上接入了 DYA Studio。`main` 保持 ZMK 官方
> `v0.2` 的纯净配置，两者互不影响，可按需选择。

## 关于 Sofle

[Sofle](https://github.com/josefadamcik/Sofle) 由 Josef Adamčík 设计，
名字来自斯洛伐克语"Sofle"，是一款 6 键宽分体人体工学键盘。

它的核心特征：

- **4 行 + 拇指键的对称布局** —— 单侧 5 列 × 4 行 + 3 枚拇指键
- **每半一枚 EC11 旋转编码器** —— 可映射音量、翻页、滚轮等
- **分体无线** —— 每半各一块 nice!nano，左右通过 BLE 配对
- **热插拔轴座** —— 3×6 MX 热插拔底

本配置针对 Sofle 的标准版本（nice!nano v2 + EC11 编码器 + 0.91" OLED 或 nice!view）。

## 固件变体

Actions → 最新一次构建 → Artifacts → `zmk-firmware`，内含以下 `.uf2`：

| Artifact | 说明 |
| --- | --- |
| `sofle_left` / `sofle_right` | **无屏版**，余量最大，推荐日常使用 |
| `sofle_left_oled` / `sofle_right_oled` | SSD1306 OLED 显示 |
| `sofle_left_niceview` / `sofle_right_niceview` | nice!view 适配板 |
| `sofle_left_studio_unlocked` / `sofle_right_studio_unlocked` | 无屏 + Studio 常开（免解锁） |
| `settings_reset` | 清除所有持久化设置（含 DYA Studio 保存的配置） |

左右半分别刷对应文件。`settings_reset` 刷完后再刷正常固件即可。

**要免解锁就直接刷 `*_studio_unlocked`**：这组变体带了
`-DCONFIG_ZMK_STUDIO_LOCKING=n`。若刷完仍锁定，说明该 cmake 参数没生效
（正常两个 uf2 体积应有明显差异），按[排查](#排查)一节核对。

## 左右半分配

**本键盘的 central（主控）是左手** —— USB 与 BLE 都接左手，DYA Studio 网页也只会连上左手。

| | 左手 · central | 右手 · peripheral |
| --- | --- | --- |
| **刷哪个固件** | **`sofle_left*`** | **`sofle_right*`** |
| USB / BLE 连接 | 是 | — |
| Studio 通信 | 是 | — |
| 连接管理、OS 识别、按 OS 切层 | 是 | — |
| 设置中继 | 发起 | 接收并应用 |
| 抖键诊断、看门狗 | 汇总两半 | 检测本半并上报 |

这与 ZMK 内置 sofle shield 一致（`app/boards/shields/sofle/Kconfig.defconfig` 里
就是 `if SHIELD_SOFLE_LEFT → ZMK_SPLIT_ROLE_CENTRAL default y`）。
**Sofle 默认就是左手主控。**

> ⚠️ **刷反了整把键盘会完全没有输入**（central 与 peripheral 配不上）。
> 记住：**插 USB 的那一半 = 左手 = 刷 `sofle_left*`**。

## 图层设计

| 层 | 进入方式 | 内容 |
| --- | --- | --- |
| 0 DEFAULT | — | 主键区，**与原 `main` 分支逐键一致**（仅右手第 2 枚拇指键有变，见下） |
| 1 LOWER | 左手第 4 枚拇指键 | F 键 / 数字 / 符号 |
| 2 RAISE | 右手第 2 枚拇指键（短按退格 / 长按进层） | 蓝牙配置档 / 导航 / 编辑 |
| 3 ADJUST | LOWER + RAISE 同按，或 RAISE 中 `&mo 3` | 媒体 / 亮度 / 蓝牙 / bootloader |

> **相对原 `main` 分支的唯一改动**：DEFAULT 层右手第 2 枚拇指键由
> `&kp RCTRL` 改为 `&lt 2 BACKSPACE`（短按退格 / 长按进 RAISE），
> 以腾出分层入口并与 Corne 参考方案保持一致的习惯。其余 59 个键位逐键未动。
> 原来的右 Ctrl 在本分支中未另置他处；若你依赖它，
> 把该位改回 `&kp RCTRL` 即可（改回后 RAISE 层需另找入口）。

拇指键布局（每侧 3 枚）：

```
左半                                右半
GUI   ALT  CTRL                     SPC   RAISE  RCTRL
LWR   ENT                          ALT   RGUI
```

- `DEFAULT → LOWER`：左手 `&mo 1`
- `DEFAULT → RAISE`：右手第 2 枚 `&lt 2 BACKSPACE`（短按退格 / 长按进层）
- `RAISE → ADJUST`：左手第 2 枚 `&mo 3`
- `LOWER + RAISE` 同按 → ADJUST（conditional layer）
- LOWER / RAISE / ADJUST 的兜底位均为 `&mo 0`，按住可临时回 DEFAULT

### 编码器

每半各一枚 EC11，四层映射一致：

| | 默认层（未在网页配置时的回落值） |
| --- | --- |
| 左编码器（sensor 0） | 音量（`C_VOL_UP` / `C_VOL_DN`） |
| 右编码器（sensor 1） | 翻页（`PG_UP` / `PG_DN`） |

**编码器映射可以在 DYA Studio 网页上按层实时配置，无需重新编译固件。**
本分支接入了 [cormoran/zmk-behavior-runtime-sensor-rotate](https://github.com/cormoran/zmk-behavior-runtime-sensor-rotate)，
`keymap` 里为左右编码器各定义了一个行为实例：

```
rsr_left_vol   → sensor 0（左）  cw-binding = <&kp C_VOL_UP>  ccw-binding = <&kp C_VOL_DN>
rsr_right_pg   → sensor 1（右）  cw-binding = <&kp PG_UP>     ccw-binding = <&kp PG_DN>
```

四层的 `sensor-bindings` 统一为 `<&rsr_left_vol &rsr_right_pg>`。行为规则：

- **默认回落**：某层还没在网页上配置过时，按上面写的 `cw-binding` / `ccw-binding` 动作
  （与原main 分支的 `&inc_dec_kp` 手感一致）
- **网页优先**：在 DYA Studio 上配了某层后，该层以网页配置为准
- **配置持久化**：绑定写入 settings，重启后保留
- 两项都不写时该实例完全透明（不产生动作）

依赖 `config/sofle.conf` 里的 `CONFIG_EC11=y` 与
`CONFIG_EC11_TRIGGER_GLOBAL_THREAD=y` —— 这两行不能删，删掉后传感器仍在
但不再产生事件，编码器会完全失效。

> 该模块的 Kconfig 只在 **central（左半）** 开启（见
> `snippets/sofle-central/sofle-central.conf`）。Studio RPC 的 protobuf 桩代码
> 只在 central 生成，右手开启会编译失败 —— 但这不影响使用：**编码器事件经由
> 分体链路汇总到 central 处理**，两半的旋转都会生效。
>
> 它依赖 settings 后端做持久化，该后端已由 `zmk-module-settings-rpc` 提供
> （`CONFIG_ZMK_SETTINGS_RPC=y`）。

> `config/sofle.conf` 里没有 `CONFIG_ZMK_DISPLAY`。显示由 `build.yaml` 的
> snippet（`sofle-oled` / `sofle-no-display`）控制。

### 外设电源

DEFAULT 层第 4 行保留了原 `main` 分支的 `&ext_power EP_TOG`（外设电源开关），
位置在第 3 行 MUTE 键右侧。Sofle 的 OLED / nice!view 与外设供电相关，
若你的版本没有这个功能，删掉该绑定即可。

## DYA Studio

浏览器打开 <https://studio.dya.cormoran.works/>，按 splash 页面选择连接方式：

| 方式 | 说明 | 支持的浏览器 |
| --- | --- | --- |
| USB | 插线后选择串口（Web Serial） | Chrome / Edge（桌面） |
| Bluetooth | 配对后通过 BLE 连接（Web Bluetooth） | Chrome / Edge（桌面）、Android Chrome、iOS 用 Bluefy |
| Demo | 无需键盘，模拟键盘试用全部界面 | 同上 |

Firefox 与 Safari 不支持。

### 可用功能

- **Keymap Editor** — 可视化编辑键位与图层，按键实时高亮
- **Macros & Combos** — 网页上直接创建宏与组合键，**无需重新编译固件**
- **Sensor Rotate** — 编码器顺/逆时针行为**按层配置**，无需重新编译，设置持久化
- **Connection Management** — BLE 配置档重命名 / 切换 / 解绑，设置 USB 与蓝牙优先级
- **Device Settings** — 电源管理（idle / deep sleep 超时）、各分半独立设置
- **Troubleshooting** — 电量、固件版本、uptime；逐键抖键检测；复制完整支持报告

> Sofle 有编码器但无轨迹球，DYA Studio 的 Trackball Tuning 与相关模块
> （PMW3610 驱动）不适用；编码器的方向映射在 **Sensor Rotate** 面板配置，
> Input Processor 面板针对的是指点设备（鼠标/轨迹球）。

### Studio 锁定

**默认固件会锁定 Studio**：断开连接时立刻锁定，空闲后也会重新锁定。
锁定状态下浏览器的 Web Bluetooth **扫不到这块键盘**，必须先在键盘上按一次解锁键：

```
进 RAISE 层：DEFAULT 层右手第 2 枚拇指键 —— 短按退格 / 长按进 RAISE
解锁 Studio：RAISE 层上的左手第 1 枚拇指键（GUI 位）
备用解锁位：ADJUST 层上的左手第 3 枚拇指键
```

## 本地构建

需要 [Zephyr SDK](https://docs.zephyrproject.org/latest/develop/toolchains/index.html) 和
[`west`](https://docs.zephyrproject.org/latest/develop/west/install.html)。

需要 **Python 3.12**（Zephyr 4.1 不支持 3.13/3.14）和 Zephyr SDK 0.17.0：

```bash
brew install python@3.12
python3.12 -m venv .venv && ./.venv/bin/pip install west
export ZEPHYR_SDK_INSTALL_DIR=~/zephyr-sdk-0.17.0

# 依赖下载到 ./dependencies
west init -l config
# 或下载到仓库上一级（多个 config 仓库共享同一份 ZMK）
# west init -l . --mf config/west-workspace.yml

west update --narrow
west zephyr-export
# Studio 的 nanopb 生成器需要这两个包，否则构建会在生成 proto 时失败
./.venv/bin/pip install "protobuf>=5.27" grpcio-tools

west zmk-build                   # 构建 build.yaml 里的全部变体
west zmk-build -a sofle_right  # 只构建某一个
west zmk-build -a sofle_left -p always   # 改过配置后必须 pristine 重建

# 烧录必须指定单个变体（多目标时 --flash 会直接报错退出）
west zmk-build -a sofle_left --flash

# 排查问题时编一个带 USB 日志的固件（会多占 flash，日常固件不受影响）
west zmk-build -a sofle_left -S zmk-usb-logging --flash
```

固件输出在 `./build/<artifact>/zephyr/zmk.uf2`。

## 排查

固件能编译通过不代表功能都打开了。最典型的例子：DYA Studio 的全部 Kconfig
如果没被正确加载，固件照样能编过、刷进去也能打字，但 Studio 协议栈压根没被链接进去 ——
表现就是"连不上 Studio"。

**1. 先看构建日志里的 Kconfig warning —— 本分支应为 0 条。** 每条
`assigned the value 'y' but got 'n'` 都意味着某个功能没开，warning 里带符号名：

```bash
grep -iE "warning: (assigned|attempt to assign|choice symbol)" \
  build/sofle_left/stdout_and_stderr.log
```

**2. 确认构建实际合并了哪些 `.conf`。** 这是定位"配置没生效"最快的办法——
如果列表里出现的是 `dependencies/zmk/app/boards/shields/sofle/sofle_left.conf`
（ZMK 里的空文件）而不是本仓库的 snippet，就说明配置根本没被读到：

```bash
grep -oE "[^ ']*\.conf" build/sofle_left/stdout_and_stderr.log | sort -u
```

正确的输出里应该能看到 `snippets/sofle-central/sofle-central.conf`。

**3. 核对最终生效的 `.config`。** 下面这些必须全是 `y`，任何一行是 `n`
说明对应功能没打开（最典型的是 `ZMK_BEHAVIOR_LOCAL_ID_TYPE_CRC16`，
它必须和 DYA Studio 网页端一致，否则 Macros / Combos / Keymap Editor 写入的
绑定对不上现有按键，表现为「改了没反应」）：

```bash
grep -E "ZMK_STUDIO|PHYSICAL_LAYOUTS|FAST_KEYMAP|KSCAN_DIAG|WATCHDOG|OS_DETECTION|DEFAULT_LAYER|BLE_MANAGEMENT|RUNTIME_(MACRO|COMBO|INPUT)|INPUT_STREAM|DEVICE_INFO|LOCAL_ID_TYPE" \
  build/sofle_left/zephyr/.config | grep -v "=n$"
```

**4. 编码器没反应**，确认 `grep EC11 build/sofle_left/zephyr/.config` 里
`CONFIG_EC11=y`，且 overlay 中对应 encoder 的 `status = "okay"`（已默认开启）。

**5. 抓设备日志。** 日常固件关了 `CONFIG_CONSOLE` 以省 flash，排查时用
`-S zmk-usb-logging` 编一个带日志的版本。

**6. 键盘完全没有输入**，先确认没刷反：插 USB 的那半必须是 `sofle_left*`。

## 目录结构

```
config/
  sofle.keymap           键位定义（4 层布局，含注释图示）
  sofle.json             外部 keymap-editor / keymap-drawer 用的布局描述
                         （不参与编译；DYA Studio 的键盘图形来自 ZMK fork 内置的
                          physical-layout，不是这个文件）
  sofle.conf             两半共用的 Kconfig（蓝牙、电池、sleep、EC11 编码器……）
  west-dependency.yml    依赖清单：ZMK fork + DYA Studio 模块栈
  west.yml               默认 west 入口
  west-isolated.yml      依赖装到 ./dependencies
  west-workspace.yml     依赖装到 west topdir
boards/shields/sofle/
  Kconfig.defconfig      只放键盘名 + 分体角色 + 上游继承的 ZMK_SPLIT / 显示 / LVGL
  Kconfig.shield         shield 标识
snippets/
  sofle-central/         左手（central）专用：Studio 及全部
                         *_STUDIO_RPC、OS 识别、默认层、连接管理、各扩展模块
  sofle-peripheral/      右手（peripheral）专用：只接收 / 应用设置中继
  sofle-oled/            OLED 显示
  sofle-no-display/      禁用板载 OLED，回收 flash
zephyr/module.yml        告诉 west 在哪里找 boards / snippets
build.yaml               构建矩阵
```

> `sofle.dtsi` / `sofle_left.overlay` / `sofle_right.overlay` 以及物理布局
> （`josefadamcik_sofle_layout`）都来自 `cormoran/zmk` 的 `main+dya` 分支，
> 本仓库不重复定义。

### 按半边拆分配置：为什么用 snippet 而不是 `<shield>.conf`

DYA Studio 的 Kconfig 必须按半边隔离（`*_STUDIO_RPC` 依赖的 protobuf 桩代码
只在 central 生成，右手一旦开启就会因缺少 `proto/zmk/custom.pb.h` 而编译失败）。
本仓库通过 `build.yaml` 里每个变体的 `snippets:` 来做这个隔离：

| 文件 | 作用域 | 放什么 |
| --- | --- | --- |
| `Kconfig.defconfig` | shield | 键盘名、分体角色、上游继承的 `ZMK_SPLIT` / 显示 / LVGL |
| `snippets/sofle-central` | 仅左手 | Studio / RPC / OS 识别 / 默认层 / 各扩展模块 |
| `snippets/sofle-peripheral` | 仅右手 | 只接收并应用设置中继 |
| `config/sofle.conf` | 两半 | 蓝牙、电池、sleep、EC11 编码器、分体事件中继、线程栈 |

**踩过的坑：把配置放进 `boards/shields/sofle/sofle_left.conf` 是无效的。**
Zephyr 的 `SHIELD_ROOT` 只解析到**第一个**包含该 shield 的 board root，而 ZMK 的
`app/boards` 排在前面，于是构建日志里出现的是

```
dependencies/zmk/app/boards/shields/sofle/sofle_left.conf   ← ZMK 里的空文件
```

而不是仓库里的那份。`Kconfig.defconfig` 之所以有效，是因为 Zephyr 会对
**全部** board root 做 `osource`。所以想按半边隔离配置，只有两条路：

1. `Kconfig.defconfig` + `default y`（能生效，但依赖不满足时会被**静默丢弃、
   不产生任何警告**，很容易出现"编译通过但功能没开"）；
2. **snippet**（本仓库的做法，走 `EXTRA_CONF_FILE`）。它是显式赋值，
   依赖不满足时会明确报 warning，同时天然按半边隔离。

## 依赖来源

本分支不使用 ZMK 官方 `main` / `v0.3`，而是 [cormoran/zmk](https://github.com/cormoran/zmk)
的 `main+dya` 分支 —— 它在 ZMK main 之上加了 DYA Studio 所需的
自定义 Studio RPC 协议与分体中继支持。**DYA Studio 的扩展功能依赖这个 fork**，
换回官方 ZMK 只能保留基础的 keymap 编辑功能。

> ⚠️ cormoran 的 ZMK fork 为实验性分支，针对 DYA 键盘优化，可能包含不稳定的变更。
> 自己用着没问题，但不建议用于需要高稳定性的场合。仓库配置了每周五一次的
> 定时构建，用于监控上游变更带来的破坏。
>
> 相比之下 `main` 分支用的是官方 ZMK `v0.2`，更保守稳定，但没有 DYA Studio。

## License

MIT