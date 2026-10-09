# EC6108V9C 刷入 Linux 内核 + Debian 系统 — 初步调研报告

> 文档状态：**草稿（初版）** · 创建于 2026-10-09
> 目标硬件：华为悦盒 EC6108V9C（电信 IPTV 机顶盒，SoC 为海思 Hi3798MV100）
> 目标系统：完整 Linux 内核 + Debian（armhf）

---

## 1. 结论摘要

在 EC6108V9C 上运行完整 Linux + Debian 是**可行且社区已有成熟先例**的，但"完整内核"有三种含义截然不同的技术路线，选型决定了工作量与功能上限：

| 路线 | 内核 | 优点 | 代价 |
| --- | --- | --- | --- |
| A. 厂商 SDK 内核 | HiSTBLinux `linux-4.4.y`（`hi3798mv100_defconfig`） | 驱动最全：HDMI/GPU/音频/CA/解码都在；Docker 可跑 | 非 mainline，licensing 与维护性差，版本锁死 4.4 |
| B. Mainline 内核移植（推荐） | `loonpn/ec6108v9c-linux-kernel`（本仓库子模块，上游已到 v6.14.3） | 干净的内核源码、可跟上游、SMP 4 核可用、SD 卡支持 | HDMI/GPU/音频**无驱动**，只能无头（headless）使用 |
| C. 社区发行版内核 | histb（海纳思）`4.4.35` | 开箱即用，生态文档全 | 本质是变相 A，且以 Ubuntu 为主 |

**建议采用 B 为主、A 为硬件参考与兜底**：即用 mainline 系内核获得一个可长期维护的 Debian 系统（SSH/网络/USB/存储/看门狗全部可用），把图形与视频输出留给后续 HDMI/GPU 适配工作。这样本仓库的定位 = *B 路线的 Debian 化发行版*。

关键前提条件（缺一不可）：

1. 盒子必须是 **eMMC 版本且非高安（non-CA / pub）版**；NAND 版与 CA 版无法刷机。
2. reg name 需为 `hi3798mdmo1g`（1G DDR3 + 8G eMMC），对应 DDR 初始化文件必须匹配，否则跑码/启动即失败。
3. 需要 USB-TTL（CH340，3.3V）串口线，用于首次烧写与救砖。
4. 需自行承担变砖风险；`fastboot` 分区不可破坏，它是唯一的 TTL 救砖入口。

---

## 2. 硬件规格与资源盘点

数据来源：`arch/arm/boot/dts/hi3798mv100.dtsi`、`hi3798mv100-ec6108v9c.dts`（本仓库子模块 `ec6108v9c-linux-kernel`）以及 histb 设备适配表。

| 项目 | 参数 |
| --- | --- |
| SoC | HiSilicon Hi3798MV100 |
| CPU | 4 × ARM Cortex-A7（ARMv7-A，32-bit，支持 LPAE），`enable-method = hisilicon,hi3798-smp` |
| 内存 | 1 GB DDR3（reg: `hi3798mdmo1g_hi3798mv100_ddr3_1gbyte_16bitx2_4layers_emmc.reg`） |
| 存储 | 8 GB eMMC（实测可用 7456 MB，MMC 5.1，DDR50，8-bit 位宽，52 MHz） |
| 网口 | 内置 FEPHY（`hisilicon,hisi-femac-v2`，MII 模式，PHY 地址 1）；MAC 由 bootloader 覆写 |
| 串口 | UART0 = PL011 @ `0x8b00000`，115200 8N1（DTS 中 `stdout-path = "serial0:115200n8"`） |
| USB | USB 2.0 × 2（OHCI/EHCI，走内部 `usb2_phy1` / `usb2_phy2`）；xhci0 与 combphy（USB 3.0）在 DTS 中为 `disabled` |
| 存储扩展 | SD/TF 卡槽（`snps,dw-mshc` @ `0x9820000`，mainline 内核已启用） |
| eMMC 控制器 | `hisilicon,hi3798cv200-dw-mshc` @ `0x9830000`（不可移除，需 `non-removable`） |
| 看门狗 | SP805 @ `0x8a2c000` |
| 定时器 | SP804 × 4（`timer1`~`timer4`，24 MHz 固定时钟） |
| GPIO | PL061 × 7（`gpio0`~`gpio6`，每组 8 bit，经 `pmx0` pinconf-single 复用到寄存器） |
| I2C / SPI | `hisilicon,hix5hd2-i2c` × 3；`arm,pl022` SPI × 1 |
| 红外 | `hisilicon,hix5hd2-ir` @ `0x8001000` |
| LED | 3 颗：`green:network`(gpio0_4)、`red:network`(gpio2_2)、`green:ir`(gpio5_2，带 `panic-indicator`) |
| GPU | Mali-450（`arm,mali-utgard` @ `0x9200000`，OPP 150~500 MHz，比例电压 0.9 V~1.32 V） |
| 音频/视频 | SoC 带 I2S/S/PDIF，但 mainline 内核**无驱动** |
| WiFi | 部分批次板载 RTL8188FTV（SDIO，2.4 GHz，150 Mbps） |

