# OpenMPTCProuter 定制固件 · 交接手册

> 面向第一次接手本项目的人，只讲"做什么、为什么"。命令级别的细节在两份专业文档里：
> [build.md](build.md)（构建）、[lite-menu-verify.md](lite-menu-verify.md)（lite 菜单验证）。

---

## 1. 这个项目在做什么

OpenMPTCProuter（OMR）是一个基于 OpenWrt 的软路由系统，把多条上网链路（有线、4G/5G…）
聚合成一条，经 VPS 汇总后出网。

本项目的定制点是**一台设备、两个管理入口**：

| 入口 | 界面 | 给谁用 |
|---|---|---|
| `:80` / `:443` | 完整 LuCI 菜单 | 管理员 |
| `:9091` | 精简菜单（lite） | 普通使用者 |

实现方式一句话：9091 上跑第二个 uhttpd 实例，构建期两个补丁让 CGI 入口看到
"请求来自 9091"时，改读另一个菜单目录 `/usr/share/luci/menu.d-lite/`。

## 2. 电脑里的四个目录（工作区地图）

工作区在 `/Users/qingcheng/Desktop/workspace/suko/omr/`，**构建不在本机**，
在 Ubuntu 构建机的 `/omr-build/openmptcprouter/`。

| 目录 | 是什么 | 你会不会改它 |
|---|---|---|
| `openmptcprouter/` | 构建框架（flXu12 fork，分支 `feature/tag-063`） | 改补丁、build.sh、docs/ 文档 |
| `openmptcprouter-feeds/` | 109 个软件包（同一个 fork 的另一个仓库） | **最常改**——lite 菜单 json 在这里 |
| `luci/` | OpenWrt 上游 LuCI 参考快照 | 只读，不改 |
| `openwrt/` | OpenWrt 上游参考快照 | 只读，不改 |

## 3. 改什么 → 动哪里 → 怎么生效（速查表）

| 想做的事 | 改的文件 | 生效方式 |
|---|---|---|
| lite 菜单增删页面 | feeds → `luci-app-openmptcprouter/root/usr/share/luci/menu.d-lite/*.json` | 见第 6 节两条路 |
| 9091 实例 / 端口 / DNAT | feeds → 同包 `root/etc/uci-defaults/2100-omr-lite-menu` | 增量重建 |
| 菜单目录切换机制 | `openmptcprouter/patches/luci-base-menu-dir.patch`（dispatcher） | 增量重建 |
| 9091 端口注入 | `openmptcprouter/patches/luci-base-lite-cgi.patch`（CGI 入口） | 增量重建 |
| 构建流程本身 | `openmptcprouter/build.sh` | 重跑（慎用，见坑 3） |

改任何代码后，**先 commit 再 push 到 GitHub**（fork 仓库 `flXu12/*`，分支
`feature/tag-063`）——构建机是从 GitHub 拉代码的，本地工作区的改动构建机看不到。

## 4. 从源码到固件（正常流程）

```
本机改代码 → commit + push → 构建机 git pull → 构建 → 产物目录拿固件 → 刷机验证
```

1. 构建：`OMR_TARGET=x86_64 OMR_KERNEL=6.12 OMR_PACKAGES=mini ./build.sh -j4`
   （详见 build.md §4；首次构建几小时，建议 tmux）
2. 产物在构建机 `x86_64/6.12/source/bin/targets/x86/64/`（详见 build.md §7）
3. **两个必背警告**：
   - `OMR_KERNEL` 默认是 5.4，**必须显式写 6.12**，否则固件里根本没有 lite 菜单
   - `build.sh` 会把源码 `checkout -f` 回上游，冲掉所有本地修改——续跑用 `make`（build.md §6）

## 5. 刷机与首次验证

### 选哪个文件

| 用途 | 文件 |
|---|---|
| 物理机 UEFI 启动 | `*-generic-squashfs-combined-efi.img.gz` |
| 物理机传统 BIOS | `*-generic-squashfs-combined.img.gz` |
| VirtualBox / VMware | 对应 `.vdi` / `.vmdk` |

`squashfs` 与 `ext4` 制品**功能完全一致**（2026-10-08 用 ext4 在物理机验证过——
那次发现的 9091 404 与制品类型无关，见坑 1）。正式交付用 squashfs。

### 写盘（物理机）

```sh
gunzip openmptcprouter--*.img.gz
lsblk                                  # ⚠️ 先确认目标盘
dd bs=4M if=openmptcprouter--*.img of=/dev/sdX conv=fsync
```

⚠️ `of=` 写错设备会抹掉那块盘。建议先用 U 盘试，不动机器原有系统。

### 第一次开机

1. 浏览器打开 `http://192.168.100.1`，root 登录（默认空密码）
2. **立即设置密码**（页面上有黄色提醒条）
3. SSH 可连（dropbear），后续排查都在命令行做

### 验证双端口双菜单

浏览器：

- `http://192.168.100.1/cgi-bin/luci/` → 完整菜单（状态/系统/服务/网络/VPN）
- `http://192.168.100.1:9091/cgi-bin/luci/` → 精简菜单，**出现登录页**才算正常

设备上 4 条命令（判读详见 lite-menu-verify.md）：

