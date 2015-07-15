title: PKU校园网IPv6环境下的透明代理配置
date: 2015-01-10 22:00:00
categories: network
tags: [shadowsocks, openwrt, debian]
---

## 相关说明

服务器信息：

> Ramnode 128MB CVZ SEA
> Debian 7.0 32bit minimal
> 1 Core CPU
> 128/64 MB RAM/VSwap
> 80/500 GB Storage/Bandwidth
> IPv4 addr x1
> IPv6 addr x16

路由器信息：

> Mercury 4530R OpenWrt-Dreambox 12.09
> 560MHz CPU
> 16M Flash
> Ar71xx Generic

## 基本思路

通过shadowsocks走IPv6渠道在路由器上建立一个透明代理，将全部收费地址范围的流量在路由器端重定向走透明代理，全部免费地址范围的流量继续走IPv4渠道。

在路由器上配置定时登录脚本防止IP网关中断IPv4连接，在路由器上利用hosts文件防止DNS污染。

## 服务器的配置

安装shadowsocks：

    $ echo "deb http://shadowsocks.org/debian wheezy main" >> /etc/apt/sources.list
    $ wget -O- http://shadowsocks.org/debian/1D27208A.gpg | apt-key add -
    $ apt-get update
    $ apt-get install shadowsocks-libev

配置shadowsocks，启用IPv6支持，使用rc4-md5加密算法：

    $ vi /etc/shadowsocks-libev/config.json
    {
        "server":["[::0]","0.0.0.0"],
        "server_port": 8080,
        "local_port": 1080,
        "password": "xxxxxxxx",
        "timeout": 60,
        "method": "rc4-md5",
    }

注意，新版本的shadowsocks要同时启用IPv4与IPv6支持，需要使用`["[::0]","0.0.0.0"]`这样的格式，而不仅仅是使用`"::"`这样的格式。使用`"::"`会导致shadowsocks只监听IPv6，不监听IPv4。

启动shadowsocks服务端：

    $ /etc/init.d/shadowsocks-libev restart

至此，服务器配置完毕。

## 路由器的配置

创建`/usr/bin/connect`，用于连接IP网关免费地址范围：

    $ vi /usr/bin/connect
    #!/bin/sh
    wget -Y off -T 10 -t 3 -O /dev/null --no-check-certificate "https://its.pku.edu.cn:5428/ipgatewayofpku?uid=校园网账号&password=校园网密码&range=2&operation=connect&timeout=1">/dev/null 2>&1

创建`/usr/bin/disconnect`，用于中断IP网关全部连接：

    $ vi /usr/bin/connect
    #!/bin/sh
    wget -Y off -T 10 -t 3 -O /dev/null --no-check-certificate "https://its.pku.edu.cn:5428/ipgatewayofpku?uid=校园网账号&password=校园网密码&range=2&operation=disconnectall&timeout=1">/dev/null 2>&1

设定可执行权限：

    $ chmod +x /usr/bin/connect
    $ chmod +x /usr/bin/disconnect

配置路由器启动后自动登录IP网关，编辑`rc.local`：

    $ vi /etc/rc.local

在`exit 0`之前加入：

    /usr/bin/disconnect
    /usr/bin/connect

查看`/etc/init.d/cron`：

    $ cat /etc/init.d/cron

检查是否有如下三条命令：

    rm -rf /etc/crontabs/root
    config_load cron
    config_foreach task_get task

这三条命令会导致`crontab -e`添加的计划任务在OpenWrt系统重启之后被清除，并加载`/etc/config/cron`中的配置项。

配置`/etc/config/cron`添加计划任务，每小时尝试连接一次IP网关，以确保连接持续稳定：

    config task
        option enabled '1'
        option task_name 'reconnect'
        option task_Everyday '0'
        option task_Monday '0'
        option task_Tuesday '0'
        option task_Wednesday '0'
        option task_Thursday '0'
        option task_Friday '0'
        option task_Sartuday '0'
        option task_Sunday '0'
        option task_time 'everyh_1'
        option task_task '/usr/bin/connect'

如果`/etc/init.d/cron`中没有上述三条命令，则可直接通过`crontab -e`创建计划任务：

    $ crontab -e
    0 */1 * * * /usr/bin/connect

