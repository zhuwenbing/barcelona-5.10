# Barcelona Linux 5.10.1 内核 编译与打包说明

> 目标：把 Barcelona（Intel CE5300）平台的定制 Linux 5.10.1 内核，以**上游 4.3.3 的 `.config` 为基线**做全功能编译，产出 ABI `5.10.1.001+` 的 `.deb` 安装包。
> 本说明在 Ubuntu 24.04 / WSL2 环境实测通过，适用于任何 x86_64 Linux（Debian/Ubuntu 系）。

---

## 0. 背景速览

| 项 | 说明 |
|---|---|
| 上游原版（提供定制与 `.config` 基线） | `https://github.com/matianfu/barcelona-4.3.3` |
| 5.10 移植仓库（含已合并的定制补丁） | `https://github.com/zhuwenbing/barcelona-5.10` |
| 定制内容 | CE5300 PCI 配置模拟、CE5xx SPI NOR、inventec 板级（看门狗/LED/关机）、cf9 关机重启、SATA/HDD LED、GEN3 高时钟串口 |
| 关键提示 | 5.10 仓库 `master` 已含 PR#1（提交 `df64b832c` 修正），**新克隆即可用，无需再打补丁** |

---

## 1. 环境准备

### 1.1 系统依赖
```bash
# Debian/Ubuntu
sudo apt update
sudo apt install -y build-essential dpkg-dev flex bison bc rsync \
    libelf-dev libssl-dev
# dwarves（pahole）仅在开启 CONFIG_DEBUG_INFO_BTF 时必需；本配置默认可不装
```

### 1.2 磁盘与内存
- 源码树约需 **10 GB+** 可用磁盘（全功能编译产物很大）。
- 建议内存 ≥ 8 GB，多核 `-j$(nproc)` 并行。

### 1.3 网络（如需走代理）
GitHub 拉取/推送在受限网络下可带代理：
```bash
git -c http.proxy=http://<PROXY>:<PORT> -c https.proxy=http://<PROXY>:<PORT> clone <url>
```

---

## 2. 获取源码

```bash
git clone https://github.com/matianfu/barcelona-4.3.3   # 上游，仅作 .config 基线
git clone https://github.com/zhuwenbing/barcelona-5.10  # 5.10 移植（已含全部定制修正）
```

---

## 3. 配置迁移（关键步骤）

以上游 4.3.3 的 `.config` 为基线，用 `olddefconfig` 迁移到 5.10，再确认板卡关键选项：

```bash
cd barcelona-5.10

# ① 用上游 4.3.3 配置做基线
cp ../barcelona-4.3.3/.config ./.config
make olddefconfig

# ② 开启板卡与关键驱动（板载 eMMC / SATA / 串口 / SPI-NOR）
./scripts/config --enable X86_INTEL_CE_GEN3 BARCELONA_BOARD
./scripts/config --enable MMC MMC_SDHCI MMC_SDHCI_PCI MMC_BLOCK
./scripts/config --enable SATA_AHCI BLK_DEV_SD
./scripts/config --enable SERIAL_8250 SERIAL_8250_CONSOLE
./scripts/config --enable SPI MTD MTD_SPI_NOR

# ③ 固定版本串（决定最终 ABI）
./scripts/config --set-str LOCALVERSION ".001"
./scripts/config --disable LOCALVERSION_AUTO

# ④ 重新规整配置
make olddefconfig
```

校验最终版本串：
```bash
make kernelrelease        # 应输出：5.10.1.001+
```

> **⚠️ 关键注意**：`LOCALVERSION` 只能写 `.001`，**不要带 `+`**（否则会叠出 `5.10.1.001++` 的错误版本串）。ABI 的 `+` 是内核自动附加的。

### 期望的关键配置值（核验用）
```bash
grep -E '^(CONFIG_MMC=|CONFIG_MMC_SDHCI=|CONFIG_MMC_SDHCI_PCI=)' .config
# MMC=y、MMC_SDHCI=y、MMC_SDHCI_PCI=y（内建进内核 → eMMC root 可挂载）
grep -E '^(CONFIG_SATA_AHCI=|CONFIG_BLK_DEV_SD=)' .config   # 通常为 =m（模块）
grep -E '^(CONFIG_SERIAL_8250=|CONFIG_SERIAL_8250_CONSOLE=)' .config  # =y
# 预期整体规模：约 2094 个 =m、1658 个 =y
```