### 2.1 内存映射要点（排障时会反复用到）

```
0x8000000   sysctrl  (system-controller, uart0/ir 时钟与复位)
0x8a20000   perictrl (USB2 PHY / combphy)
0x8a21000   pmx0     (pinconf-single 复用)
0x8a22000   crg      (clock-reset-controller)
0x8a2c000   wdt0     (SP805 看门狗)
0x8b00000   uart0    (PL011)
0x8b10000   i2c0 / i2c1 / i2c2
0x8b1a000   spi0     (PL022)
0x8b20000   gpio0 ~ gpio6 (PL061)
0x9200000   gpu      (Mali-450)
0x9820000   sd       (SD/TF, snps,dw-mshc)
0x9830000   mmc      (eMMC, hisilicon,hi3798cv200-dw-mshc)
0x9840000   eth0     (hisi-femac-v2) + 0x9841100 mdio0
0x9880000   ohci0/ehci0 / 0x98a0000 xhci0(未用)
0x9920000   ohci1/ehci1
0xf8a01000  gic      (dist 0x1000 + cpu 0x100)
```

`hisilicon,hi3798mv100-crg` 同时挂在 `hisilicon,cpuctrl` / `syscon` / `simple-mfd`，SMP 启动方法 `hi3798-smp` 实现在 `arch/arm/mach-hisi/platsmp.c`。

---

## 3. 启动链与 eMMC 分区

### 3.1 启动链

```
Mask ROM (Bootrom)  ──从 eMMC 引导──▶  Vendor Fastboot 3.3.0 (U-Boot 衍生)
                                              │
                          bootcmd = mmc read 0 0x1FFFFC0 0x1000 0x4000; bootm 0x1FFFFC0
                                              │
                          从 mmc0 偏移 0x1000 块 (=2 MiB) 起读 0x4000 块 (=8 MiB)
                          到内存 0x1FFFFC0，按 legacy uImage 启动
                                              │
                                              ▼
                                    Linux Kernel (ARM 32-bit, 入口 0x2000000)
                                              │
                                              ▼
                                    rootfs (ext4) → Debian
```

U-Boot 侧关键环境变量（源自 `glinuz/hi3798mv100` 的 `bootargs.txt`）：

```sh
bootcmd=mmc read 0 0x1FFFFC0 0x1000 0x4000;bootm 0x1FFFFC0
bootargs=console=ttyAMA0,115200 root=/dev/mmcblk0p4 rootfstype=ext4 rootwait \
         blkdevparts=mmcblk0:1M(fastboot),1M(bootargs),8M(kernel),128M(rootfs),-(system)
bootdelay=3
baudrate=115200
```

**U-Boot 会按 `blkdevparts=` 画出分区表**，因此 Linux 看到的分区布局来自 bootargs，而不是设备树。修改分区必须同步修改 `bootargs`（用厂商 `mkbootargs` 重新生成 bootargs.bin，或在 U-Boot 里 `setenv`/`saveenv`）与刷机用的 XML 分区表。

### 3.2 分区方案

（a）**参考方案 A**（`glinuz` / histb 生态通用，128 MiB rootfs 的极简版）：

