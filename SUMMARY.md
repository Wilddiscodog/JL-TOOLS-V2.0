# xiaolu-r2 (AC7911AB) 逆向与烧录工具 —— 信息汇总

> 更新日期：2026-09-24
> 工作目录：`J:\xiaolu-r2\pc-tools`（本文档所在地）、`D:\fw-AC79_AIoT_SDK`（杰理 SDK）

---

## 1. 设备与芯片信息

| 项目 | 值 |
|---|---|
| 设备代号 | xiaolu-r2（云知声 Unisound 语音方案产品） |
| 芯片 | AC7911AB（杰理 AC791N / wl82 平台，pi32v2 双核 DSP，最高 320MHz） |
| 固件 PID | `AC791N_STORY`（杰理 SDK 默认值，厂商未修改） |
| 固件版本 | 0.01 |
| Flash | SPI NOR 16MB，JEDEC ID `0x684018` |
| SDRAM | 8MB（来自 flash 明文配置 `ENABLE_SDRAM=1, SDRAM_SIZE=0x800000`） |
| Flash UUID | `50 03 23 21 78 6E B2 1C D6 32 06 66 5D 23 5B 51` |
| 芯片版本 | B |
| **芯片 key** | **已烧录，值 = `0x7DF1`**（2 字节；2026-10-19 经 JlFlash loader FC09 实测读取：raw[6..7] 孟黎 CrcCipher 解码，多次上电稳定）。**ROM 强制 eFuse 解密：固件必须用 0x7DF1 派生密钥流加密才能启动**（2026-10-19 实测：默认流镜像不启动，recrypt 后正常运行） |
| 蓝牙名 | `EFBB10607280010426` / `EFBB1060728001042673W` |
| 芯片 MAC | `38:4B:5B:9A:D6:7B` |

---

## 2. Flash 布局（16MB 整片 dump 分析）

备份文件：`J:\xiaolu-r2\device-flash-bin\flash_dump_20260924_092843.bin`

| 地址范围 | 内容 | 状态 |
|---|---|---|
| `0x000000–0x240000` (2.25MB) | 固件区：isd 头 + uboot + app + cfg | 加密（**chipkey=0x7DF1 派生密钥流**，ROM 强制解密；生成器非线性、非孟黎算法，见 §4/§14） |
| `0x240000–0x248000` (32KB) | 加密 blob（预留/备份区） | 加密 |
| `0x248000–0x4F0000` | 空（全 0） | — |
| `0x4F0000–0x500000` (64KB) | **VM 配置区（明文）** | 明文 |
| `0x500000–0xED0000` (9.8MB) | 资源区（提示音等） | 加密 |
| `0xEE0000–0xFF8000` | 空 | — |
| `0xFF8000–0xFF8080` | 出厂设备信息块（MAC/蓝牙名） | 明文 |

### VM 区明文发现（0x4F0000）
- `Units_of_Unisound` / `Unauthorized Version of Unisound`（云知声 SDK，未授权版标识）
- 产品 ID `hyundai415`，序列号 `3fa7fu9h`，日期 `20170101`，热点名 `PDCN`
- 存储的 WiFi 凭据（WPS TLV）：ZTE `ZXHN Fxxxx` / SSID `Fxxxx` / 密码 `12345678`；AP `CU_y27y`
- base64：`VW5pdHNfb2ZfVW5pc291bmQ=` = "Units_of_Unisound"

### 明文系统配置（固件区 0x3700–0x3A50，与 isd_config.ini 字段对应）
- SPI flash：`2_1_0_0`；`ENABLE_SDRAM=1`，`SDRAM_SIZE=8MB`
- `VDDIO=7`(3.5V)、`DVDD=13`(1.32V)、`DCDC14=3`(1.40V)
- `UTBD=1000000`（uboot 调试串口 1M）、`RESET=PB01_08_0`
- **原厂调试串口使用 PB2/PB8**（用户确认）

---

## 3. CPU 架构

- 杰理自研 **pi32v2** 指令集（非 ARM/MIPS/RISC-V），只能用杰理定制 LLVM（`C:\JL\pi32`）编译
- 特征：16/32 位变长指令、`r0`–`r15` + `ssp/sp/rete/reti`、DSP 指令（`rep`、`h[]`、`btbclr`）
- 反汇编方法：clang `-target pi32v2` 把裸 bin 包成 ELF（`.incbin` + `.section .text,"ax"`），再 `llvm-objdump -D --triple=pi32v2`
- 已反汇编：`uboot.boot`（未加密）、`uboot.boot_debug`、MaskROM 运行区（见 §6）

---

## 4. 固件加密机制（逆向结论）

