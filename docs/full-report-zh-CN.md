# HP Compaq dc7900 USDT POST 卡死 / Intel ME 修复项目完整技术报告

[返回文档目录](README.md)

> **项目周期：2026-08-19 — 2026-09-02**  
> **文档版本：1.0 / 项目结案版**  
> **结案状态：A、B 两台机器均已恢复正常**
>
> 两台机器均已能够正常完成 POST、进入 F10 BIOS Setup、启动操作系统。后续观察未再出现原始 POST 卡死，也未再复现 2233/2206 MEI/HECI 错误。

---

## 核心结论

本项目通过两台同型号 HP Compaq dc7900 Ultra-Slim Desktop 的实际维修、SPI 固件取证和单变量实验，将典型故障：

> POST / 内存计数完成附近冻结，F10 被识别并显示 `SETUP`，但无法真正进入 BIOS Setup，也不能继续正常引导

定位到：

**Intel ME 5.x Region 的持久化配置 / 状态层异常，重点位于 EFFS / NVAR / OEM configuration 一带。**

本项目的直接实验同时排除了：

- System BIOS 主体随机损坏；
- 单纯 BIOS 版本过旧；
- ME CODE / NFTP 可执行代码本身；
- HECI 物理控制器损坏。

最终验证成功的修复原则是：

1. 保留目标机器自己的 Flash Descriptor；
2. 保留目标机器自己的 GbE NVM / MAC；
3. 保留目标机器自己的 PDR；
4. 保留目标机器自己的 BIOS runtime / DMI / Serial / UUID；
5. 用 HP 官方健康 OEM ME Region 整块替换故障 ME Region；
6. System BIOS 主体使用 HP 官方 BIOS v1.27。

本报告按照：

**现象 → 取证 → 排除 → 根因 → 修复 → 验证 → SOP**

的顺序整理。

过程中的无效猜测和已经被实验推翻的路线，仅在对理解证据链有必要时保留。

---

# 1. 项目对象、原始故障与最终结论

## 1.1 两台机器与共同原始症状

项目对象为两台：

**HP Compaq dc7900 Ultra-Slim Desktop（USDT）**

BIOS family：

```text
786G1
```

两台机器在任何大规模拆焊、固件实验之前，原始症状基本一致：

1. 机器能够开机并显示 POST；
2. 内存计数可以正常完成；
3. 如果提前按 F10，右下角能够显示 `SETUP`；
4. 但系统随后永久停在该画面；
5. 无法真正进入 BIOS Setup；
6. 无法继续正常引导操作系统。

因此画面虽然停留在“内存计数完成”的位置，但最终实验已经证明：

**故障并不是内存检测本身。**

更准确地说，内存计数只是屏幕上最后成功刷新的 UI 状态。

真正的阻塞发生在更靠后的平台初始化阶段。

---

## 1.2 A 与 B 在维修过程中的分化

两台机器原始故障相同，但维修过程中逐渐叠加了不同问题。

### B 机

原始：

```text
POST 完成附近冻结
F10 → SETUP
无法进入 Setup
```

维修过程中因为多次拆焊：

```text
U19 左侧 Pin 1–4 焊盘脱落
```

后续使用：

- U21 SOP16 替代位置；
- 飞线；
- 16-pin Flash；

重新建立 SPI 网络。

最终 B 机成为整个项目的主要**因果隔离实验平台**。

### A 机

A 原始状态与 B 相同：

```text
POST freeze
F10 → SETUP
无法进入
```

但后期在大量拆装之后又出现：

```text
完全黑屏
无正常 POST
风扇高速运行
```

同时：

```text
U19 Pin3 / WP# 焊盘脱落
```

后来确认：

A 除了原始 ME 状态故障之外，还叠加了一个独立的：

**CPU / Socket 接触问题。**

重新清洁 CPU 触点和 Socket 后，再使用已经在 B 上验证成功的健康固件，A 恢复 POST。

---

## 1.3 最终根因结论与证据边界

### 已经通过实机证明

原始：

```text
POST freeze
F10 显示 SETUP
不能进入 Setup
```

与目标机器原来的 **Intel ME Region 内容 / 状态存在直接因果关系**。

B 机在：

```text
完整替换 HP OEM 健康 ME Region
```

后立即跨过过去永远无法通过的冻结点。

A 机在排除 CPU / Socket 接触问题后，同样通过健康 ME 路线恢复。

### 强烈支持

问题位于 ME 的：

```text
persistent configuration / state
EFFS
NVAR
OEM configuration
```

而不是：

```text
ME CODE
NFTP
```

可执行代码本身。

主要证据包括：

- 单独把 System BIOS 主体更新到 1.27，故障完全不变；
- 单独把 ME CODE/NFTP 更新到 5.2.50.1039，故障完全不变；
- FITC 对故障镜像报告：

```text
ME_CFG_DEF NVAR Not found.
```

- 同一个 FITC 对 HP 官方 ME 镜像可以正常解析 OEM Configuration；
- 完整替换 HP OEM ME 后，故障消失。

### 尚未证明

本项目没有逐个替换每一个 NVAR 对象，因此不能声称已经找到：

> 唯一导致故障的某一个固定 NVAR 或某几个具体字节。

同样，本项目只有两台实机，虽然网络上存在大量同症状案例，但不能据此宣称：

> 全球所有 dc7900 的同类卡死 100% 都是同一个根因。

---

# 2. dc7900 SPI / ME 架构与主板服务接口

## 2.1 4 MiB SPI Flash 最终确认布局

完整 SPI：

```text
4,194,304 bytes
0x000000–0x3FFFFF
```

最终确认布局如下：

| Region | 地址 | 大小 | 本项目处理原则 |
|---|---:|---:|---|
| Flash Descriptor | `0x000000–0x000FFF` | 4 KiB | 保留目标机原始值 |
| GbE NVM | `0x001000–0x002FFF` | 8 KiB | 保留目标机 MAC / NVM |
| PDR | `0x003000–0x00AFFF` | 32 KiB | 保留目标机原始值 |
| Intel ME | `0x00B000–0x25FFFF` | 0x255000 bytes | 故障核心，替换为健康 HP OEM ME |
| BIOS runtime / DMI | `0x260000–0x261FFF` | 8 KiB | 保留目标机身份 / 运行时数据 |
| System BIOS body | `0x262000–0x3FFFFF` | 0x19E000 bytes | 使用 HP 官方 v1.27 主体 |

