---
title: 配置NAT实现网络共享
date: 2017-12-07 15:35:00
categories: Network
tags:
  - linux
  - NAT
---

```sh
echo 1 >/proc/sys/net/ipv4/ip_forward
iptables -t nat -A POSTROUTING -o ppp0 -j MASQUERADE
iptables -A FORWARD -i ppp0 -o wlan0 -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i wlan0 -o ppp0 -j ACCEPT
```
<!-- more -->