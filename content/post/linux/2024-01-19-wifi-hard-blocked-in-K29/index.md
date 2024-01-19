---
title: "解决昭阳 K29 更换硬盘后导致 WiFi 不能启用的问题"
slug: wifi-hard-blocked-in-K29
date: 2024-01-19T10:43:28+08:00
tags: [linux"]
categories: ["Linux"]
image: "zhaoyang-k29.png"
---

## 原由

用于安装系统的 240G mSATA 固态硬盘一直空间不够用，而且也出现 `S.M.A.R.T` 警告很长时间了。

系统越发缓慢，更换升级刻不容缓。

## 入手新盘

本来应该在去年最香的时段入手新盘的，由于我一直把 `K29` 的固态硬盘记错成 `mSATA 半` 接口，因此完美错过了最香价格。

思来想去，最终在海鲜市场入手一块 0 通电时长的 `Thinklife SSD ST600` 固态硬盘，容量 `512G` 。`S.M.A.R.T` 信息显示通电次数仅 `4` 次，通电时长 `0` 小时，总写入  `156G` 。

## 更换升级

更换前先用 `dd` 命令把系统备份到一个机械硬盘。

更换后再用 `dd` 命令把备份写入新的固态硬盘。

最后修改更新 `GRUB` 和 `fstab` 配置文件，更新新的分区 `UUID` 。

一顿操作过后系统正常启动。

## WiFi 不可用

进入系统后迫不及待进行测试，软件启动、系统响应都比之前快很多，所有数据都没问题，算是一次成功的升级。

等等。。

`WiFi` 无法启动。。

## 经验失效

其实这个问题我之前也遇到过。`Linux` 下无法启用无线网卡就需要重启进入 `Windows` 系统，在 `Windows` 系统下连接一下无线网络再回到 `Linux` 下就可以了。

但是，这次这个经验失效了。。在 `Windows` 下也无法正常使用无线网卡。

设备管理里面删除网卡重新安装也无效。去设置重置网络也无效。

这下。。矇了。。

## Hard blocked

Linux 和 Windows 都能正确被别到无线网上，但是都无法正常工作。

Linux 下会提示无线网上的硬件开关没有打开之类的，设置里无法启用无线网卡。

Windows 下在设置里面打开无线功能也无效，切换页面回来还是关闭状态。

Linux 下运行 `rfkill list all` 命令，输出提示 `Hard Blocked` ：

```bash
0: phy0: Wireless LAN
 Soft blocked: no
 Hard blocked: yes
```

使用笔记本的 `Fn+F5` 切换也只是更新 `Soft blocked` 的状态，对于 `Hard blocked` 状态无效。

Windows 下使用笔记本的 `Fn+F5` 切换也没有任何效果。

## 解决

网上搜索了一番有一些解决办法：

1. 运行 `lspci -v | grep -i "network\|kernel" | grep -A 1 -i network` 命令查找网卡驱动模块，运行 `sudo modprobe -r xxx` 移除此模块再运行 `sudo modprobe xxx` 重新加载。

2. 运行 `lsmod | grep thinkpad` 查找 `ThinkPad` 驱动模块，运行 `sudo modprobe -r xxx` 移除此模块再运行 `rfkill unblock all` 。

3. 更新 `BIOS` 。

前两种方法我都进行了尝试，但是并没能解决。

最后一种方法我没有尝试，因为在我尝试之前已经找到解决办法了。

当看到第三种方法的时候脑子灵光一闪，想起更换硬盘之后系统提示过 `System Configuration Updated` ，然后重新启动，`BIOS` 部分选项也被自动修改过。

而且，更新 `BIOS` 后设置会被重置。会不会是这个原因？

重启进入 `BIOS` ，重置 `BIOS` ，再修改好自己的配置保存重启。

重启后先进入 `Windows` 系统验证，这次无线网络恢复正常了。

再重启进入 `Linux` 无线网络也能正常使用。

问题完美解决！

## PS

如果你是使用 `UEFI` 方式启动的话，重置 `BIOS` 设置会丢失你的启动选项。

需要使用工具恢复。

我这里使用了 2 个工具：

1. `DiskGenius` 用户挂载隐藏的 `UEFI` 启动分区

2. `BOOTICE` 此工具我使用的是 `1.3.3` 版本，使用 `UEFI` 标签页下的修改启动序列功能添加我的启动选项

注意：`BOOICE` 不能使用新版，原因是作者硬盘坏了，丢失了代码。新版部分功能并没有实现，例如 `1.3.4` 版本的 `UEFI` 修改启动序列功能就没有实现。
