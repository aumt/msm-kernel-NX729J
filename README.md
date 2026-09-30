# 红魔 8S Pro (NX729J) 通用内核

> 红魔 8S Pro / 8S Pro+ (NX729S / NX729J, **SM8550 / kalama**, **GKI 2.0**) 自定义内核，按内核版本分为多个分支（见下方「分支说明」），内核版本串与手机固件**严格对齐**（`vendor_dlkm` vermagic 匹配，可正常加载原厂内核模块）。集成 **KernelSU (ReSukiSU 分支)** 与 SUSFS 等特性，支持 GitHub Actions 云编译与 Linux 服务器本地编译两种方式。

---

## 项目介绍

- **内核版本**：`5.15.x`（GKI 2.0，`BOOT_IMAGE_HEADER_V3` / AK3 `split_boot` 刷入）
- **分支说明**：

  | 分支 | 内核版本 | 基底 | 说明 |
  |------|----------|------|------|
  | `nx729j-5.15.104` | 5.15.104 | ACK `android13-5.15-2023-07_r3` | 与红魔8S Pro 原厂出货内核（**RedMagicOS 9.0.11MR**）版本号严格一致；**已实机验证** |
  | `nx729j-5.15.144` | 5.15.144 | ACK `android13-5.15-2024-02` | 与设备出货内核版本号严格一致，适配**移植版 HyperOS 4**；**已实机验证** |
  | `nx729j-5.15.167` | 5.15.167 | Qualcomm `kernel.lnx.5.15.c5` 重建树 | 用 `.scmversion` 对齐手机固件完整版本串，`vendor_dlkm` 模块可正常加载 |
  | `main` | 5.15.41 | nubia 官方 5.15.41 树 | 官方树，版本串由 git hash 生成 |

- **构建方式差异**：`nx729j-5.15.104` / `nx729j-5.15.144` 使用仓库自带的单文件 `arch/arm64/configs/nx729j_defconfig`（这两个分支**不含** `.github/workflows/`，但 `main` 上的工作流已支持编译它们）；`main` / `nx729j-5.15.167` 使用三合一 `merge_config.sh`。**四个版本均可走「构建方式一」云编译**，也可走「构建方式二」本地编译。
- **版本串对齐机制**：`UTS_RELEASE = 内核版本号 + CONFIG_LOCALVERSION + .scmversion`。5.15.167 树将手机固件的完整版本串（如 `5.15.167-android13-8-00017-gb1f32b310a30-ab12826353`）写入 `.scmversion`，编译出的内核版本号与手机原厂内核完全一致。`nx729j-5.15.104` / `nx729j-5.15.144` 开启了 `CONFIG_MODVERSIONS`，模块校验以符号 CRC 为准、比对时跳过版本串前缀，因此无需 `.scmversion` 对齐。
- **编译工具链**：AOSP LLVM/Clang（云端 `clang-r450784d`，android13 时代 clang-14），`LLVM=1` 全 LLVM 链接，配合 ccache 加速。

---

## 功能特性（workflow 开关）

以下功能通过 workflow `workflow_dispatch` 输入开关控制（默认全部开启），关闭后从 `.config` 移除对应内核符号：

| 功能 | 开关 input | 默认 | 内核配置 |
|------|-----------|------|----------|
| SUSFS（内核级隐藏 root） | `enable_susfs` | ✅ 开 | `CONFIG_KSU_SUSFS` |
| ZRAM LZ4 压缩算法 | `enable_lz4` | ✅ 开 | `CONFIG_ZRAM_DEF_COMP_LZ4` |
| 网络功能增强（ipset + IPv6 NAT，OpenClash 等依赖） | `enable_network` | ✅ 开 | `CONFIG_IP_SET` / `CONFIG_IP6_NF_NAT` |
| BBR / Brutal 等拥塞控制算法 | `enable_bbbrutal` | ✅ 开 | `CONFIG_TCP_CONG_BBR` / `CONFIG_TCP_CONG_BRUTAL` |
| Droidspaces 容器支持（SYSVIPC / 命名空间） | `enable_droidspaces` | ✅ 开 | `CONFIG_SYSVIPC` / `CONFIG_POSIX_MQUEUE` / `CONFIG_IPC_NS` / `CONFIG_USER_NS` |
| 内核级基带保护（防格机） | `enable_bbg` | ✅ 开 | `CONFIG_BBG` + LSM `baseband_guard` |
| eBPF 支持（BPF/JIT/BTF/IKHEADERS，**daed 必需**） | `enable_ebpf` | ✅ 开 | `CONFIG_BPF_SYSCALL` / `CONFIG_BPF_JIT` / `CONFIG_DEBUG_INFO_BTF` / `CONFIG_IKHEADERS` |