原始镜像 Descriptor signature：

```text
5A A5 F0 0F
```

A、B 两台机器的 GbE NVM 均通过了：

```text
0xBABA
```

checksum 验证。

因此最终修复不应该覆盖目标机自己的：

- Descriptor；
- MAC；
- GbE NVM；
- DMI；
- Serial；
- Product；
- UUID。

---

## 2.2 HP 官方服务跳线与项目标准基线

HP dc7900 Technical Reference Guide 的 System Board Component Designations 中：

```text
E1       Descriptor table override header
E14      SPI ROM boot block header
E49/JP49 Password clear header / jumper
```

项目中还使用：

```text
SW50     Clear CMOS
```

最终标准基线：

| 项目 | 正常状态 | 用途 |
|---|---|---|
| E49 / JP49 | 1–2 短接 | 正常运行 |
| E1 | Open | Descriptor override，正常运行不短接 |
| E14 | Open | SPI ROM Boot Block，正常运行不短接 |
| SW50 | 断 AC、拆电池后操作 | Clear CMOS |
| CR2032 | 正常安装 | RTC / CMOS |

SW50 清 CMOS 时项目采用：

```text
断开 AC
拆下 CMOS 电池
按住 SW50 约 10 秒
```

### 历史纠正

早期维修过程中曾经在其它指导下移动过多个跳线。

因此后来不能再根据：

> 当前绿色跳线帽插在哪里

反推：

> 出厂状态就一定在哪里。

最终以 HP 技术资料重新建立基准：

```text
E49 1–2
E1 open
E14 open
```

---

# 3. 普通 BIOS / Recovery 路线为何无法解决

进入 SPI 直接分析之前，两台机器经历过大量传统恢复尝试，包括：

- Boot Block；
- FDO；
- Win+B；
- USB Recovery；
- 跳线组合；
- Clear CMOS；
- 普通 BIOS 更新；
- 各种 BIOS Recovery 路线。

这些实验最终留下的真正价值不是：

> 某一步差一点成功。

而是证明：

**普通 System BIOS 刷写链没有触及真正出故障的持久化 ME 状态。**

HP SP54033：

```text
786G1 BIOS v1.26 Rev.A
2011-07-25
```

提供：

- DOS Flash；
- HPQFlash；
- BIOS CD；
- BootBlock Emergency Recovery；

这些方案主要针对 System BIOS 更新。

而最终故障却位于：

```text
Intel ME Region persistent state
```

因此遇到高度相同症状时，不应无限重复：

```text
HPQFlash
Boot Block
USB Recovery
CMOS clear
```

而应该较早进入：

```text
完整 SPI dump
↓
分区分析
↓
ME Region 对照
```

---

# 4. SPI 芯片定位、读取基线与编程器可靠性

## 4.1 U19 与 U21

主板提供：

```text
U19 SOIC8
U21 SOP16
```

两个 SPI Flash 兼容 / 替代布局。

原机 Flash 属于：

**Macronix MX25L3205D 系列**

规格：

```text
32 Mbit
4 MiB
3.3 V
```

后期实际购买并成功使用：

```text
MX25L3205DMI-12G
SOP16
4 MiB
3.3 V
```

---

## 4.2 U19 ↔ U21 信号映射

最终确认：

| U19 SOIC8 | 信号 | U21 SOP16 |
|---:|---|---:|
| 1 | CS# | 7 |
| 2 | SO / MISO | 8 |
| 3 | WP# | 9 |
| 4 | GND | 10 |
| 5 | SI / MOSI | 15 |
| 6 | SCLK | 16 |
| 7 | HOLD# | 1 |
| 8 | VCC 3.3 V | 2 |

SOP16 其它：

```text
3–6
11–14
```

为 NC。

这个映射后来成为 B 机 U19 焊盘损伤后恢复 SPI 总线的关键。

---

## 4.3 三次 dump 一致才建立可信基线

A 机原始读取：

```text
SPI_A_1.bin
SPI_A_2.bin
SPI_A_3.bin
```

三份均为：

```text
4,194,304 bytes
```

并且：

```text
逐字节完全一致
```

A 原始 SHA-256：

```text
209577D93CA7B70DE336864A4B771499C7234318955B9C5F1571697E9B9991E4
```

A 原始 MD5：

```text
DA0E898F5831D4521F15A1A1B285AF5E
```

B 机：

```text
SPI_B_1.bin
SPI_B_2.bin
SPI_B_3.bin
```

同样三次读取逐字节一致。

B 原始 SHA-256：

```text
70B3A1229420AE9FB7363B2A96C9B4C506EA1CCE5D8A8B266DE2A012B80DB967
```

B 原始 MD5：

```text
968737F068528F86324618A35CC8C25C
```

### 这一原则非常重要

一次成功 Read 并不能证明读取可靠。

必须：

```text
Read
Read
Read
↓
SHA-256 完全一致
```

之后，后续观察到的二进制差异才有资格被解释为：

> 固件内容差异

而不是：

> 夹具、供电、接触或者通信噪声。

---

## 4.4 EZP2019+ / EZP2023 的 Win11 主机陷阱

项目中使用：

```text
EZP2019+
```

曾经出现一个非常容易误判的问题。

在一台 Windows 11 主机上：

- 芯片 ID 可以识别；
- Read 可以工作；
- Page Program 甚至可以写入全 0；
- 但是 Erase 只运行几格；
- 芯片只被部分擦除；
- RUN 灯可能异常常亮；
- 软件有时报告编程器断开。

后来又测试：

```text
EZP2023 V3.0
```

在同一台 Win11 主机上仍然出现类似 Erase 异常。

但是：

**把完全相同的 EZP2019+、芯片和转接板换到另一台 Windows 10 电脑以后，Erase / Write / Read Back 立即全部恢复正常。**

因此最终判断：

问题不在：

- SPI 芯片；
- EZP2019；
- 转接板；

而在原 Win11 主机的：

```text
USB / 驱动 / 供电 / 软件环境
```

后续项目统一采用：

```text
Windows 10
```

作为已经验证的编程主机。

### 测试文件

#### 4 MiB 全 0

