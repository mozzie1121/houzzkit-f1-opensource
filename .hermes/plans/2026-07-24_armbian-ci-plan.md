# Plan: Armbian CI Build for HOUZZkit Force 1 (RK3568)

## Goal

在 GitHub Actions 上为 HOUZZkit Force 1 板子自动构建 Armbian 系统镜像，包含：
- 板子专用内核（6.1 + `jl_v1_linux_defconfig` + `rk3568-jl-rm01.dts`）
- 板子专用 U-Boot
- Armbian rootfs（Debian Bookworm）
- 完整可烧录镜像

用户 Push 代码自动触发，每周或手动可触发。

---

## Current Context

板子信息：
- SoC: Rockchip RK3568
- 设备树: `rk3568-jl-rm01.dts` + `rk3568-jl-rm01.dtsi`
- 内核 config: `jl_v1_linux_defconfig`
- 内核版本: 6.1
- WiFi: RTL8822CE（PCIe）
- BSP SDK 结构: 标准 Rockchip Linux SDK（buildroot 体系）
- 原本运行 Ubuntu 24.04，通过 FIT 镜像启动

目录位置：
- BSP: `houzzkit-f1-bsp-k6199/`
- 内核: `houzzkit-f1-bsp-k6199/kernel-6.1/`
- U-Boot: `houzzkit-f1-bsp-k6199/u-boot/`
- rkbin: `houzzkit-f1-bsp-k6199/rkbin/`
- 硬件资料: `houzzkit-f1-hardware/`
- 原有 Ubuntu rootfs: `houzzkit-f1-rootfs-u2404/`

Fork 仓库: `mozzie1121/houzzkit-f1-opensource`
NAS 本地已拉完（16GB），路径 `/volume1/volume1/hermers/houzzkit-f1-opensource/`

---

## Approach

**不手工拼镜像，直接用 Armbian 官方构建系统交叉编译。**

Armbian 已经原生支持 `rockchip-rk3568` 系列板卡，但需要注入自定义设备树和内核配置。核心思路：

1. 在 GitHub Actions 上拉 Armbian 官方构建系统
2. 把板子的 dts/dtsi 注入到 Armbian 的内核源码树
3. 用板子的 defconfig 编译内核
4. Armbian 自动打包成完整 .img

> 为什么不用 ArmBian 的 `USERPATCH` 方式？因为 USERPATCH 只 patch 文件，不能替换整个 defconfig。最佳方案是 fork Armbian 构建仓库然后加 board config，但太重。折中方案：**先编译内核+dtb 为 .deb，再让 Armbian 构建系统用这个 deb 而不是自己编译内核**。

---

## Step-by-step Plan

### Step 1: 内核编译为 .deb 包 ✅ 纯交叉编译

在 GitHub Actions 上用 `aarch64-linux-gnu-` 交叉工具链编译：
- 用 `jl_v1_linux_defconfig`
- 产出 `linux-image-*.deb` + `linux-dtb-*.deb` + `linux-headers-*.deb`

关键命令：
```bash
export ARCH=arm64
export CROSS_COMPILE=aarch64-linux-gnu-
make jl_v1_linux_defconfig
make -j$(nproc) Image dtbs
make -j$(nproc) modules
make INSTALL_MOD_PATH=$PWD/../modules modules_install
make -j$(nproc) bindeb-pkg
```

### Step 2: U-Boot 编译

用 rkbin 工具的 mkimage.sh 打包 uboot.img + MiniLoaderAll.bin。
原本的 SDK 已经有完整的工具链。

### Step 3: 调用 Armbian 构建 rootfs

```
sudo ./compile.sh \
  BOARD=rockchip-rk3568 \
  BRANCH=current \
  RELEASE=bookworm \
  BUILD_DESKTOP=no \
  KERNEL_ONLY=no \
  KERNEL_CONFIGURE=no \
  EXPERT=yes \
  SKIP_KERNEL_CHECK=yes
```

### Step 4: 替换内核

用 Step 1 编译的 .deb 替换 Armbian 打包的内核：
```
dpkg -i linux-image-*.deb linux-dtb-*.deb
```

### Step 5: 打包完整 .img

替换 U-Boot + 内核后，Armbian 重新打包为可烧录的 .img。

---

## 文件改动

新文件：
- `.github/workflows/build-armbian.yml` — CI 主流程
- `patches/README.md` — 说明如何为 Armbian 添加板子支持

仓库源码无需改动，所有 CI 在外部完成。

---

## 验证方式

1. CI 跑完后下载 artifact (.img 文件)
2. 用 `RKDevTool` 或 `dd` 烧录到 SD 卡/eMMC
3. 上电看串口输出是否正常启动
4. 检查网络、WiFi、GPIO、USB 等外设是否驱动

---

## 风险与待定

| 风险 | 应对 |
|------|------|
| Armbian 构建系统 8GB runner 磁盘不够 | 加 `swap` 或减少并发 |
| rkbin 闭源部分（DDR init/ATF）需要板子匹配 | 直接用原 BSP 的 rkbin |
| Armbian 默认不含 rtl8822ce 驱动 | 内核 config 已打开，如果模块化了需打包 firmware |
| Actions 运行时长超 6h | 开启 preemptible runner 或缩小并发 |
| ArmBian rootfs 打包出来的分区布局与板子不匹配 | 用 parameter-ubuntu-fit.txt 的分区参数调整 |

---

## Next

确认方案后，落地：
1. 写 `.github/workflows/build-armbian.yml`
2. Push 到 fork 仓库
3. 手动触发一次验证
