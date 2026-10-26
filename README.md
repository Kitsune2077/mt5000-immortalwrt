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
| `iwrt_profile` | 基础版本配置档:master=滚动快照(内核 6.18)/25.12=稳定版(内核 6.12) | `master` |
| `iwrt_ref` | ImmortalWrt 基础版本(commit/tag/branch,留空则按 profile 取默认基线) | 空 |
| `lan_ip` | 默认 LAN IP(含 failsafe IP) | `10.0.0.1` |
| `timezone` | 时区(上海/香港/台北/东京/新加坡/柏林/UTC) | `Asia/Shanghai` |
| `ui_lang` | LuCI 界面语言(zh-cn / en) | `zh-cn` |
| `rootfs_size` | 软件包分区大小(MB,可选 1024~8192) | `2048` |
| `luci_flavor` | LuCI 套件(full / light) | `full` |
| `enable_store` | 集成 iStore 软件商店 | `true` |
| `bake_daede` | 将 daede(dae/daed)编译进固件 | `false` |
| `bake_doona` | 编入 doona+honk(daed 归档后的新一代 Web UI + Rust eBPF 引擎;alpha 实验性,与 `bake_daede` 互斥;内核需 >=6.12,两配置档均满足) | `false` |
| `extra_services` | 附加服务,四选一:`none` / `mwan3`(多线管理,LuCI 网络→Multi WAN;**编入时自动打上"仅接管 IPv4"修复**,见下文) / `docker`(Docker+Dockerman) / `docker+mwan3`(两者) | `none` |
| `pppoe` | 预置 PPPoE 拨号(WAN = eth1) | `false` |
| `pppoe_username` | PPPoE 宽带账号 | 空 |
| `pppoe_password` | PPPoE 密码(留空则用仓库 Secret) | 空 |
| `enable_ipv6` | 启用 IPv6(WAN 委托前缀 + LAN RA/SLAAC 下发,见下文) | `false` |
| `lan_ipv6_assign` | LAN IPv6 分配长度 `ip6assign`(64/60/56/disabled) | `64` |
| `lan_ipv6_suffix` | LAN 接口自身 IPv6 后缀 `ip6ifaceid`(eui64/random/::1) | `eui64` |
| `lan_ipv6_dhcpv6` | LAN DHCPv6 服务(disabled = 纯 SLAAC;server = 有状态分配) | `disabled` |
| `lan_ipv6_dns` | 向客户端通告路由器为 IPv6 DNS | `false` |
| `hostname` | 路由器主机名 | `ImmortalWrt` |
| `root_password` | 默认 root 密码(留空保持无密码;仅首启且尚未设密码时写入) | `root` |
| `luci_extra_apps` | 集成 LuCI 应用清单 `config/luci-extra.seed`(argon 主题/diskman/openclash/rclone/ttyd/upnp,全部官方源;passwall 已注释停用) | `true` |
| `enable_quickstart` | 集成 luci-app-quickstart 首页/向导(iStoreOS 风格;自动带入 linkease nas/nas-luci 源,硬依赖 iStore 与磁盘工具链 e2fsprogs/parted/smartmontools 等,约增加十几 MB;核心为 aarch64 预编译二进制) | `false` |
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
| WAN 接口 | 清掉 `wan` 上的 `ip6assign/ip6class/ip6hint/ip6weight`,写 `delegate='1'`;**删除 `wan.ipv6`**(该值留空/`auto` 才会触发自动 `wan_6`) | 1.2 方案A |
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
- WAN 侧拨号方式不限:`pppoe=true/false` 都支持,但两种拨号获取 PD 的机制不同:
  - **PPPoE**:脚本删除 `wan.ipv6`(留空=auto)后,拨号成功时 ppp 的 `ppp6-up`
    钩子会**自动生成动态 `wan_6` 接口**(DHCPv6 客户端,挂在 `@wan` 上,
    PD 默认开启)——即教程 1.2 方案A 描述的行为。早期版本(r9 时代)脚本显式
    写了 `ipv6='1'`,而 `ppp.sh` 只在 `ipv6` 留空或 `auto` 时才启用自动
    `wan_6`,显式 `1` 只做 IPv6CP——这正是当时 `wan_6` 不出现、需要手动
    `wan6` 的根因。脚本同时删除静态 `wan6`(出厂默认或旧固件残留),避免
    与动态 `wan_6` 形成双 DHCPv6 客户端重复请求 PD
  - **DHCP WAN**:`proto dhcp` **不会**自动生成 `wan_6`,PD 必须由显式
    `wan6` 接口(DHCPv6 客户端,与 `wan` 同端口,`reqaddress=try`、
    `reqprefix=auto`)承载,即教程 1.2 方案B;脚本会确保其存在并规范化选项
