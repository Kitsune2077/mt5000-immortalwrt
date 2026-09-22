# ImmortalWrt 自动编译 — GL.iNet MT5000 (Brume 3)

使用 GitHub Actions 在线全自动编译带完整定制的 ImmortalWrt 固件,
并支持**编译时自定义**路由 IP、PPPoE 拨号、IPv6、分区大小等选项。

> **为什么不用 ImageBuilder(如 wukongdaily/ImmortalWrt-ImageBuilder)?**
> ImageBuilder 只能重新组装**官方已发布**的固件和软件包,且其自定义能力
> 仅限镜像层。MT5000 的设备支持来自尚未合并的上游 PR(openwrt#24237),
> 包含内核补丁,因此必须**全源码编译**——这也让我们能自定义到内核级
> (如 eBPF/BTF),这是 ImageBuilder 做不到的。

## 编译时自定义选项(Run workflow 时填写)

| 选项 | 说明 | 默认值 |
|---|---|---|
| `iwrt_ref` | ImmortalWrt 基础版本(commit/tag/branch) | `f44d1535b4`(已知可用) |
| `lan_ip` | 默认 LAN IP(含 failsafe IP) | `10.0.0.1` |
| `timezone` | 时区(上海/香港/台北/东京/新加坡/柏林/UTC) | `Asia/Shanghai` |
| `ui_lang` | LuCI 界面语言(zh-cn / en) | `zh-cn` |
| `rootfs_size` | 软件包分区大小(MB,可选 1024~8192) | `2048` |
| `luci_flavor` | LuCI 套件(full / light) | `full` |
| `enable_store` | 集成 iStore 软件商店 | `true` |
| `bake_daede` | 将 daede(dae/daed)编译进固件 | `false` |
| `include_docker` | 集成 Docker + Dockerman 管理界面 | `false` |
| `pppoe` | 预置 PPPoE 拨号(WAN = eth1) | `false` |
| `pppoe_username` | PPPoE 宽带账号 | 空 |
| `pppoe_password` | PPPoE 密码(留空则用仓库 Secret) | 空 |
| `enable_ipv6` | 启用 IPv6(WAN 委托前缀 + LAN RA/SLAAC 下发,见下文) | `false` |
| `lan_ipv6_assign` | LAN IPv6 分配长度 `ip6assign`(64/60/56/disabled) | `64` |
| `lan_ipv6_suffix` | LAN 接口自身 IPv6 后缀 `ip6ifaceid`(eui64/random/::1) | `eui64` |
| `lan_ipv6_dhcpv6` | LAN DHCPv6 服务(disabled = 纯 SLAAC;server = 有状态分配) | `disabled` |
| `lan_ipv6_dns` | 向客户端通告路由器为 IPv6 DNS | `false` |
| `hostname` | 路由器主机名 | `ImmortalWrt` |
| `extra_feeds` | 额外软件源(多个用 `\|` 分隔) | 空 |
| `extra_packages` | 额外编译进固件的软件包(空格分隔) | 空 |

**PPPoE 密码安全建议**:workflow 输入会显示在运行记录里。推荐把密码配置为
仓库 Secret(Settings → Secrets and variables → Actions → 新建 `PPPOE_PASSWORD`),
运行时 `pppoe_password` 留空即可自动使用。

## IPv6 设置(勾选 `enable_ipv6` 后生效)

默认**关闭**,即不打任何 IPv6 相关配置,保持上游 ImmortalWrt 默认行为。
勾选后,首启脚本 `99-ipv6` 会按
[Aethersailor wiki《OpenWrt IPv6 设置方案》](https://github.com/Aethersailor/Custom_OpenClash_Rules/wiki/OpenWrt-IPv6-%E8%AE%BE%E7%BD%AE%E6%96%B9%E6%A1%88)
(主路由 + 运营商下发 PD 前缀的架构)写入:

| 位置 | 写入的配置 | 对应 wiki 步骤 |
|---|---|---|
| WAN 接口 | 删除单独的 `wan6`,改在 `wan` 上 `delegate='1'`(旧版 netifd 另写兼容项 `ipv6='1'`),并清掉 `ip6assign/ip6class/ip6hint/ip6weight` | 1.2 方案A |
| WAN DHCP | `dhcp.wan.ignore='1'`(WAN 侧不对外提供 DHCP) | 1.2 方案A 第 5 步 |
| LAN 接口 | `ip6assign`(默认 `64`)、`ip6ifaceid`(默认 `eui64`) | 1.3.1 / 1.3.2 |
| LAN RA | `ra='server'` + `ra_slaac='1'`,客户端 SLAAC 自动生成地址 | 1.3.3 |
| LAN DHCPv6 | 默认 `disabled`(纯 SLAAC,兼容不支持有状态 DHCPv6 的安卓);选 `server` 时自动给 RA 带上 M/O 标志 | 1.3.3 |
| LAN NDP | `ndp='disabled'`(NDP 代理关闭) | 1.3.3 |
| LAN DNS | 默认 `dns_service='0'`:不通告路由器为 IPv6 DNS,客户端继续用路由器 IPv4 地址(默认 `10.0.0.1`)解析,保住 OpenClash 的分流链路 | 1.3.3 |
| dnsmasq | `filter_aaaa='0'`:不过滤 IPv6 AAAA 记录 | 1.1 |

`dhcpv6=disabled` 时脚本把 `ra_flags` 写成 `none`(RA 不带 M/O 标志);
`dhcpv6=server` 时写成 `managed-config` + `other-config`。若不这么做,
odhcpd 的默认 O 标志会让客户端以为"还有 DHCPv6 可以问 DNS",徒增等待与超时。

### 几个选项怎么选

- `lan_ipv6_assign` 默认 **64**:上游只委派 /64 时也能用(60 会分不出来);
  若还要往二级路由继续委派子网,选 `60`/`56`;选 `disabled` 则 LAN 只保留 ULA,
  不下发公网 IPv6。
- `lan_ipv6_suffix` 默认 **eui64**:路由器 LAN 接口自身的地址后缀由 MAC 派生,
  上游前缀变化时后缀不变,便于防火墙按后缀放行(见 wiki 第 3 节)。
- `lan_ipv6_dns` 默认 **false**:按 wiki 的思路"DNS 走 IPv4、业务流量走 IPv6",
  避免运营商 IPv6 DNS 抢答绕过 OpenClash。确有需要再打开。

### 前提与注意

- 需要光猫已开启 IPv6(建议桥接)且宽带**运营商下发 PD 前缀**;拿不到 PD 前缀,
  这套配置无法让内网获得公网 IPv6
- WAN 侧拨号方式不限:`pppoe=true/false` 都支持(脚本通过 `wan.delegate` 触发
  内建 IPv6 客户端,PPPoE 下会走 IPv6CP + DHCPv6-PD)
- 本方案面向**主路由**架构,不适用于旁路由
- 只对**首启**生效:首启脚本执行过后修改 workflow 选项不会改动已刷机的配置,
  重新构建后请用 `sysupgrade -n`(不保留配置)刷入,或在 LuCI 手动改
- 若把 `iwrt_ref` 换成 2024 年之前的老基线,`delegate` 这个新选项名会被忽略,
  脚本里同时写入了旧名 `ipv6='1'` 做兼容
- 构建时的选项汇总会写进 Release 说明的 `IPv6` 一行,方便回溯某个固件用了哪套参数

## 固件内置内容(默认选项下)

- MT5000 设备支持(上游 PR #24237 移植)
- 默认 LAN `10.0.0.1` / 时区 `Asia/Shanghai` / 简体中文 LuCI
- autocore 状态页(CPU 型号/频率/温度)
- 内核 BTF + XDP_SOCKETS(eBPF CO-RE 前置)
- daede 运行前置依赖全套内置:kmod 五件套 + v2ray geo 数据 + ca-bundle
- iStore 软件商店
- rootfs 2048MB;首启自动清理会 404 的 apk 源行

`bake_daede=false`(默认)时,刷机后用官方脚本安装代理本体:
```
wget -O - https://raw.githubusercontent.com/kenzok8/openwrt-daede/refs/heads/main/scripts/install.sh | ash
```
`bake_daede=true` 时直接编译进固件,并自动应用 **600 秒 reload 超时**补丁
(解决 1G 内存设备 daed 面板应用配置 75s 超时被杀的问题)。
注意:daede 上游采用滚动源码包,若 CI 在下载 daed 源码时失败,多为上游
tarball 更替所致,过几天重试或在其 feed 提 issue。

## 使用方法

1. 在 GitHub 上新建一个仓库,把本目录全部内容推送上去
   (网页上传也可:`.github/workflows/`、`patches/`、`config/`、`files/`、`README.md`)
2. 仓库 → Actions → **Build ImmortalWrt for GL.iNet MT5000** → Run workflow,
   按需填写上述选项
3. 约 2.5~3.5 小时后(docker/daede 选项会增加时长),在 Artifacts 下载固件
4. 刷机:优先用 `*glinet_gl-mt5000-squashfs-sysupgrade.bin`
   (原厂固件升级页直接刷;或 `sysupgrade -n` 不保留配置)

## 产物发布

构建成功后会自动:

- 上传 **Artifacts**(保留 30 天,登录后可下载)
- 创建 **Release**,tag 为 `mt5000-r<运行编号>`,附件包含:
  固件镜像、`sha256sums` 校验、`*.manifest` 软件清单、`config.buildinfo` 构建配置;
  Release 说明自动生成本次构建的选项汇总表、内置功能清单、刷机方法与
  daede 安装指引(永久保留,方便随时回溯某个版本)

## 如何添加软件包 / 软件源

### 方式一:编译时临时添加(不改仓库)

Run workflow 时填写:

- **`extra_packages`** — 空格分隔的包名,直接编入固件。
  例:`luci-app-zerotier htop fdisk`
- **`extra_feeds`** — 额外软件源,多个用 `|` 分隔,每个是标准 feeds.conf 行。
  例:`src-git ken https://github.com/kenzok8/openwrt-packages;main`

  与官方源出现**同名包冲突时,额外源会强制接管**(如需官方版请勿添加同名包)。
  名称写错或依赖不满足时,sanity check 会红牌列出未生效的包。

### 方式二:固化到仓库(永久默认)

- **软件包**:编辑 `config/mt5000.seed`,添加 `CONFIG_PACKAGE_xxx=y`
- **软件源**:参考 workflow 中 istore 的写法,在 feeds 步骤把
  `echo 'src-git <名称> <地址>[;分支]' >> feeds.conf.default` 写死
- 也可以把包源码直接放进仓库 `packages/` 目录随仓库携带(无需外部 feed)

### 常用第三方源示例

| 仓库 | 内容 |
|---|---|
| `src-git ken https://github.com/kenzok8/openwrt-packages;main` | 常用 LuCI 插件合集(去广告/多播/网易云等) |
| `src-git daede https://github.com/kenzok8/openwrt-daede` | dae/daed 代理(勾选 bake_daede 已内置此源) |
| `src-git istore https://github.com/linkease/istore;main` | iStore 商店(enable_store 已内置) |

### 注意事项

- **kmod 类包**:编译期添加没有问题(会针对本固件内核一起编译);
  限制仅在"刷机后运行时在线安装 kmod"的场景(无匹配内核的在线仓库)
- 新增的 LuCI 应用会随 `LUCI_LANG_zh_Hans=y` 自动带上中文翻译(如有)
- Go 类大型插件(如 daed)会显著增加编译时长

## 维护说明

- 想跟进 ImmortalWrt master:`iwrt_ref` 填 `master`;设备补丁应用失败时
  Action 会明确报错,需基于 PR #24237 最新 head 重新生成 `patches/0001-*.patch`
- 改编译配置:编辑 `config/mt5000.seed`(menuconfig 风格种子,defconfig 展开;
  workflow 的输入项会动态覆盖其中的 IP/分区大小/语言)
- 改首启行为:输入项之外的固化定制放 `files/etc/uci-defaults/`;
  IPv6 那套 uci 写入由 workflow 里 `enable_ipv6` 分支生成 `99-ipv6`,
  要加别的 IPv6 项直接改 workflow 那段即可
- 提速:`dl/` 缓存已按选项组合分键;可再加 ccache(seed 加
  `CONFIG_DEVEL=y` + `CONFIG_CCACHE=y`)
- 1G 内存设备建议保持 `bake_daede=false` + 运行时安装;若 baking 后 daed
  面板仍慢,可参考局域网编译机 `~/custom-pkgs/` 中的定制包

## 目录结构

```
.github/workflows/build-mt5000.yml   # CI 流程(含 20 个自定义输入)
patches/0001-*.patch                 # MT5000 设备支持(上游 PR #24237)
config/mt5000.seed                   # 固件配置种子(基础定制)
files/etc/uci-defaults/99-clean-apk-feeds  # 首启清理 404 apk 源
files/etc/uci-defaults/99-ipv6       # 仅 enable_ipv6=true 时由 CI 生成
```