| 分区 | 起始 | 大小 | 用途 |
| --- | --- | --- | --- |
| fastboot | 0 | 1M | U-Boot/fastboot，**救砖唯一入口，绝对不要覆盖** |
| bootargs | 1M | 1M | U-Boot 环境变量 |
| kernel | 2M | 8M | Linux 内核 |
| rootfs | 10M | 128M | 根文件系统（ext4） |
| system | 138M | 剩余 | 数据/扩展 |

（b）**建议方案 B**（本项目的目标布局，塞完整 Debian，8G eMMC 用满）：

| 分区 | 起始 | 大小 | 用途 |
| --- | --- | --- | --- |
| p1 fastboot | 0 | 1M | 保留不动 |
| p2 bootargs | 1M | 1M | 保留（改 bootargs） |
| p3 kernel | 2M | **16M** | mainline uImage（zImage+appended DTB 约 7 MB）留足余量 |
| p4 rootfs | 18M | **剩余 ~7.3G** | Debian armhf ext4 |

方案 B 必须同时改动三处：

1. `bootargs` 的 `blkdevparts=` 改为 `mmcblk0:1M(fastboot),1M(bootargs),16M(kernel),-(rootfs)`，`root=/dev/mmcblk0p4`；
2. `bootcmd` 读取长度从 `0x4000`（8 MiB）改为 `0x8000`（16 MiB）：
   `bootcmd=mmc read 0 0x1FFFFC0 0x1000 0x8000;bootm 0x1FFFFC0`
   （`0x1000` 块 × 512 B = 2 MiB 偏移；块数 × 512 B = 读取长度。）
3. HiTool 的 XML 分区表同步修改。

> mainline 系内核的 v6.14.3 已改为"**启动参数由内核解析**"并移除了 initramfs 支持，所以 `root=` 必须能直接挂载，不能再依赖 initramfs 兜底。

---

## 4. 内核方案

### 4.1 路线 A：厂商 SDK（HiSTBLinux）

本仓库已引入子模块 `HiSTBLinuxV100R005C00SPC060`（`07bug/HiSTBLinuxV100R005C00SPC060`）。

```sh
cd HiSTBLinuxV100R005C00SPC060
sh server_install.sh   # 或 apt install gcc make gettext bison flex bc zlib1g-dev \
                        #   libncurses5-dev lzma
cp configs/hi3798mv100/hi3798mdmo1g_hi3798mv100_cfg.mak ./cfg.mak
source ./env.sh
make menuconfig
make build -j$(nproc) 2>&1 | tee buildlog.txt
```

产物在 `out/hi3798mv100/`：`fastboot-burn.bin`（U-Boot）、`bootargs.bin`、`hi_kernel.bin`（内核）。
自定义内核时改 `source/kernel/linux-4.4.y/arch/arm/configs/hi3798mv100_defconfig`，走
`make ARCH=arm hi3798mv100_defconfig` → `menuconfig` → `savedefconfig` → 拷回 → `make linux` 的流程。
**要跑 Docker 必须确认打开**：`CONFIG_DEVTMPFS`、`CONFIG_FHANDLE`（open by fhandle）、`CONFIG_CGROUPS`。

### 4.2 路线 B：Mainline 系移植内核（本仓库主推）

本仓库子模块 `ec6108v9c-linux-kernel`（`loonpn`）即此路线：
基于 mainline 6.3-rc5 合入 hi3798mv100 SoC 支持，上游 master 已跟进到 **v6.14.3**（2025-04 发布，附带 `uImage` + `modules.tar.gz`）。

板级文件：

- `arch/arm/boot/dts/hi3798mv100.dtsi`（SoC 层）
- `arch/arm/boot/dts/hi3798mv100-ec6108v9c.dts`（板级：model = "Huawei EC6108V9C"）
- `arch/arm/configs/hi3798mv100_defconfig`

内核配置关键点（摘自 `hi3798mv100_defconfig`）：

