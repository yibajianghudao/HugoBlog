---
weight: 100
title: VPN
slug: vpn
summary: VPN
draft: false
author: jianghudao
tags:
isCJKLanguage: true
date: 2026-02-24T09:31:20+08:00
lastmod: 2026-02-24T16:13:44+08:00
---

## 使用 NetworkManager VPN 代替 Windows VPN

有一个使用场景是:

在 Windows 环境下，使用 VPN 远程连接主机通常通过配置 VPN 并配合 `.bat` 脚本来实现拨号和路由添加:

```bash
@echo off
COLOR E
rasdial.exe "vpn" "123" "123123"
route  add 10.25.4.0 mask 255.255.0.0 172.16.0.1 
route  add 100.45.98.0 mask 255.255.252.0 172.16.0.1 
route  add 101.25.11.129 mask 255.255.255.0 172.16.0.1 
route  add 10.32.110.0 mask 255.255.255.0 172.16.0.1
timeout /nobreak /t 300
exit
pause
```

我们可以手动切换 Windows 设置的 VPN 类型来确定 VPN 的类型：

![](assets/VPN/使用NetworkManager%20VPN代替Windows%20VPN-20260224150332350.png)

从图中可以看出，这是一个 L2TP/IPsec 类型的 VPN。由于数据加密选项为可选，且连接过程中未手动配置预共享密钥 (PSK)，可以确定该连接未开启 IPsec 加密，即纯 L2TP 模式。

在 Linux 系统上可以直接使用 `NetworkManager` 的命令行工具 `nmcli` 来替代这个脚本，还可以通过 Linux 的路由机制实现更优雅的流量接管。

### 原理

#### VPN 分流

上述 `.bat` 脚本中的 `route add` 命令用于配置路由来实现 VPN 分流。如果不配置这些路由，连接 VPN 后，默认网关可能会改变，导致所有流量都经过 VPN 服务器。配置特定网段的路由后，只有访问 `10.25.4.0` 等特定网段时才走 VPN 隧道，其他流量正常走本地物理网卡。

#### Linux 路由配置

Windows 脚本中指定了固定的网关 IP：`172.16.0.1`。在 Linux 上处理 PPP (如 L2TP) 等点对点协议时，建议不指定网关 IP，而是直接将流量转发到虚拟网卡设备（如 `dev ppp1`）。

原因是，如果服务器动态分配的虚拟网关 IP 发生变化，指定固定 IP 的路由将会失效；而直接指向网卡设备的路由不受对端 IP 变化的影响。

### 配置

#### 安装依赖组件

安装 NetworkManager 的 L2TP 插件及底层加密组件：

```bash
sudo apt install network-manager-l2tp network-manager-l2tp-gnome xl2tpd strongswan libstrongswan-extra-plugins libcharon-extra-plugins
```

停用底层服务的独立自启动，避免占用端口导致 NetworkManager 拨号失败：

```bash
sudo systemctl stop xl2tpd strongswan-starter ipsec
sudo systemctl disable xl2tpd strongswan-starter ipsec
```

重启 NetworkManager:

```bash
sudo systemctl restart NetworkManager
```

#### 创建和配置 VPN

通过 `nmcli` 创建和配置 VPN，将子网掩码转换为 CIDR 格式（如 `255.255.255.0` 转换为 `/24`）：

```bash
# 1. 创建 L2TP VPN 连接
nmcli connection add type vpn vpn-type l2tp con-name "MyL2TP_CLI" ifname "*"

# 2. 配置服务器 IP 和认证协议（注意关闭 ipsec）
nmcli connection modify "MyL2TP_CLI" vpn.data "gateway=101.20.138.165, user=123, refuse-pap=yes, refuse-chap=yes, refuse-eap=yes, ipsec-enabled=no"

# 3. 设置密码
nmcli connection modify "MyL2TP_CLI" vpn.secrets "password=123123"

# 4. 开启分流模式（仅配置的网段走 VPN）
nmcli connection modify "MyL2TP_CLI" ipv4.never-default yes

# 5. 添加路由
nmcli connection modify "MyL2TP_CLI" +ipv4.routes "10.25.4.0/16"
nmcli connection modify "MyL2TP_CLI" +ipv4.routes "100.45.98.0/22"
nmcli connection modify "MyL2TP_CLI" +ipv4.routes "101.25.11.0/24"
nmcli connection modify "MyL2TP_CLI" +ipv4.routes "10.32.110.0/24"
```

#### 启动和测试

启动 VPN：

```bash
nmcli connection up "MyL2TP_CLI"
```

通过 `ip a` 命令查看 `ppp` 网卡：

```bash
$ ip a
11: ppp1: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1400 qdisc fq_codel state UNKNOWN group default qlen 3
    link/ppp 
    inet 172.16.0.51 peer 172.16.0.1/32 scope global ppp1
       valid_lft forever preferred_lft forever
```

开启分流模式后，没有配置的网段不会走 VPN。使用 `ip route get` 查看一个没有配置的 IP 的路由：

```bash
# 未配置的 IP 走本地物理网卡 (enp1s0)
$ ip route get 20.0.0.123    
20.0.0.123 via 192.168.88.1 dev enp1s0 src 192.168.88.7 uid 1000 
    cache 
```

手动把网段添加路由之后就可以走 VPN 了:

```bash
$ nmcli connection modify "MyL2TP_CLI" +ipv4.routes "20.0.0.0/24"

$ nmcli connection down "MyL2TP_CLI"
Connection 'MyL2TP_CLI' successfully deactivated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/7)

$ nmcli connection up "MyL2TP_CLI" 
Connection successfully activated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/8)
```

测试路由:

```bash
$ ip route get 20.0.0.123  
20.0.0.123 dev ppp1 src 172.16.0.51 uid 1000 
    cache 
```
