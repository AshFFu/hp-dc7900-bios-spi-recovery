# Image Evidence Archive

[Project README](../README.md) | [中文首页](../README.zh-CN.md) | [中文完整报告](../docs/full-report-zh-CN.md)

本目录收录 **HP Compaq dc7900 BIOS / SPI / Intel Management Engine 修复项目**中形成的关键照片、诊断截图和技术标注图。

这些图片用于补充完整技术报告中的关键证据，包括：

- 主板 SPI Flash 型号与实际编程工具；
- U19 / U21 区域的焊盘损伤与信号恢复；
- Intel FITC 5.0.0.1167 分析环境；
- 完整 HP OEM ME Region 替换后机器首次越过原 POST 冻结点；
- 修复过程中出现的暂态 HECI / MEBx 错误；
- Linux 下对 HECI、`mei_me`、`/dev/mei0` 和 MEI sysfs 状态的最终验证。

完整故障分析、实验过程、SPI 区域结构、镜像构建方法和结论请参见：

**[中文完整技术报告](../docs/full-report-zh-CN.md)**

---

## 图像处理与证据完整性

本目录中的公开图片均来源于本项目实际维修、实验和验证过程。

为保护设备及操作者隐私，公开版本可能移除了与技术结论无关的信息，例如：

- MAC 地址；
- Serial Number；
- UUID / Asset Tag；
- 用户名、账户信息；
- 本地文件路径；
- 其它可关联到具体设备或个人的标识。

除上述隐私处理外，公开图片仅可能进行以下处理：

- 裁剪无关边缘；
- 旋转；
- 轻度亮度或对比度调整；
- 隐私字段遮挡；
- 添加箭头、引脚编号、信号名称等明确的技术标注。

图片未进行生成式重绘，也未使用 AI 修改焊盘状态、芯片状态、POST 信息、错误代码、终端输出或其它与技术结论有关的内容。

技术标注图是在原始照片副本上增加说明形成，其底层硬件图像仍来自实际维修现场。

---

## 图像索引

| 图号 | 文件 | 内容 |
|---|---|---|
| 图 1 | [`01-mx25l3205dmi-12g-sop16.jpg`](01-mx25l3205dmi-12g-sop16.jpg) | 实际使用的 Macronix MX25L3205DMI-12G SOP16 SPI Flash |
| 图 2 | [`02-ezp2019-plus-programmer.jpg`](02-ezp2019-plus-programmer.jpg) | 本项目使用的 EZP2019+ USB SPI 编程器 |
| 图 3 | [`03-b-u19-u21-pad-damage.jpg`](03-b-u19-u21-pad-damage.jpg) | B 机 U19 / U21 区域及 U19 左侧焊盘损伤 |
| 图 4 | [`04-b-u19-u21-signal-map.jpg`](04-b-u19-u21-signal-map.jpg) | B 机损伤区域的 CS# / MISO / WP# / GND 信号标注 |
| 图 5 | [`05-a-u19-pin3-wp-pad-damage.jpg`](05-a-u19-pin3-wp-pad-damage.jpg) | A 机 U19 区域，主要损伤为 Pin 3 / WP# 焊盘 |
| 图 6 | [`06-fitc-5.0.0.1167-platform-select.jpg`](06-fitc-5.0.0.1167-platform-select.jpg) | Intel FITC 5.0.0.1167 Platform Select |
| 图 7 | [`07-b-first-post-after-oem-me-replacement.jpg`](07-b-first-post-after-oem-me-replacement.jpg) | B 机替换完整 HP OEM ME Region 后首次越过原冻结点 |
| 图 8 | [`08-heci-2233-2206-transient-error.jpg`](08-heci-2233-2206-transient-error.jpg) | 修复早期一次性的 2233 / 2206 HECI / MEBx 错误 |
| 图 9 | [`09-linux-heci-pci-mei.jpg`](09-linux-heci-pci-mei.jpg) | Linux 下 HECI PCI 设备、`mei_me` 与 `/dev/mei0` |
| 图 10 | [`10-linux-mei-sysfs-status.jpg`](10-linux-mei-sysfs-status.jpg) | Linux 下 MEI sysfs 最终状态验证 |

---

## 图 1 — MX25L3205DMI-12G SPI Flash

![MX25L3205DMI-12G SPI Flash](01-mx25l3205dmi-12g-sop16.jpg)