```
CONFIG_ARCH_HISI=y          CONFIG_SMP=y               CONFIG_NR_CPUS=8
CONFIG_ARM_LPAE=y           CONFIG_HIGHMEM=y           CONFIG_MCPM=y
CONFIG_ARM_APPENDED_DTB=y   # DTB 附加在 zImage 尾部，mkimage 打包
CONFIG_KERNEL_XZ=y          CONFIG_HZ_250=y            CONFIG_PREEMPT_VOLUNTARY=y
CONFIG_SERIAL_AMBA_PL011=y  CONFIG_SERIAL_AMBA_PL011_CONSOLE=y
CONFIG_GPIO_PL061=y         CONFIG_WATCHDOG=y + ARM_SP805_WATCHDOG=y
CONFIG_HISI_FEMAC=y         CONFIG_MDIO_HISI_FEMAC=y   CONFIG_LED_TRIGGER_PHY=y
CONFIG_MMC_DW=y             CONFIG_MMC_DW_HI3798CV200=y
CONFIG_USB_XHCI_HCD=y       CONFIG_USB_EHCI_HCD=y      CONFIG_USB_OHCI_HCD=y
CONFIG_I2C_HIX5HD2=y       CONFIG_SPI_PL022=y         CONFIG_PINCTRL_SINGLE=y
CONFIG_PHY_HISI_INNO_USB2=y  CONFIG_IR_* / RC_CORE=y   CONFIG_LEDS_GPIO=m
```

这条路线是"为 Debian 准备"的理想基线：文件系统极其完整（ext4/f2fs/btrfs/xfs/ubifs/erofs/overlay/squashfs…）、网络栈与 netfilter/nftables 齐全、容器与安全（cgroup v1、namespaces、seccomp、SELinux/AppArmor/Landlock）齐备。

> **Windows 开发环境注意**：Linux 内核源码树含 `drivers/gpu/drm/nouveau/nvkm/subdev/i2c/aux.c`，`aux` 是 Windows 保留设备名，`git checkout` 会以
> `error: invalid path ... aux.c` 整体中止，导致工作区为空。
> 因此在 Windows 上**不要尝试 `git submodule update --init` 拉取内核源码**；请改用 WSL2 / Linux 实体机或容器构建，
> 或像本次调研一样通过 `git --git-dir` + `ls-tree` / `show` 直接读取 git 对象获取信息。


**v6.14.3 已知问题**（直接影响 Debian 可用性，需在文档中明示）：

- L1/L2 cache 不工作（性能显著受损）
- HDMI 输出与 GPU 不工作（→ 只能无头使用，或后续补 lima/panfork 与 HDMI 驱动）
- PCM 与 S/PDIF 输出不工作
- RTL8188FTV 随机断线
- 内置网口 MAC 无法由 U-Boot 填充 → 需要在内核参数或 DT 中指定 `mac-address`

### 4.3 内核构建与打包（路线 B）

```sh
sudo apt install -y gcc make gettext bison flex bc zlib1g-dev libncurses5-dev \
                    libssl-dev lzma gcc-arm-linux-gnueabihf
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- hi3798mv100_defconfig
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- menuconfig
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- -j$(nproc)

cat arch/arm/boot/zImage arch/arm/boot/dts/hi3798mv100-ec6108v9c.dtb > zImage-dtb
mkimage -A arm -O linux -T kernel -C none -a 0X2000000 -e 0X2000000 -n linux -d zImage-dtb uImage
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- INSTALL_MOD_PATH=out modules_install
```

> `CONFIG_ARM_APPENDED_DTB=y` 时 DTB 附加在 zImage 尾部；加载/入口地址固定 `0x2000000`。U-Boot 是 legacy uImage 引导，必须用 `mkimage` 打包（`-C none` 不压缩）。

---

## 5. Debian 根文件系统

目标：armhf + LPAE（内核 `CONFIG_ARM_LPAE=y`），架构 `armhf` 即可，无需 `arm64`。

推荐 `debootstrap` + `qemu-user-static` 两步法（debian bookworm/trixie 均可；示例用已镜像的 USTC 源）：

