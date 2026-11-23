# fanchmwrt-tr3000 — 自编译 FanchmWrt 固件（Cudy TR3000 256MB v1）

基于 [fanchmwrt/fanchmwrt](https://github.com/fanchmwrt/fanchmwrt) 分支 `fanchmwrt-25.12.6`，
用 GitHub Actions 云端编译，产出可直接刷机的 `sysupgrade.bin`。

## 和目标设备的匹配

| 项目 | 值 |
| --- | --- |
| 设备 | Cudy TR3000 256MB v1 |
| 平台 | mediatek / filogic (aarch64_cortex-a53) |
| 刷机方式 | 系统 → 备份/升级 → 刷写固件（sysupgrade，保留配置可选） |

## 在 FanchmWrt 默认包之外新增的内容

- **USB 相关**：`kmod-usb3` `kmod-usb-net` `cdc-ether` `rndis` `cdc-ncm` `huawei-cdc-ncm`
  `qmi-wwan` `cdc-mbim` `ipheth` `rtl8152` `asix-ax88179`、`usb-storage-uas`、
  `usb-serial(-option,-wwan)`、`usb-wdm`、`usb-acm`、`usb-modeswitch`、`usbutils`、`usbids`
- **拨号备用**：`comgt` `comgt-ncm` `chat` `uqmi` `umbim` + `luci-proto-qmi/mbim/3g`
- **存储**：`kmod-fs-exfat` `kmod-fs-ntfs3` `kmod-fs-f2fs` `e2fsprogs` `dosfstools`
  `ksmbd-server` `ksmbd-utils` `luci-app-ksmbd`（ext4/vfat 与 block-mount 已是默认）
- **代理**：`luci-app-openclash`、`luci-app-passwall2`、`luci-i18n-passwall2-zh-cn`、`xray-core`
- **应用商店**：`luci-app-store`（iStore，来自 linkease/istore）
- **预置配置**：`/etc/config/network` 首次启动即带 `usb`(eth2, dhcp, metric 20) 与
  `wwan`(usb0, dhcp) 两个接口；`/etc/hotplug.d/net/30-usbwan` 会在插入 USB 网卡后
  自动绑定设备并 `ifup usb`（中兴 F50 / 手机 USB 共享即插即用）

## 软件源已换成清华 TUNA

镜像内的 `/etc/apk/repositories.d/distfeeds.list` 由 `include/version.mk` 的 `VERSION_REPO`
生成，所以流水线在 clone 之后追加一行覆盖它（步骤 `Point package feeds at TUNA (Tsinghua) mirror`）：

```
VERSION_REPO:=https://mirrors.tuna.tsinghua.edu.cn/openwrt/releases/25.12.6
```

产物镜像里所有 feed 就变成：

```
https://mirrors.tuna.tsinghua.edu.cn/openwrt/releases/25.12.6/targets/mediatek/filogic/packages/packages.adb
https://mirrors.tuna.tsinghua.edu.cn/openwrt/releases/25.12.6/packages/aarch64_cortex-a53/{base,luci,packages,routing,telephony,video}/packages.adb
https://mirrors.tuna.tsinghua.edu.cn/openwrt/releases/25.12.6/targets/mediatek/filogic/kmods/<kernelver>/packages.adb
```

流水线还会 `curl -fsIL` 探一下
`$MIRROR_REPO/targets/mediatek/filogic/packages/packages.adb`，
源缺版本时直接 fail，不会闷头烧一张装不了包的固件。
注意清华只镜像 `releases/targets/packages/kmods`，**没有** `git.openwrt.org` 的 feeds 源码，
所以 `./scripts/feeds update` 仍走 GitHub / git.openwrt.org（Actions 机器在美国，无影响）。

### 已经刷好的机器上换源（一条命令）

```sh
# OpenWrt 25.x 用 apk
cp /etc/apk/repositories.d/distfeeds.list /etc/apk/repositories.d/distfeeds.list.orig
sed -i 's#https://downloads.openwrt.org#https://mirrors.tuna.tsinghua.edu.cn/openwrt#g' \
  /etc/apk/repositories.d/distfeeds.list
apk update

# 老版本 opkg
sed -i 's#downloads.openwrt.org#mirrors.tuna.tsinghua.edu.cn/openwrt#g' /etc/opkg/distfeeds.conf
opkg update
```

## 怎么跑

Actions → **build-fanchmwrt** → Run workflow：

- `mode = check`：只做 feeds + `.config` 解析，10 分钟内出结果，用来检查依赖是否齐全
- `mode = build`：完整编译（约 2～5 小时），产出 `sysupgrade.bin` 并自动建 Release

## 目录

```
.github/workflows/build.yml   编译流水线
config/diffconfig             目标/设备与少量开关
config/packages.txt           需要额外加进去的软件包列表
files/                        打包进固件根目录的预置文件（uci-defaults、hotplug）
```

## 许可

固件与源码遵循 FanchmWrt / OpenWrt 的 GPL-2.0。原始项目版权归 FanchmWrt 作者所有。
