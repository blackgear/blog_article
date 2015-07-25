title: PKU校园网IPv6环境下的网络配置
date: 2015-07-25 22:00:00
categories: network
tags: [shadowvpn, chinadns, openwrt, debian]
---

## 相关说明

服务器信息：

> ConoHa 1GB KVM Tokyo
> 2 Core CPU
> 50GB SSD
> Debian 8.0 64bit
> IPv4 addr x1
> IPv6 addr x17

路由器信息：

> Mercury 4530R OpenWrt BarrierBreaker 14.07
> 560MHz CPU
> 16M Flash
> Ar71xx Generic

## 服务器的配置

面前ConoHa的Debian 8.0 64bit系统尚不能自动绑定IPv6地址，所以首先需要绑定IPv6地址。从ConoHa的控制台查看IPv4、IPv6的地址、掩码、网关数据，ConoHa提供了17个IPv6地址，随意选择一个即可，随后修改`/etc/network/interfaces`绑定静态IP：

    $ vi /etc/network/interfaces
    # This file describes the network interfaces available on your system
    # and how to activate them. For more information, see interfaces(5).

    source /etc/network/interfaces.d/*

    # The loopback network interface
    auto lo
    iface lo inet loopback
    iface lo inet6 loopback

    # The primary network interface
    auto eth0
    iface eth0 inet6 static
            address IPv6地址
            netmask 64
            gateway IPv6网关
    iface eth0 inet static
            address IPv4地址
            netmask IPv4掩码
            gateway IPv4网关

随后重启服务器即可完成IP地址的绑定工作。

接下来安装ShadowVPN：

    $ echo "deb http://shadowvpn.org/debian wheezy main" >> /etc/apt/sources.list
    $ apt-get update
    $ apt-get install shadowsocks-libev

修改ShadowVPN的配置文件：

    $ vi /etc/shadowvpn/server.conf

将`server`设置为`::0`以监听IPv6地址，将`port`设置为`53`以绕过QoS限速，将`mtu`设置为`1420`以适配IPv6网络，设置一个复杂的密码。重启ShadowVPN使设置生效：

    $ service shadowvpn restart

至此，服务器的配置已经完成。

## 路由器的配置

路由器使用OpenWrt BarrierBreaker 14.07初始系统，第一次启动时先通过telnet进行访问并修改root用户密码，再通过ssh登录：

    $ telnet 192.168.1.1
    $ passwd
    $ exit
    $ ssh root@192.168.1.1

在配置前，首先添加软件源进行系统更新，并安装软件包：

    $ echo "src/gz openwrt_dist http://openwrt-dist.sourceforge.net/releases/ar71xx/packages" >> /etc/opkg.conf
    $ echo "src/gz openwrt_dist_luci http://openwrt-dist.sourceforge.net/releases/luci/packages" >> /etc/opkg.conf
    $ opkg update
    $ opkg list-upgradable | cut -d " " -f1 | xargs -r opkg upgrade
    $ opkg install ShadowVPN ChinaDNS wget

针对PKU校园网络进行IPv6配置，修改`/etc/config/dhcp`中的如下两段：

    $ vi /etc/config/dhcp
    config dhcp 'lan'
        option interface 'lan'
        option start '100'
        option limit '150'
        option leasetime '12h'
        option dhcpv6 'server'
        option ndp 'relay'
        option ra 'relay'

    config dhcp 'wan'
        option interface 'wan'
        option ignore '1'
        option ndp 'relay'
        option ra 'relay'
        option master '1'

配置ShadowVPN：

    $ vi /etc/config/shadowvpn
    config shadowvpn
        option enable '1'
        option concurrency '1'
        option intf 'tun0'
        option port '53'
        option password '密码'
        option mtu '1420'
        option server '服务器IPv6地址'
        option route_mode '1'
        option route_file '/etc/chinadns_chnroute.txt'

配置ChinaDNS：

    $ vi /etc/config/chinadns
    config chinadns
        option compression '1'
        option chnroute '/etc/chinadns_chnroute.txt'
        option port '5353'
        option server '114.114.114.114,8.8.4.4'
        option enable '1'
        option bidirectional '0'

根据[lsylsy2的此项目](https://gist.github.com/lsylsy2/fe94ca41a8f52b78772e)修改路由表配置文件：

    $ vi /etc/chinadns_chnroute.txt
    1.0.1.0/24
    1.0.2.0/23
    1.0.8.0/21
    1.0.32.0/19
    ……
    223.220.0.0/15
    223.240.0.0/13
    223.254.0.0/16
    223.255.0.0/17

或者是直接下载修改好的版本：

    $ wget -O /etc/chinadns_chnroute.txt --no-check-certificate https://gist.githubusercontent.com/lsylsy2/fe94ca41a8f52b78772e/raw/e51449a7d76d153d3df6934d285d7871cb0862ae/cidr_merge

配置校园网自动登录，修改`/etc/rc.local`：

    $ vi /etc/rc.local
    # Put your custom commands here that should be executed once
    # the system init finished. By default this file does nothing.

    wget -q -Y off -T 10 -t 3 -O /dev/null --no-check-certificate "https://162.105.129.65:5428/ipgatewayofpku?uid=校园网帐号&password=校园网密码&range=2&operation=connect&timeout=1"

    exit 0

配置校园网保持登录：

    $ crontab -e
    0 * * * * wget -q -Y off -T 10 -t 3 -O /dev/null --no-check-certificate "https://162.105.129.65:5428/ipgatewayofpku?uid=校园网帐号&password=校园网密码&range=2&operation=connect&timeout=1"

配置无线网络，提升无线网络速度上限与信号强度：

    $ vi /etc/config/wireless
    config wifi-device 'radio0'
        option type 'mac80211'
        option channel '11'
        option hwmode '11ng'
        option path 'platform/ar934x_wmac'
        option txpower '30'
        option country 'TW'
        option noscan '1'
        option htmode 'HT40'

    config wifi-iface
        option device 'radio0'
        option network 'lan'
        option mode 'ap'
        option ssid '2.4G网络SSID名称'
        option encryption 'psk2+ccmp'
        option key '2.4G网络WIFI密码'

    config wifi-device 'radio1'
        option type 'mac80211'
        option hwmode '11na'
        option path 'pci0000:00/0000:00:00.0'
        option country 'TW'
        option txpower '30'
        option htmode 'HT40'
        option noscan '1'
        option channel '149'

    config wifi-iface
        option device 'radio1'
        option network 'lan'
        option mode 'ap'
        option ssid '5G网络SSID名称'
        option encryption 'psk2+ccmp'
        option key '5G网络WIFI密码'

至此，路由器的配置已经完成。