> ⚠️ **关闭某些开关有副作用面**（如 `enable_ebpf` 关掉后 daed 无法运行，`enable_droidspaces` 影响容器应用），默认均开启，如非必要请保持默认。

**Always-on**：KernelSU（ReSukiSU 分支，内核级 root，不受开关控制）。

---

## ⚠️ 刷机风险警告

- 刷写内核**有风险**，可能导致无法开机、WIFI/指纹/基带异常等。
- 刷写前务必备份 `boot` 分区（TWRP 或 Android 工具箱）。
- 刷入后如遇问题，请回刷官方 `boot.img`。
- 首次使用内核级 root（KernelSU/SUSFS）请安装对应管理器：**ReSukiSU 管理器** 见 [ReSukiSU_CI](https://github.com/cctv18/ReSukiSU_CI/releases)。

---

## 构建方式一：GitHub Actions（四个版本通用）

无需本地环境，在 GitHub 云端完成全部编译、打包、发布。工作流文件 `build-gki.yml` 只存在于 `main` 分支，由它按所选 `kernel_version` 检出对应分支来编译，因此 `nx729j-5.15.104` / `nx729j-5.15.144` 同样可以在云端编译。

1. **Fork 本仓库** → 点击右上角 **Fork**。
2. 进入你的仓库 → **Actions** → 左侧选择 **Build NX729J GKI (Red Magic 8S Pro)** → **Run workflow**。
3. **填写参数**：

   | 参数 | 说明 |
   |------|------|
   | `kernel_version` | 四选一：`5.15.41`（官方树）/ `5.15.104`（ACK 重建树，RedMagicOS 9.0.11MR）/ `5.15.144`（ACK 重建树，移植版 HyperOS 4）/ `5.15.167`（c5 重建树） |
   | `ccache_update` | 源码/配置变更后开启，强制重建缓存（一般保持关闭） |
   | `package_ak3` | 打包 AnyKernel3 刷机包（默认开启） |
   | `custom_version` | **自定义完整版本串**，留空用默认。**获取方法见文末附录**。系统 OTA 更新后固件版本串变化但内核版本号（如 5.15.167）没变时，用它对齐 `vendor_dlkm`。 |
   | `enable_susfs` / `enable_lz4` / `enable_network` / `enable_bbbrutal` / `enable_droidspaces` / `enable_bbg` / `enable_ebpf` | 各功能开关，见上方特性表 |
   | `create_release` | 构建成功后自动创建 GitHub Release（默认开启） |

4. **等待构建**（约 40–60 分钟，二次构建因 ccache 缓存约 15–30 分钟）。
5. 构建成功后：
   - 若 `create_release` 开启 → 在 **Releases** 页下载 `AnyKernel3-NX729J-<完整版本串>-ReSukiSU.zip` 刷机包；
   - 或到本次运行 **Summary** 页的 **Artifacts** 下载 `Image` / `compiled.config` / 刷机包。
6. **刷机**：见下方「刷机」小节。

---

## 构建方式二：Linux 服务器本地编译（详细教程）

以下为在任意 x86_64 Linux 服务器 / 桌面上完整复现云编译的步骤（含打包 AnyKernel3）。

### 1. 环境要求

- **系统**：Debian / Ubuntu 系（其他发行版自行替换包名）
- **内存**：建议 **≥ 20 GB**（`LTO_CLANG_FULL` 的 `vmlinux` 链接是单进程内存大户，16 GB 会在链接阶段被系统终止）
- **磁盘**：≥ 30 GB 可用
- **依赖**：

  ```bash
  sudo apt-get update
  sudo apt-get install -y binutils-aarch64-linux-gnu gcc-aarch64-linux-gnu \
    binutils python-is-python3 libssl-dev libelf-dev libdw-dev bc dwarves \
    ccache zip unzip git curl
  ```

### 2. 克隆源码 + 切分支

```bash
git clone https://github.com/aumt/msm-kernel-NX729J.git msm-kernel
cd msm-kernel
git checkout nx729j-5.15.104        # 5.15.104 ACK 树（与 RedMagicOS 9.0.11MR 出货内核一致）
# 或 git checkout nx729j-5.15.144   # 5.15.144 ACK 树（移植版 HyperOS 4）
# 或 git checkout nx729j-5.15.167   # 5.15.167 c5 重建树
# 或 git checkout main              # 5.15.41 官方树
```

### 3. 准备 clang（二选一）

**方式 A（推荐，与云端一致）** — AOSP clang-14：

```bash
mkdir -p clang
curl -fL "https://android.googlesource.com/platform/prebuilts/clang/host/linux-x86/+archive/refs/tags/android-platform-13.0.0_r28/clang-r450784d.tar.gz" -o clang.tar.gz
tar -xzf clang.tar.gz -C clang
rm -f clang.tar.gz
clang/bin/clang --version | head -n1
```

**方式 B** — 系统 LLVM（clang-16+ 亦可编译 5.15）：

```bash
sudo apt-get install -y clang lld llvm
```

### 4. 准备 defconfig

**`nx729j-5.15.104` / `nx729j-5.15.144`**：使用仓库自带的单文件 `arch/arm64/configs/nx729j_defconfig`，本步骤无需操作（步骤 6 直接用该名字生成 `.config`）。

**`main` / `nx729j-5.15.167`**：与 `build.config.msm.common` 同参数的 `merge_config.sh`，把 GKI 基础 + 厂商 kalama 片段 + NX729J 差分配置合并为完整 defconfig：

```bash
KCONFIG_CONFIG=arch/arm64/configs/vendor/kalama-NX729J-gki_defconfig \
  bash scripts/kconfig/merge_config.sh -m -r -y \
  arch/arm64/configs/gki_defconfig \
  arch/arm64/configs/vendor/kalama_le_GKI.config \
  arch/arm64/configs/vendor/NX729J-perf_diff.config
```

> `main`（5.15.41）分支将 `kalama_le_GKI.config` 换成 `kalama_GKI.config`。

### 5. 版本串对齐（仅 `nx729j-5.15.167` 需要）

`nx729j-5.15.104` / `nx729j-5.15.144` 已开启 `CONFIG_MODVERSIONS`，模块校验比对时跳过版本串前缀，本步骤可跳过。

把手机固件的完整版本串**去内核版本号前缀后的部分**写入 `.scmversion`（`.scmversion` 只含后缀，否则版本串会重复拼接）：

```bash
# 完整版本串获取方法见文末附录；示例：
echo "-android13-8-00017-gb1f32b310a30-ab12826353" > .scmversion
cat .scmversion
```

### 6. 生成 .config + 编译 Image

```bash
export PATH="$PWD/clang/bin:$PATH"       # 方式 A 需要；方式 B（系统 clang）可省略
export LLVM=1 LLVM_IAS=1 ARCH=arm64 SUBARCH=arm64
export CROSS_COMPILE=aarch64-linux-gnu-
export CC="ccache clang"

# 生成 .config
# main / nx729j-5.15.167：步骤 4 合并出的 defconfig
make -j$(nproc) O=out LLVM=1 ARCH=arm64 CC="$CC" LD=ld.lld OBJCOPY=llvm-objcopy vendor/kalama-NX729J-gki_defconfig
# nx729j-5.15.104 / nx729j-5.15.144：仓库自带的单文件 defconfig
# make -j$(nproc) O=out LLVM=1 ARCH=arm64 CC="$CC" LD=ld.lld OBJCOPY=llvm-objcopy nx729j_defconfig

# 编译内核（约 40–60 分钟）
make -j$(nproc) O=out LLVM=1 ARCH=arm64 CC="$CC" LD=ld.lld OBJCOPY=llvm-objcopy Image
```

> 如需自定义功能开关，参照上方「功能特性」表，用 `./scripts/config --file out/.config -d <符号>` 关闭对应符号，再执行 `make O=out LLVM=1 ARCH=arm64 olddefconfig` 收敛配置。

### 7. 校验产物

```bash
ls -lh out/arch/arm64/boot/Image

# 校验版本串是否与固件完整版本串精确匹配（仅 nx729j-5.15.167）
strings -a out/arch/arm64/boot/Image | grep -oE "5.15.167-android13-8-00017-gb1f32b310a30-ab12826353" | sort -u
```

### 8. 打包 AnyKernel3 刷机包

GKI 2.0 的 `boot` 分区只含内核（ramdisk 在 `init_boot`），因此 AnyKernel3 使用 `split_boot` / `flash_boot` 跳过 ramdisk 解包/重打包，只替换内核：

```bash
git clone --depth=1 https://github.com/osm0sis/AnyKernel3 anykernel3
rm -rf anykernel3/.git
cd anykernel3
cp ../out/arch/arm64/boot/Image ./Image

# 修改 anykernel.sh：机型 / 跳过设备校验 / GKI 2.0 只刷内核
sed -i 's|kernel.string=.*|kernel.string=RedMagic8SPro-GKI|' anykernel.sh
sed -i 's|do.devicecheck=.*|do.devicecheck=0|' anykernel.sh
sed -i 's|supported.manufacturers=.*|supported.manufacturers=|' anykernel.sh
sed -i 's|^BLOCK=.*|BLOCK=boot;|' anykernel.sh
sed -i 's|^dump_boot;|split_boot; # GKI 2.0: ramdisk 在 init_boot, boot 仅内核|' anykernel.sh
sed -i 's|^write_boot;|flash_boot; # 仅替换内核, 跳过 ramdisk 重打包|' anykernel.sh

# 打包（文件名包含完整版本串，便于识别）
zip -r9 "../AnyKernel3-NX729J-5.15.167-android13-8-00017-gb1f32b310a30-ab12826353-ReSukiSU.zip" ./* -x '*.git*'
cd ..
ls -lh AnyKernel3-NX729J-*.zip
```

### 9. 刷机

> 我们默认你已经拥有一定的刷机基础能力，和基本的救砖知识，所以这一部分的文档并不会写得很详细。

任选其一：

- **内核管理器**：使用支持 AnyKernel3 刷机包的内核管理器刷入本 zip（刷入前建议先备份 boot）。
- **TWRP / 卡刷**：重启到 TWRP → 安装本 zip → 重启。

> ⚠️ 刷写前务必备份 `boot` 分区；如遇无法开机，回刷官方 `boot.img`。

---

## 附录：获取固件完整版本串

`custom_version`（云端）或 `.scmversion`（本地）需要手机固件的**完整版本串**。获取方式：

1. **从手机获取（最准确）**：
   ```bash
   adb shell cat /proc/version
   ```
   输出类似：
   ```
   Linux version 5.15.167-android13-8-00017-gb1f32b310a30-ab12826353 (clang ...)
   ```
   取包含 `Linux version` 那一行的**第一个空格段**，即 `5.15.167-android13-8-00017-gb1f32b310a30-ab12826353`。

2. **从已发布的 Release 获取**：直接复制本仓库 Releases 页中最新版「内核版本号」。

**用途**：手机系统 OTA 更新后，固件的完整版本串会变化（如 `...-00017-...` → `...-00020-...`），但内核版本号（如 `5.15.167`）可能不变。此时内核模块（`vendor_dlkm`）的 vermagic 要求与新固件一致，若用旧版本串编译会导致模块加载失败。把新固件的完整版本串填入 `custom_version`（云端）或写入 `.scmversion`（本地），即可让编译出的内核版本串与新固件严格对齐。

> `nx729j-5.15.104` / `nx729j-5.15.144` 开启了 `CONFIG_MODVERSIONS`，模块校验改为比对符号 CRC、比对时跳过版本串前缀，OTA 后无需重新对齐版本串。

---

## 目录结构（与本项目相关）

- `.github/workflows/build-gki.yml` — GitHub Actions 构建工作流（编译 + 打包 + 自动发布），**仅存在于 `main` 分支**
- `arch/arm64/configs/` — `gki_defconfig`；`nx729j_defconfig`（`nx729j-5.15.104` / `nx729j-5.15.144`）；`vendor/kalama*`、`vendor/NX729J-perf_diff.config`（`nx729j-5.15.167` / `main`）
- `vendor/` — nubia 厂商层 / 厂商内核模块源码