```text
MX25L3205D_4MB_ALL_ZERO_TEST.bin
```

SHA-256：

```text
BB9F8DF61474D25E71FA00722318CD387396CA1736605E1248821CC0DE3D3AF8
```

用途：

证明 Page Program 通路能够工作。

#### 1 MiB 测试图样

```text
MX25L8005E_1MB_TEST_PATTERN.bin
```

SHA-256：

```text
FBBAB289F7F94B25736C58BE46A994C441FD02552CC6022352E3D86D2FAB7C83
```

用途：

辅助排除芯片 / 编程器基本写入问题。

---

# 5. 焊盘脱落与 U19 / U21 硬件修复

## 5.1 B 机：U19 左侧四个焊盘脱落

B 机在反复拆焊过程中失去：

```text
U19 Pin1
U19 Pin2
U19 Pin3
U19 Pin4
```

对应：

```text
Pin1 CS#
Pin2 MISO
Pin3 WP#
Pin4 GND
```

后续发现一个很关键的 PCB 结构事实：

**U21 部分信号并不是绕过 U19 焊盘以后独立直达芯片组。**

一些网络仍经过：

```text
U19 pad / 附近走线
```

继续连接下游。

因此：

> 单纯把 SOP16 Flash 焊在 U21 上，并不一定能够绕过已经断掉的 U19 网络。

最终处理原则：

### Pin1 / CS#

必须连接到：

```text
真正的下游 CS# 网络
```

不能只接 U21 侧残留 stub。

### Pin2 / MISO

同样必须接回真正下游数据网络。

### Pin3 / WP#

不需要强行找回原断线。

维修方案可以：

```text
约 4.7–10 kΩ
上拉至 3.3 V
```

使 WP# 稳定为 High，避免悬空。

### Pin4 / GND

连接到附近通过万用表确认的可靠 GND。

### Pin5–8

U19 原焊盘仍然完整，因此继续使用原 PCB 网络。

这套：

```text
U21 SOP16 + 飞线
```

最终成功恢复 B 机完整 SPI 读写。

对于需要反复编程的维修实验，它甚至比不断拆焊 SOIC8 更方便。

---

## 5.2 A 机：仅 Pin3 / WP# 焊盘脱落

A 主要损伤：

```text
U19 Pin3 / WP#
```

WP# 不是正常 SPI Read 所需的数据线。

只要：

```text
稳定上拉 VCC
```

单独丢失 WP# pad 不应该导致：

```text
完全黑屏
风扇全速
```

后来 A 的 CPU / Socket 接触问题进一步验证了这一判断。

---

# 6. 原始 BIN 的结构化取证

## 6.1 A / B 机器身份与 NVM

| 项目 | A 机 | B 机 |
|---|---|---|
| MAC | `00:22:64:XX:XX:XX` | `00:23:7D:XX:XX:XX` |
| GbE NVM checksum | `0xBABA` | `0xBABA` |
| Product | `FX808PA#***` | `KP722**` |
| Serial | `JPA84*****` | `JPA90*****` |
| Model | HP Compaq dc7900 Ultra-Slim Desktop | HP Compaq dc7900 Ultra-Slim Desktop |

这直接决定：

**最终不能把 B 的完整 4 MiB 镜像永久刷到 A。**

也不能简单把 HP 通用 4 MiB 镜像整颗覆盖到任意 dc7900。

正确方法必须保留目标机器自己的：

```text
Descriptor
GbE / MAC
PDR
BIOS runtime / DMI
```

---

## 6.2 System BIOS 主体的排除证据

B 原始 SPI：

```text
0x262000–0x3FFFFF
```

与 HP 官方 1.26 的 System BIOS 主体：

```text
逐字节一致
```

这是非常强的反证。

它说明：

**B 原始 POST freeze 不是因为 System BIOS 主体发生随机 bit corruption。**

A 与 B 的静态 System BIOS 主体同样一致。

两机差异主要集中于：

```text
BIOS 前 8 KiB
0x260000–0x261FFF
```

以及 ME 的可写状态区域。

这些位置原本就会包含：

- 运行时信息；
- 机器特定数据；
- 状态数据。

B 的 BIOS 前两个 4 KiB 冗余块之间，历史分析只发现：

```text
2 bytes difference
```

而每个 4 KiB block 的字节和校验均为：

```text
0
```

因此更像：

```text
带 generation / checksum 的冗余记录
```

而不是随机 bit rot。

---

## 6.3 HP 官方 1.26 与 1.27 的关键关系

一个早期判断曾认为：

> 1.27 可能同时升级 ME。

后续直接比较证明：

**这个判断是错误的。**

HP 官方：

```text
1.26
1.27
```

两版的：

```text
ME Region 完全相同
```

1.26 与 1.27 的主要变化位于：

```text
System BIOS body
```

因此：

**后面 B 机修复成功绝不能解释为“BIOS 1.27 带来了更新 ME”。**

项目提取的 HP 1.26 完整 ROM：

```text
HP_786G1_v01.26_OFFICIAL_Rom.bin
```

SHA-256：

```text
EAC16981A32DBFDC312B3C12597522F4520D5A7D16BE4F3422F2983A2652EDAF
```

MD5：

```text
09B5628F42134F0C5F28702C9F3B56C7
```

---

# 7. B 机逐步排除实验

B 机承担了整个项目最重要的单变量因果隔离工作。

## 7.1 原始基线

镜像：

```text
SPI_B_1.bin
```

结果：

```text
POST 到固定位置冻结
F10 → SETUP
仍然冻结
```

建立了可重复故障基线。

---

## 7.2 V2 失败路线

早期实验曾一次性大范围替换：

```text
PDR
ME
BIOS
```

结果：

```text
B 黑屏
```

这个实验变量改变太多，因此无法定位根因。

随后重新刷回：

```text
SPI_B_1.bin
```

B 又精确恢复：

```text
原始 POST freeze
```

这个失败实验仍然有一个重要价值：

它证明：

- 芯片可重复编程；
- 板上 SPI 通路可恢复；
- 外置编程过程可控；
- 后续单变量实验结果可信。

### 禁止继续使用

```text
DC7900_A_127_CLEAN_TEST_V2.bin
DC7900_B_127_CLEAN_TEST_V2.bin
```

这两个文件属于已经废弃的实验路线。

---

