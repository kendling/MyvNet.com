---
title: "解决京东京造键盘在 Linux Fx 功能键失效的问题"
slug: ixed-keychron-keyboards-fx-key-not-work-at-linux
date: 2022-02-22T14:35:18+08:00
tags: [linux"]
categories: ["Linux"]
image: "keychron-k1.png"
---

## 不支持 Linux 系统？

最近入手了一块二手的京东京造 K1 键盘。

到手试了一下，`Windows` 下和没有问题，可到了 `Linux` 下 `F1-F12` 全失败，包括按 `Fn+Fx` 都不起作用。

## 尝试切换模式

网上搜了一下，尝试过 `Fn+C+L` 和 `Fn+X+L` 切换模式都失效。

切换 `Win/Android`  、 `Mac/iOS` 、有线、蓝牙，组合上面的模式切换都失效。

## 解决方法

不死心，不可能说 `Linux` 不支持某个键盘，不接受这个事情。

继续网上搜英文资料，终于在 https://gist.github.com/andrebrait/961cefe730f4a2c41f57911e6195e444 找到解决的办法。

## 原因

Keychron（也就是京东京造）键盘在 `Linux` 下会被 `hid_apple` 驱动程序识别并驱动。而且不管你在键盘设置的是 `Win/Android` 模式还是 `Mac/iOS` 模式，照识别无误。

然而 `hid_apple` 驱动程序默认把 `F1-F12` 键设置为媒体键，也就出现了上面所说的 `Fx` 键失效的问题。

## 解决

问题找到了就容易解决了，`hid_apple` 驱动程序提供了 `fnmode` 选项修改上面提到的行为：

- 0 = `disabled`: 禁用 `Fn` 键，按 `Fn+F8` 和直接按 `F8` 效果一样。

- 1 = `fkeyslast`: 优先触发第二定义，按 `F8` 优先触发特殊键，也就是多媒体键（播放/暂停），按 `Fn+F8` 触发 `F8`。

- 2 = `fkeysfirst`: 优先触发第一定义，按 `F8` 优先触发 `F8` ，按 `Fn+F8` 触发特殊键，也就是多媒体键（播放/暂停）。

先临时测试一下：

```bash
# 把下面的 <value> 替换成 0, 1 or 2
# 例如: echo 2 | sudo tee /sys/module/hid_apple/parameters/fnmode
echo <value> | sudo tee /sys/module/hid_apple/parameters/fnmode
```

然后测试一下你的键盘，确保它按你想要的模式运行。

最后我们更新一下 `initramfs` 让它在系统启动的时候自动生效：

```bash
# 把下面的 <value> 替换成上一步你测试好的值 (0, 1 or 2)
# 例如: echo "options hid_apple fnmode=2 | sudo tee /etc/modprobe.d/hid_apple.conf"
# this will erase any pre-existing contents from /etc/modprobe.d/hid_apple.conf
echo "options hid_apple fnmode=<value>" | sudo tee /etc/modprobe.d/hid_apple.conf
# 参数 "-k all" 并非必要, 但是它可以帮你一次更新所有内核的 initramfs
sudo update-initramfs -u -k all
sudo systemctl reboot
```

上面的链接还有解决蓝牙连接问题的方法，遇到蓝牙连接问题的小伙伴可以进去看看。

## 组合键

- `Fn+1/2/3` 短按切换蓝牙设备，长按 4 秒进入蓝牙匹配
- `Fn+K+C` 长按 4 秒切换 `Mac` 系统的功能键与多媒体键
- `Fn+X+L` 长按 4 秒切换 `Win` 系统的功能键与多媒体键
- `Fn+S+O` 长按 3 秒禁用/启用键盘自动进入睡眠状态
- `Fn+B` 长按显示电量（第一排键盘不亮，其他按键都亮，表明电量剩余 30%-70% ）
- `Fn+J+Z` 长按 3 秒恢复出厂设置，蓝牙复位

## 多媒体键

- `Fn+F1` 降低屏幕亮度
- `Fn+F2` 提高屏幕亮度
- `Fn+F3` `Win` - 切换任务 / `Mac` - Mission Control
- `Fn+F4` `Win` - 打开文件管理器 / `Mac` - Launchpad
- `Fn+F5` 降低键盘背景灯亮度
- `Fn+F6` 提高键盘背景灯亮度
- `Fn+F7` 上一曲
- `Fn+F8` 播放/暂停
- `Fn+F9` 下一曲
- `Fn+F10` 静音
- `Fn+F11` 降低音量
- `Fn+F12` 提高音量
