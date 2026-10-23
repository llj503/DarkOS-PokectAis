# dArkOS on Pocket AIS

基于 dArkOS NGP45HT（Trixie）移植到 Pocket AIS 的 Linux 系统。镜像可写入 TF 卡，
由 eMMC 内置 U-Boot 加载 TF 卡 p3 的 `extlinux.conf` 启动。

## 状态

已实现并验证：

- 面板、背光、DSI 时序正常工作；屏幕方向正确。
- 左摇杆方向正确（DTB `left-stick-invert`），按键/肩键/摇杆可用。
- AIC8800D80 SDIO WiFi（RG52 Mini 驱动/固件），图形化 WiFi 与 SSH 设置可用。
- EmulationStation 中文菜单、主题、色温/BCSH 面板调节。
- RetroArch 简体中文与 CJK 字体；退出/热键正常。
- EmulationStation 单游戏高级菜单：进入游戏前可切换模拟器/核心。
- 电池图标裁剪修复；RK817“永远 Charging”假状态用 shim 修正。
- 关机充电/USB host 角色切换；睡眠/背光恢复修复。
- 首次启动自动扩展 `/roms` 分区（exFAT）。

仍待处理/已知限制：

- 内置 eMMC AisOS 未修改；eMMC 阶段的第一个启动 logo 不属于 TF 镜像。
- 若重新打包镜像后只启动内置系统，检查 p3 的 GPT 可启动标志（0x4）以及 p1/p2 的
  分区类型 GUID；`image_build/fix_gpt_20261020.py` 是修复脚本。
- 镜像构建脚本是在运行中的 dArkOS 设备上执行的参考脚本（固定 `/roms`、`/dev/mmcblk1`
  等路径），不是可在 PC 上直接交叉构建整个发行版的脚本。

## 镜像

| 文件 | 大小 | SHA-256 |
| --- | --- | --- |
| `pocketAis-dArkOS-20261020-autoexpand.img` | 13,019,137,024 B | `9090d128ffe51aef8b4d29735ef6c5f317164bde3f3375d8ffb46ba4c4d9b25f` |
| `pocketAis-dArkOS-20261020-autoexpand.img.xz` | 3,518,490,016 B | `2e0a0202916c42c1a204ebbfc8e3b3fbf5191271ed653bef99f0356289b8e656` |

镜像不纳入 Git。刷写前先解压 xz，使用 Rufus **DD** 模式或 `dd`；刷写后首次启动会扩展
`/roms`。

## 目录

```text
image_build/
  build_image.sh                  # 早期版本的镜像组装脚本
  build_image_20261020.sh         # 最终镜像组装（p1-p4 + p5 占位 + GPT + p3 清理）
  fix_gpt_20261020.py             # 恢复 p1/p2 类型 GUID、p3 可启动标志、GPT CRC
  fix_suspend_20261020.py         # 修复 GPU governor 路径、睡眠背光、ES 屏保
  pocketais-autoexpand.sh         # 首次启动扩展 /roms
  pocketais-autoexpand.service
  raw2png.js                      # 调试用帧缓冲转换
dtb/
  ro520c.dts                      # Rockchip RK3562 EVB1 基础 DTS
  tf_patch/ro520c_aic.dts         # Pocket AIS 适配后的 DTS（WiFi/摇杆/音频）
  tf_patch/extlinux.conf          # 指向 rk3562-ro520c.dtb 的启动项
  patch_dtb.py                    # 早期按键映射替换脚本
rootfs/
  pocketais-battery-shim*         # 修正 RK817 电池状态
  pocketais-usb-mode              # 手动切换 OTG host/peripheral
  pocketais-usb-role-*            # 自动探测充电器/USB 设备并切换角色
kernel_config.txt                 # AisOS 内核配置（参考）
*.sha256                          # 镜像校验
```

## DTB 与启动

- 基础 DTB 来自 RK3562 EVB1（`rk3562-ro520c.dtb`），Pocket AIS 的 dArkOS 使用它。
- 在 DTS 中加入/保留：
  - `left-stick-invert`（左摇杆方向）。
  - AIC8800D80 SDIO WiFi 节点与供电。
  - RK817 codec 的 `use-ext-amplifier`、`hp2extamplifier`、`hp-ctl-gpios`，
    扬声器走 HP 输出 + 外部功放。
- 编译后的 DTB 放在 p3 的 FAT 分区，`extlinux.conf` 的 `FDT` 指向它。
- `.dtb` 二进制不纳入仓库；请用 `dtc` 从 DTS 编译，或从镜像 p3 中提取。
  上游 DTS 的基础来源见顶层 README 的“上游与致谢”。

## 镜像组装要点

`build_image_20261020.sh` 在设备上执行，主要步骤：

1. 读取 p5 起始扇区，确定镜像总大小（p1-p4 + 256 MiB 的 p5 占位 + 备份 GPT）。
2. 停止 EmulationStation，复制 p1-p4，写入 p5 占位并截断镜像。
3. 用 `sgdisk` 重建 GPT，保留 p1-p4 的 GUID/名称，p3 设为可启动。
4. 挂载 p3，删除首次启动标记和日志，确保新卡会执行自动扩容。
5. 在 PC 上压缩为 xz；压缩后校验 SHA-256。

`fix_gpt_20261020.py` 用于修复“镜像启动后进入内置 eMMC 系统”的问题：恢复 p1/p2
的特殊类型 GUID、p3 的 `LegacyBIOSBootable` 属性，并重算主/备份 GPT CRC。

## 电池与 USB

- `pocketais-battery-shim` 用 tmpfs 文件 bind-mount 覆盖
  `/sys/class/power_supply/battery/status`，根据 `current_now` 的符号和滞回判断
  Charging/Discharging，解决 ES 不插电也显示闪电图标的问题。
- `pocketais-usb-role-watch` 在无 USB 设备时每 8 秒短暂探测 peripheral；检测到充电器
  则保持 peripheral，否则切回 OTG host。相关 systemd 单元在 `rootfs/` 中。
- 安装时把脚本放到 `/usr/local/sbin/`，单元放到 `/etc/systemd/system/`，`systemctl enable`
  对应服务。注意不要把 `pocketais-usb-role.service` 加 `After=multi-user.target`，
  否则会形成 systemd 顺序环，导致 watcher 不启动、充电只有约 2 W。

## 相关上游

- dArkOS NGP45HT：<https://github.com/llj503/dArkOS_ngp45ht>
- ArkOS4Clone：<https://github.com/lcdyk0517/arkos4clone>
- ROCKNIX distribution：<https://github.com/AveyondFly/distribution_rocknix>
- RetroArch / EmulationStation：各自上游项目。