- 本方案面向**主路由**架构,不适用于旁路由
- 只对**首启**生效:首启脚本执行过后修改 workflow 选项不会改动已刷机的配置,
  重新构建后请用 `sysupgrade -n`(不保留配置)刷入,或在 LuCI 手动改
- 老基线兼容:`ipv6` 留空在旧版 netifd 上同样触发自动 `wan_6`(`ppp.sh` 的
  `autoipv6` 逻辑久已存在),无需额外兼容项
- 构建时的选项汇总会写进 Release 说明的 `IPv6` 一行,方便回溯某个固件用了哪套参数

## mwan3 与 IPv6(可选组件)

`mwan3` 默认**不编入**,需要时把 `extra_services` 选成 `mwan3`(或 `docker+mwan3`)。
(`docker` / `mwan3` 合并成一个选择项,是因为 GitHub 的 `workflow_dispatch` 单文件最多 25 个输入框。)

**为什么默认关、而且编入时必须打补丁**:mwan3 自带的默认配置里有一条

```
config rule 'default_rule_v6'
	option dest_ip '::/0'
	option family 'ipv6'
	option use_policy 'balanced'
```

而默认的 `balanced` 策略里只有 IPv4 成员(`wan` 是 `family ipv4`,
`wan6` 是 `enabled 0`)。mwan3 对"策略里没有该协议族的可用成员"的处理是
`last_resort unreachable`:它会把**每一个 IPv6 包**打上 `fwmark 0x3e00`,
正好命中 mwan3 自己装的内核规则

```
2062:	from all fwmark 0x3e00/0x3f00 unreachable
```

这条 `ip -6 rule` 的优先级 2062 小于 `main` 表的 32766(数值越小越先命中),于是路由器自身和
**所有下游设备**的 IPv6 流量在路由查找阶段就直接 `ENETUNREACH`。IPv4 走的
是有成员的策略,完全正常——所以典型症状是"**下游能拿到公网 IPv6 地址,却访问
不了任何 IPv6 网站**",并且 `ip -6 route show default` 看着有路由也照样报
`Network is unreachable`。同一条链路还会让 odhcpd 认为"本机没有默认路由",
进而把 RA 的 router lifetime 发成 0(日志 `No default route present, setting
ra_lifetime to 0!`),下游连默认网关都拿不到。

所以 `extra_services` 含 `mwan3` 时,构建流程会额外生成首启脚本 `99-mwan3-ipv4`:

```sh
[ -n "$(uci -q get mwan3.https)" ] && uci -q set mwan3.https.family='ipv4'
uci -q delete mwan3.default_rule_v6
uci -q commit mwan3
```

即**让 mwan3 的规则只作用于 IPv4**(`https` 那条规则原本没写 `family`,
默认 v4/v6 都会装),mwan3 的健康检查、优先级、故障转移对 IPv4 多线上行
照常工作,IPv6 交回内核主路由表。

> 要真正让 mwan3 管理 IPv6,不是把 `mwan3.wan6.enabled` 置 `1` 就行:
> 还得建 `family ipv6` 的接口 + 成员 + 策略(Policy),否则依旧会退回
> `unreachable`。本项目的默认方案是 mwan3 管 IPv4、IPv6 直连主表。

**遇到"有地址、上不了网"的排查顺序**(确认与 mwan3 有关还是无关):