---

## 4. 编译与打包

> **关键**：Ubuntu 24.04 **没有 `kernel-package`（make-kpkg）**，请使用内核原生的 `bindeb-pkg` 目标。

```bash
cd barcelona-5.10
make -j$(nproc) bindeb-pkg
```

产物生成在**源码树的上一级目录**（即 `barcelona-5.10/..` 下）：

```
linux-image-5.10.1.001+_5.10.1.001+-N_amd64.deb    # N = 构建版本号（如 -3）
linux-headers-5.10.1.001+_5.10.1.001+-N_amd64.deb
linux-libc-dev_5.10.1.001+-N_amd64.deb
```

### 4.1 中断后增量续跑
内核编译可安全中断，`.o` 增量保留。直接**重新执行**上面的 `make -j$(nproc) bindeb-pkg` 即可自动续编剩余部分，无需清理。

---

## 5. 产物验证

```bash
# 包元数据 / ABI
dpkg-deb -f linux-image-5.10.1.001+_*.deb Package Version Architecture

# 内容清单：vmlinuz + 模块总量
dpkg-deb -c linux-image-5.10.1.001+_*.deb | grep -E 'boot/vmlinuz'
dpkg-deb -c linux-image-5.10.1.001+_*.deb | grep -cE '\.ko'

# 关键驱动（应存在：mmc_block.ko、ahci.ko、sata_mv.ko、r8169/igb/e1000/e1000e）
dpkg-deb -c linux-image-5.10.1.001+_*.deb | grep -E '(mmc/block|ata/(ahci|sata_mv)|net/.*(r8169|igb|e1000))'
```

**验证要点**：
- 模块总数应达 **2000+**（全功能配置，非精简版那十几个）。
- `MMC/SDHCI/SDHCI-PCI` 为**内建**（在 `vmlinuz` 内，无独立 `.ko`）——板载 eMMC（sdhci-pci `8086:070b`）才能被枚举、`root=/dev/mmcblk0p1` 才可挂载。
- `mmc_block.ko`、`ahci.ko`、`sata_mv.ko` 应在包内。

---

## 6. 刷机（wisnuc-installer 引导契约）

板级引导约定（刷机脚本 `zhuwenbing/wisnuc-installer` 已适配本内核）：
- 引导文件：`/boot/bzImage`（指向 `vmlinuz-5.10.1.001+`）、`/boot/ramdisk`（initrd）、`/boot/cmdline`
- 内核 cmdline：
  ```
  console=tty0 console=ttyS0,115200 root=/dev/mmcblk0p1 rootwait
  ```

---

## 7. 常见问题排查

| 现象 | 排查 |
|---|---|
| 版本串出现 `5.10.1.001++` | `LOCALVERSION` 误写为 `.001+`，改为 `.001` 后重跑 `make olddefconfig` |
| 板子起不来 / eMMC 无 `/dev/mmcblk0` | 确认 `MMC_SDHCI_PCI=y`（内建），`mmc_block` 在 initrd 内 |
| 串口控制台乱码 | 确认 `SERIAL_8250_CONSOLE=y`，走 GEN3 高时钟（BASE_BAUD=14745600/16） |
| SATA 盘不见 | 确认 `SATA_AHCI`/`sata_mv` 模块加载（`modprobe ahci` / `sata_mv`） |
| 编译很慢 / 中断 | 直接重跑 `make -j$(nproc) bindeb-pkg`，增量续编安全 |
| 缺少依赖报错 | 按第 1.1 节装齐 `libelf-dev`、`libssl-dev`、`bc`、`rsync` 等 |

---

## 8. 参考命令速查

```bash
# 全流程一键（在 barcelona-5.10 内）
cp ../barcelona-4.3.3/.config ./.config && make olddefconfig
./scripts/config --enable X86_INTEL_CE_GEN3 BARCELONA_BOARD \
  MMC MMC_SDHCI MMC_SDHCI_PCI MMC_BLOCK SATA_AHCI BLK_DEV_SD \
  SERIAL_8250 SERIAL_8250_CONSOLE SPI MTD MTD_SPI_NOR
./scripts/config --set-str LOCALVERSION ".001" && ./scripts/config --disable LOCALVERSION_AUTO
make olddefconfig && make kernelrelease
make -j$(nproc) bindeb-pkg
```