注意PKU校园IP网关每日连接上限为100次，超过上限将导致IP网关无法登陆。

配置路由器IPv6访问，PKU校内静态IPv6地址规则如下：

静态IPv6地址 = 校园网地址:本网段地址::本机地址
- 校园网地址为`2001:da8:201`
- 如果本机的IPv4地址为`162.105`开头，则本网段地址为IPv4地址的第3组数字加1000，如本机IPv4地址为`162.105.10.100`，则本网段地址为1010。
- 如果本机的IPv4地址为`222.29`开头，则本网段地址为IPv4地址的第3组数字加1300，如本机IPv4地址为`222.29.10.100`，则本网段地址为1310。
- 本机地址为任意1-4位十六进制数，但不得为1。

IPv6网关地址 = 校园网地址:本网段地址::1

配置路由器启动后自动设置IPv6地址，编辑`rc.local`：

    $ vi /etc/rc.local

在`exit 0`之前加入：

    ifconfig eth0.2 inet add 静态IPv6地址
    route -A inet6 add ::/0 gw IPv6网关地址

首先检查路由器固件中预装的SSL库：

    $ opkg info *ssl

若提示中有如下字样，说明路由器固件中已经安装了`libopenssl`：

    Package: libopenssl
    ......
    Status: install ok installed

若提示中有如下字样，说明路由器固件中已经安装了`libporalssl`：

    Package: libporalssl
    ......
    Status: install ok installed

若无提示信息出现，则说明路由器固件中尚未安装SSL库。