## 7.3 BIOS-only 1.27：直接排除 System BIOS 主体

镜像：

```text
DC7900_B_v1.27_VALIDATED_BIOS_ONLY.bin
```

结构：

```text
0x000000–0x261FFF
保留 B 原始

0x262000–0x3FFFFF
替换 HP 官方 BIOS v1.27 主体
```

SHA-256：

```text
5357E612C21655EBA27D84190343884556CE148C21A7099A37A53D6870B3F3C4
```

MD5：

```text
3F98191396CB3DE8E9735CB0594EA2CC
```

实机结果：

- BIOS 显示 v1.27；
- 原始冻结位置完全不变；
- F10 → SETUP 后行为完全不变。

因此可以直接得出：

**单纯升级 System BIOS 不能解决这个故障。**

System BIOS 主体被基本排除。

---

## 7.4 ME CODE / NFTP 5.2.50.1039：排除 ME executable

HP：

```text
SP54355
```

包含：

```text
52501039.BIN
```

从 Intel `$MAN` headers 识别版本：

```text
5.2.50.1039
```

B 原始 CODE/NFTP：

```text
5.0.1.1111
```

当次实验只替换：

```text
CODE
NFTP
```

而保留原 B：

```text
EFFS
runtime
persistent state
```

历史项目划分：

| ME 子区 | ME 相对地址 | SPI 绝对地址 | 处理 |
|---|---:|---:|---|
| EFFS | `0x002000–0x0D1FFF` | `0x00D000–0x0DCFFF` | 保留 B |
| CODE | `0x0D2000–0x161FFF` | `0x0DD000–0x16CFFF` | 换 5.2.50.1039 |
| NFTP | `0x162000–0x241FFF` | `0x16D000–0x24CFFF` | 换 5.2.50.1039 |
| Tail / Padding | — | `0x24D000–0x25FFFF` | 保留 |

> 注：上述 ME 子区地址为项目当时的历史划分，使用时应以实际映射再次核对。

生成：

```text
DC7900_B_v1.27_ME_5.2.50.1039_CODE_NFTP_TEST.bin
```

SHA-256：

```text
884CC0F4C2376ED093702D9797B99E74B1DC79136C007311BD72A22BDC5A7903
```

MD5：

```text
D041C15F5E60BEE6F446020186D8F6A3
```

实机结果：

```text
仍然在完全相同位置冻结
```

因此：

如果根因是：

- ME executable code 损坏；
- ME 版本过旧；

替换 CODE/NFTP 应该至少改变机器行为。

实际完全没有改变。

所以嫌疑进一步集中到：

```text
EFFS
runtime state
persistent configuration
NVAR
```

---

# 8. FITC 证据：ME_CFG_DEF NVAR 缺失

## 8.1 FITC 版本与平台

使用：

```text
Intel ME Flash Image Tool
5.0.0.1167
```

Platform Select 提供：

```text
Boulder Creek - EL ICH10
McCreary - EL ICH10D
```

针对 Q45 / ICH10 dc7900，本项目最终使用：

```text
Boulder Creek - EL ICH10
```

---

## 8.2 故障 B 镜像关键日志

FITC 对 B / B127 故障基础镜像 Decompose：

```text
Decomposing data...
Descriptor Region
GbE Region
PDR Region
ME Region
BIOS Region
Configuration
ME_CFG_DEF NVAR Not found.
...
KernFixedData NVAR Not found.
 Adding NVAR.
```

同时产生的：

```text
Configuration.txt
```

为：

```text
0 bytes
```

---

## 8.3 HP 官方 7G1_0126.BIN 对照

对 HP 官方：

```text
7G1_0126.BIN
```

使用相同：

```text
FITC 5.0.0.1167
```

可以正常生成：

```text
Configuration.txt
```

约：

```text
4 KB
```

而且没有：

```text
ME_CFG_DEF NVAR Not found.
```

官方 OEM Configuration 中可以解析出：

```text
Manageability Mode = 1
Intel® iQST Enabled = 1
Intel® QST Supported = 1
ASF2 Supported = 1
Intel® AMT Supported = 1
Intel® Standard Manageability Supported = 1
ME Flash Protection Override Enabled = 0
Remote Connectivity Service Enabler Name = HP Business PC Group
```

以及：

- Remote Configuration；
- OEM CA hashes；
- HP OEM 管理功能配置。

### 证据意义

同一个 FITC：

```text
HP 官方 OEM ME
→ 能找到 ME_CFG_DEF
→ 能正常生成 Configuration

故障 B
→ ME_CFG_DEF NVAR Not found
→ Configuration.txt = 0 bytes
```

再结合：

```text
完整替换 HP OEM ME 后实机立即恢复
```

使：

> ME persistent configuration / NVAR / EFFS 状态异常

从单纯猜测提升为强证据。

---

## 8.4 为什么没有采用 FITC Build 的 outimage

对故障基础镜像直接 Build 时，FITC 会：

```text
Adding KernFixedData NVAR
```

更重要的是，生成的：

```text
outimage
```

还曾经意外改变 Flash Descriptor straps。

项目中观察到过：

```text
5 bytes Descriptor difference
```

以及另一次：

```text
88 bytes Descriptor difference
```

Descriptor 属于平台关键配置。

因此本项目建立规则：

> **FITC Build 成功 ≠ 生成镜像可以直接刷。**

任何 FITC Build 输出，都必须先做：

```text
Descriptor
GbE
PDR
```

逐区二进制比较。

所有未经解释的 Descriptor 改动都应被视为高风险。

---

# 9. B 机最终修复

## 9.1 最终成功镜像

文件：

```text
DC7900_B_v1.27_HP127_ME_TEMPLATE_ONLY_TEST.bin
```

最终结构：

```text
0x000000–0x00AFFF
B 原始 Descriptor + GbE/MAC + PDR

0x00B000–0x25FFFF
HP 官方完整 OEM ME Region

0x260000–0x261FFF
B 原始 BIOS runtime / DMI

0x262000–0x3FFFFF
HP 官方 BIOS v1.27 主体
```

SHA-256：

```text
2F42B1A1B12F0D85B8DE4A8546687EBD6A11447AE91FC3FE3E88992EE8927123
```

MD5：

```text
0E05E16CA72EE1CBAD86B53A380B0448
```

