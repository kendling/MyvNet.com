---
title: "Ubuntu 环境把宝塔 5.9 升级到 6.8"
slug: "upgrade-bt-panel-to-6.8-from-5.9-under-ubuntu"
date: 2019-02-22
tags: ["Ubuntu", "Linux",
       "乌班图", "宝塔"]
categories: ["Linux"]
---

## 贱

好久没看宝塔，今天看了一下，发现升级到 6.8 了。我还在使用 5.9 。

于是，跟着升级提示打开了官方的升级教程。

但是，由于没有了解清楚就跑了升级命令 `curl http://download.bt.cn/install/update6.sh|bash`。

PS: 官方也没有说明此脚本不能在 Ubuntu 环境下平滑升级。

## 升级失败

估计你们比我知道到结果：升级失败！报一些红色的错误。

按官方说明，继续跑了 2 次升级命令，结果还是一样。

`bt restart` 也重启失败。

只能问问度娘，问了一圈也没结果。倒是知道了升级命令并不适用于 Ubuntu 环境。

没办法，只能靠自己了。

## 分析问题

`bt restart` 报 gunicorn 启动失败。于是跑了一遍。

```
apt update
apt upgrade
apt install gunicorn
```

结果还是你们清楚，还是无法安装。

那只能分析安装脚本找环境了。

`wget http://download.bt.cn/install/install-ubuntu_6.0.sh` 把脚本下载回来打开。

发现 `gunicorn` 是通过 pip 安装的，难怪 `apt` 找不到。

## 解决

按安装脚本一个个依赖库安装。

```
apt install libffi-dev -y
wget --no-check-certificate https://bootstrap.pypa.io/get-pip.py
python get-pip.py
rm -f get-pip.py

pip install --upgrade setuptools
pip install itsdangerous==0.24
pip install paramiko==2.0.2
pip install flask-socketio==3.0.2
pip install python-socketio==2.1.2
pip install psutil
pip install chardet
pip install virtualenv
pip install Flask
pip install Flask-Session
pip install Flask-SocketIO
pip install flask-sqlalchemy
pip install Pillow
pip install gunicorn
pip install gevent-websocket
pip install paramiko

wget -O Pillow-3.2.0.zip $download_Url/install/src/Pillow-3.2.0.zip -T 10
unzip Pillow-3.2.0.zip
rm -f Pillow-3.2.0.zip
cd Pillow-3.2.0
python setup.py install
cd ..
rm -rf Pillow-3.2.0
```

一系列依赖环境安装下来再支持 `bt restart` 。已经可以正常启动了。

再打开面板登陆，一切正常。

至此结束。

## 奇怪的安装脚本

大家可以看到安装依赖环境的时候，同一个包会安装 2 遍。非常奇怪。

看原脚本，某些包会安装 3 遍、 4 遍。。。

```
pip install xx=x.x.x  # 指定版本安装一次
pip install xx        # 不指定版本安装一次
for xin xx yy; do pip install xx done # 循环安装一次
pip install xx yy     # 批量安装一次，此命令比上面循环安装只多了一个包名
# 最后还有从宝塔下载地址 `http://bt.cn/install/src/xx-1.0.0.tar.gz 下载源代码安装一次
```

这样的代码质量，令人担忧。。。