- 若已经安装`libopenssl`，从[这里](http://sourceforge.net/projects/OpenWrt-dist/files/shadowsocks-libev/ "shadowsocks-libev")下载最新版`shadowsocks-libev`。
- 若已经安装`libporalssl`，从[这里](http://sourceforge.net/projects/OpenWrt-dist/files/shadowsocks-libev/ "shadowsocks-libev")下载最新版`shadowsocks-libev-polarssl `。
- 若尚未安装SSL库，从[这里](http://sourceforge.net/projects/OpenWrt-dist/files/depends-libs/ "depend-libs")寻找对应路由器架构的依赖包目录并下载`libporalssl`，从[这里](http://sourceforge.net/projects/OpenWrt-dist/files/shadowsocks-libev/ "shadowsocks-libev")下载最新版`shadowsocks-libev-polarssl`。

最后，从[这里](http://downloads.OpenWrt.org/ "OpenWrt")寻找对应OpenWrt版本与路由器架构下的packages目录，下载最新版`kmod-ipt-nat-extra`和`iptables-mod-nat-extra`。

将全部包上传到路由器上并安装：

    $ opkg install --force-depends *

编辑shadowsocks配置文件：

    $ vi /etc/shadowsocks.json
    {
        "server":"服务器IPv6地址",
        "server_port":8080,
        "local":"0.0.0.0"
        "local_port":1080,
        "password":"xxxxxxxx"
        "timeout":60,
        "method":"rc4-md5",
    }

配置`ss-iptables`转发脚本：

    $ vi /usr/bin/ss-iptables
    #!/bin/sh

    # Create a new chain named SHADOWSOCKS
    iptables -t nat -N SHADOWSOCKS
    iptables -t nat -F SHADOWSOCKS

    # Ignore shadowsocks server's IPv4 addresses
    # iptables -t nat -A SHADOWSOCKS -d [IP] -j RETURN

    # Ignore LANs IP address
    iptables -t nat -A SHADOWSOCKS -d 0.0.0.0/8 -j RETURN
    iptables -t nat -A SHADOWSOCKS -d 10.0.0.0/8 -j RETURN
    iptables -t nat -A SHADOWSOCKS -d 127.0.0.0/8 -j RETURN
    iptables -t nat -A SHADOWSOCKS -d 169.254.0.0/16 -j RETURN
    iptables -t nat -A SHADOWSOCKS -d 172.16.0.0/12 -j RETURN
    iptables -t nat -A SHADOWSOCKS -d 192.168.0.0/16 -j RETURN
    iptables -t nat -A SHADOWSOCKS -d 224.0.0.0/4 -j RETURN
    iptables -t nat -A SHADOWSOCKS -d 240.0.0.0/4 -j RETURN

    # Ignore CERNET FREE IP address
    iptables -t nat -A SHADOWSOCKS -d 1.8.1.0/255.255.255.0 -j RETURN
    iptables -t nat -A SHADOWSOCKS -d 1.8.6.0/255.255.255.0 -j RETURN
    iptables -t nat -A SHADOWSOCKS -d 1.8.8.0/255.255.254.0 -j RETURN
    ......
    iptables -t nat -A SHADOWSOCKS -d 223.252.192.0/255.255.192.0 -j RETURN

    # Redirect all tcp traffic
    iptables -t nat -A SHADOWSOCKS -p tcp -j REDIRECT --to-ports 1080

    # Apply the rules
    iptables -t nat -I PREROUTING -p tcp -j SHADOWSOCKS


将中间省略部分将免费地址范围以`网络号/网络掩码`的格式填入其中，注意CERNET提供的[免费地址范围](http://www.nic.edu.cn/RS/ipstat/internalip/ "cernet-free-ip-list")与PKU官方提供的[免费地址范围](https://its.pku.edu.cn/oper/liebiao.jsp "pku-free-ip-list")并不相同，推荐参考PKU官方提供的[免费地址范围](https://its.pku.edu.cn/oper/liebiao.jsp "pku-free-ip-list")，为了减少文件体积，可以使用十进制位掩码格式而不是十进制点分式数掩码格式，即将`1.8.1.0/255.255.255.0`表示为`1.8.1.0/24`

另外也可以直接下载制作好的脚本：

    $ wget --no-check-certificate https://darknode.in/i/iptables -O /usr/bin/ss-iptables

注意：`iptables -t nat -A SHADOWSOCKS -d [IP] -j RETURN `这一句最好注释掉，因为shadowsocks走的是IPv6渠道，`iptables`并不会处理IPv6流量，不注释掉这一句，从IPv4访问shadowsocks所在的服务器不会经过shadowsocks的处理，而是直接连接，没有开IP网关收费地址访问权限的话是无法访问的。这就意味着你不能直接在路由器内侧的电脑上直接ssh到服务器，必须手动开IP网关收费地址访问。对于非IPv6环境下的路由器才需要使用这一句确保shadowsocks能够直接走IPv4连接服务器，而不会被重定向到shadowsocks自身，导致循环。

设定可执行权限：

    $ chmod +x /usr/bin/ss-iptables

修改`shadowsocks`启动脚本：

    $ vi /etc/init.d/shadowsocks

注释掉ss-local并去掉ss-redir的注释，并加入`ss-iptables`脚本：

    start() {
            #service_start /usr/bin/ss-local -c $CONFIG -b 0.0.0.0
            service_start /usr/bin/ss-redir -c $CONFIG -b 0.0.0.0
            #service_start /usr/bin/ss-tunnel -c $CONFIG -b 0.0.0.0 -l 5353 -L 8.8.8
            /usr/bin/ss-iptables
    }

    stop() {
            #service_stop /usr/bin/ss-local
            service_stop /usr/bin/ss-redir
            #service_stop /usr/bin/ss-tunnel
            /etc/init.d/firewall restart
    }


为了解决DNS污染的问题，采用路由器端hosts文件的方式处理，这样可以保证墙内资源能够得到准确快速的解析结果，墙外资源得到正确的解析结果，从[这里](https://raw.githubusercontent.com/phoenixlzx/imouto.host/master/imouto.host.txt "imouto-host")获得hosts文件，将文件编码从`UTF-8 with BOM`转换为`UTF-8`，将换行符从`CRLF`转换为`LF`，随后上传至路由器覆盖`/etc/hosts`，也可以直接下载经过修改后的文件镜像：

    $ wget --no-check-certificate https://darknode.in/i/hosts -O /etc/hosts

修改配置文件，使hosts生效：

    $ vi /etc/config/dhcp

在`dnsmasq`段的末尾插入：

    list addnhosts '/etc/hosts'

由于shadowsocks服务器地址为IPv6地址，shadowsocks应在IPv6访问正常后再启动，编辑`rc.local`：

    $ vi /etc/rc.local

在`exit 0`之前加入：

    $ /etc/init.d/shadowsocks start

重启路由器：

    $ reboot

至此，路由器端配置完成。