---

## 9.2 第一次跨过冻结点

写入该镜像后，B 第一次真正跨过了过去永远不能通过的 POST 阶段。

出现：

```text
512-Rear Chassis fan not detected
162-System Options Not Set
CMOS checksum invalid / defaults loaded
Intel Boot Agent initialization
F1 Save Changes
```

这些提示在：

- Clear CMOS；
- 裸机；
- 系统配置重新初始化；

的背景下是可解释的。

最重要的事实不是这些 Warning，而是：

**机器已经继续运行到了过去永远无法到达的位置。**

按 F1 保存以后：

- 自动重启；
- HP Logo 正常出现；
- 语言选择出现；
- F10 BIOS Setup 可以真正进入；
- Mint Live 可以正常启动。

至此 B 原始 POST freeze 被修复。

---

## 9.3 一次性的 2233 / 2206 HECI 错误

修复初期，在 Full / Quick Boot 切换附近曾出现一次：

```text
2233-HECI error during MEBx execution
MEBX Status = 0302
2206-End Of POST HECI Failure
```

后续：

- 切回 Full Boot；
- 多次冷启动；
- 多次热启动；

该错误未再出现。

因此本项目没有把这一次暂态错误解释成：

> 修复失败。

更合理的解释是：

完整替换 OEM ME persistent state 后，早期启动过程中可能存在：

```text
ME / MEBx 状态重新建立
```

但这一具体内部机制没有被直接证明，因此只能视为推断。

---

# 10. Linux HECI / MEI 验证

Mint Live 使用：

```bash
lspci -nnk -d 8086:2e14
lsmod | grep -E 'mei|heci'
ls -l /dev/mei*
cat /sys/class/mei/mei0/dev_state
cat /sys/class/mei/mei0/fw_ver
cat /sys/class/mei/mei0/fw_status
cat /sys/class/mei/mei0/hbm_ver
```

B 实际结果包括：

```text
Intel 4 Series HECI
PCI ID 8086:2e14
rev 03

Subsystem:
HP 103c:3036

Kernel driver:
mei_me

/dev/mei0:
存在

dev_state:
ENABLED

fw_status:
300A965A

hbm_ver:
1.0
```

`fw_ver` 曾返回：

```text
0:0.0.0.0
```

对于 ME5 / 老驱动接口，不能仅凭这个字段判断：

> ME firmware 不存在。

### 这组验证说明什么

至少证明：

- HECI PCI device 能枚举；
- `mei_me` 驱动能够正常绑定；
- `/dev/mei0` 存在；
- MEI HBM 1.0 可以建立。

因此强烈反对：

> ICH10 HECI 物理控制器损坏

这一假设。

原始故障更应该位于：

```text
更高层的 ME state
client
configuration
initialization
```

而不是 HECI 硬件链路本身。

---

# 11. A 机：第二故障与最终个性化固件

## 11.1 为什么早期 donor 测试曾经造成误导

A 原始故障与 B 完全相同。

但大量拆装以后，A 变成：

```text
黑屏
无蜂鸣
风扇全速
```

当时甚至曾经把：

```text
能够在 B 上 POST 的 Flash
+
SPI_B_1.bin
```

装到 A 上。

结果 A 仍然无法 POST。

当时这个实验合理地提示：

> A 可能已经叠加了固件之外的板级故障。

后来 B 已经获得真正健康的成功镜像。

再次测试 A 前：

- 清洁 CPU 触点；
- 清洁 CPU Socket；

随后写入 B 成功镜像。

A 立即进入：

```text
与 B 首次修复成功时相同的 POST / 初始化画面
```

因此最终解释：

A 实际叠加两个独立问题：

### 原始问题

```text
ME persistent state 故障
→ POST freeze
```

### 后期新增问题

```text
CPU / Socket 接触异常
→ 黑屏 + 风扇全速
```

所以早期：

```text
B donor → A 仍黑屏
```

并不是对 ME 根因的反证。

---

## 11.2 A 最终个性化镜像

B 成功镜像只能用于 A 的：

```text
生死诊断
```

不能长期保留，因为 B 的：

- MAC；
- Serial；
- DMI；
- Descriptor；

不是 A 的。

最终 A 拼接结构：

```text
0x000000–0x00AFFF
A 原始 Descriptor + GbE/MAC + PDR

0x00B000–0x25FFFF
B 成功镜像中的 HP OEM ME
本质上来自 HP 官方健康 OEM ME

0x260000–0x261FFF
A 原始 BIOS runtime / DMI / 身份

0x262000–0x3FFFFF
B 成功镜像中的 HP 官方 BIOS v1.27 主体
```

A 原始 SHA-256：

```text
209577D93CA7B70DE336864A4B771499C7234318955B9C5F1571697E9B9991E4
```

B 成功 donor SHA-256：

```text
2F42B1A1B12F0D85B8DE4A8546687EBD6A11447AE91FC3FE3E88992EE8927123
```

最终输出：

```text
DC7900_A_FIXED_v1.27_HP_OEM_ME.bin
```

大小：

```text
4,194,304 bytes
```

SHA-256：

```text
8AFDA8295E4CE3325DD3FD409B444655CDE9D4E0F5B3FF8D79CFB301E2978683
```

MD5：

```text
0AC25FEB2BE91CD15814B710D890EB73
```

脚本同时验证 A 的：

```text
MAC:
00:22:64:XX:XX:XX

Product:
FX808PA#***

Serial:
JPA84*****

Model:
HP Compaq dc7900 Ultra-Slim Desktop
```

均得到保留。

最终镜像实机写入后：

**A 正常运行。**

项目至此完成两机闭环。

---

# 12. 最终故障机制模型

## 12.1 最小故障链

基于两机实验，可以建立：

```text
CPU / 内存 / 基础 BIOS 初始化正常
        ↓
POST 进入后期平台初始化
        ↓
BIOS / MEBx 通过 HECI 与 ME 交互
        ↓
ME 读取 / 建立 EFFS、NVAR、OEM persistent state
        ↓
故障机：
关键 configuration / state 结构异常
FITC: ME_CFG_DEF NVAR Not found
        ↓
ME / BIOS 后期初始化无法完成
        ↓
屏幕停在“内存计数已经完成”的最后 UI 状态
        ↓
F10 仍被捕获
右下角显示 SETUP
        ↓
但无法真正进入 Setup
```