本项目实际涉及的 SPI Flash 为 **Macronix MX25L3205DMI-12G**。

主要规格：

- 32 Mbit；
- 4 MiB；
- 3.3 V；
- SPI NOR Flash；
- SOP16 封装。

完整 SPI 镜像大小为 4 MiB，与该器件容量一致。

本项目后期使用 SOP16 器件进行离线编程和焊接修复。

---

## 图 2 — EZP2019+ USB SPI Programmer

![EZP2019+ USB SPI Programmer](02-ezp2019-plus-programmer.jpg)

项目中用于 SPI Flash 离线读写的编程器为 **EZP2019+**。

实际维修流程包括：

1. 多次读取原始 Flash；
2. 比较各次 dump 是否完全一致；
3. 保存原始镜像及哈希；
4. 构建实验镜像；
5. 擦除；
6. 写入；
7. 回读；
8. 对回读结果进行二进制比较或哈希校验。

本项目实际测试中，EZP2019+ 在 Windows 10 环境下工作可靠。

---

## 图 3 — B 机 U19 / U21 区域及焊盘损伤

![B board U19 U21 pad damage](03-b-u19-u21-pad-damage.jpg)

B 机在 SPI Flash 拆焊过程中出现了明显焊盘损伤。

U19 左侧 Pin 1–4 对应焊盘脱落，因此无法再简单地将替换芯片直接焊回原焊盘。

这使修复工作从普通的 SPI Flash 更换进一步演变为：

**SPI 芯片替换 + PCB 信号恢复。**

后续通过 U21 等可访问节点恢复了对应 SPI 信号。

---

## 图 4 — B 机 U19 / U21 信号映射

![B board U19 U21 signal map](04-b-u19-u21-signal-map.jpg)

该图是在实际 B 机主板照片基础上制作的技术标注图。

重点标出了受损区域相关的 SPI 信号：

- `CS#`
- `MISO`
- `WP#`
- `GND`

完整维修过程中确认的 U19 与 U21 对应关系为：

| U19 | 信号 | U21 |
|---|---|---|
| Pin 1 | CS# | Pin 7 |
| Pin 2 | MISO | Pin 8 |
| Pin 3 | WP# | Pin 9 |
| Pin 4 | GND | Pin 10 |
| Pin 5 | MOSI | Pin 15 |
| Pin 6 | SCLK | Pin 16 |
| Pin 7 | HOLD# | Pin 1 |
| Pin 8 | VCC | Pin 2 |

SOP16 器件的其余 NC 引脚不参与上述 SPI 信号恢复。

---

## 图 5 — A 机 U19 Pin 3 / WP# 焊盘损伤

![A board U19 WP pad damage](05-a-u19-pin3-wp-pad-damage.jpg)

A 机的焊盘损伤程度明显轻于 B 机。

主要问题集中在：

**U19 Pin 3 / WP#**

因此 A 机的 PCB 修复复杂度低于 B 机，并未出现 B 机那种左侧四个焊盘同时脱落的情况。

---

## 图 6 — Intel FITC 5.0.0.1167

![Intel FITC 5.0.0.1167 Platform Select](06-fitc-5.0.0.1167-platform-select.jpg)

本项目使用 **Intel Flash Image Tool 5.0.0.1167** 对 dc7900 的 Intel ME 5.x 镜像进行分析。

FITC 对健康 HP 官方镜像能够正常解析配置。

而在故障 B 机镜像中，FITC 出现了关键异常，例如：

```text
ME_CFG_DEF NVAR Not found.
KernFixedData NVAR Not found. Adding NVAR.
```

并且故障镜像解析得到的 `Configuration.txt` 异常为空。

这成为判断问题位于 **ME Region 持久化配置 / NVAR / EFFS 附近** 的重要证据之一。

需要特别说明：

FITC 的成功 Build 并不代表生成镜像一定适合直接写回目标机器。

本项目曾发现 FITC Build 会改变 Descriptor strap，因此 FITC 主要被用于分析，而不是直接作为最终镜像生成工具。

---

## 图 7 — 完整 HP OEM ME Region 替换后的首次成功 POST

![First POST after OEM ME replacement](07-b-first-post-after-oem-me-replacement.jpg)

这是整个项目最关键的实验结果之一。

此前已经分别验证：

