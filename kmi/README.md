# kmi/ —— 内核模块 ABI（KMI）参考表

这里放的是**手机厂商模块对内核导出面的要求**，供 CI 的「KMI 闸门」比对。

## 为什么需要它

手机上的厂商模块（`/vendor/lib/modules` 的 276 个 + `vendor_boot.img` ramdisk 里的 305 个 `.ko`）
是**预编译**的。它们能不能装进我们自编的内核，只取决于两件事：

1. 模块依赖的符号，内核有没有导出（少了 ⇒ 装载失败）
2. 那些符号的 CRC 对不对（`CONFIG_MODVERSIONS=y`，对不上 ⇒ `-ENOEXEC`）

CRC 由符号的**函数原型**算出，所以只要导出面没变，CRC 就不会变。反过来，**任何一个 CRC 变了都说明
ABI 真的动了** —— 这种改动会让手机开不了机或掉功能，必须在 CI 阶段就红，而不是等刷机。

最典型的一次翻车：编译时用了 GNU `ld` 而不是 `ld.lld` ⇒ Kconfig 自动 `LTO_NONE=y` ⇒
`CFI_CLANG` 连带消失 ⇒ 一大批符号 CRC 变样 ⇒ 刷进去直接开不了机。
`module_layout` 这一个数就是那次事故的照妖镜。

## 表怎么来的

```
# 1) 在手机上取厂商模块（两套）
adb shell 'su -c "tar -C /vendor/lib/modules -cf - ."' > vendor-modules.tar   # vendor_dlkm
# 2) vendor_boot.img 第一段 ramdisk 里的 .ko 解出来（legacy LZ4 帧，见 analysis/）
# 3) 逐个取 __versions 段
for ko in *.ko; do modprobe --dump-modversions "$ko"; done \
  | awk 'NF>=2{print $2"\t"$1}' | sort -u > vendor-all.tsv        # 符号, CRC
# 4) 与一次【验收通过】的本机构建取交集
awk -F'\t' 'NR==FNR{h[$2]=1;next} ($1 in h)' out/vmlinux.symvers vendor-all.tsv \
  | sort > kmi/nx729j-<版本>.crc.tsv
```

`vmlinux.symvers` 由 `make Image` 生成（第 1 列 CRC、第 2 列符号名）。
**必须用验收通过的那次构建**：验收 = 与手机厂商模块逐符号比对 CRC 不符 0、`module_layout` 一致、
CFI 反汇编带阳性对照。否则等于把错误固化进闸门。

## 换取新版（例如 ROM 升级后）

按上面的步骤重新生成同名文件，然后**先在本机核对一遍**：新表对本机构建应当
`未导出 0, CRC 不符 0`。若不为 0，说明是新内核真的动了 ABI —— 那是要先解决的问题，不是改表能绕过的。

## 目前覆盖情况

| 内核版本 | 参考表 | 说明 |
|---|---|---|
| 5.15.41  | ✅ `nx729j-5.15.41.crc.tsv`（2777 条） | 默认分支，2026-10-02 验收基线 |
| 5.15.104 | ❌ | 尚无"构建提交 == 分支 tip"的验收构建可用 |
| 5.15.144 | ❌ | 本机无该版本构建与厂商模块数据 |
| 5.15.167 | ❌ | 同上 |

没有表的版本，闸门会**明确打印"跳过"**并继续（不会静默变绿）。补齐办法：对该分支跑一次完整
本机验收，按上面的步骤生成 `nx729j-<版本>.crc.tsv` 并提交即可。