1. **打包链**：`sdk.elf` → objcopy 抽节 → `app.bin` → (可选 LZ4 压缩，`app.lz4`) → `isd_download.exe` 按 `isd_config.ini` 分区打包并加密 → `jl_isd.bin`（= flash 镜像）→ `fw_add/ufw_maker` → `jl_isd.fw/.ufw`
2. **固件体加密 = 固定密钥流 XOR**：`密文[i] = 明文[i] ^ KS[i]`，密钥流与地址/长度/内容无关，**按区块类型固定**（uboot 块、app 块各一条），编译在 isd_download 内
3. 密钥流生成器为非线性算法（BM 线性复杂度打满、非 MINSTD/LFSR/crc_cipher），带 **0x400 准周期**结构；已实测提取 **app 块密钥流 2.3MB**（2026-10-19 确认：该流即 **chipkey=0x7DF1 派生流**——recrypt 烧录可启动、默认流不启动）（`Temp\wl82dis\KS.bin`）
4. 明文先经**杰理私有 LZ4 打包**压缩再加密（`uboot_lz4.exe`：16 字节头 = 8B 保留 + 4B 入口地址 + 4B 原始大小 + LZ4 块；支持字典）
5. **线上传输加密 = 孟黎密码**（`jltech/cipher.py`）：`crc_cipher`——CRC16(poly 0x11021) 链式状态 + 魔串 **"孟黎我爱你，玉林"**(GB2312: `C3 CF C0 E8 CE D2 B0 AE C4 E3 A3 AC D3 F1 C1 D6`)，key 默认 `0xFFFFFFFF`；另有 `enc_cipher`（16 位 LFSR）和 `rxgp_cipher`（MINSTD）
6. **AES-128** 存在于 isd_download（S-box 在 .data 0x108a020，密钥扩展在 0x455750），用于密钥文件处理而非固件体

### 密钥文件体系（isd_download 4.2.74，UPX 壳，Qt 编写）
- `key2018/2021/2022/2024.bin`、`v2mkey.bin`、`-key / -key1+mkey / -v2mkey / -use-chip-key`
- `-use-chip-key`（从芯片读 key 并加密镜像）**需要杰理授权**（机器码验证），不可用
- 本设备的芯片已烧 key。isd_download 的 "Key mismatch/ERR_KEY" 在**芯片在线时是硬性要求**（须提供匹配的 -key 文件做 recrypt）；`-use-chip-key` 需杰理授权不可用，但**自行生成 key 文件可绕过**（见 §14，已实测）。**默认流镜像不能启动**（2026-10-19 实测两次）

---

## 5. USB 下载协议（参考 kagaimiq/jl-uboot-tool，已实测验证）

### 传输层
- 芯片下载模式枚举为 **USB Mass Storage**，SCSI Inquiry：`WL82 / UBOOT1.00 / 1.00`
- Windows 通过 `\\.\K:` 卷 + `IOCTL_SCSI_PASS_THROUGH_DIRECT` 直发 16 字节 CDB
- 命令格式：`[opcode][sub-opcode][args...]`，尾部 0xFF 填充到 16 字节

### 三层协议
| 阶段 | 实现 | 命令 |
|---|---|---|
| ① ROM (UBOOT1.00) | 芯片内置 | `FB06` 写内存（wl82 上 ROM 原地解密孟黎密文）、`FD07` 读内存（返回孟黎密文视图）、`FB08` 跳转（带 16bit arg：bit0-3 目标介质/1=SPI NOR，bit4-11 分频，bit12-15 SPI 模式） |
| ② loader 下载 | — | `wl82loader.bin`（本身就是孟黎密文）按 **512B 块**写 **0x1C02000**，然后 `FB08` 跳转，实测 arg=`0x0011` 可用 |
| ③ 烧录 (Loader v2) | wl82loader 运行后 | 见下 |

### Loader v2 命令集（wl82 实测状态）
| 命令 | 功能 | 实测 |
|---|---|---|
| `FB00 addr(4)` | 擦 64K 块，结果在 resp[2] | ✅ |
| `FB01/FB02` | 擦扇区/整片 | 未测 |
| `FB04 addr(4) len(2) 00 crc16(2LE)` + data | 写 flash | ✅ |
| `FD05 addr(4) len(2)` | 读 flash | ✅（用于校验） |
| `FC0A` | 在线设备：resp[2]=类型(3=SPI0 NOR)，resp[4:8]=JEDEC | ✅ |
| `FC09 addr(4)` | 读 eFuse 字（**addr=字索引 0~8**），resp[4:6]=孟黎密文 key，尾带会话混淆流 | ⚠️ 可读但解码未定 |
| `FC14` | USB 缓冲区大小（实测 5372） | ✅ |
| `FC0C arg(4)` | 复位/跑 app | ✅ |
| `FC0B/FC13/FC16/FC15/FD07` | 读 ID/CRC16/MaskROM ID/版本/读内存 | ❌ 会令芯片崩溃失联，**禁用** |

### 已知坑
- loader 运行中会**随机 USB 重枚举**（错误 121 超时 / 1167 设备断开），每条命令失败后需重新扫描盘符、重开句柄重试（JlFlash 已实现）
- 复位时若 USB 数据线连接，芯片可能重新进入下载模式而不启动 flash 固件 → **测试运行必须断开数据线、用充电线供电**

---

## 6. MaskROM / RAM dump

| 区域 | 地址 | 说明 |
|---|---|---|
| MaskROM 运行代码 | `0x1E00000`（约 7KB 有效） | ROM 模式 FD07 可直接 dump（文件 `wl82_rom.bin`） |
| SDRAM 起始 | `0x1C00000` | loader 加载区（0x1C02000） |
| RAM 数据页 | `0x00000000` | MSC 描述符 "USBS UBOOT1.00" |
| loader 解压后代码 | — | FD07 在 loader 模式下会崩溃，尚未 dump 成功；loader 文件本体为"解压 stub + 私有压缩体"（熵 7.99，无标准 LZ4 帧） |

