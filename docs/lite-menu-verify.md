# 双端口 lite 菜单 · 验证步骤

验证「9090 出完整菜单、9091 出 lite 菜单」这套机制是否正常工作。

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
| lite 菜单定义（8 个 json） | `openmptcprouter-feeds` → `luci-app-openmptcprouter/root/usr/share/luci/menu.d-lite/` |
| 创建 9091 实例 + DNAT 转发 | 同包的 `root/etc/uci-defaults/2100-omr-lite-menu` |

## 前置条件

- 已按 `OMR_KERNEL=6.12` 构建出 x86_64 镜像（补丁只在 6.12/6.17 生效）
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

lite 菜单默认是完整菜单的**逐字节拷贝**（裁剪是实现者后续要做的内容）。
两棵树内容相同时，任何路径在两边都存在，测不出区别——所以要先挪走一个。

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

---

## 判读表

| 结果 | 含义 |
|---|---|
| `lite 404` / `full 403` | ✅ 机制正常 |
| 两边都 `403` | `menu.d-lite` 内容与 `menu.d` 相同——**未裁剪**时的正常状态，不是故障 |
| 两边都 `404` | 目录被误删，或 `$P` 写错 |
| lite 连接失败 / `000` | 9091 实例没起来 → 回到验证点 1、2 |

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

- 菜单内容本身（默认仍是完整拷贝，需人工裁剪）
- `vpn` 区那条 DNAT 转发规则（本机 localhost 测试不经过它）
- 跨端口单点登录（需浏览器登录才能确认）

## 已知未完成项

- **菜单裁剪**：`menu.d-lite/*.json` 需按产品需求删条目。改完 `rm -f /tmp/luci-indexcache*`
  再 `/etc/init.d/uhttpd restart` 即生效。注意**删条目 = 该页面对该端口彻底不可达（404）**，
  别误删 lite 用户需要的页面。
- **真机目标**：本流程验的是 x86_64 功能。BPI-R4-PRO-4E 目前不在支持列表内，
  需要先解决该板子的 target/DTS 支持才能产出真机固件。