```sh
netstat -ltn | grep -E ':(80|9091)'                    # 两个口都该 LISTEN
uci show uhttpd | grep lite                            # 应有 lite.cgi_prefix='/cgi-bin'
ls /usr/share/luci/menu.d-lite/ | wc -l                # 应为 9（含 luci-base.json）
curl -s -o /dev/null -w '%{http_code}\n' 127.0.0.1:9091/cgi-bin/luci/   # 应为 302
```

## 6. 改了菜单之后怎么落地（两条路）

lite 菜单一共 **9 个 json**（`menu.d-lite/` 目录下）：
`luci-base.json`（鉴权 + 分类骨架，**缺它入口就 404**，见坑 1）+ 8 个模块菜单。

### 路A · 临时替换（约 5 分钟，验证用，不动固件）

菜单 json 是运行时读取的普通文件，在已刷机的设备上直接替换：

```sh
# 本机：把 9 个 json scp 到设备
scp openmptcprouter-feeds/luci-app-openmptcprouter/root/usr/share/luci/menu.d-lite/*.json root@192.168.100.1:/usr/share/luci/menu.d-lite/

# 设备上：清缓存 + 重启
ssh root@192.168.100.1 'rm -f /tmp/luci-indexcache* && /etc/init.d/uhttpd restart'
```

### 路B · 增量重建（出正式固件，几十分钟）

前提：改动已 push。在构建机上（详见 build.md §8）：

```sh
cd /omr-build/openmptcprouter/feeds/openmptcprouter
git fetch origin && git checkout -f origin/feature/tag-063 -B build

cd /omr-build/openmptcprouter/x86_64/6.12/source
export FORCE_UNSAFE_CONFIGURE=1
./scripts/feeds install -a -d y -f -p openmptcprouter
make -j4 IGNORE_ERRORS=m 2>&1 | tee /tmp/omr-build3.log
```

## 7. 踩坑实录（读一遍，少走弯路）

**坑 1 · 9091 入口 404（2026-10-08 物理机实测踩中）**

- **现象**：`:80` 一切正常；`:9091` 打开 `/cgi-bin/luci/` 显示
  `404 No page is registered at '/'`，LuCI 页脚正常渲染
- **根因**：旧固件 `menu.d-lite` 是 8 个模块菜单的拷贝，**缺 `luci-base.json`**。
  LuCI 入口跳转要求 `admin` 根节点**同时有 title 和 action**（`resolve_firstchild()`，
  dispatcher.uc），这个节点只存在于 luci-base.json 里。缺了它，入口找不到落点就 404
- **修复**：补上 lite 版 `luci-base.json`（已在 2026-10-13 的未提交改动里），
  按第 6 节两条路任选其一落地
- **教训**：验证不能只测深层 URL——深层 URL 可达、入口 404，两件事同时成立
- **附带安全问题**：缺这个文件的固件，9091 上**没有登录页也不强制登录**，
  直接输页面 URL 是否可见只取决于该页的 ACL

**坑 2 · `OMR_KERNEL` 默认 5.4**：不显式写 6.12，lite 补丁根本不应用，9091 无菜单。

**坑 3 · `build.sh` 冲掉本地修改**：它对源码和每个 feed 执行 `git checkout -f`。
ppp 哈希等本地补丁会丢。做过本地修改后，续跑只用 `make`。

**坑 4 · qemu 验证时 curl 得到 `000`**：不是固件问题，是 OMR 默认防火墙挡了
WAN 侧入站（qemu 端口转发恰好从 WAN 进）。设备里 `/etc/init.d/firewall stop` 再测。

**坑 5 · 构建环境**：Ubuntu 默认 `mawk` 过不了 OpenWrt 检查，必须装 `gawk`；
x86_64 镜像生成需要 `qemu-utils`；root 构建必须 `export FORCE_UNSAFE_CONFIGURE=1`
（同一个 shell 里生效）。完整清单见 build.md §1、§5。

## 8. 已知限制与待办

| 项 | 状态 |
|---|---|
| lite 菜单按规格裁剪（5 改 1 增，含 `luci-base.json`） | 改动完成，**未提交**——提交 push 后按 §6 路B 出正式固件 |
| OMR Link RTT（`admin/network/omr-rtt`） | 定制项，代码库中不存在，本期暂不处理（2026-10-13 确认） |
| 5G 模块（Modemdata） | 定制项，lite 中保持"仅 Modemdata Status"现状，暂不处理 |
| 9091 端口号 | 硬编码在 lite-cgi 补丁和 DNAT 规则两处，改端口需同步 |
| `admin/system/openmptcprouter` 路径段 | 硬编码为 `openmptcprouter`，改菜单名需同步 |
| 真机目标 BPI-R4-PRO-4E | 不在支持列表，仅 x86_64 验证过 |
| `OMR_PACKAGES=full` | 会编 rust 工具链，未跑通；目前用 mini |

## 9. 求助时提供什么信息

1. 固件版本号：浏览器页面底部 `Powered by openmptcprouter x.x.x`，或构建产物里的 `version.buildinfo`
2. 设备上这三条的输出：
   ```sh
   uci show uhttpd | grep lite
   ls /usr/share/luci/menu.d-lite/
   curl -s -o /dev/null -w '%{http_code}\n' 127.0.0.1:9091/cgi-bin/luci/
   ```
3. 你参考了哪份文档的哪一节——文档本身也可能有错，欢迎修订