---

## 7. C# 烧录工具 JlFlash（`J:\xiaolu-r2\pc-tools\JlFlash`）

.NET 8 (`net8.0-windows`)，零第三方依赖。

```
Transport/ScsiTransport.cs   IOCTL_SCSI_PASS_THROUGH_DIRECT，16B CDB
Protocol/JlCrc.cs            CRC16(0x11021 非反射) / CRC32
Protocol/MengLi.cs           孟黎密码（crc_cipher / enc_cipher）
Protocol/UbootRom.cs         ROM 层：FB06(写内存,512B块,孟黎线上加密) / FD07 / FB08
Protocol/LoaderV2.cs         Loader 层全部安全命令
Flow/FlashFlow.cs            握手→下loader→跳转→擦→写→FD05校验→复位，带重枚举自动恢复
```

### 命令
```bash
cd J:\xiaolu-r2\pc-tools\JlFlash\bin\Debug\net8.0-windows
JlFlash.exe list                          # 扫描设备
JlFlash.exe scan                          # 列出所有卷的 SCSI Inquiry
JlFlash.exe flash <jl_isd.bin>            # 烧录（FD05 逐512B校验）
JlFlash.exe flash <jl_isd.bin> --no-verify
JlFlash.exe readkey                       # 读 eFuse chipkey（解码存疑，见 §8）
JlFlash.exe readmem <addrHex> <len>       # ROM 模式读内存（dump MaskROM 用）
JlFlash.exe probe                         # FC09 各字索引原始应答
```

### 标准烧录流程
```bash
# 1. SDK 编译（设备不要接：在线已烧 key 芯片会触发 ERR_KEY 硬拦截；离线打包的 jl_isd.bin 仍为默认流，**烧录时须带 -key 做 recrypt**，否则不启动）
cd D:\fw-AC79_AIoT_SDK && tools\utils\make.exe ac791n_demo_helloworld
# 2. 板子进下载模式（插 USB 数据线）
JlFlash.exe flash D:\fw-AC79_AIoT_SDK\cpu\wl82\tools\jl_isd.bin
```

---

## 8. eFuse 读取（✅ 已解决，2026-10-19）

- `FC09` 的 address 参数 = **eFuse 字索引（0~8）**，与 SDK `efuse.h` 的 getter 一一对应（chip_id / vbat_trim / vbg_trim / lrc_ws_inc / lrc_ws_init / btosc_ws_inc / btosc_ws_init / lrc_change_mode / boot_flag）；`0xAC6900` 魔数返回值与 word6 相同
- 每次会话可读出全部 10 个 16 位字的**原始密文**，但 resp[4:6] 的孟黎解码**跨会话不一致**（三次会话得 0x90F0 / 0x164B / 0x30B0）——应答外层有按会话/调用序推进相位的混淆流
- 破解路径（按优先级）：
  1. **烧录含 `boot_info` 打印的 helloworld**（已内置：chip_id / trim_value 会经 UART 输出）→ 用 chip_id 真值校准 FC09 解码 ← 当前方案
  2. dump loader 解压后代码分析 FC09 应答构造（FD07 在 loader 模式崩溃，待解决）
  3. 分析 MaskROM 的 efuse 例程（7KB 已 dump：`wl82_rom.bin`）
- 注意：本芯片已烧 key，若 key 被读保护，FC09 返回的可能是屏蔽值

### ✅ 解决（2026-10-19，JlFlash 修复解码后实测）
- **解码修正**：key 不在 loader 应答的 `resp[4:6]`（那里混入了按会话推进的相位字节，即上面三次不一致的根因），而在 **ROM 原始 16 字节应答的 `raw[6..7]`**（大端对）做孟黎 CrcCipher 解码——**跨会话稳定**（多芯片多次上电复验）
- 实测真值：**AC7911(xiaolu-r2) = `0x7DF1`**（4 次会话稳定）；**AC6956C(词易帮 Y20) = `0xB165`**（2 次会话稳定）
- 工具：JlFlash `FlashFlow.ReadChipKey` 已修复解码并新增 `ReadChipKeyAndReset`（读完 FC0C 复位退出 UBOOT，避免挡住后续打包）；仓库 JL-TOOLS 已同步

---

## 9. 调试串口（当前待验证）

- 固件已改为 **UART1，TX=PB8**（原厂调试引脚），1000000 波特率，8N1
- **测试必须断开 USB 数据线、用充电线供电**，否则芯片复位后重新进入下载模式，串口无输出
- 已烧录"**全端口扫描固件**"（app_main.c 的 scan_table）：循环轮询 17 种串口映射（UART0/1/2 全部固定组 + PB3/PB8 重映射），每种打印 `>>> I_AM_HERE: <名字> <<<` 8 行，约 1 分钟一轮——接任意候选引脚都能看到自己的身份，一次定位调试口
- 若所有配置均无输出 → 固件未运行，问题转向"芯片 key 拦截启动"
- 注意：原厂产品用 PB2/PB8；PB2 在 UART0 固定映射里是 RX（TX=PB1），PB8 不在任何固定映射组，只能走 REMAP 输出通道

---

## 10. SRAM 运行（不擦除 FLASH）—— runram

