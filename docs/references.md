# 资料索引（References）

> 收集于 2026-10-09。分为**本仓库内资源**与**外部社区资源**两类。

---

## 1. 本仓库子模块

| 路径 | 上游 | 用途 |
| --- | --- | --- |
| `ec6108v9c-linux-kernel` | `git@github.com:loonpn/ec6108v9c-linux-kernel.git` | Mainline 系 hi3798mv100 内核（本仓库主推）。`arch/arm/boot/dts/hi3798mv100-ec6108v9c.dts`、`hi3798mv100.dtsi`、`arch/arm/configs/hi3798mv100_defconfig` |
| `HiSTBLinuxV100R005C00SPC060` | `git@github.com:07bug/HiSTBLinuxV100R005C00SPC060.git` | 海思厂商 SDK（`linux-4.4.y`），驱动最全，作为硬件参考与兜底内核 |
| `hi3798mv100` | `git@github.com:glinuz/hi3798mv100.git` | 编译/烧录/rootfs 制作全流程文档，含 `bootargs.txt`、`emmc_partitions.xml`、`hitool-burn.png`、TTL 接线图 |

> 当前 pin 状态：`ec6108v9c-linux-kernel` 固定在 `8677b406968b`（基于 6.3-rc5 的树），上游 master 已推进到 `v6.14.3`；后两个子模块尚未 init（`git submodule update --init <path>`）。

---

## 2. 内核与硬件源码要点

均来自子模块 `ec6108v9c-linux-kernel` 的 git 对象，可离线查阅：

- `arch/arm/boot/dts/hi3798mv100.dtsi` — SoC 级描述：4×Cortex-A7、GIC、CRG/sysctrl、GPIO×7、SP804×4、SP805 WDT、PL011 UART、hix5hd2 I2C/IR、PL022 SPI、Mali-450、eMMC/SD、FEMAC、USB。
- `arch/arm/boot/dts/hi3798mv100-ec6108v9c.dts` — 板级：`model = "Huawei EC6108V9C"`，`stdout-path = "serial0:115200n8"`，3 颗 LED 的 GPIO 定义，各 IP 置 `okay`。
- `arch/arm/configs/hi3798mv100_defconfig` — 完整内核配置参考。
- `drivers/clk/hisilicon/crg-hi3798.c` + `include/dt-bindings/clock/histb-clock.h` — CRG 时钟/复位驱动与绑定。
- `drivers/net/ethernet/hisilicon/hisi_femac.c` — 内置网口驱动（`CONFIG_HISI_FEMAC`）。
- `drivers/mmc/host/dw_mmc-hi3798cv200.c` — eMMC（`CONFIG_MMC_DW_HI3798CV200`）。
- `drivers/media/rc/ir-hix5hd2.c` — 红外。
- `arch/arm/mach-hisi/platsmp.c` — `hi3798-smp` 多核启动。

---

## 3. 外部资源

### 3.1 内核 / 编译 / 烧录

- loonpn/ec6108v9c-linux-kernel（含预编译 `uImage` / `modules.tar.gz`）：
  <https://github.com/loonpn/ec6108v9c-linux-kernel>
  发布页（v6.14.3 已知问题清单，见 report.md §4.2）：
  <https://github.com/loonpn/ec6108v9c-linux-kernel/releases/latest>
- glinuz/hi3798mv100（编译 + HiTool 烧录 + debootstrap rootfs 全流程，本报告主要事实来源之一）：
  <https://github.com/glinuz/hi3798mv100>
  - `bootargs.txt`：<https://raw.githubusercontent.com/glinuz/hi3798mv100/master/bootargs.txt>
  - `emmc_partitions.xml`：<https://raw.githubusercontent.com/glinuz/hi3798mv100/master/emmc_partitions.xml>
- lucamot/HiSTB（同上内容的英文版，附 `Dockerfile` 便于复现编译环境）：
  <https://github.com/lucamot/HiSTB>
- zknb-666/hi3798m、wdznb/EC6108V9C_HiSTBLinuxV100R005C00SPC060（SDK SPC060 + Ubuntu 22.04 rootfs + Docker defconfig）：
  <https://github.com/zknb-666/hi3798m> ·
  <https://github.com/wdznb/EC6108V9C_HiSTBLinuxV100R005C00SPC060>
- CSDN：hi3798 编译 linux / EC6108V9 刷 Ubuntu 16.04（启动日志参考）：
  <https://blog.csdn.net/weixin_42659791/article/details/116671157>

### 3.2 社区发行版 histb / 海纳思（ecoo.top, histb.com）

- 设备支持适配表（EC6108V9C = hi3798mv100, 1G, eMMC 8G, reg `mdmo1g`）：
  <https://doc.ecoo.top/devices>
- HiTool（海兔）TTL 线刷指南：<https://ecoo.top/hitool.html>
- TTL 刷机图文：<https://www.ecoo.top/docs/tutorial-basics/ttl-flash>
- 固件下载（TTL/USB 刷机包）：<https://histb.com/download> · <https://dl.ecoo.top>
- 海思内核升级/手动 `dd` 升级内核教程：<https://histb.com/docs/nas-skill/kernel>
- 论坛：<https://bbs.histb.com>

### 3.3 型号鉴别 / 通用刷机（Android 侧）

- ZNDS 华为悦盒 EC6108V9/V9U 通刷教程（pub/ca 版区分、进 Recovery、TTL 救砖）：
  <https://www.znds.com/tv-1198363-1-1.html>
- ZNDS 6108V9 系列教程（各子型号与芯片对应关系）：
  <https://www.znds.com/tv-1166925-1-1.html>
- 博客园：EC6108V9U 破解过程全记录：
  <https://www.cnblogs.com/mq0036/p/18756348>
- 博客园：EC6108V9 刷 Ubuntu + CUPS 打印服务器实战（含 histb 刷入、RTL8188FTV 安装）：
  <https://www.cnblogs.com/oliudaneng/p/18608382>
- ADB 卡刷 Linux（Ubuntu）思路：<http://huitaihen.com/article/dhecij.html>

### 3.4 同类 SoC 的 Debian 化参考

- leandrotsampa/e2d-hi3798mv200（Hi3798Cv200/Mv200 上跑 Debian Buster + Enigma2 + KODI，SD 卡镜像）：
  <https://github.com/leandrotsampa/e2d-hi3798mv200>

---

## 4. 工具与镜像

| 工具 | 说明 | 来源 |
| --- | --- | --- |
| HiTool (Windows) | 华为官方烧录工具 | `https://oss.histb.com/HiTool-windows.7z` |
| HiTool (Linux) | 同上 Linux 版 | `https://oss.histb.com/hitool_linux.tar.gz` |
| 内核预编译 | `uImage` + `modules.tar.gz` | loonpn releases |
| `gcc-arm-linux-gnueabihf` | 32-bit ARM 交叉编译器 | Debian/Ubuntu apt |
| `debootstrap` + `qemu-user-static` | armhf rootfs 构建 | Debian/Ubuntu apt |
| `u-boot-tools`（`mkimage`/`mkbootargs`） | uImage 打包 | Debian/Ubuntu apt |
