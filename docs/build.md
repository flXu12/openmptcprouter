# 构建操作步骤

在 Ubuntu 服务器上构建 OpenMPTCProuter 固件（以 x86_64 / 内核 6.12 为例）。

> 新接手项目先看 [handover-guide.md](handover-guide.md)（交接手册）。
> 构建完成后如何验证 lite 菜单，见 [lite-menu-verify.md](lite-menu-verify.md)。

## 0. 适用范围

本文记录的是**实际验证通过**的流程：`OMR_TARGET=x86_64` + `OMR_KERNEL=6.12` + `OMR_PACKAGES=mini`。

**未经验证**：`OMR_PACKAGES=full`（会编译 rust 工具链，那是整条链上最重的一步，本机未跑通）。
其他 target（mediatek 等）另需对应的 `config-<target>` 文件。

## 1. 服务器准备

### 依赖（Debian / Ubuntu）

```sh
sudo apt update
sudo apt install -y build-essential gawk libncurses-dev zlib1g-dev libssl-dev libelf-dev \
  git rsync patch diffutils coreutils findutils util-linux bash tar \
  gzip unzip bzip2 wget file debianutils python3 python3-dev python3-setuptools \
  perl ccache qemu-utils
```

两点容易漏：

- **`gawk` 必需**。Debian/Ubuntu 默认装的是 `mawk`，OpenWrt 的 `include/prereq-build.mk`
  检查 GNU awk，`mawk` 通不过。
- **`qemu-utils`** 不在 OpenWrt 的依赖清单里，但 x86_64 会生成 VDI/VMDK/VHDX 镜像，
  缺了它会在**最后一步** `target/linux/install` 失败。

### 资源

| 项目 | 要求 |
|---|---|
| 磁盘 | **≥50 GB**（80 GB+ 更从容）。`OMR_ALL_PACKAGES=yes` 会暴涨，别开 |
| 内存 | ≥8 GB |
| 网络 | 必须能上外网 |
| 时间 | 几小时，建议用 `tmux` 挂住 |

### root 构建的处理

若以 root 构建，**必须**先导出：

```sh
export FORCE_UNSAFE_CONFIGURE=1
```

- 拒绝 root 的不是 OpenWrt（整套代码里没有 root 检查），而是 **GNU autoconf 生成的 `configure`**
- 症状：`configure: error: you should not run configure as root`
- 该变量**必须在执行 `build.sh` 的同一个 shell 里生效**。在一个终端 export、在另一个终端跑构建是无效的
- 更彻底的做法是用普通用户构建（但需把构建树放在 `/home` 下，`/root` 普通用户进不去）

## 2. 拿代码

```sh
git clone -b feature/tag-063 git@github.com:flXu12/openmptcprouter.git
cd openmptcprouter

git fetch origin --tags
git tag | wc -l          # 应为 170 左右

git checkout -f origin/feature/tag-063 -B build
git log --oneline -1     # 确认是最新提交
```

**`--tags` 不能省**。缺 tag 会导致 `build.sh` 里的 `git describe` 失败，`OMR_RELEASE` 变成空值，
固件里的软件源地址会畸形（`http://IP:80/release/-6.12/x86_64`）。

## 3. 关键参数

```sh
OMR_TARGET=x86_64        # 默认值；其他设备见仓库根目录的 config-<target>
OMR_KERNEL=6.12          # ⚠️ 默认是 5.4，必须显式指定
OMR_PACKAGES=mini        # 默认 full
```

⚠️ **`OMR_KERNEL` 必须显式写 `6.12`（或 `6.17`）**。lite 菜单的两个补丁
（`patches/luci-base-menu-dir.patch`、`patches/luci-base-lite-cgi.patch`）的门控就在这两个内核上，
用默认的 5.4 构建出来的固件里**根本没有 lite 菜单**。

## 4. 构建

```sh
export FORCE_UNSAFE_CONFIGURE=1

OMR_TARGET=x86_64 OMR_KERNEL=6.12 OMR_PACKAGES=mini \
  ./build.sh -j4 2>&1 | tee /tmp/omr-build.log
```

