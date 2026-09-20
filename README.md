# ImmortalWrt 自动编译 — GL.iNet MT5000 (Brume 3)

使用 GitHub Actions 在线全自动编译带完整定制的 ImmortalWrt 固件,
并支持**编译时自定义**路由 IP、PPPoE 拨号、分区大小等选项。

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
| `hostname` | 路由器主机名 | `ImmortalWrt` |

**PPPoE 密码安全建议**:workflow 输入会显示在运行记录里。推荐把密码配置为
仓库 Secret(Settings → Secrets and variables → Actions → 新建 `PPPOE_PASSWORD`),
运行时 `pppoe_password` 留空即可自动使用。

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

## 维护说明

- 想跟进 ImmortalWrt master:`iwrt_ref` 填 `master`;设备补丁应用失败时
  Action 会明确报错,需基于 PR #24237 最新 head 重新生成 `patches/0001-*.patch`
- 改编译配置:编辑 `config/mt5000.seed`(menuconfig 风格种子,defconfig 展开;
  workflow 的输入项会动态覆盖其中的 IP/分区大小/语言)
- 改首启行为:输入项之外的固化定制放 `files/etc/uci-defaults/`
- 提速:`dl/` 缓存已按选项组合分键;可再加 ccache(seed 加
  `CONFIG_DEVEL=y` + `CONFIG_CCACHE=y`)
- 1G 内存设备建议保持 `bake_daede=false` + 运行时安装;若 baking 后 daed
  面板仍慢,可参考局域网编译机 `~/custom-pkgs/` 中的定制包

## 目录结构

```
.github/workflows/build-mt5000.yml   # CI 流程(含 13 个自定义输入)
patches/0001-*.patch                 # MT5000 设备支持(上游 PR #24237)
config/mt5000.seed                   # 固件配置种子(基础定制)
files/etc/uci-defaults/99-clean-apk-feeds  # 首启清理 404 apk 源
```