因此更准确的故障名称可以写为：

> **Intel ME 5.x OEM persistent configuration / EFFS/NVAR corruption induced late-POST initialization deadlock**

中文：

> **Intel ME 持久化配置 / 状态区异常导致的 POST 后期初始化死锁**

---

## 12.2 为什么不是“BIOS 坏了”

因为：

1. B 的 System BIOS 主体与官方 1.26 一致；
2. 只换官方 1.27 BIOS 主体以后仍卡；
3. A/B 静态 BIOS 主体相同；
4. 整块健康 ME 替换后故障立即消失。

因此：

```text
“BIOS ROM 主程序随机损坏”
```

不能解释实验结果。

---

## 12.3 为什么不是 ME executable code

仅把：

```text
CODE / NFTP
```

从：

```text
5.0.1.1111
```

升级到：

```text
5.2.50.1039
```

故障完全不变。

因此：

```text
ME 可执行代码版本过旧
ME CODE 随机损坏
```

同样不能解释结果。

---

## 12.4 为什么不是 HECI 物理控制器

修复后 Linux：

```text
PCI 8086:2e14
mei_me
/dev/mei0
HBM 1.0
```

均正常。

因此：

```text
ICH10 HECI hardware failure
```

与现有证据不符。

---

# 13. 为什么两台机器会在相近时期接连发生

这部分属于：

**高概率机制解释，而不是逐字节证明的事实。**

关键在于：

> 用户没有刷 BIOS，并不代表整颗 SPI Flash 十几年都不发生写入。

System BIOS 主体通常接近静态。

但 ME 的：

```text
EFFS
NVAR
persistent configuration
runtime state
```

可能在平台生命周期中持续发生变化。

两台机器具有：

- 相同平台；
- 相同 ME 5.x 架构；
- 相同 HP OEM 配置体系；
- 相近生产年代；
- 相似使用寿命；
- 可能相似的掉电 / 开关机历史。

如果失效机制包含：

- 状态累计到某个阈值；
- 动态 sector 长期擦写；
- 某次 NVAR garbage collection 异常；
- 某次持久化事务被断电中断；
- Flash 老化；
- 老 ME5 firmware 的状态管理边界条件；

那么同龄设备在生命周期后段：

```text
开始接连出现相同故障
```

是合理的。

Macronix MX25L3205D 属于这一代产品。

数据手册给出的典型耐久参数包括：

```text
100,000 P/E cycles
20 year data retention
```

但本项目并不支持：

> “单纯因为 20 年 retention 到期，所以机器坏了”

这一简单结论。

因为：

- Descriptor 大体正常；
- BIOS 主体正常；
- ME CODE 大体正常；
- GbE NVM 正常；
- 异常更集中在动态 ME state 层。

因此物理老化更可能是：

```text
助因之一
```

而不是已经证明的唯一根因。

---

# 14. 再次遇到同症状时的标准修复 SOP

## 14.1 适用症状

优先适用于 dc7900 / 786G1：

```text
POST 可以进行
内存计数完成
F9/F10/F12 可能仍然能被捕获
F10 显示 SETUP
随后机器永久冻结
不能真正进入 Setup
不能继续正常引导
```

---

## 14.2 第 0 步：恢复标准硬件基线

1. 断开 AC；
2. E49 / JP49 恢复 1–2；
3. E1 保持 Open；
4. E14 保持 Open；
5. 安装正常 CR2032；
6. 正确 Clear CMOS；
7. 移除不必要外设；
8. 保留 CPU、已知良好内存、基本显示。

如果机器已经变成：

```text
黑屏 + 风扇全速
```

不要自动继续把问题归因于 ME。

先检查：

- CPU / Socket；
- 供电；
- Reset；
- clock；
- SPI CS#；
- SPI CLK。

---

## 14.3 第 1 步：获取可信原始 dump

1. 确认 Flash 为兼容的 3.3 V / 4 MiB 器件；
2. 最好离板读取，或者使用已经验证可靠的 ISP 条件；
3. 连续 Read 至少三次；
4. 不进行 Erase；
5. 不进行 Program；
6. 每个文件必须严格：

```text
4,194,304 bytes
```

7. 三份 SHA-256 必须完全一致；
8. 保存至少两份不可修改原始证据母本。

---

## 14.4 第 2 步：检查 Region 与机器身份

确认：

```text
Descriptor
0x000000–0x000FFF

GbE
0x001000–0x002FFF

PDR
0x003000–0x00AFFF

ME
0x00B000–0x25FFFF

BIOS runtime / DMI
0x260000–0x261FFF

System BIOS body
0x262000–0x3FFFFF
```

提取并记录：

- MAC；
- Product；
- Serial；
- UUID；
- DMI。

基本原则：

> **目标机身份不换，故障 ME 状态换掉。**

---

## 14.5 第 3 步：构造“目标机 + HP 健康 ME”镜像

最小通用逻辑：

```python
out = target_original

out[0x00B000:0x260000] = hp_oem_me[0x00B000:0x260000]
out[0x262000:0x400000] = hp_bios_127[0x262000:0x400000]

# 其余区域保持 target_original
```

也就是：

```text
000000–00AFFF
保留目标机

00B000–25FFFF
健康 HP OEM ME

260000–261FFF
保留目标机

262000–3FFFFF
HP 官方 BIOS 1.27
```

如果使用本项目 B 成功镜像作为 donor，则复制以上两个区间是等价的，因为 B donor 中：

```text
ME = HP 官方健康 OEM ME
BIOS body = HP 官方 v1.27
```

---

## 14.6 第 4 步：烧录与读回

标准流程：

```text
Erase
↓
Blank Check
↓
Program
↓
Verify
↓
重新完整 Read Back
↓
SHA-256
```

Read Back 的 SHA-256 必须：

```text
与待写入镜像完全一致
```

不一致：

> **绝不上机。**

### EZP 特别注意

如果出现：

```text
能读
能 Page Program
但是 Erase 异常
```

不要立即判断 Flash 损坏。

本项目已经验证：

```text
换编程电脑
Win11 → Win10
```

可以直接解决相同问题。

---

## 14.7 第 5 步：首次启动与稳定性验证