- `-j4` 按 CPU 核心数调整
- **`tee` 存日志很重要**：并行构建会把真实报错吞掉，有日志才能事后定位

### 中断后如何续跑

编译成果靠 stamp 保留，**不要删 `build_dir`**。续跑时：

```sh
cd x86_64/6.12/source
export FORCE_UNSAFE_CONFIGURE=1
make -j4 IGNORE_ERRORS=m 2>&1 | tee /tmp/omr-build2.log
```

⚠️ **用 `make`，不要再用 `build.sh`** —— 原因见第 6 节。

## 5. 已知问题与处理

| 症状 | 原因 | 处理 |
|---|---|---|
| `configure: error: you should not run configure as root` | autoconf 的 root 检查 | `export FORCE_UNSAFE_CONFIGURE=1`（同一 shell） |
| `fatal: No names found, cannot describe anything` | 克隆缺 tag | `git fetch origin --tags` |
| `Hash mismatch for file ppp-2.5.2.tar.zst` | 上游镜像的同名文件被覆盖 | 见下方"ppp 特例" |
| `WARNING: Install qemu-img ...`（`target/linux/install` 阶段） | 缺 `qemu-utils` | `apt install qemu-utils` |
| `ERROR: unable to select packages: kmod-ipt-compat-xtables` | feeds 里的幽灵依赖 | 更新 feeds 到含修复的提交 |
| 构建失败但日志里找不到真实错误 | 并行构建吞掉了 | `make -j1 V=s`（不带 `-j`） |
| 报错提到 `ONLY_PREPARE` 相关 | 该变量会**跳过源码克隆** | 首次构建不要设它 |

### ppp 特例

`PKG_MIRROR_HASH` 在 `package/network/services/ppp/Makefile` 里记录的哈希，
与上游镜像现在提供的文件已不匹配（镜像按文件名存放，同名文件被后续构建覆盖过）。

处理：让本地现场生成，把生成的哈希写回。

```sh
cd x86_64/6.12/source

make package/network/services/ppp/compile -j4 2>&1 | tail -20
# 报错里会给出 "got <hash>"，抄下这个哈希

rm -f dl/ppp-2.5.2.tar.zst
sed -i 's/^PKG_MIRROR_HASH:=.*/PKG_MIRROR_HASH:=<上一步抄到的哈希>/' \
  package/network/services/ppp/Makefile

make package/network/services/ppp/compile -j4 2>&1 | tail -20
```

本机生成的哈希是机器相关的，**换台构建机需重新做一遍**。

### 可能的坑：OMR_HOST

`build.sh` 开头有 `set -e`，而 `OMR_HOST` 默认取 `curl -sS ifconfig.co`。
若服务器连不上 `ifconfig.co`，脚本会直接中断且报错看不出与网络有关。
遇到时显式指定：

```sh
OMR_HOST=<服务器公网IP> OMR_TARGET=... ./build.sh -j4
```

## 6. ⚠️ build.sh 会冲掉本地修改

`build.sh` 里的 `_get_repo` 对 OpenWrt 源码和每个 feed 执行：

```sh
git fetch origin -f
git checkout -f origin/<ref> -B build
```

**任何本地修改都会被丢弃**，包括上面那个 ppp 哈希。

所以：

- 做过本地修改后，**续跑用 `make`**（不要 `build.sh`）
- 一旦因为别的原因重跑了 `build.sh`，**必须重新应用所有本地修改**

## 7. 产物

```
x86_64/6.12/source/bin/targets/x86/64/
```

### 该选哪个

| 用途 | 文件 |
|---|---|
| **物理机**（UEFI） | `*-generic-squashfs-combined-efi.img.gz` |
| **物理机**（传统 BIOS） | `*-generic-squashfs-combined.img.gz` |
| 虚拟机 VirtualBox | `*-generic-squashfs-combined.vdi` |
| 虚拟机 VMware | `*-generic-squashfs-combined.vmdk` |

