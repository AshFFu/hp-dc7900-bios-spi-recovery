# Image Evidence Archive

[Project README](../README.md) | [中文首页](../README.zh-CN.md) | [中文完整报告](../docs/full-report-zh-CN.md)

本目录保存 HP Compaq dc7900 BIOS / SPI / Intel ME 修复项目中具有直接证据价值的公开版照片和诊断截图。

## 公开版隐私规则

上传前必须检查图片中是否出现下列信息：

- 完整 MAC 地址
- Serial Number
- UUID / Asset Tag
- Windows 用户名、账户名、邮箱
- 本地个人目录路径
- 原始相机/手机文件名或其它内部归档标识
- 无关设备的序列号或标签
- 浏览器账号、通知、聊天侧栏等无关个人信息

如存在，应先裁剪、遮挡或重新命名后再上传。

不要为了“美观”对证据图片进行内容生成、重绘或 AI 修复。允许的处理仅限于：

- 裁剪无关边缘
- 旋转
- 亮度 / 对比度的轻度调整
- 对隐私字段进行遮挡
- 添加明确的技术标注箭头或引脚编号

原始未处理照片、原始文件名与公开文件之间的对应关系，应仅保存在私人项目归档中，不进入公开 GitHub 仓库。

## 文件命名

公开仓库使用与完整报告图号一致的稳定文件名：

| 图号 | GitHub 文件名 | 内容 |
|---|---|---|
| 图 1 | `01-mx25l3205dmi-12g-sop16.jpg` | 实际使用的 Macronix MX25L3205DMI-12G SOP16 Flash |
| 图 2 | `02-ezp2019-plus-programmer.jpg` | 项目使用的 EZP2019+ USB 编程器 |
| 图 3 | `03-b-u19-u21-pad-damage.jpg` | B 机 U19/U21 区域，U19 左侧 1–4 脚焊盘脱落 |
| 图 4 | `04-b-u19-u21-signal-map.jpg` | B 机损伤区域信号标注：CS# / MISO / WP# / GND |
| 图 5A | `05a-a-u19-wp-pad-damage.jpg` | A 机 U19/U21 局部，主要损伤为 Pin3 / WP# |
| 图 5B | `05b-b-u19-four-pad-damage.jpg` | B 机局部，左侧四个焊盘损伤 |
| 图 6 | `06-fitc-5.0.0.1167-platform-select.png` | Intel FITC 5.0.0.1167 Platform Select |
| 图 7 | `07-b-first-post-after-oem-me-replacement.jpg` | B 机完整替换 HP OEM ME 后首次越过原冻结点 |
| 图 8 | `08-heci-2233-2206-transient-error.jpg` | 修复早期一次性的 2233 / 2206 HECI / MEBx 错误 |
| 图 9 | `09-linux-heci-pci-mei.jpg` | Linux 中 HECI 8086:2e14、mei_me 与 /dev/mei0 |
| 图 10 | `10-linux-mei-sysfs-status.jpg` | Linux sysfs：dev_state / fw_status / hbm_ver / fw_ver |

## 图像选择原则

公开报告只保留能够直接支撑技术结论的图片。

### 图 3 / 图 4 / 图 5A / 图 5B

从私人原始照片中选择能够清楚表现以下内容的版本：

- A 机单个 WP# 焊盘损伤；
- B 机 U19 左侧四个焊盘损伤；
- U19 / U21 区域实际走线关系；
- CS# / MISO / WP# / GND 的有效信号路径。

其中图 4 可以在图 3 的基础上制作公开版技术标注图，但不得改变原始硬件内容。

### 图 9 / 图 10

Linux 验证阶段只保留两张公开截图：

- 图 9：能够同时看见 `8086:2e14`、`mei_me`、`/dev/mei0`；
- 图 10：能够看见 `dev_state=ENABLED`、`fw_status=300A965A`、`hbm_ver=1.0`，以及 `fw_ver=0:0.0.0.0`。

如果原始截图中出现用户名、主机名、本地路径或其它可关联信息，应先裁剪或遮挡后再上传。

## 使用原则

这些图片是技术证据，不是装饰素材。

每张图至少应回答一个明确问题，例如：

- 使用的 Flash 芯片到底是什么；
- 哪些焊盘实际脱落；
- U19 / U21 信号如何对应；
- FITC 是否能够解析故障镜像的配置；
- 完整 OEM ME 替换后机器是否真正越过原冻结点；
- HECI 硬件是否能够在 Linux 中正常枚举和建立 MEI 通信。

不上传重复、模糊或不能增加证据价值的过程照片。