```sh
sudo apt-get install -y binfmt-support debootstrap qemu qemu-user-static
mkdir -p ~/ec6108v9c/rootfs && cd ~/ec6108v9c
sudo debootstrap --arch=armhf --variant=minbase --foreign \
  --include=locales,util-linux,apt-utils,ifupdown,systemd-sysv,iproute2,curl,wget,expect,\
ca-certificates,openssh-server,isc-dhcp-client,vim-tiny,bzip2,cpio,usbutils,netbase,parted,\
jq,bc,crda,wireless-tools,iw \
  trixie rootfs http://mirrors.ustc.edu.cn/debian/

cd rootfs
sudo cp /usr/bin/qemu-arm-static usr/bin/
sudo mount -v --bind /dev dev
sudo mount -vt devpts devpts dev/pts -o gid=5,mode=620
sudo mount -t proc proc proc
sudo mount -t sysfs sysfs sys
sudo mount -t tmpfs tmpfs run
sudo chroot . /debootstrap/debootstrap --second-stage
sudo chroot . dpkg --configure -a
# 之后在 chroot 内完成：hostname、timezone、locale、sources.list
# 串口登录：ln -s /lib/systemd/system/getty@.service \
#   /etc/systemd/system/getty.target.wants/getty@ttyAMA0.service
# sshd：PermitRootLogin yes（首次），随后改密/禁 root
```

chroot 内建议固定项：

- `/etc/hostname` = `ec6108v9c`；`/etc/timezone` = `Asia/Shanghai`
- `en_US.UTF-8` + `zh_CN.UTF-8` locale
- `sources.list` 指向 `deb.debian.org` 或国内镜像，含 `main contrib non-free non-free-firmware`
- 若板载 RTL8188FTV：安装 `firmware-realtek`

打包为可烧写的 ext4 镜像（eMMC 可用约 7456 MB，p4 自 18 MiB 起，故镜像取 7400 MiB，可按需调整）：

```sh
dd if=/dev/zero of=rootfs.ext4 bs=1M count=7400
mkfs.ext4 -L debian-rootfs rootfs.ext4
mkdir -p /mnt/rootfs && sudo mount -o loop rootfs.ext4 /mnt/rootfs
sudo cp -a rootfs/. /mnt/rootfs/
sudo umount /mnt/rootfs
```

> 由于 U-Boot 没有 `boot.scr` 动态加载流程，最简单可靠的做法是直接把 Debian 装在 p4 上，`root=/dev/mmcblk0p4` 直接挂 ext4。

---

## 6. 刷机流程与要点

### 6.1 TTL 线刷（首次，必须）

工具：HiTool（Windows 版 `HiTool-windows.7z` 或 Linux 版 `hitool_linux.tar.gz`），配 USB-TTL（推荐免驱 CH340，3.3 V）。

硬件接线风险提示：**只接 GND / TX / RX 三个点，TX 与 RX 需对调；切勿接 VCC**，否则可能烧板。

步骤：

1. 串口接好，上电跑码，读出芯片型号与 reg name（EC6108V9C 应为 `hi3798mdmo1g`）。
2. 盒子网线接电脑/同网段路由器。
3. HiTool：`芯片` 菜单确认型号 → `本地配置` 选串口号 → `服务器IP` 选与本地同网段的那个 → 传输方式选**网口** → 选 **eMMC 烧写** → 浏览选择分区表 XML（如 `emmc_TTL-hi3798mv100-32.xml`）→ 烧写。
4. 中途按提示断电重启，等待烧写完成。
5. 关闭防火墙与杀软，避免 69 端口被拦截。

常见失败原因：TTL 接触不良导致 fastboot 下载失败、网线/网络不通卡在 `TTTT`、高安板或 NAND 板（放弃）、部分盒子 TTL 被阻断需飞线。

### 6.2 已刷系统后的增量更新

刷过一次之后，网络/SSH 可用，内核可以走 `dd` 直接写分区，不必再走 HiTool：

```sh
# 以 histb 布局（内核在第 6 分区 /dev/mmcblk0p6，32 位内核占 20 MiB）为例
dd if=uImage of=/dev/mmcblk0p6 bs=1024 count=20480
```

> 本项目方案 B 的分区布局不同（内核在 p3、16 MiB），对应改成 `dd if=uImage of=/dev/mmcblk0p3 bs=1024 count=16384`。**务必先核对分区表再 dd。**

