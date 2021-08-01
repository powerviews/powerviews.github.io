---
title: "精致小巧：Nuc搭建Homelab流水账"
date: 2021-08-01T16:59:33+08:00
tags: ["折腾"]
categories: "流水账"
author: "Me"
# author: ["Me", "You"] # multiple authors
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "玩一玩闲置很久的Nuc8i5"
disableHLJS: true # to disable highlightjs
disableShare: false
hideSummary: false
searchHidden: false
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
cover:
    image: "<image path/url>" # image path/url
    alt: "<alt text>" # alt text
    caption: "<text>" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
editPost:
    URL: "https://github.com/powerviews/powerviews.github.io/content"
    Text: "Suggest Changes" # edit text
    appendFilePath: true # to append file path to Edit link
---
## 序
去年购置的Nuc8i5，搭配了32G内存 + 1T m.2 SSD，花费了差不多四千多？机器本身比较便宜，咸鱼可以不到两千入手一个。    
  
![配置图](https://cdn.jsdelivr.net/gh/powerviews/picture@main/blog/1627809036301-1627809036294.png)  

在装了manjaro之后，遇到了出现很多次问题，最后换新了两次后成功吃灰。当然作为穷狗，肯定不能浪费。想来想去，还是做一个Home lab吧，在申请了宽带外网IP后，配合DDNS，便可以当作家用服务器来使用。

## 安装ProxmoxVE
经过短时间调研后，各类互联网评测结果都倾向PVE更加适合作为家用服务器的虚拟化平台，故此先试试PVE。  
>Proxmox VE（英语：Proxmox Virtual Environment，通常简称为PVE、Proxmox），是一个开源的服务器虚拟化环境Linux发行版。Proxmox VE基于Debian，使用基于Ubuntu的定制内核，包含安装程序、网页控制台和命令行工具，并且向第三方工具提供了REST API，在Affero通用公共许可证第三版下发行。Proxmox VE支持两类虚拟化技术：基于容器的LXC（自4.0版开始，3.4版及以前使用OpenVZ技术）和硬件抽象层全虚拟化的KVM。  
> 
> PVE下载地址  
https://www.proxmox.com/en/downloads/category/iso-images-pve

balenaEtcher刷入U盘，然后BIOS修改硬盘顺序，常规操作刷入NUC。

## 连接WIFI
由于Nuc拥有wifi和有线双模功能的，为了更加方便使用（见不得那么多线），决定最终得使用WIFI来出网。Debian连接WIFI有几种方案： https://wiki.debian.org/WiFi/HowToUse   。    
  
笔者测试后认为还是Intel的ConnMan工具套件最为便捷，先接入网线，用于安装一些工具。如果有使用端口映射将PVE到公网的需求，那么一定要提前在路由器上设置好DHCP分配静态IP，以免PVE端的IP变动。
1.  安装connman  
```
sudo apt install connman ifupdown2
sudo systemctl start connman
```

2.  执行connmanctl
启动wifi模块，然后扫描wifi，等待扫描结束，通过services指令获得SSID，使用connect指令连接对应WIFI，输入密码成功后，则提示连接成功，随后quit指令退出Connman即可。
```
root@pve:~# connmanctl
connmanctl> enable wifi
Error wifi: Already enabled
connmanctl> scan wifi
Scan completed for wifi
connmanctl> services
    ChinaNet-Ze4p        wifi_c8e26537e042_4368696e614e65742d5a653470_managed_psk
    TP-802               wifi_c8e26537e042_54502d383032_managed_psk
    pb                   wifi_c8e26537e042_7062_managed_psk
                         wifi_c8e26537e042_hidden_managed_psk
    TP-LINK_home         wifi_c8e26537e042_54502d4c494e4b5f686f6d65_managed_psk
connmanctl> agent on
Agent registered
connmanctl> connect wifi_c8e26537e042_4368696e614e65742d5a653470_managed_psk
```

## 配置网络接口
如果上述操作没有遇到问题，那么就可以看到我们被分配到了IP，并且可以成功使用WIFI上网。  
  
![网络配置](https://cdn.jsdelivr.net/gh/powerviews/picture@main/blog/1627825408811-1627825408800.png)  
  
接下来，需要配置NAT，让虚拟机使用PVE的IP来传出流量，这里需要使用到iptables来转发。具体操作可参考：
https://pve.proxmox.com/wiki/Network_Configuration#_masquerading_nat_with_tt_span_class_monospaced_iptables_span_tt  
简单修改一下，我这个场景下，已经使用路由器DHCP分配了固定的IP，所以无需在客户端配置静态IP。  
```
auto lo
iface lo inet loopback

auto wlp0s20f3
iface wlp0s20f3 inet dhcp

iface eno1 inet manual

auto vmbr0
iface vmbr0 inet static
	address 10.10.10.1/24
	bridge-ports none
	bridge-stp off
	bridge-fd 0
#NAT出网，网关10.10.10.1

        post-up   echo 1 > /proc/sys/net/ipv4/ip_forward
        post-up   iptables -t nat -A POSTROUTING -s '10.10.10.0/24' -o wlp0s20f3 -j MASQUERADE
        post-down iptables -t nat -D POSTROUTING -s '10.10.10.0/24' -o wlp0s20f3 -j MASQUERADE
```

如此一来，网络配置基本处理完了。对了，这里推荐一个PVE主题 [**PVEDiscordDark**](https://github.com/Weilbyte/PVEDiscordDark) 。接下来，就只需要重启网卡或者直接设备重启即可落实修改了。

## 测试虚拟机
输入wifi网卡的地址的8006端口，记得用https，即可访问到PVE的web管理界面。  
  
![Dark mode](https://cdn.jsdelivr.net/gh/powerviews/picture@main/blog/1627826602556-1627826602550.png)
  
在 `本地节点-local存储-ISO镜像` 处，可以上传系统ISO镜像，也就是常规的VM格式。  
  
![ISO镜像上传](https://cdn.jsdelivr.net/gh/powerviews/picture@main/blog/1627826702986-1627826702980.png)  
  
而后，直接创建虚拟机，跟着走就行。我直接跳到安装后的虚拟机中，设置好网卡属性，子网10.10.10.1/24，网关为10.10.10.1，然后一定要记住设置DNS，不然无法解析域名。最后，重启一下网卡或者机器即可。
  
![IPV4设置](https://cdn.jsdelivr.net/gh/powerviews/picture@main/blog/1627826857866-1627826857857.png)
  
![测试网络](https://cdn.jsdelivr.net/gh/powerviews/picture@main/blog/1627827003051-1627827003047.png)