> 2026-10-14 新增。目标芯片：AC695N（BR23 平台，RAM 0x0~0x30000 ≈192KB，代码 XIP 于 0x1E00100）。

原理：UBOOT ROM 的 FB06/FD07/FB08 直接操作 RAM，**全程不执行任何擦写 FLASH 命令、不下载 wl82loader**，芯片原固件不受影响，断电重启即恢复原固件。

### 上位机新命令（JlFlash，已实现）
```
JlFlash.exe runram <payload.bin> [addrHex=10000] [--arg=XXXX] [--mailbox=XXXXXXXX] [--enc|--plain] [--no-verify]
JlFlash.exe kick        # 单独执行：把运行中的芯片踢进下载模式
```
- **runram 自动处理进模式**：已有 UBOOT → 直接用；只看到 UDISK/DEVICE（固件 U 盘）→ 自动发 SCSI 0xFB 踢重启进下载模式（对应 SDK msd_upgrade.c 的 go_mask_usb_updata）再继续
- 自动探测线上格式：先明文+回读校验，失败自动换 crc_cipher（chips.yaml 证实 br23 也是 memory-rw-mengli-crypt: yes，预计会选中密文模式）
- FB06 写 512B 块 → FD07 逐块校验 → FB08 跳转（arg 默认 0）
- 跳转后尝试读 RAM 邮箱（默认 0x2FC00，魔数 "JLRM"）验证载荷确实执行过
- 注意：Program.cs 已从 GBK 转为 UTF-8(BOM) 保存

### GUI 版（JlFlashGui，2026-10-14 更新）
- 结构：共享流程类 `JlFlash/Flow/SramRun.cs`（EnterDownloadMode + 探线密 + RunPayload，CLI/GUI 共用）
- GUI 三个分组：设备（扫描/kick/停止）、**AC695N SRAM 运行**（载荷路径默认指向 SDK apps/hello_ram/hello_ram.bin、加载地址/邮箱/arg/线密模式、"SRAM 运行"一键按钮）、wl82 原有功能（烧录/恢复 dump/ChipKey/loader 路径 RAM 运行）
- 含中文的源码文件必须存为 UTF-8 **with BOM**（否则中文 Windows 的 csc 按 GBK 误读）
- 启动：`JlFlashGui\bin\Debug\net8.0-windows\JlFlashGui.exe`

### 操作顺序（runram 标准流程）
1. 板子断电 → 插 USB 数据线 → 上电（是否先开电脑软件无所谓，命令会等设备）
2. `JlFlash.exe runram hello_ram.bin` 或 GUI 点「SRAM 运行」（bin 已复制在 JlFlash 输出目录）
3. 结束后拔线、断电重上电 → 原固件照常启动

### br23 载荷工程（J:\witerpaper\AC695N_Soundbox_SDK-master\SDK\apps\hello_ram\）
- freestanding C，杰理 clang 编译（-target pi32v2 -mcpu=r3），pi32v2-ld 链接，`make` 出 `hello_ram.bin`（182B）
- `payload_main` 必须位于加载基址（`__attribute__((section(".text.entry")))` 在**定义**上 + ld 脚本 KEEP，否则入口错位）
- 载荷行为：写邮箱（JLRM+长度+字符串）→ 若 UT0(PB04@1M，ROM 下载模式已配好) TXEN 已开则 UART 打印 → 返回
- br23 UART0 寄存器：CON0@0x1E2000（BIT0=TXEN、BIT15=TX完成、BIT13=w1c）、BUF@0x1E2006（参考 SDK uart_dev.c:82-86）
- 加载地址 0x10000 为保守选择（避开低端 ROM 状态区、0x2BDC0 矢量表、0x2FF80 UPDATA 区）；isd 的 uboot 加载地址未确认，冲突时换 `runram xxx.bin 8000`

### 两种跳转结果的处理
- FB08 返回（call 语义）→ 上位机读回邮箱打印 Hello World，芯片继续留在下载模式
- FB08 不返回 → 设备消失（正常现象），断电重启即回原固件；串口 PB04@1M 已能看到打印

---

## 10.5 AC695N 屏幕测试工程 + BR23 烧录（2026-10-14）

### SDK 侧：apps/soundbox/lcd_test/
- 外接 ST7305 miPis 屏（300x400 单色，FD042），原厂 Arduino 例程：`J:\witerpaper\fd042-mn_-st7305-v0.1`
- 接线 = **SPI2 port A 硬件映射**（cpu/br23/spi.c spi_io_map）：**PB9=SCL(CLK)、PB10=SDA(DO)、PB8=MISO(DI，W25Q 用)**、**PC2=CS(GPIO)**、**PA10=RS(D/C)**；LCD 无独立复位线（软复位 0x01）。**PB11 = 芯片 SDPG（SD 卡电源闸）**，由 SD 驱动 `sd_set_power()`（sdmmc 平台数据 .power 钩子）控制，曾被误配为 LCD RST 已修正
- **W25Q128 (16MB) 同挂 SPI2**：CS=**PA0**（驱动 w25q128.c：JEDEC ID/读/页编程/扇区/块/整片擦除，开机仅探测 ID 不做数据操作）；demo 板 PA0 的 CTMU 触摸/段码屏/SPI-LCD 框架全是关闭态，无冲突
- **TF 卡 = SD1 'A' 组（CMD:PC4 CLK:PC5 DAT0:PC3），1-bit 模式**：board_ac695x_demo_cfg.h 里 SD0 禁用（原 'B' 组引脚正是 PB8/9/10，与 SPI2 冲突）、SD1 启用；LED7 UI 引脚自动从 PC 组切到 PA02-09
- st7305_lcd.c 里补了 spi0_p_data 占位定义（spi.c 查表需要；换板需删）
- 固件已并入 app_main（lcd_test_init()），上电循环显示：黑→白→条带→条纹→反色，2s/步
- 产物：`cpu/br23/tools/download/standard/jl_isd.bin`

