# HP Compaq dc7900 BIOS / SPI Flash 恢复

[English](README.md) | [简体中文](README.zh-CN.md)

这是一次针对两台 HP Compaq dc7900 Ultra-Slim Desktop 相同晚期 POST 卡死故障进行完整调查并最终成功修复的技术记录。最终证据将原始故障定位到 Intel ME 5.x 的持久化配置 / 状态层。

本项目记录了从硬件识别、SPI Flash 访问、重复读取校验、二进制比较、单变量固件实验、外置编程、主板焊盘修复，到最终镜像重构和修复后验证的完整过程。

本仓库的目标不是提供一个可供所有 dc7900 直接刷写的通用 BIN，而是记录一套在保留机器唯一数据前提下，可以理解、验证并复现的恢复方法。

## 项目状态

**已完成 —— 已在两台真实机器上完成修复并验证。**

两台机器目前均可以：

- 正常完成 POST；
- 进入 F10 BIOS Setup；
- 启动操作系统；
- 在 Linux 下正常枚举 HECI / MEI。

修复后未再复现原始晚期 POST 卡死。

## 硬件

- 设备：HP Compaq dc7900 Ultra-Slim Desktop
- BIOS family：`786G1`
- 主板标识：
  - HP 462433-001
  - HP 460954-001
- 固件存储：4 MiB 板载 SPI NOR Flash
- 恢复过程中使用的外置编程器：EZP2019+

## 核心结论

实机单变量实验将原始故障定位到 Intel ME 5.x 的持久化配置 / 状态层，重点指向：

```text
EFFS
NVAR
OEM configuration
```

以下项目已经被直接排除为原始卡死的主要原因：

- System BIOS 主体；
- 单纯 BIOS 1.26 版本过旧；
- ME executable CODE / NFTP 本身；
- HECI 物理控制器损坏。

最终验证成功的恢复原则：

```text
保留目标机 Descriptor + GbE/MAC + PDR
完整替换为健康的 HP OEM ME Region
保留目标机 BIOS runtime / DMI
System BIOS 主体使用 HP 官方 BIOS v1.27
```

## 仓库内容

```text
docs/
  README.md
  full-report-en.md
  full-report-zh-CN.md
  recovery-procedure.md
  spi-flash-analysis.md

images/
  README.md
  10 张证据图片

tools/
  BUILD_DC7900_A_FINAL_ONECLICK_V2.cmd
```

### 文档

- [`docs/full-report-en.md`](docs/full-report-en.md) —— 完整英文技术报告
- [`docs/full-report-zh-CN.md`](docs/full-report-zh-CN.md) —— 完整中文技术报告
- [`docs/recovery-procedure.md`](docs/recovery-procedure.md) —— 可直接复用的恢复操作流程
- [`docs/spi-flash-analysis.md`](docs/spi-flash-analysis.md) —— SPI 区域、二进制比较、实验镜像、哈希与拼接逻辑
- [`images/README.md`](images/README.md) —— 图像证据档案

### 工具

- [`tools/BUILD_DC7900_A_FINAL_ONECLICK_V2.cmd`](tools/BUILD_DC7900_A_FINAL_ONECLICK_V2.cmd)

公开版脚本来自项目最终实际使用的构建器，已删除机器完整 MAC / Product / Serial 等私有硬编码信息。脚本仍然逐字节保留并验证目标机的身份相关区域。

## 重要提示

在没有确认以下信息之前，**不要直接将本项目中的任何固件镜像写入另一台机器：**

- 主板版本；
- SPI Flash 芯片型号和容量；
- 固件布局；
- 机器唯一数据；
- 原始 dump 的完整性；
- 是否具备恢复和重新编程能力。

任何修改和写入之前，都应至少保存多份完整原始 dump 并验证一致性。

本项目采用的可信基线为：

```text
连续读取 3 份完整 4 MiB dump
+
SHA-256 完全一致
```

写入后还必须完整 Read Back，并用 SHA-256 与目标镜像逐文件验证。

## 固件二进制文件

本仓库**不公开完整 SPI dump 和最终重构 BIN**。

主要原因：

- 完整镜像中包含机器唯一标识及运行时数据；
- 涉及 HP / Intel 固件再分发问题；
- 机器专属完整镜像被误刷到其它设备存在明显风险。

因此仓库公开的是：

- 精确 SPI Region 布局；
- 地址偏移；
- 文件哈希；
- 单变量实验结果；
- 重构逻辑；
- 脱敏后的构建脚本；
- 完整恢复与验证流程。

## 图像证据

仓库目前包含 10 张实际维修与验证图片，涵盖：

- SPI Flash 芯片；
- EZP2019+ 编程器；
- U19 / U21 焊盘损伤；
- SPI 信号映射；
- Intel FITC 分析；
- 完整 OEM ME 替换后首次跨过原冻结点；
- 暂态 HECI / MEBx 报错；
- Linux HECI / MEI 最终验证。

详见 [`images/README.md`](images/README.md)。

## 免责声明

本项目记录的是一次独立进行的硬件维修与技术研究过程。

HP、Intel 以及其它相关商标均属于其各自权利所有者。本项目与 HP 或 Intel 没有隶属、授权或官方支持关系。

固件修改、焊接以及外置 SPI 编程均可能导致硬件无法启动。本仓库内容仅用于技术研究、故障分析及硬件维修参考。
