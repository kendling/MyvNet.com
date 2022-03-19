---
title: "当你的 systemd 失效之后如何优雅的重启或关闭系统？"
slug: "restart-linux-even-systemd-failed"
date: 2021-07-23
tags: ["Linux", "systemd", "shutdown"]
categories: ["Linux"]
---

## shutdown 失效后你要怎么重启系统？

- reset 键？
- 长按关机键？
- 拔电源？

我这里有一个方法可以让你优雅的重启或者关闭系统，只需要有 root shell 就足够了。

方法非常简单，只有 2 句命令：

```bash
# 必须在 root shell 运行以下命令

# 重启系统
echo 1 > /proc/sys/kernel/sysrq
echo b > /proc/sysrq-trigger

# 关闭系统
echo 1 > /proc/sys/kernel/sysrq
echo o > /proc/sysrq-trigger
```

以后遇到 CAD 按键/reboot/shutdown 命令都失效的时候试试吧。