### 上位机：BR23 烧录支持 + SRAM 读保护（2026-10-14 下午）
- **BR23 read-safe mode**（依据 BR23工具包 实测：UBOOT1.00 ROM 的 FD07 读 >16B 或连续读会挂设备）：
  - Inquiry vendor=="BR23" 时自动启用：跳过探线密（直接用已知孟黎模式）、跳过逐块校验、邮箱只做**单次 16B 读**
  - 载荷邮箱格式改为 16B 窗口友好：`[0]='J' [1]='L' [2..]=消息(NUL 结尾)`，hello_ram.bin 消息缩为 "Hello, World!"（13 字符恰好在 16B 内）
  - 非 BR23（wl82）行为不变
- hello_world 功能已从 SDK 固件移除（app_main/Makefile/cbp/目录全清，2026-10-14 16:04 的 jl_isd.bin 已无 [HELLO] 输出）

### 上位机：BR23 烧录支持
- FlashFlow 增加 `Chip` 属性：Br23 用 kagaimiq 的 `br23loader.bin`(11KB, MengLi 存储) @ **RAM 0x12000**（与 isd_download 的 `-boot 0x12000` 互证）；wl82 用 wl82loader @ 0x1C02000
- **SDK 自带 cpu/br23/tools/br23loader.bin(24KB) 是 UART 版，不能用于此 USB 流程**
- CLI：`JlFlash.exe flash <jl_isd.bin> --br23`（flash/runram 都会自动进下载模式）
- GUI：AC695N 分组第三行「固件: [jl_isd.bin 路径] [浏览] [烧录到 FLASH]」，默认路径指向 SDK 产物