### 6.3 U-Boot 参数即时调整

上电时按 `Ctrl+C` 进 U-Boot，可现场改启动参数验证，确认无误再保存：

```sh
setenv bootargs console=ttyAMA0,115200 root=/dev/mmcblk0p4 rootfstype=ext4 rootwait \
  blkdevparts=mmcblk0:1M(fastboot),1M(bootargs),16M(kernel),-(rootfs)
setenv bootcmd 'mmc read 0 0x1FFFFC0 0x1000 0x8000;bootm 0x1FFFFC0'
saveenv
reset
```

mainline 内核的网口 MAC 缺陷（见 R6）建议在设备树中静态指定，例如在板级 DTS 中：

```
&mdio0 {
	phy1: phy@1 {
		mac-address = [00 11 22 33 44 55];
	};
};
```

（`hi3798mv100.dtsi` 中该字段是占位值，注释已说明"会被 bootloader 覆写"，而 mainline 内核收不到这个覆写。）

### 6.4 救砖

只要 `fastboot` 分区（p1，第 0~1 MiB）完好，就能用同样的 TTL + HiTool 流程重刷。因此**任何方案都不应触碰 p1**。

---

## 7. 风险与开放问题

| 编号 | 风险 | 影响 | 缓解 |
| --- | --- | --- | --- |
| R1 | 高安（CA）版 / NAND 版盒子 | 完全无法刷机 | 刷前用 TTL 跑码核对 reg name 与存储介质 |
| R2 | reg name / DDR 初始化不匹配 | 启动即挂 | 严格使用 `hi3798mdmo1g` 对应的 fastboot 与 reg 文件 |
| R3 | mainline 内核无 HDMI/GPU 驱动 | 无视频输出 | 无头部署（SSH）；图形适配单独立项 |
| R4 | mainline L1/L2 cache 缺陷 | 性能明显下降 | 后续调研 cache 补丁；短期以可跑通为主 |
| R5 | 1 GiB 内存 | 无法跑重负载 | 关图形、zram、精简 systemd 服务 |
| R6 | 网口 MAC 不被 U-Boot 填充 | 随机 MAC | DT/U-Boot 静态指定 |
| R7 | WiFi（RTL8188FTV）不稳定 | 联网不稳 | 优先有线；固件放 `non-free-firmware` |
| R8 | eMMC 磨损 | 长期运行掉盘 | 关 swap（或 zram）、日志限流、`noatime` |

---

## 8. 建议的实施路线（里程碑）

1. **M1 环境与资料（本仓库已完成）**：子模块引入、硬件/启动链/分区事实固化成本报告。
2. **M2 基线跑通**：按路线 B 构建 `hi3798mv100_defconfig`，产出 `uImage`；用 HiTool 刷 p1/p2/p3 + 极简 rootfs；串口看到 kernel 启动日志并登录。
3. **M3 Debian 化**：debootstrap 出完整 armhf Debian，落到 p4；打通网络、SSH、apt、systemd；固化可重复的构建脚本。
4. **M4 稳定性**：看门狗接守护进程、`zram`、`fstab`（noatime）、MAC 固化、WiFi 固件、eMMC 健康策略。
5. **M5 增值能力**：按需补齐 Docker（依赖 A/Docker 相关 Kconfig 全开）、Samba/NFS、Cups、容器化应用；图形/HDMI 单独立项评估。

---

## 9. 待补充/待验证清单

- [ ] 确认目标盒子为非高安 eMMC 版，reg name = `hi3798mdmo1g`
- [ ] 实测 mainline uImage 在 p3=16 MiB 布局下的启动
- [ ] 验证设备树中静态 `mac-address` 方案可稳定工作
- [ ] 评估 mainline 内核的 L1/L2 cache 缺陷是否有上游修复可移植
- [ ] 决定 Debian 版本（bookworm 稳定 vs trixie 测试）
- [ ] 补齐 `docs/references.md` 中失效链接的镜像来源

---

*本文事实性内容尽量以上游源码（子模块内的 DTS/defconfig）与社区公开资料为准；未经实测的推断已标注"待验证"。*