```sh
ip -6 rule show                       # 有没有 fwmark 0x3e00/0x3f00 unreachable
nft list chain ip6 mangle mwan3_rules # 有内容 = mwan3 正在给 IPv6 打标
ip -6 route get 2400:3200::1          # 报 unreachable 但 ping 正常 = 打标所致
uci show mwan3 | grep -E 'rule_v6|family'
```

`ip6 mangle mwan3_rules` 为空、`ip -6 rule` 里那条 2062 虽然仍在但没有包命中,
就是修好了。

## 固件内置内容(默认选项下)

- MT5000 设备支持(上游 PR #24237 移植 + 审阅反馈修复)
- 默认 LAN `10.0.0.1` / 时区 `Asia/Shanghai` / 简体中文 LuCI
- 默认 root 密码 `root`(`root_password` 留空则保持无密码;首启脚本仅在 root
  尚无密码时写入,保留配置升级**不会**覆盖你已改过的密码)
- autocore 状态页(CPU 型号/频率/温度)
- luci-app-run 网页上传/运行安装器(必装, 支持 .run/.sh/.ipk/.apk)
- **USB 口网络全家桶**:`kmod-usb3` 主机栈 + USB 有线网卡(RTL815x/AX88179/ASIX/
  LAN78xx/CDC-Ether)+ **4G/5G 转接板**(QMI/MBIM/NCM + 串口,含 uqmi/umbim)+
  **手机 USB 共享**(安卓 RNDIS/NCM、iPhone ipheth);插上即插即用,
  LuCI「接口-新建协议」里选 QMI/MBIM/NCM/DHCP 即可
- **随身 WiFi 即插即用**:hotplug 钩子 `30-usbmodem-auto` 自动识别
  QMI/MBIM/NCM/RNDIS 设备,插入即自动创建接口 `modem`(自动选对协议、
  wdm 控制口自动探测、APN 默认 `ctnet` 可自行改)、挂入防火墙 wan 区域、
  metric=50 使其默认作为**备份上联**(不抢主 WAN 路由);零点击,
  删除接口后重新插拔即重建;配合 mwan3 可拖入策略设优先级
- **mwan3 多线管理**(可选,仅 `extra_services` 含 `mwan3` 时编入):有线 WAN + 随身 WiFi +
  手机共享等多条上行同时在线,支持**优先级策略**(如 有线优先、随身 WiFi 备份)
  与**健康检查自动切换**(Ping 探测失败自动切到下一条线);配置入口
  LuCI「网络 → Multi WAN」;注意每个上行接口需在防火墙 wan 区域或单独区域。
  编入时首启脚本 `99-mwan3-ipv4` 会把它限定为**只接管 IPv4**(原因见上节
  「mwan3 与 IPv6」),所以默认不会影响 IPv6
- 内核 BTF + XDP_SOCKETS(eBPF CO-RE 前置)
- daede 运行前置依赖全套内置:kmod 五件套 + v2ray geo 数据 + ca-bundle
- iStore 软件商店
- `enable_quickstart=true` 时额外编入 **luci-app-quickstart**(iStoreOS 风格首页/
  网络向导,见选项表;LuCI 菜单进入「向导」/Quickstart,也可自行设为默认首页)
- LuCI 应用精选清单(`luci_extra_apps`,全部官方源):
  argon 主题(3.0.0 起设置页内置,不再有独立配置应用;25.12 基线的 luci feed
  仍为 2.4.7 且无设置页,构建时由 workflow 用 luci master 的 3.x 覆盖注入,
  见「维护说明」)、diskman(含 btrfs/lsblk)、openclash、
  rclone(含 webui/ng)、ttyd、upnp、luci-compat(passwall 已注释停用,
  需要时去掉 `config/luci-extra.seed` 中的注释即可)
- rootfs 2048MB;首启自动清理会 404 的 apk 源行

`bake_daede=false`(默认)时,刷机后用官方脚本安装代理本体:
```
wget -O - https://raw.githubusercontent.com/kenzok8/openwrt-daede/refs/heads/main/scripts/install.sh | ash
```
`bake_daede=true` 时直接编译进固件,并自动应用 **600 秒 reload 超时**补丁
(解决 1G 内存设备 daed 面板应用配置 75s 超时被杀的问题)。
注意:daede 上游采用滚动源码包,若 CI 在下载 daed 源码时失败,多为上游
tarball 更替所致,过几天重试或在其 feed 提 issue。

`bake_doona=true` 时改为编入 **doona + honk**(与 `bake_daede` 互斥,默认关闭)。
这是 daed 归档(2026-09)后 daeuniverse 的新一代组合:doona 是"引擎无关"的静态
Web UI(当前对接 honk,dae 后续接入同一原生 API),honk 是 Rust 重写的 eBPF
透明代理引擎(**alpha 实验性**,兼容 dae 配置语法,TC eBPF + UDP NFQUEUE,
要求内核 >=6.12——两个配置档均满足)。官方不提供 OpenWrt 软件包,构建时从
Zakkaus/doona 的 release 下载(SHA256 双 pin)经 `files/` overlay 编入:
`/usr/bin/honk-core` + `/usr/share/doona` + `/etc/honk/`(含 procd 服务)。
刷机后访问 `http://<LAN_IP>:9527`,首次登录创建管理员;`/etc/honk` 会自动
加入 sysupgrade 备份。honk 明确不建议生产使用(用户态代码多为 AI 生成、
未经完整审计),保守选择请继续用 daede。

## 使用方法

1. 在 GitHub 上新建一个仓库,把本目录全部内容推送上去
   (网页上传也可:`.github/workflows/`、`patches/`、`config/`、`files/`、`README.md`)
2. 仓库 → Actions → **Build ImmortalWrt for GL.iNet MT5000** → Run workflow,
   按需填写上述选项
3. 约 2.5~3.5 小时后(docker/daede 选项会增加时长),在 Artifacts 下载固件
4. 刷机:优先用 `*glinet_gl-mt5000-squashfs-sysupgrade.bin`
   - **从原厂固件过来**:升级页直接上传,或在 SSH 中 `sysupgrade -n`(必须不保留配置)
   - **已在本项目固件之间升级**:直接上传即可,**会保留配置**(设备补丁已把
     `glinet,gl-mt5000` 加入 `platform_copy_config()`);要恢复出厂再加 `-n`

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
- 也可以把包源码直接放进仓库 `packages/` 目录随仓库携带(无需外部 feed):
  workflow 需自行加一段把 `packages/*` 拷入构建树 `package/` 的逻辑
  (曾经的 Glint 主题就是这种集成方式,现已移除)

### 常用第三方源示例

| 仓库 | 内容 |
|---|---|
| `src-git ken https://github.com/kenzok8/openwrt-packages;main` | 常用 LuCI 插件合集(去广告/多播/网易云等) |
| `src-git daede https://github.com/kenzok8/openwrt-daede` | dae/daed 代理(勾选 bake_daede 已内置此源) |
| `src-git istore https://github.com/linkease/istore;main` | iStore 商店(enable_store 已内置) |
| `src-git nas https://github.com/linkease/nas-packages;master` | iStoreOS NAS 包(quickstart 核心二进制,enable_quickstart 已内置) |
| `src-git nasluci https://github.com/linkease/nas-packages-luci;main` | iStoreOS LuCI 应用(luci-app-quickstart 等,enable_quickstart 已内置) |

### 注意事项

- **kmod 类包**:编译期添加没有问题(会针对本固件内核一起编译);
  限制仅在"刷机后运行时在线安装 kmod"的场景(无匹配内核的在线仓库)
- 新增的 LuCI 应用会随 `LUCI_LANG_zh_Hans=y` 自动带上中文翻译(如有)
- Go 类大型插件(如 daed)会显著增加编译时长

## 维护说明

- **配置档与基线**:`iwrt_profile=master`(默认)使用快照基线 `f44d1535b4`(内核 6.18,
  补丁在 `patches/`);`iwrt_profile=25.12` 使用稳定线基线 `84b665b5de`
  (ImmortalWrt `openwrt-25.12` 分支 HEAD @2026-10-24,内核 6.12,已并入上游
  25.12.x(.6 世代)修复——ImmortalWrt 尚无 v25.12.3+ tag,故 pin 分支提交;
  补丁在 `patches-25.12/`)。`iwrt_ref` 显式填写时优先,但需自行确保对应补丁能套用。
  跟进上游时:master 改基线需重验 `patches/0001~0004`;25.12 可跟进新 tag(如
  `v25.12.3`)或 dmsza 上游分支更新,补丁套用失败时 Action 会明确报错,基于新基线
  重新生成 `patches-25.12/0001-*.patch` 即可(源自
  [dmsza/openwrt](https://github.com/dmsza/openwrt) 的 `[25.12]` 回移 commit
  (现为 `af67b61`,原 `e4415712` rebase 而来);为适配 ImmortalWrt 树已剔除
  仅涉及 jiorouter 设备的 hunk,其终态在 ImmortalWrt 上游已存在)。
  `patches-25.12/0002`:795-11 段是上游 `8fbeeb6` 原形态(拒绝 VLAN-aware 桥的
  PPE 卸载);795-12 段采用 dmsza 分支当前形态(`b325c2b` 上下文重制 + `8fbeeb6`
  的 `if (eth->dummy_dev)` 清理防护)——基线 `84b665b5de`(内核 6.12.112)上
  上游 stable 已把 `register_netdev()` 失败路径改为 `err_unreg_netdev`,0001 的
  旧 hunk 形态在内核打补丁阶段 Hunk #2 套不上(且其 device_notifier 清理在
  本树无对应字段、无法编译)
- 想跟进 ImmortalWrt master:`iwrt_ref` 填 `master`;设备补丁应用失败时
  Action 会明确报错,需基于 PR #24237 最新 head 重新生成 `patches/0001-*.patch`
- 设备补丁分三层:`0001` 是 PR #24237 的原始提交(设备支持本体),
  `0002` 是该 PR 的**审阅反馈修复**,`0003` 是上游 `8fbeeb6` 的**6.18 适配版**
  (拒绝 VLAN-aware 桥的 PPE 卸载 + dummy NAPI 分配失败时清理 QDMA notifier),
  均移植自 [dmsza/openwrt](https://github.com/dmsza/openwrt)(0002 对齐 `102a02f`)。
  **注意**:`8fbeeb6` 的 6.18 伴随提交(`b669f86d`,795-12 上下文重写)针对
  dmsza 自己的 openwrt main 树,在 master 固定基线上**内核打补丁阶段套不上**
  (CI 实测失败),因此 0003 不采用其重写形态,而是把功能修复折叠进 0001 的
  原始 hunk(r13 时代已验证可套)。25.12 侧则相反:基线升到 `84b665b5de`
  (6.12.112)后,0001 的旧 hunk 形态反而套不上,`0002` 已采用 dmsza 分支当前
  的重制形态(`b325c2b` + `8fbeeb6`,已对 6.12.112 + pending-6.12 中间态
  逐字节验证)。上游分支再更新时优先增补后续层,
  并确认全链能按顺序干净套用到 `iwrt_ref` 基线
- `patches/0002-*.patch` 的内容:RTL8366UB 默认 VLAN 表清零(默认 PVID 不
  承载任何数据 VLAN)、CPU 口强制只收 tagged 帧、DSA tag 协议号 32→200(避开
  上游已占用 id)、`795-12` 错误路径只回滚已成功注册的 netdev 并注销
  device notifier、`platform.sh` 把 `glinet,gl-mt5000` 加入
  `platform_copy_config()`(eMMC 上 sysupgrade 才会保留配置)、DTS 改名
  `mt7987a-glinet-gl-mt5000` 并补 serial0/stdout-path 等。重新生成 0001 后
  需同步核对 0002(其 hunk 上下文依赖 0001 生成的文件内容)
- `patches/0003-*.patch` 的内容(795-11 段):`mtk_ppe_offload` 拒绝 VLAN-aware
  桥的硬件卸载(tagger 对 VLAN-aware 出口不打标签,PPE 无法用管理 VID 替代);
  (795-12 段):dummy NAPI 分配失败改跳 `err_unreg_netdev` 以清理已注册的
  QDMA notifier,清理路径加 `if (eth->dummy_dev)` 防护
- 改编译配置:编辑 `config/mt5000.seed`(menuconfig 风格种子,defconfig 展开;
  workflow 的输入项会动态覆盖其中的 IP/分区大小/语言)
- argon 主题版本:master 配置档直接用 luci feed master 的 3.x;25.12 配置档
  的 openwrt-25.12 luci feed 仍是 2.4.7(无设置页,该分支也没有独立的
  argon-config 应用),workflow 在 feeds update 后把 luci master 的主题包
  覆盖注入 `feeds/luci` 并校验覆盖后版本为 3.x(3.0 的 `ucode-mod-math`
  依赖在 25.12 基线已具备)。上游把 argon 3.x 回移 openwrt-25.12 luci 分支
  后,删除 workflow 中 "25.12 argon overlay" 那段即可
- 改首启行为:输入项之外的固化定制放 `files/etc/uci-defaults/`;
  IPv6 那套 uci 写入由 workflow 里 `enable_ipv6` 分支生成 `99-ipv6`,
  要加别的 IPv6 项直接改 workflow 那段即可;
  mwan3 的 IPv4-only 修复在 `extra_services` 含 `mwan3` 时生成 `99-mwan3-ipv4`
  (mwan3 默认配置会打标 IPv6,细节见「mwan3 与 IPv6」一节)
- 提速:`dl/` 缓存已按选项组合分键;可再加 ccache(seed 加
  `CONFIG_DEVEL=y` + `CONFIG_CCACHE=y`)
- 1G 内存设备建议保持 `bake_daede=false` + 运行时安装;若 baking 后 daed
  面板仍慢,可参考局域网编译机 `~/custom-pkgs/` 中的定制包
- doona/honk 版本跟进:改 workflow `env` 里的 `DOONA_VERSION` 与两个 SHA256
  (`DOONA_TARBALL_SHA256` / `HONK_CORE_SHA256`,取自对应 release 的 SHA256SUMS);
  honk-core 用 aarch64-musl(mimalloc)资产,升级换版时注意 init 脚本与配置项
  是否随上游文档变动(doona-docs 的 OpenWrt 安装/服务管理页)

## 目录结构

```
.github/workflows/build-mt5000.yml   # CI 流程(含 25 个自定义输入, GitHub 上限)
patches/0001-*.patch                 # MT5000 设备支持(上游 PR #24237, master 基线)
patches/0002-*.patch                 # PR #24237 审阅反馈修复(自 dmsza/openwrt 移植, master 基线)
patches/0003-*.patch                 # VLAN 卸载/NAPI 清理修复(dmsza 8fbeeb6 的 6.18 原形态适配)
patches-25.12/0001-*.patch           # MT5000 设备支持 25.12 回移(自 dmsza/openwrt, 含审阅修复)
patches-25.12/0002-*.patch           # VLAN 卸载修复(8fbeeb6)+ 795-12 重制(b325c2b, 6.12.112 基线)
config/mt5000.seed                   # 固件配置种子(基础定制)
config/luci-extra.seed               # LuCI 应用精选清单(luci_extra_apps 选项控制)
files/etc/uci-defaults/99-clean-apk-feeds  # 首启清理 404 apk 源
files/etc/uci-defaults/99-ipv6       # 仅 enable_ipv6=true 时由 CI 生成
files/etc/uci-defaults/99-mwan3-ipv4 # 仅 extra_services 含 mwan3 时由 CI 生成(限定 mwan3 只管 IPv4)
```

`config/luci-extra.seed` 的维护:这是本固件预置的 LuCI 应用精选清单,已剔除
由 `enable_store`/`bake_daede`/`luci_flavor` 等选项管理的部分。要增删应用
直接编辑该文件(包必须存在于官方 ImmortalWrt feeds,否则 sanity check 会
红牌报出未生效条目)。
