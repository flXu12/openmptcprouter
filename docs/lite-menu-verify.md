# 双端口 lite 菜单 · 验证步骤

验证「9090 出完整菜单、9091 出 lite 菜单」这套机制是否正常工作。

> 新接手项目先看 [handover-guide.md](handover-guide.md)（交接手册）。
> 构建固件的步骤见 [build.md](build.md)。

## 机制一句话

9091 上跑一个独立的 uhttpd 实例，它把 `/cgi-bin/luci` 交给**打过补丁的 CGI 脚本**。
该脚本看到 `SERVER_PORT` 是 `9091` 时，往环境里注入 `LUCI_MENU_DIR=/usr/share/luci/menu.d-lite`；
`dispatcher.uc` 读到这个变量后，改用 lite 的菜单目录来构建路由树。

所以两个端口读的是**两个不同的目录**，菜单和路由都是独立的。

## 涉及的代码位置

| 内容 | 位置 |
|---|---|
| `dispatcher.uc` 支持 `LUCI_MENU_DIR` | `patches/luci-base-menu-dir.patch` |
| CGI 入口按端口注入该变量 | `patches/luci-base-lite-cgi.patch` |
| 两个补丁的门控（仅 6.12/6.17） | `build.sh` 的补丁段落 |
| lite 鉴权 + 分类节点 + ubus/uci 端点 | `openmptcprouter-feeds` → `menu.d-lite/luci-base.json` |
| lite 菜单定义（按规格裁剪后） | `openmptcprouter-feeds` → `luci-app-openmptcprouter/root/usr/share/luci/menu.d-lite/` |
| 创建 9091 实例 + DNAT 转发 | 同包的 `root/etc/uci-defaults/2100-omr-lite-menu` |

**关于 `luci-base.json`**：分类节点（`admin`、`admin/status`、`admin/system`…）和登录鉴权
（`auth.login`）都定义在 luci-base 的菜单里，而 lite 实例只读 `menu.d-lite/`。
缺了它，9091 不仅渲染不出一级菜单，而且**没有登录页**——未登录请求会一直 403。
lite 版 `luci-base.json` 只保留 `admin` 鉴权节点、5 个分类、`admin/ubus/*`、
`admin/uci/*`（JS 视图的 RPC 与配置应用端点）和 `admin/logout`，
砍掉了需要 Lua controller 页面的 `admin/vpn` 等分支。

## 前置条件

- 已按 `OMR_KERNEL=6.12` 构建出 x86_64 镜像（补丁只在 6.12/6.17 生效）
- ⚠️ 2026-09-26 那批固件构建于菜单裁剪之前：其 `menu.d-lite` 仍是完整拷贝，
  且没有 `luci-base.json`（9091 无登录页）。要验证裁剪效果，先按
  [build.md](build.md) 第 8 节增量重建，或在运行中的设备上临时替换 json（同节末尾）
- 在构建服务器上操作

---

## 0. 进入镜像目录

```sh
cd /root/omr-build/openmptcprouter/x86_64/6.12/source/bin/targets/x86/64
IMG=$(ls *squashfs-combined.vdi)
```

## 1. 启动虚拟机

```sh
qemu-system-x86_64 -m 1024 -smp 2 -nographic \
  -drive file="$IMG,format=vdi" \
  -netdev user,id=n0,hostfwd=tcp::9090-:80,hostfwd=tcp::9091-:9091 \
  -device e1000,netdev=n0
```

等出现登录提示（约 1–2 分钟）。

- **退出 qemu：`Ctrl-A` 然后 `X`**
- `hostfwd` 把宿主 `9090`→客机 `80`、`9091`→客机 `9091`，复刻生产环境的入口结构

## 2. 登录

```
OpenMPTCPRouter login: root
Password:               ← 直接回车（默认空密码）
```

## 3. 关掉防火墙

```sh
/etc/init.d/firewall stop
```

**这一步必须做。** OMR 默认拦 WAN 侧入站，而 qemu 的端口转发正是从客户机的 WAN 侧进来——
不关防火墙的话，从宿主机 curl 会得到 `000`（连不上），看起来像功能故障，其实是防火墙挡的。

（测试环境，关掉无妨。客户机内部测试用 localhost，其实不受影响。）

---

## 验证点 1 · 9091 实例有没有建起来

```sh
netstat -ltn | grep -E ':(80|9091)'
```

**期望**：`80` 和 `9091` **都**在 LISTEN。只有 80 → uci-defaults 脚本没生效。

## 验证点 2 · 配置对不对

```sh
uci show uhttpd | grep lite
```

**期望**含这两行：

```
uhttpd.lite.cgi_prefix='/cgi-bin'
uhttpd.lite.listen_http='0.0.0.0:9091' '[::]:9091'
```

`cgi_prefix` 是让 lite 实例走补丁过的 CGI 脚本的关键字段，缺了它 9091 只会提供静态文件。

## 验证点 3 · 端口→菜单目录切换 ★ 核心

### 先制造差异

lite 菜单已按产品规格裁剪，与完整菜单本就不同。这一步要制造一个**可控**的差异：
挑一个在 `menu.d-lite/` 里**确实存在**的文件挪走，让它在 9091 上消失。

```sh
mv /usr/share/luci/menu.d-lite/luci-mod-status.json /tmp/
rm -f /tmp/luci-indexcache*
/etc/init.d/uhttpd restart; sleep 2
```

### 然后测

```sh
P=/cgi-bin/luci/admin/status/overview

curl -s -o /dev/null -w 'lite %{http_code}\n' 127.0.0.1:9091$P
curl -s -o /dev/null -w 'full %{http_code}\n' 127.0.0.1:80$P
```

