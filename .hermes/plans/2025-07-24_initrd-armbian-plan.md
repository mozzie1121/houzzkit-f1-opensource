# Plan: CI 生成 initrd + Armbian 迁移

## 目标

1. CI 构建的 boot.img 自带 initrd，不再依赖原始 boot 分区的 initrd
2. 将 rootfs 从 Debian Bookworm 改为 Armbian（Ubuntu 24.04-based，RK3568 优化）
3. CI firmware 全量刷入即可工作（uboot.img + boot.img + rootfs.img）

## 当前状态

- CI 用 `debootstrap --arch=arm64 bookworm` 构建最小 rootfs
- boot.img 只放 kernel Image + dtb + extlinux.conf，**没有 initrd**
- extlinux.conf 有 `initrd` 行但指着不存在的文件 → 靠原始 boot 分区里的 initrd 启动
- 原始系统是 Ubuntu 24.04（houzzkit-f1-rootfs-u2404），有完整的 rfsbuild.sh
- 内核编译时 `CONFIG_INITRAMFS_SOURCE=""`，不走内置 initramfs
- CMDLINE 为空，靠 DTB 的 chosen 节点或 bootloader 传参

## 方案

### 第一部分：CI 生成 initrd

在 `Build rootfs with debootstrap` 步骤末尾，chroot 后运行 `update-initramfs -u` 生成 initrd，然后把它拷贝到 boot.img 里。

**改动文件**：`.github/workflows/build-firmware.yml`

**关键步骤**：
1. debootstrap 安装完包后，chroot 内执行：
   ```bash
   apt-get install -y initramfs-tools
   update-initramfs -u -k all
   ```
2. 在 "Assemble firmware package" 的 boot.img 创建阶段，从 rootfs 里把 initrd 拷贝出来：
   ```bash
   INITRD=$(ls mnt-root/boot/initrd.img-* 2>/dev/null | head -1)
   if [ -n "$INITRD" ]; then
     cp "$INITRD" mnt-boot/initrd-${KERNEL_VER}
   fi
   ```
3. extlinux.conf 添加 `initrd /initrd-${KERNEL_VER}` 行
4. 删除 boot.img 创建阶段末尾的 `|| true` 吞错误，让 CI 失败时能暴露问题

### 第二部分：rootfs 迁移到 Armbian

Armbian 本质上是 Ubuntu 24.04 + RK 优化 + 定制 kernel 包。但我们的 kernel 是自己编译的，所以不需要 Armbian 的 kernel，只需要 Armbian 的 userspace（包源、配置、工具）。

**方案 A（推荐）**：直接用现在的 debootstrap 流程，但把包源换成 Armbian 源，安装 Armbian 工具包。

**具体步骤**：

1. 修改 debootstrap 阶段：
   - 用 `debootstrap noble`（Ubuntu 24.04）取代 `bookworm`
   - chroot 后添加 Armbian 源：
     ```bash
     echo "deb https://apt.armbian.com noble main" > /etc/apt/sources.list.d/armbian.list
     ```
   - 安装 Armbian 基础包：`armbian-config armbian-bsp-cli-* armbian-firmware`
   - 保留我们自己的 kernel + modules（不装 Armbian kernel）

2. 添加 Houzzkit 板子特有的配置：
   - `/boot/armbianEnv.txt` → 指定 dtb、fdtfile
   - 网络配置（NetworkManager 预配置）
   - console 配置（ttyFIQ0 / ttyS2 串口）

**方案 B（更轻量）**：只添加 Armbian 的 firmware 包和工具，不改包源，保持 Debian 基础。

推荐方案 A，因为 Armbian 的 userspace 包含了大量 RK3568 优化（温度监控、风扇控制、硬件编解码器、GPU 支持等）。

### 风险与注意事项

| 风险 | 缓解措施 |
|------|----------|
| Armbian 源在中国可能慢 | 可加 mirrors.ustc.edu.cn 源替代 |
| initramfs 生成失败导致 CI 全红 | 保留 fallback：initrd 不存在时不加 initrd 行 |
| Armbian 包体积大 → rootfs.img 更大 | 调整 ROOTFS_SIZE 从 2800 → 3500 |
| Armbian kennel 和我们的冲突 | 明确 exclude linux-image-* armbian-kernel |

### 验证步骤

1. CI 构建成功后下载 artifact
2. 用原始 uboot.img + 新 boot.img + 新 rootfs.img 刷机
3. 验证：系统启动、网络通、LED 亮、SSH 可登
4. `ls /sys/class/leds/` 检查 LED 设备
