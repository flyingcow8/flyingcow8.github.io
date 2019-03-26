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
```sh
vi /etc/network/interfaces

>auto lo
>iface lo inet loopback

>auto ens3
>iface ens3 inet dhcp
>iface ens3 inet6 auto

>auto ens3:1
>iface ens3:1 inet static
>        address 207.246.83.xxx
>        netmask 255.255.254.0
```
重启网络，
```sh
service networking restart
```

## 部署shadowsocks服务
详细安装方式请参考官网[shadowsocks.org][1]。本文选择安装使用最多的python版本ss。也可以选择go、libev(适合嵌入式设备)、QT_C++、perl等版本。
## 安装
```sh
pip install shadowsocks
```
## 配置
新建json配置文件，/etc/shadowsocks.json
```json
{
    "server":"111.111.111.111",
    "server_port":8388,
    "local_port":1080,
    "password":"123456",
    "timeout":600,
    "method":"aes-256-cfb"
}
```
启动shadowsocks server(后台运行)
```sh
ssserver -c /etc/shadowsocks -d start
```
加入到开自启动脚本~/.bashrc，防止重启服务器后需要手动重启服务

***
从油管学到一种更加方便的部署方式，如下。
* 在vultr部署一台CentOS 7 x64的VPS。
* 使用root登陆后，运行以下命令安装ss：
```
wget --no-check-certificate -O shadowsocks-all.sh https://raw.githubusercontent.com/teddysun/shadowsocks_install/master/shadowsocks-all.sh
chmod +x shadowsocks-all.sh
./shadowsocks-all.sh 2>&1 | tee shadowsocks-all.log
```
* 开启BBR加速，油管4K视频无压力！
```
wget --no-check-certificate https://github.com/teddysun/across/raw/master/bbr.sh && chmod +x bbr.sh && ./bbr.sh
```
最后附上视频地址：https://www.youtube.com/watch?v=pwZkEYjHNUA

[1]: http://www.shadowsocks.org "shadowsocks"
