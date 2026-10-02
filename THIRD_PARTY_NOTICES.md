# 第三方代码与许可证说明

本仓库 `aumt/msm-kernel-NX729J` 是在 Android Common Kernel 之上叠加改动的内核树。
内核主体与仓库根 `COPYING` 一致：`GPL-2.0 WITH Linux-syscall-note`（即 GPL-2.0-only）。

下列第三方代码以内嵌副本形式随内核树分发，本文件记录其来源与许可证。
（预编译的 KernelSU 管理器 App 不在本仓库内，不属本文件范围。）

---

## 1. KernelSU（`KernelSU/`）

- **来源**：ReSukiSU 分支 — https://github.com/ReSukiSU/ReSukiSU
- **版本**：`KSU_VERSION = 35137`（对应上游 `git rev-list --count HEAD = 4437`）
- **改动**：无。`KernelSU/` 是上游副本，本仓库未修改其内容。

### 为什么这里有两份 LICENSE

| 文件 | 内容 | sha256（前 16 位） | 与上游同路径文件 |
|---|---|---|---|
| `KernelSU/LICENSE` | GPL-3.0 | `3972dc9744f6499f` | 逐字节相同 |
| `KernelSU/kernel/LICENSE` | GPL-2.0 | `f9c375a1be4a41f7` | 逐字节相同 |

两份并存是**上游自己的安排**，不是本仓库放错了文件：

- `KernelSU/kernel/LICENSE`（GPL-2.0）覆盖内核态模块部分，即 `KernelSU/kernel/`
  下的代码 —— 本仓库编译并分发的正是这一部分。
- `KernelSU/LICENSE`（GPL-3.0）覆盖上游仓库的其余部分（管理器 App、脚本、文档），
  那些内容**不在本仓库内**。

两份均按上游原样保留，未作改动。

> 附注：同源的 `SukiSU-Ultra/SukiSU-Ultra` 已把 `kernel/LICENSE` 换成另一份文本
> （sha256 `8177f975…`），与本仓库不同 —— 本仓库对应的是 **ReSukiSU**。

---

## 2. SUSFS（`fs/susfs.c`、`include/linux/susfs.h`、`include/linux/susfs_def.h`）

- **来源**：SUSFS4KSU — https://gitlab.com/simonpunk/susfs4ksu
- **版本**：`SUSFS_VERSION "v2.2.0"`（定义在 `include/linux/susfs.h`）
- **许可证**：GPL-3.0
- **改动**：**有**。为适配本内核树（5.15 与配套 KernelSU 的 SUSFS 接口）作过修改，
  与上游 v2.2.0 并非逐字一致，差异见提交历史。

由 `CONFIG_KSU_SUSFS` 接入（`fs/Makefile`）；三份源文件顶部均带来源与许可头。

---

## 3. Baseband-guard（`security/baseband-guard/`）

- **来源**：Github@showdo 编写的内核模块（见该目录 `Kconfig`）
- **许可证**：GPL-2.0（该目录自带 `LICENSE`）

---

## 4. Brutal TCP 拥塞控制（`net/ipv4/brutal.c`）

- **许可证**：`SPDX-License-Identifier: GPL-2.0-only`（文件首行自带）

---

## 关于许可证兼容性

本内核树整体为 `GPL-2.0-only`。上面以 **GPL-3.0** 提供的部分（SUSFS 的三份文件，
以及 KernelSU 仓库整体那份）与 `GPL-2.0-only` **不兼容** —— 二者不能在法律上合并成
单一作品再分发。

本仓库的处理方式是**如实保留上游的许可证声明**：不改标成 GPL-2.0，也不删除上游的
LICENSE 文件。再分发者请自行评估这一许可组合。
