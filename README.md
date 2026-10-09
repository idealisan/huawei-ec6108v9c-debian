# huawei-ec6108v9c-debian

在**华为悦盒 EC6108V9C**（海思 Hi3798MV100，4×Cortex-A7 / 1 GiB DDR3 / 8 GiB eMMC）上刷入完整 Linux 内核并运行 **Debian (armhf)** 的方案与工具集。

> 警告：刷机有变砖风险。请先完整阅读 [docs/report.md](docs/report.md)，确认你的盒子是 **eMMC 版且非高安（non-CA / pub）版**，并准备好 USB-TTL 串口线用于救砖。

---

## 方案概览

| | 内容 |
| --- | --- |
| 内核 | **主推**：mainline 系移植内核 `loonpn/ec6108v9c-linux-kernel`（`hi3798mv100_defconfig`，上游 v6.14.3）<br>**兜底**：厂商 SDK `HiSTBLinuxV100R005C00SPC060`（`linux-4.4.y`） |
| 用户空间 | Debian armhf（`debootstrap` + `qemu-user-static` 构建） |
| 引导 | 保留原厂 U-Boot（Fastboot 3.3.0），`legacy uImage` 引导，DTB 附加在 zImage 尾部 |
| 分区 | `1M(fastboot)` / `1M(bootargs)` / `16M(kernel)` / 剩余 `rootfs(ext4)` |
| 使用形态 | 无头（headless）服务器/轻量 NAS，SSH 登录；图形与视频输出暂不支持 |

---

## 文档

| 文档 | 内容 |
| --- | --- |
| [docs/report.md](docs/report.md) | **初步调研报告**：结论、硬件规格、启动链与分区、内核三条路线、Debian rootfs、刷机流程、风险与里程碑 |
| [docs/references.md](docs/references.md) | 资料索引：子模块源码要点、外部社区资源、工具与镜像下载地址 |

---

## 快速开始

```bash
# 1. 拉取子模块（本仓库依赖三个上游仓库）
#    注意：内核源码树含 Windows 保留文件名，Windows 上 checkout 会失败，需在 WSL/Linux 下执行
git submodule update --init ec6108v9c-linux-kernel
git submodule update --init HiSTBLinuxV100R005C00SPC060
git submodule update --init hi3798mv100

# 2. 构建 mainline 系内核
sudo apt install -y gcc make gettext bison flex bc zlib1g-dev libncurses5-dev \
                    libssl-dev lzma gcc-arm-linux-gnueabihf u-boot-tools
cd ec6108v9c-linux-kernel
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- hi3798mv100_defconfig
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- -j"$(nproc)"

# 3. 打包 U-Boot 可引导的 uImage（DTB 附加在尾部）
cat arch/arm/boot/zImage \
    arch/arm/boot/dts/hi3798mv100-ec6108v9c.dtb > zImage-dtb
mkimage -A arm -O linux -T kernel -C none \
        -a 0X2000000 -e 0X2000000 -n linux -d zImage-dtb uImage

# 4. 构建 Debian armhf rootfs（详见 docs/report.md §5）
```

烧录与刷入步骤（TTL 线刷、HiTool 分区表、U-Boot 参数、`dd` 增量更新内核）见
[docs/report.md §6](docs/report.md)。

---

## 仓库结构

```
.
├── docs/                               # 文档
│   ├── report.md                       # 初步调研报告
│   └── references.md                   # 资料索引
├── ec6108v9c-linux-kernel/             # 子模块：mainline 系内核（主推）
├── HiSTBLinuxV100R005C00SPC060/        # 子模块：厂商 SDK 内核（兜底）
└── hi3798mv100/                        # 子模块：编译/烧录/rootfs 教程
```

---

## 已知限制

- mainline 内核目前**不支持 HDMI 输出与 GPU（Mali-450）**，也无法输出 PCM/S-PDIF 音频 → 只能无头使用。
- L1/L2 cache 未正常工作，性能低于硬件潜力。
- 内置网口 MAC 地址不会由 U-Boot 自动填充，需静态指定。
- 部分批次板载 RTL8188FTV WiFi 存在随机断线。

详细说明与缓解措施见 [docs/report.md §7](docs/report.md)。

---

## 免责声明

本项目仅用于个人学习与自有设备研究。刷机可能导致设备失去原厂保修或无法恢复；因操作不当造成的任何后果由操作者自行承担。