- 原 System BIOS 主体并不是故障根因；
- 单独升级 BIOS 到 1.27 后故障依旧；
- 单独替换 ME CODE / NFTP 后故障依旧。

当 B 机的完整 ME Region 被替换为健康的 HP OEM ME Region 后，机器第一次成功越过原来的 POST 冻结位置。

因此直接证据支持：

> 故障位于原 ME Region 中的持久化状态 / 配置区域，而不是 System BIOS 主体，也不是单纯的 ME 可执行 CODE / NFTP。

本项目没有进一步证明究竟是哪一个单独 NVAR 或哪几个具体字节导致故障，因此报告不会把结论扩大到尚未验证的粒度。

---

## 图 8 — 2233 / 2206 暂态 HECI / MEBx 错误

![HECI 2233 2206 transient error](08-heci-2233-2206-transient-error.jpg)

在修复早期，完整替换 OEM ME Region 后曾出现一次性的：

- `2233`
- `2206`

HECI / MEBx 相关错误。

该现象随后没有持续存在，因此最终被判断为修复后首次启动过程中产生的暂态状态，而不是新的永久故障。

这张图被保留，是为了完整记录修复过程中的异常。

---

## 图 9 — Linux 下 HECI / MEI 设备验证

![Linux HECI PCI MEI](09-linux-heci-pci-mei.jpg)

修复完成后，使用 Linux 对 Intel ME / HECI 通信链进行了进一步验证。

系统能够识别 Intel HECI PCI 设备：

```text
8086:2e14
```

并能够看到：

```text
mei_me
/dev/mei0
```

这说明：

- HECI PCI 功能能够正常枚举；
- Linux `mei_me` 驱动能够绑定；
- MEI 字符设备能够建立。

因此，修复后的机器并不是单纯“能够通过 POST”，ME 与主机之间的基础 HECI / MEI 通信链也能够正常建立。

---

## 图 10 — Linux MEI sysfs 最终状态

![Linux MEI sysfs status](10-linux-mei-sysfs-status.jpg)

Linux sysfs 进一步显示：

```text
dev_state = ENABLED
fw_status = 300A965A
hbm_ver = 1.0
```

同时可以看到：

```text
fw_ver = 0:0.0.0.0
```

对于这套较老的 Intel ME 5.x 平台及现代 Linux MEI 驱动组合，`fw_ver = 0:0.0.0.0` 本身不能作为“ME 固件不存在”或“ME 未运行”的证据。

因为同一系统已经同时确认：

- HECI PCI 设备存在；
- `mei_me` 已绑定；
- `/dev/mei0` 存在；
- `dev_state = ENABLED`；
- `fw_status` 可读取；
- HBM 协议版本可读取。

因此最终判断应基于整条证据链，而不是孤立解释 `fw_ver` 一项。

---

## 与完整报告的关系

这些图片并不独立构成完整的修复教程。

它们的主要作用，是为完整技术报告中的以下关键阶段提供可核查的视觉证据：

```text
POST 冻结
    ↓
排除 System BIOS 主体
    ↓
排除单纯 ME CODE / NFTP
    ↓
FITC 发现故障 ME 配置异常
    ↓
完整替换健康 HP OEM ME Region
    ↓
机器越过冻结点
    ↓
恢复 SPI 硬件连接
    ↓
Linux HECI / MEI 最终验证
```

完整实验过程、SPI 地址结构、各实验镜像哈希、故障边界及最终重建方法，请参见：

**[HP Compaq dc7900 BIOS / SPI / Intel ME 完整修复报告](../docs/full-report-zh-CN.md)**

---

## 结论

本图像档案支持本项目的核心结论：

> 在本项目涉及的两台 HP Compaq dc7900 USDT 上，POST 冻结故障的直接证据最终指向 Intel ME 5.x Region 中的持久化配置 / 状态区域，而不是 System BIOS 主体，也不是单独的 ME CODE / NFTP。

对故障机器完整替换健康的 HP OEM ME Region，同时保留目标机器自身的 Descriptor、GbE / MAC、PDR 以及 BIOS runtime / DMI 数据，并配合官方 BIOS 主体，可以恢复机器正常启动。

该结论严格限定于本项目实际验证的设备和实验结果，不主张所有 dc7900 POST 冻结故障都具有完全相同的根因。