1. Clear CMOS；
2. 恢复标准跳线；
3. 完全 AC-off 后首次启动；
4. 允许出现：

```text
162
CMOS checksum invalid
defaults loaded
```

等初始化提示；

5. 按 F1 Save Changes；
6. 确认 F10 可以真正进入 Setup；
7. 设置 Full Boot；
8. 至少执行：

```text
3 次热重启
+
3 次完整 AC-off 冷启动
```

9. 启动 Linux Live；
10. 执行 HECI / MEI 验证；
11. 最后再安装 / 启动目标操作系统。

dc7900 使用 Legacy BIOS。

Windows 安装 U 盘应优先使用：

```text
MBR
BIOS / CSM
```

模式。

---

## 14.8 若仍然黑屏：停止继续拼 BIN

在已经确认：

- 健康、同型并实机验证过的镜像；
- SPI 写入校验正确；
- 3.3 V 正常；
- WP# 不悬空；
- 跳线正确；

以后，如果机器仍然：

```text
完全黑屏
风扇全速
```

应该停止继续制造更多 BIN。

优先检查：

```text
SPI Pin1 CS#
SPI Pin6 CLK
SPI Pin2 MISO
```

的启动活动。

如果：

```text
CS# / CLK 完全无活动
```

优先检查：

- 芯片组供电；
- Reset；
- 32.768 kHz / 主时钟；
- straps；
- CPU；
- Socket。

A 机最终经历正是这个原则的实例。

---

# 15. BIN / 工具 / 哈希总表

## 15.1 原始与最终镜像

| 文件 | 大小 | SHA-256 | 定位 |
|---|---:|---|---|
| `SPI_A_1.bin` | 4 MiB | `209577D93CA7B70DE336864A4B771499C7234318955B9C5F1571697E9B9991E4` | A 原始证据母本 |
| `SPI_B_1.bin` | 4 MiB | `70B3A1229420AE9FB7363B2A96C9B4C506EA1CCE5D8A8B266DE2A012B80DB967` | B 原始故障基线 |
| `HP_786G1_v01.26_OFFICIAL_Rom.bin` | 4 MiB | `EAC16981A32DBFDC312B3C12597522F4520D5A7D16BE4F3422F2983A2652EDAF` | HP 官方 1.26 |
| `DC7900_B_v1.27_VALIDATED_BIOS_ONLY.bin` | 4 MiB | `5357E612C21655EBA27D84190343884556CE148C21A7099A37A53D6870B3F3C4` | BIOS-only 实验，仍卡 |
| `DC7900_B_v1.27_ME_5.2.50.1039_CODE_NFTP_TEST.bin` | 4 MiB | `884CC0F4C2376ED093702D9797B99E74B1DC79136C007311BD72A22BDC5A7903` | ME CODE/NFTP 实验，仍卡 |
| `DC7900_B_v1.27_HP127_ME_TEMPLATE_ONLY_TEST.bin` | 4 MiB | `2F42B1A1B12F0D85B8DE4A8546687EBD6A11447AE91FC3FE3E88992EE8927123` | B 最终成功基准 / A donor |
| `DC7900_A_FIXED_v1.27_HP_OEM_ME.bin` | 4 MiB | `8AFDA8295E4CE3325DD3FD409B444655CDE9D4E0F5B3FF8D79CFB301E2978683` | A 最终成功镜像 |
| `DC7900_B_127_CLEAN_TEST_V2.bin` | 4 MiB | `91EE3D03E1DF23D31515B7E642713BD0DE6591D05D5ADF496DBB7034D07FBE9B` | 废弃 |
| `DC7900_A_127_CLEAN_TEST_V2.bin` | 4 MiB | `1A403F5DAB7FD3631FE6BEEBE3A0343C425BCA6A84C70362BB3FE279176AF4DE` | 废弃 |

---

## 15.2 SP54355 / ME 5.2.50.1039

`sp54355.exe`

SHA-256：

```text
E713F0358B06FFFDE5AC9A0B807EF1E58054B3AAF03AB5899749DF92905182BD
```

`52501039.BIN`

大小：

```text
1,510,212 bytes
0x170B44
```

SHA-256：

```text
298087A41EDD12013C78CE149F5ECCDECC156FE4238E2C5A75800B2118C43A65
```

目标版本：

```text
5.2.50.1039
```

B 原 CODE/NFTP：

```text
5.0.1.1111
```

---

## 15.3 工具与已验证环境

| 工具 | 版本 / 环境 | 项目结论 |
|---|---|---|
| EZP2019+ | Version 2.0 / Win10 | 最终可靠烧录环境 |
| EZP2023 | V3.0 / 原 Win11 | 同样出现 Erase 异常 |
| Intel Flash Image Tool | 5.0.0.1167 | ME5 分解 / 比较；Build 输出不可直接信任 |
| Linux Mint Live | 项目现场 | HECI / mei_me / 系统启动验证 |
| Rufus | Legacy 安装介质 | Windows 使用 MBR + BIOS/CSM |

---

# 16. 禁止使用 / 高风险项

以下内容已经由项目证明不应该继续使用：

### 废弃 BIN

```text
DC7900_A_127_CLEAN_TEST_V2.bin
DC7900_B_127_CLEAN_TEST_V2.bin
```

### FITC 自动 Build 输出

任何出现未经解释的 Descriptor 改动的：

```text
outimage
```

均不得直接刷写。

### HP 官方完整 BIN 整颗覆盖

不要未经检查把：

```text
7G1_0127.bin
```

直接从：

```text
0x000000
```

整颗覆盖目标机 SPI。

否则可能丢失：

- Descriptor；
- GbE；
- MAC；
- DMI；
- Serial；
- UUID。

### B 完整镜像长期留给 A

B donor 可以用于：

```text
诊断
```

但不能作为 A 最终镜像长期使用。

### `fw_ver = 0:0.0.0.0`

不要只凭：

```text
/sys/class/mei/mei0/fw_ver
```

显示：

```text
0:0.0.0.0
```

就判断：

> ME firmware 不存在。

### 反复拆焊

出现焊盘损伤以后不要继续无目的反复拆焊。

应尽早建立：

```text
SOP16 / 飞线 / 稳定可维护 SPI 结构
```

---

# 17. 关键验证命令

## 17.1 Windows SHA-256

