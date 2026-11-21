# linux-brokestar 打包文件

本目录属于内核源码仓库（分支 `mykernel`），PKGBUILD 直接以仓库自身为源，
构建出 **7.3.5-Brokestar**。

## 内核来源与改动

- 基线：CachyOS 内核树 tag `cachyos-7.3.5-2`（上游 `CachyOS/linux`）
- 以普通提交形式合入（不使用 patch 文件）：
  - `BORE 7.0.1`（firelzrd/bore-scheduler）
  - `lru_cast v0.2.11`（firelzrd/lru_cast）
  - `sched/fair: adapt BORE to 7.3 protect_slice`（本地适配）
- UKSM 已放弃（与 zswap 冲突），改用标准 KSM；实验分支保留为 `uksm-port`

## 配置

`pkgbuild/config` 里已固化：

| 项 | 值 |
|---|---|
| 时钟 | HZ=500, NO_HZ_FULL, PREEMPT full |
| 编译 | -O3, X86_NATIVE_CPU (znver5), GCC |
| 调度 | SCHED_BORE=y, CACHY=y |
| 内存 | THP=always, KSM=y, LRU_CAST=y (MGLRU 仍在，二选一) |
| 网络 | DEFAULT_TCP_CONG="bbr3" |
| 无线 | MT7925E=m（MT7925 网卡） |

内核版本字符串由 `CONFIG_LOCALVERSION="-Brokestar"` 决定；
构建时额外导出 `LOCALVERSION=""` 以抑制脚本追加的 `+`。

## 构建

```bash
cd pkgbuild
export PATH=/usr/lib/ccache/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/bin
export MAKE=/usr/bin/make LOCALVERSION=""
# CCACHE_DIR / CCACHE_SLOPPINESS / CCACHE_UMASK 由 ~/.zshrc 提供
makepkg -s
```

依赖与 ccache 说明见 PKGBUILD 顶部注释。

## 安装

```bash
sudo pacman -U linux-brokestar-7.3.5-1-x86_64.pkg.tar.zst \
               linux-brokestar-headers-7.3.5-1-x86_64.pkg.tar.zst
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

mkinitcpio 的 pacman hook 会依据 `/usr/lib/modules/7.3.5-Brokestar/pkgbase`
生成 `/boot/vmlinuz-linux-brokestar` 与 initramfs。

## 更新 CachyOS 基线

```bash
git fetch upstream tag cachyos-7.4.x-y
git rebase --onto cachyos-7.4.x-y cachyos-7.3.5-2 mykernel
```

冲突通常只会出现在 BORE 改过的 `kernel/sched/*` 与 lru_cast 改过的 `mm/*`。
