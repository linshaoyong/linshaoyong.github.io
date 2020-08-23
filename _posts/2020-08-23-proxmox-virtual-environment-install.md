---
layout: post
title: "基于Proxmox VE搭建办公环境（一）"
date: 2020-08-23 15:50:00 +0800
tags: [proxmox virtual]
---

现在分布式应用越来越多，作为程序员，如果可以在家里搭建各种分布式集群（比如Kafka/Flink/Elasticsearch/TiDB），就可以很方便地研究这些较为复杂的技术。简单调研之后决定使用较为轻量的Proxmox Virtual Environment (VE) 来实现。

PVE是一个开源的企业级虚拟化平台， 集成了KVM与LXC、软件定义存储（Software-defined storage）以及网络功能。按照官方的说法，计算、网络、存储在一个平台全解决。

PVE基于Debian，如果你已经有Debian系统，只需安装PVE相关软件包。否则，就在官方网站下载ISO，制作安装媒介来安装。

## 准备材料

- 拼多多百亿补贴，斥资2500元购入16G内存/256G NVMe的主机
- 多年前的Sandisk酷刃U盘
- [Proxmox VE ISO Installer](https://www.proxmox.com/en/downloads)

## 制作安装盘

[官方提供了各种系统上的制作文档](https://pve.proxmox.com/wiki/Prepare_Installation_Media#installation_prepare_media)，现在用光驱的应该极少了，我们以苹果系统为例上制作U盘安装盘为例

```shell
# 假设下载的IOS文件名为proxmox-ve.iso，用hdiutil转换为dmg文件
$ hdiutil convert -format UDRW -o proxmox-ve.dmg proxmox-ve.iso

# 获取设备列表，假设U盘为/dev/diskX，将它unmount
$ diskutil list
$ diskutil unmountDisk /dev/diskX

# 将安装文件写入U盘
sudo dd if=proxmox-ve.dmg of=/dev/rdiskX bs=1m
```

## 安装

- 本来以为安装很容易，结果还是费了一番周折。一开始我用的是CHIPFANCIER高速固态盘USB3.1，但是进入安装界面后就提示search ISO for id xxxxxx，然后多次timeout无法继续。经过Google搜索，[发现不少人遇到类似问题，还有人在论坛吵起来的](https://forum.proxmox.com/threads/unable-to-install-no-cdrom-found.42043/)。后来我怀疑是PVE安装程序对这款U盘兼容性不够好，换了个老款的Sandisk就好了。
- 图形化安装界面，没啥好说的。最主要的是IP配置，无线网络配置较麻烦，建议先用网线连局域网，因为安装后需要用另一台电脑的浏览器配置。

## 配置

如果一开始IP没配置好，需要在两个地方修改

/etc/network/interfaces

```
auto lo
iface lo inet loopback

iface eno1 inet manual

auto vmbr0
iface vmbr0 inet static
        address 192.168.1.100 # 确保IP地址正常
        netmask 255.255.255.0
        gateway 192.168.10.1
        bridge_ports eno1
        bridge_stp off
        bridge_fd 0
```

/etc/hosts

```
127.0.0.1 localhost
192.168.1.100 <FQDN> <host name>
```

修改后重启，就可以通过另一台电脑访问 https://192.168.1.100:8006 来配置了。（注意，用https访问）

至此安装完毕，等我好好摸索一番，我们再来聊聊使用细节。