```powershell
Get-FileHash .\SPI_A_1.bin -Algorithm SHA256
Get-FileHash .\SPI_B_1.bin -Algorithm SHA256
Get-FileHash .\DC7900_B_v1.27_HP127_ME_TEMPLATE_ONLY_TEST.bin -Algorithm SHA256
Get-FileHash .\READBACK.bin -Algorithm SHA256
```

核心原则：

> **软件提示 Verify OK 以后仍然执行完整 Read Back，并对整份文件计算 SHA-256。**

---

## 17.2 Linux HECI / MEI

```bash
lspci -nnk -d 8086:2e14
lsmod | grep -E 'mei|heci'
ls -l /dev/mei*
cat /sys/class/mei/mei0/dev_state
cat /sys/class/mei/mei0/fw_ver
cat /sys/class/mei/mei0/fw_status
cat /sys/class/mei/mei0/hbm_ver
```

本项目 B：

```text
PCI: 8086:2e14 rev03
Subsystem: HP 103c:3036
Kernel driver: mei_me
/dev/mei0: present
dev_state: ENABLED
fw_status: 300A965A
hbm_ver: 1.0
```

---

# 18. 最小拼接逻辑

以下代码只用于解释最终镜像结构。

它不替代正式的带源文件哈希和逐区验证脚本。

```python
from pathlib import Path

target = bytearray(Path("target_original.bin").read_bytes())
donor = Path(
    "DC7900_B_v1.27_HP127_ME_TEMPLATE_ONLY_TEST.bin"
).read_bytes()

assert len(target) == len(donor) == 0x400000

target[0x00B000:0x260000] = donor[0x00B000:0x260000]
target[0x262000:0x400000] = donor[0x262000:0x400000]

Path("target_fixed.bin").write_bytes(target)
```

---

# 19. A 最终脚本与验证

最终 A 构建脚本并不是按照文件名盲目选择源文件。

它按照：

```text
SHA-256
```

识别输入。

A 原始要求：

```text
209577D93CA7B70DE336864A4B771499C7234318955B9C5F1571697E9B9991E4
```

B donor 要求：

```text
2F42B1A1B12F0D85B8DE4A8546687EBD6A11447AE91FC3FE3E88992EE8927123
```

最终脚本执行后进行了严格的逐字节来源验证：

```text
[PASS] 000000-00AFFF = A original Descriptor/GbE/PDR
[PASS] 00B000-25FFFF = B validated HP OEM ME
[PASS] 260000-261FFF = A original runtime/DMI
[PASS] 262000-3FFFFF = B validated BIOS v1.27 body

[PASS] A GbE MAC retained: 00:22:64:XX:XX:XX
[PASS] Identity retained: FX808PA#***
[PASS] Identity retained: JPA84*****
[PASS] Identity retained: HP Compaq dc7900 Ultra-Slim Desktop
```

最终输出：

```text
[SUCCESS] FINAL IMAGE BUILT AND VERIFIED
```

输出文件：

```text
DC7900_A_FIXED_v1.27_HP_OEM_ME.bin
```

SHA-256：

```text
8AFDA8295E4CE3325DD3FD409B444655CDE9D4E0F5B3FF8D79CFB301E2978683
```

---

# 20. 项目证据等级

为了避免把推断写成事实，本项目区分三个等级。

## A — 直接实验支持

例如：

- BIOS-only 1.27 仍卡；
- ME CODE/NFTP 替换仍卡；
- 完整 HP OEM ME 替换后 B 恢复；
- A 在排除 CPU 接触问题后重复恢复；
- Linux HECI / MEI 通信正常。

## B — 多个独立证据强烈支持

例如：

```text
EFFS / NVAR / OEM configuration
```

是故障核心区域。

虽然没有逐个 NVAR 完成最终法医级隔离，但：

- FITC；
- 单变量实验；
- 整块 ME 替换；
- 两台机器重复结果；

都指向同一结论。

## C — 合理机制解释

例如：

> 为什么多台同龄 dc7900 会在十几年后接连出现相同问题。

这里涉及：

- Flash 老化；
- NVAR 累积；
- garbage collection；
- transaction interruption；
- ME5 边界条件；

但目前没有证据可以确定其中哪一种是唯一触发机制。

---

# 21. 尚未解决的研究问题

维修层面已经闭环。

如果继续进行更底层逆向研究，仍有以下问题：

1. `ME_CFG_DEF` 具体为什么从故障镜像中无法解析；
2. 哪一个 NVAR 对象是真正触发 POST deadlock 的最小条件；
3. EFFS 中是否存在固定损坏模式；
4. A/B 故障 ME 是否存在同一组可重复异常字节；
5. 老化 Flash 是否只是背景因素，还是参与了状态失效；
6. 相同 dc7900 网络案例是否能够通过 dump 形成第三、第四个样本；
7. 是否能够只重建 ME persistent state，而不替换整个 OEM ME Region。

这些问题已经属于：

```text
ME5 firmware forensic analysis
```

而不是完成机器维修所必须继续解决的问题。

---

# 22. 最终结论

两台 HP Compaq dc7900 的共同原始故障最终被同一种方法重复修复：

```text
保留目标机器身份区
+
重置为健康 HP OEM ME Region
+
使用 HP 官方 System BIOS v1.27 主体
```

B 机建立了主要根因证据链：

```text
System BIOS 主体
→ 排除

ME executable CODE/NFTP
→ 排除

ME persistent state
→ 高度锁定

完整 HP OEM ME
→ 修复成功
```

A 机则进一步说明：

```text
同一种 ME 状态故障
```

可以与另一个完全独立的：

```text
CPU / Socket 接触故障
```

同时存在。

因此：

**相同机器出现相同症状，并不意味着维修过程中后来出现的所有新症状仍然只有一个根因。**

本项目到此已经在维修层面完成闭环。

后续工作重点不再是：

> 能不能把机器救活

而是：

> 能不能进一步把 ME5 EFFS / NVAR 中的最小故障对象精确定位出来。

---

## 项目状态

**COMPLETED / VALIDATED ON TWO MACHINES**

```text
Machine B:
Recovered

Machine A:
Recovered

F10 BIOS Setup:
Working

Operating system boot:
Working

Original POST freeze:
Not reproduced after repair

2233 / 2206:
Not reproduced during subsequent observation
```