### 打包坑（2026-10-14 实录）
- **打包时不能有 JL 设备/U 盘占卷**：isd_download.exe 会探测可移动卷，K: 上有 UBOOT 设备（或空读卡器）时直接中止（"获取设备芯片KEY失败/错误=1167"）→ 拔掉再跑
- 打包失败的残留 jl_isd.fw 会报"fw文件中已经存在要添加的文件"，先删 download/standard/*.fw
- 编译命令：`cd download/standard && cmd //c download.bat`（等价于 make 的后处理）
- ⚠️ 改 isd_config.ini 禁止用 Python decode/encode 全文重写（编码转换失败会把文件截断成 0 字节）；要改就整文件字节级替换。已从上下文备份恢复（4280B，原 4281B 差一个 EOF 换行）

---

## 12. 更换基线 SDK（2026-10-15）

**新基线 = `J:\witerpaper\QQ-SDK\AC695N_SDK\SDK`**（QQ 群友 loooo 对同一硬件 AC6956C8 学习机的移植版，LCD 已点亮、9/5 有成功构建）。

### 旧 SDK 不启动的根因（推断确认）
- ~~旧镜像 `SUPPORTED_LIST=A,B,D`，QQ SDK = `A,B,D,E`~~【已证伪：NES 工程 A,B,D 也能跑，芯片版本在 A/B/D 内】
- ~~SPI 2 线配置不匹配~~ 已排除：QQ SDK 同样用 `SPI=2_3_0_0` 且能启动

### 真正的启动失败根因（2026-10-15 最终确认）
- **芯片 eFuse 已烧 key 0xB165**（词易帮拼音机 Y20，片内 1MB NOR，flash ID 0xEB6014 为片内而非外部）
- 任何自编译镜像打包时必须用 `0xB165.key` 加密：download.bat 加 `-key . xB165.key`（key 文件已复制到 download/standard/）
- 之前所有"烧录成功但不启动"（JlFlash 校验通过但芯片不解密）均由此导致；与 SPI 配置、版本白名单无关
- NES 工程（`J:\witerpaper\QQ-SDK
es\Jieli-AC6956C`）= 同硬件的完整可用固件（NES 模拟器），其 isd/download.bat 含正确的 key 用法，可作打包范本

### 日志模式开关（2026-10-16 v4）
- 根因闭环：控制台死于设备表 "otg" 设备打开（dev 框架挂载阶段把 DP/DM 配成 USB 角色检测），日志精确消失于 lrc cfg 之后
- 新增：上电时按住 S3（≥300ms 检测窗）→ 日志模式：`otg_data.usb_dev_en=0`（OTG/USB 设备不初始化，DP/DM 保持串口全程不断）
- 日志模式下 S3 三秒进 uboot 的触发被抑制（避免打断日志会话）；要进 uboot 先断电不按 S3 正常重启
- 调试串口已从 PB04@115200 改为 **DP(TX)/DM(RX) @1M**（uart_dev.c UT1 通道原生支持 DP/DM）
- 代价：日志模式下插电脑不枚举 U 盘（拷 NES ROM 需正常模式上电）

### QQ SDK 关键信息（HARDWARE_WIRING.md = 板子说明书）
- 芯片 AC6956C8 QFN32；屏 = Waveshare 4.2" ST7305（物理 400x300，逻辑旋转 300x400）
- **PB11 = POW_FLS**（Flash/TF/LCD 共享电源+MIC 偏置），上电时序：PB0 拉高关功放 → PB11 稳定 100ms → SPI2 初始化
- 功放 AXS2005B：PB0 低电平使能、DACL 单端 `DAC_OUTPUT_MONO_L`、SPI1-A 必须禁用（CLK 复用 PB0）
- 触摸 CST836U 等多协议识别（软 I2C PA1/PA9，INT=PC7），状态"排查中"
- ST7305 驱动在 `apps/soundbox/ui/lcd_simple/my_demo.c`（3676 行，帧缓冲+脏区刷新+动画）
- 板级配置与我们此前结论一致：SD0 禁/SD1 启(PC3-5)、ADKEY=PB1、IOKEY 关

### 已移植到 QQ SDK（2026-10-15）
- `apps/soundbox/lcd_test/w25q128.c/h`：**w25q_init() 不再 spi_open/spi_set_baud**（SPI2 总线由 my_demo.c 持有 @其波特率，重设会拖慢 LCD），只在 learning_board_init 之后读 JEDEC
- `apps/soundbox/lcd_test/uboot_key.c`：S3 长按 1s → go_mask_usb_updata() 进 uboot
- `app_main.c`：learning_board_init() 之后调用 uboot_key_init() + w25q_init()
- Makefile/cbp 已注册；构建通过，jl_isd.bin 942080B @10-15 09:46（app.bin 内确认含 "S3 held 1s"/"flash not responding" 字符串）
- ⚠️ 经验：改 Makefile/ini 等含 `\` 续行符的文件，python 脚本必须用显式字节值(bytes([0x5C]))，heredoc 反斜杠会被转义层吃掉

---

## 13. 关键文件索引

| 文件 | 位置 |
|---|---|
| flash 整片 dump | `J:\xiaolu-r2\device-flash-bin\flash_dump_20260924_092843.bin` |
| C# 烧录工具 | `J:\xiaolu-r2\pc-tools\JlFlash\` |
| MaskROM dump (7KB) | `J:\xiaolu-r2\pc-tools\wl82_rom.bin` |
| uboot 反汇编 | `D:\fw-AC79_AIoT_SDK\cpu\wl82\tools\uboot.boot.disasm.txt`（及 `uboot.boot_debug.disasm.txt`） |
| isd_download 脱壳 | `D:\fw-AC79_AIoT_SDK\cpu\wl82\tools\isd_download_unpacked.exe`（反汇编见 `Temp\wl82dis\isd.asm`） |
| app 块密钥流 (2.3MB) | `%TEMP%\wl82dis\KS.bin` |
| 协议参考 | https://github.com/kagaimiq/jl-uboot-tool （docs/usb-loader-v2.md、jltech/*.py） |
| helloworld 工程 | `D:\fw-AC79_AIoT_SDK\apps\demo\helloworld\`（当前为 UART 全端口扫描版） |


---

## 11. 0xB165.key 解析与固件解密结论（2026-10-16 新增）

### 0xB165.key 是什么：杰理 chipkey 文件（已完全解出）
- 格式（参考 kagaimiq/jl-misctools `keyfile/keyfparse.py`，72 个 ASCII hex 字符）：
  - `[0:64]` = 32 字节数据：前 16B = AES-ECB 密文，后 16B = AES 密钥
  - `[64:72]` = CRC32（poly 0x104C11DB7 反射、init 0x26536734）校验值 → 本文件校验 **MATCH**（418F99A8）
- 本文件 AES 密钥 = `6efec7fa9f2216446efec7fa9f221644`（两个相同 8B 半块），解密明文 = `B165A000 00 FF×7`
- **val1=0xB165 = chipkey，val2=0xA000 = "chip key" 类型标识**；文件名 `0xB165.key` 即 key 值本身
- 用途：给 **eFuse 已烧 chipkey(≠0xFFFF) 的芯片**烧录时，isd_download/UBOOT2.00 用它把固件从默认 0xFFFF key **再加密(recrypt)** 成 chipkey 密钥流。它不是 AES 解密密钥，而是密钥流种子

### 能否解密 device-flash-bin 的固件：不能，也不需要
- **flash 固件区用 chipkey=0x7DF1 派生密钥流加密（2026-10-19 实测修正：默认流镜像不启动、-key recrypt 后正常启动）**。原"证据"辨析：
  1. 〔辨析：isd 头/分区表用固定 enc_cipher(0xFFFF) 加密、**不随 chipkey 变化**，头部一致只能证明配置区未按 chipkey 加密，不能证明固件体〕自建无 key 的 `jl_isd.bin` 与 dump 头部密文在相同偏移大量逐字节一致（`f031/dead/ff68`、`7a7c5317a5ad4521f36d`、`7cf8 22c3 a367 cc3d 5b97` 等）
  2. **isd 分区表（0x20~0x3F）= 孟黎 enc_cipher（16bit LFSR XOR, key=0xFFFF）**，直接解出第一分区条目 `uboot.boot, len=0x36B8`；自建固件同样解出 `uboot.boot_debug, len=0x52F0`。用 key=0xB165/0x65B1/0x0000 均为乱码
  3. app/uboot 体 = 复杂密钥流（§4 已提取 2.3MB——**该流即 chipkey=0x7DF1 派生流**；KS.bin 若遗失可从 dev_uboot_dec.bin 与 dump 逐字节 XOR 重建 uboot 段）
- 0x240000 的 32KB 加密 blob（PRCT 保护/备份区）：AES-ECB/CBC/CTR 全部 key 派生 + 孟黎 crc/enc（含 key=0xB165 种子）均无效 → 它是复杂密钥流 + 高熵明文（压缩数据），同样不需要 0xB165.key，需要 KS 密钥流才能解
- **FC09 解码校准线索（2026-10-19 更新）**：`0xB165` 是 **AC6956C(词易帮 Y20)** 的 chipkey，不是本机 AC7911 的——本机真值 = **`0x7DF1`**（见 §8；旧记录 0x90F0/0x164B/0x30B0 是 resp[4:6] 解码偏移错误产生的会话噪声）。0xB165.key 的 AES 格式解析仍然有效

### 快速复现
```python
def enc_cipher(data, key=0xFFFF):          # 解 isd 分区表
    out = bytearray()
    for b in data:
        out.append(b ^ (key & 0xFF))
        key = ((key << 1) ^ (0x1021 if key & 0x8000 else 0)) & 0xFFFF
    return bytes(out)
# enc_cipher(dump[0x20:0x40], 0xFFFF) → b'...  uboot.boot ÿ...'
```

---

## 14. 原厂密钥流复验 + isd_download 在线强制 -key（2026-10-19）

### ✅ 最终结论（2026-10-19 晚，设备实测闭环）
- **AC7911 的 ROM 强制 eFuse 解密**：flash 固件区 = chipkey(0x7DF1) 派生密钥流加密；默认流镜像**必然不启动**
- **正确烧录姿势**：`isd_download ... -key 0x7DF1.key`（在线 recrypt 烧录）；或 JlFlash 裸写**已经 recrypt 过的**镜像
- JlFlash 裸写**未 recrypt 的**默认流镜像 = 擦写校验全过但不启动（已实测两次）——工具本身没问题，缺的是 recrypt 环节
- key 文件可自行生成（格式/AES/CRC 全部破解，见下），无需杰理授权，绕过 `-use-chip-key` 的机器码限制
- 对比：Y20（AC6956C/br23）实测**默认流可启动**——两平台 ROM 行为不同，勿混淆

### 初验结论（2026-10-19 下午，→ 当晚已被实测推翻，见下）
- ~~flash 固件区密钥流 = isd_download 内置默认流（key=0xFFFF 语义）这一说法~~ **被推翻**：默认流镜像烧入后不启动；固件区实为 **chipkey=0x7DF1 派生流**（§11 证据中仍有效的部分：自建无 key 包与 dump 头部密文同偏移一致、分区表 enc_cipher 可解）
- 但密钥流**生成器无法用公开算法复现**：用官方工具离线打包提取内置流后，与 dump 比对穷举了 MengLi-CRC / ENC / RxGp（绝对/区域两种相位、多种子）+ 全部 4 字节对齐偏移 + 1..64 周期扫描，均不命中——与 §4 "非线性、0x400 准周期、BM 复杂度打满" 的结论一致，**实用上只能依赖已提取的 KS.bin / dev_uboot_dec.bin 重建**（§13 索引）
- **新实测（isd_download 4.2.33，AC7911 在线）**：检测到**已烧 key 的在线芯片**时，无 `-key` 直接拒绝打包（`芯片被烧写过KEY，ERR_KEY`）——把芯片复位回固件模式（FC0C）或拔线离线后即可正常打包。这就是 §7 提示"设备不要接"的机理；Key mismatch **并非仅策略检查——在线已烧 key 芯片必须提供匹配 -key 才能 recrypt 烧录，否则固件无法启动**
### 工具同步
- JlFlash/JlFlashGui 已推送到 GitHub `Wilddiscodog/JL-TOOLS`（含 FC09 解码修复、ReadChipKeyAndReset、BR23 读 key 修正）；仓库同时收录 Y20 V0.0.5 固件与 AC6956C 芯片手册

### recrypt 链路实测（2026-10-19，✅ 全链路闭环，设备成功运行）
- key 文件生成器完全闭环：chipkey 的 ASCII 串(如 "7DF1A000") + 00 + FF×7 → AES-128-ECB(自选密钥) → 32B = [密文16B][AES密钥16B] → hex 小写 64 字符 + **jl_crc32(hex串) 大写 8 字符**。生成器用已知 0xB165.key 逐字节复现校准
- 生成 `0x7DF1.key` 后 `isd_download -key 0x7DF1.key` **被接受**（ERR_KEY 消失，识别在线芯片 684018）→ 擦除 0x0-0xfff000 → 下载 → 复位全部成功——recrypt 工具链验证通过
- 被烧入的是 SDK 演示 app（tools/app.bin）——烧录后 USB 无枚举；后续用 JlFlash 裸写默认流 DevKitBoard/helloworld 镜像均**不启动**（无 LED/无串口），反证 **ROM 强制 eFuse 解密、recrypt 是必需步骤**
- 恢复路径：JlFlashGui 整片恢复 `flash_dump_20260924_092843.bin`；USB 进不了 UBOOT 时用 flash CS 短接触发 ROM 救砖
- 【2026-10-19 晚最终确认】recrypt 烧录 helloworld → **设备成功运行**（用户实测确认）——完整链路：eFuse 读 key(0x7DF1) → 生成 key 文件 → isd_download -key → UBOOT recrypt → ROM 解密启动
- 恢复路径：JlFlashGui 整片恢复 `flash_dump_20260924_092843.bin`；USB 进不了 UBOOT 时用 flash CS 短接触发 ROM 救砖（原行保留）B 层区分；待接 PB2/PB8 串口看日志，或烧入带 UDISK 的最小固件验证（→ 见 §15）
- 恢复路径：JlFlashGui 整片恢复 `flash_dump_20260924_092843.bin`；USB 进不了 UBOOT 时用 flash CS 短接触发 ROM 救砖


---

## 15. 最小 USB-MSC 固件工程（2026-10-19，进行中）

- 目的：① 验证 AC7911 能否启动自编固件（当前烧入的 SDK demo app 无 USB 枚举，无法判定是没 UDISK 还是没启动）；② 恢复设备的 USB 下载通道（现在 flash 里没有 UDISK 盘符，踢不了 0xFB）
- 工程：改用 apps/demo/helloworld（最小化、无 USB）——**PA6 置高接 LED 作为启动指示**、调试串口 PB2 @1M（board.c 波特率已改 1000000）
- 构建：board/wl82 下 make（PATH 需含 C:/JL/pi32/bin + AC695N SDK 的 tools/utils 以获得 mkdir_win）
- **时钟/SDRAM 修正（ini 生成源头 = isd_config_rule.c，勿直接改 ini）**：SYS_CLK 320M→240M、SDRAM_PLL3_EN 0→1（USB2.0 要求此组合）、SDRAM_SIZE 跟随 __SDRAM_SIZE__；SDRAM 容量修正的真正源头 = **helloworld/include/app_config.h 的 `__SDRAM_SIZE__`**（2M→8M，板载 8MB；每次 make 经 sdk_ld.c 重新生成 sdk.ld，勿直接改 sdk.ld/ini）
- **USB-MSC 已移植进 helloworld**（设备将枚举为 RAM 虚拟U盘）：app_config.h 加 CONFIG_USB_ENABLE 块（TCFG_PC_ENABLE=1、USB_DEVICE_CLASS_CONFIG=MASSSTORAGE_CLASS、TCFG_VIR_UDISK_ENABLE=1、APP_NAME_PC="pc"）；board.c 加 otg_dev_data + 设备表 {"otg", &usb_dev_ops}；app_main 任务表加 usb_server/usb_msd0/usb_msd1 + weak `app_cur_task_check` 桩（常驻 PC 模式）；Makefile 加 apps/common 路径、usb/device 源（cdc/descriptor/msd/msd_upgrade/ram_disk_circle/task_pc/usb_device/user_setup）、usb_config.c、lib_usb_syn.a、usb_epbuf_manager.c、usb_epbuf_manager.c
- 一键脚本：apps/demo/helloworld/build.bat（无参=编译+打包；`build.bat flash`=编译+recrypt 加密烧录；`build.bat clean`=清理；GBK 编码，双击可用）
- 产物：helloworld_usbmsc_jl_isd.bin（159744B 默认流）；烧录必须 **isd_download -key 0x7DF1.key**（recrypt），裸写默认流不启动
- 另存有 DevKitBoard MSC 固件 msc_default.bin（1253376B，含 UDISK，可用于恢复 USB 下载通道）
- 烧录实验：① JlFlash 裸写默认流 → 擦写校验通过但**不启动**（无 LED/无串口）→ 证明 ROM 强制 eFuse 解密；② isd_download `-key 0x7DF1.key` recrypt 烧录 → **设备成功运行**（用户确认）
- SDRAM 8M 版（240M+PLL3）烧录后：**helloworld 正常运行、串口出 Hello World**（用户日志确认）——但 USB 仍不枚举
- 日志显示 USB 栈已初始化（OTG_info/OTG_cnt/usb_memory_init ✓），判定 **OTG→事件→pc_device_event_handler→usb_start 链路未触发**
- 排查构建：USB_PC_NO_APP_MODE=1（IN 事件直接 usb_start）、device_mount.c 换成自带迷你 OTG 分发器、app_task.h 空壳适配、usb_config.c/usb_epbuf_manager.c 补编
- 最新诊断版：helloworld 开机 3s 后**强制 usb_start(0)**（绕过 OTG 事件链）——若枚举成功则问题在事件链；若仍不枚举则 usb_device_mode 内部（时钟/描述符）问题
- 注意：helloworld 无 USB，烧它之后设备不再枚举 U 盘，重刷需 flash CS 短接救砖；恢复 U 盘通道烧 msc_default.bin（DevKitBoard MSC 固件，须同样 -key recrypt 烧录）