**期望**：

```
lite 404
full 403
```

**含义**：`admin/status/overview` 在 9091 的路由树里**不存在**（404），在 :80 上存在、进入鉴权（403）。
两者不同 ⇒ **两个实例读的是不同目录** ⇒ 机制生效。

### 恢复

```sh
mv /tmp/luci-mod-status.json /usr/share/luci/menu.d-lite/
rm -f /tmp/luci-indexcache*
/etc/init.d/uhttpd restart
```

## 验证点 4 · 入口 URL 能不能打开 ★ 踩过的坑

验证点 3 测的是**深层路径** `/admin/status/overview`，它只证明"路由树不同"。
但用户实际访问的是**入口** `/cgi-bin/luci/`——这需要 `menu.d-lite/luci-base.json`
里的 `admin` 根节点（`title` + `action: firstchild` + `auth.login`）。

dispatcher 的 `resolve_firstchild()` 只把**同时有 title 和 action** 的子节点当候选落点；
`admin` 根节点缺失时，入口请求找不到任何候选 → `No page is registered at '/'`。

```sh
curl -s -o /dev/null -w '%{http_code}\n' 127.0.0.1:9091/cgi-bin/luci/
```

**期望**：`302`（跳转到登录页或概览页）。
**`404`**：`menu.d-lite` 缺 `luci-base.json` → 2026-10-08 物理机验证踩的就是这个坑。

> 报错文本还能反过来定位问题：`No page is registered at '/'` = lite 目录**有文件**
> 但缺根节点；`No root node was registered` = lite 目录**整个为空/缺失**。

---

## 判读表

| 结果 | 含义 |
|---|---|
| `lite 404` / `full 403` | ✅ 机制正常 |
| 两边都 `403` | `menu.d-lite` 的改动没生效——缓存没清或文件没挪走 |
| 两边都 `404` | 目录被误删，或 `$P` 写错 |
| lite 连接失败 / `000` | 9091 实例没起来 → 回到验证点 1、2 |
| 9091 入口 `/cgi-bin/luci/` 返回 404（深层 URL 可达） | `menu.d-lite` 缺 `luci-base.json`，根节点无法解析 → 验证点 4。2026-10-08 物理机（ext4 制品）即此情况，与制品类型无关 |

## 两个操作提醒

- **命令要短**（70 字符以内）。部分终端/SSH 客户端粘贴时会在终端宽度处插入真换行，
  长命令会被截断成多条，报 `syntax error`。上面用 `$P` 变量就是为了压缩长度。
- `curl` 的 `-o /dev/null` 是丢弃响应体，只看状态码。

## 这个步骤验证了什么 / 没验证什么

**验证了**

- 9091 实例被正确创建并监听
- 它与 :80 读取**不同的菜单目录**
- 页面从 lite 集合里移除后是**真正不可达**（404），不只是菜单里不显示

**没验证**

- `vpn` 区那条 DNAT 转发规则（本机 localhost 测试不经过它）
- 浏览器登录（login 页面由 lite 版 `luci-base.json` 提供，需浏览器确认）
- 跨端口单点登录

## 菜单内容（2026-10-13 已按产品规格裁剪）

一级菜单顺序：Status → System → Modem(5G网络) → Services → Network。

- **Status**：Overview、Realtime Graphs（含 load/bandwidth/wireless/connections 子页）
- **System**：OpenMPTCProuter（聚合设置向导 + 状态）、Administration（管理权）、Backup / Flash Firmware
- **Modem**（5G网络）：仅 Modemdata Status 一个页面——原 4 个子页
  （Modem(s) / Diagnostics / Modem(s) configuration / Package configuration）已从 lite 集合移除。
- **Services**：仅 SNMPD
- **Network**：MPTCP、Interfaces、Wireless、DHCP and DNS、Firewall

要点：

- **安全提醒**：2026-09-26 批次固件的 9091 没有任何鉴权节点（`luci-base.json` 缺失，
  `auth.login` 全部不在 lite 树里）——不仅没有登录页，直接访问具体页面 URL 也**不强制登录**，
  能否打开只取决于该页有没有 ACL 依赖。修复方式同上：补 `luci-base.json`。
- **定制项，本期暂不处理**（2026-10-13 确认）：
  - **OMR Link RTT**（`admin/network/omr-rtt`）——该页面在整个代码库中**不存在**，
    属定制内容，需先定数据来源再实现。
  - **5G 模块（Modemdata）**——同属定制内容，其菜单结构不再调整；lite 中保持
    「仅 Modemdata Status 单项」的现状，`depends` 仍挂在 `luci-app-modemdata` 包上。
- 聚合路由器的"状态"页和它的数据源 `interfaces_status`、向导的保存入口
  `wizard_add` 原本只由 Lua controller 定义，lite 实例不扫描 Lua controller，
  已在 `menu.d-lite/luci-app-openmptcprouter.json` 里以 JSON action 补齐。
- `admin/system/openmptcprouter` 的路径段硬编码为 `openmptcprouter`；
  若 uci `openmptcprouter.settings.menu` 改过菜单名，两处需同步。
- **真机目标**：本流程验的是 x86_64 功能。BPI-R4-PRO-4E 目前不在支持列表内，
  需要先解决该板子的 target/DTS 支持才能产出真机固件。
- 删条目 = 该页面对 9091 **彻底 404**。改完 json 后
  `rm -f /tmp/luci-indexcache*` 再 `/etc/init.d/uhttpd restart` 生效
  （缓存键含文件清单哈希，文件增删本身也会换缓存，但保险起见还是清一下）。