文件名里两组关键词的含义：

- **`squashfs` / `ext4`** —— rootfs 类型。`squashfs` 是 OMR 标准（只读 + overlay，
  支持恢复出厂设置），和真机设备一致；`ext4` 是纯可写分区
- **`combined` / `combined-efi`** —— 完整可启动镜像，后者用于 UEFI 启动
- 两种 rootfs 在功能上**没有差别**：包、补丁、菜单完全一致。固件行为异常时不要先怀疑
  制品类型——2026-10-08 物理机用 ext4 验出 9091 入口 404，与 ext4 无关
  （根因见 [lite-menu-verify.md](lite-menu-verify.md) 验证点 4）

### 不能用来启动的

| 文件 | 说明 |
|---|---|
| `*-roots.img.gz` | 只有 rootfs 分区，不含内核和引导 |
| `*-kernel.bin` | 只有内核 |
| `*-rootfs.tar.gz` | rootfs 的 tar 包 |
| `manifest` / `sha256sums` / `*.buildinfo` | 元数据 |

### 写盘

```sh
gunzip openmptcprouter--*.img.gz
lsblk                                  # 先确认设备
dd bs=4M if=openmptcprouter--*.img of=/dev/sdX conv=fsync
```

⚠️ `of=` 写错设备会抹掉那块盘的数据。写**整块盘**（`/dev/sdb`）而非分区（`/dev/sdb1`）。
测试建议写 U 盘并从 U 盘启动，不动机器原有系统。

## 8. 改了 feeds 包之后如何增量重建

适用场景：改动只在 `openmptcprouter-feeds` 的某个包里（例如 2026-10-13 的 lite 菜单裁剪，
只动了 `luci-app-openmptcprouter`）。不必从零重跑 `build.sh`。

前提：改动**已提交并推送到 fork**——构建机是从 GitHub 拉代码的（第 2 节）。

```sh
# 1. 更新构建机上的 feeds 克隆（做的是 build.sh 里 _get_repo 同样的事）
cd /root/omr-build/openmptcprouter/feeds/openmptcprouter
git fetch origin
git checkout -f origin/feature/tag-063 -B build

# 2. 进 source 树，重新索引并增量编译
cd /root/omr-build/openmptcprouter/x86_64/6.12/source
export FORCE_UNSAFE_CONFIGURE=1
./scripts/feeds install -a -d y -f -p openmptcprouter
make -j4 IGNORE_ERRORS=m 2>&1 | tee /tmp/omr-build3.log
```

- `make` 是增量的：只重编变过的包，再重新生成 rootfs 和镜像，通常几分钟到几十分钟
- **仍然不要跑 `build.sh`**（原因见第 6 节）：它会把 OpenWrt 源码 `checkout -f` 回去，
  ppp 哈希等本地修改再次丢失
- 产物仍在 `bin/targets/x86/64/`；新固件是否真的吃到了改动，看 `version.buildinfo` /
  `manifest` 的时间戳即可

### 只改菜单 json 时，可以完全不动固件

`menu.d-lite/*.json` 是 uhttpd 运行时读取的普通文件（squashfs + overlay 可写）。
在一台**已刷机的设备或虚拟机**上直接替换就能验证，不必重建固件：

```sh
# 把改后的 json 全部放入（scp 或直接粘贴），注意新增文件也要一起放
rm -f /usr/share/luci/menu.d-lite/*.json
# ... 放入新的 menu.d-lite/*.json ...
rm -f /tmp/luci-indexcache*
/etc/init.d/uhttpd restart
```

适合先验证裁剪效果和浏览器登录流程，确认无误后再走增量重建出正式固件。

⚠️ 2026-09-26 那批固件构建于菜单裁剪**之前**：其中 `menu.d-lite` 仍是完整拷贝，
且**没有** `luci-base.json`——所以那批固件的 9091 没有登录页。临时替换时务必把
新增的 `luci-base.json` 一并放入，否则 lite 实例还是进不了登录页。
