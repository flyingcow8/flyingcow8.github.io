---
title: "Ubuntu 16.04上搭建shadowsocks server"
date: 2018-01-21 18:30:00
categories: linux
tags:
  - shadowsocks
  - linux
---
之前在vultr（一家云服务商）上买的VPS突然ping不通了，导致上面部署的ss服务不可用了。原因估计是ip被墙了，尝试删除原来的Ubuntu实例，然后新建一个实例，结果获取的ip还是跟之前一样。无奈只能添加一个新的ip（收取一点费用）。本文以`Ubuntu 16.04`为例。
将新的ip添加到脚本中：
<!-- more -->
```
# vi /etc/network/interfaces

auto lo
iface lo inet loopback

auto ens3
iface ens3 inet dhcp
iface ens3 inet6 auto

auto ens3:1
iface ens3:1 inet static
        address 207.246.83.xxx
        netmask 255.255.254.0
```
重启网络，
```
# service networking restart
```

## 部署shadowsocks服务
详细安装方式请参考官网[shadowsocks.org][1]。本文选择安装使用最多的python版本ss。也可以选择go、libev(适合嵌入式设备)、QT_C++、perl等版本。
## 安装
```
$ pip install shadowsocks
```
## 配置
新建json配置文件，/etc/shadowsocks.json
```json
{
    "server":"207.246.83.xxx",
    "server_port":8388,
    "local_port":1080,
    "password":"123456",
    "timeout":600,
    "method":"aes-256-cfb"
}
```
启动shadowsocks server
```
# ssserver -c /etc/shadowsocks
```
加入到开自启动脚本~/.bashrc，-q选项表示安静模式，只会打印错误或者警告
```
ssserver -c /etc/shadowsocks -q &
```

[1]: http://www.shadowsocks.org "shadowsocks"