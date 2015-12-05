title: Ocserv在Debian下的安装与配置指南
date: 2014-12-18 06:40:35
categories: network
tags: [ocserv, debian, anyconnect]
description: 本文介绍了在Debian下配置Ocserv并启用证书登录与路由分流等优化措施的方案。
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

证书：

> SSL证书 x1

## 基本思路

安装配置ocserv，搭建与Cisco AnyConnect for iOS 兼容的服务器，使用有CA签名的SSL证书对服务器进行认证，配合用户证书进行登陆认证，即可实现iOS系统下自动掉线重连，一键登录，下发路由等多种功能。

## 安装说明

网上有数篇关于ocserv的编译安装方法的文章，其中关于具体编译时需要安装的依赖库有多种不同的说法，如果不想将一大批库装在自己的Linux上，那么需要仔细研究许久。这种情况下，可以在安装好`build-essential`之后不安装其他依赖库，直接执行如下命令：

    $ ./configure --prefix=/usr --sysconfdir=/etc > ./tmp

即可看到当前缺少哪些库。许多相互引用的文章中都提到要安装`libprotobuf-c0-dev`这个库，不过事实上应当安装`libprotobuf-c-dev`这个库。安装前一个库并不能消除执行上述命令时的警告信息，安装后一个库才可以消除警告信息。`libprotobuf-c-dev`库只能从`jessie`源中获得，所以需要添加`backports`和`jessie`两个源。

添加源：

    $ echo "deb http://ftp.debian.org/debian wheezy-backports main contrib non-free" >> /etc/apt/sources.list
    $ echo "deb ftp://ftp.debian.org/debian/ jessie main contrib non-free" >> /etc/apt/sources.list

修改源优先级，防止已安装的包被更新到backports或者jessie版本：

    $ cat << _EOF_ > /etc/apt/preferences
    Package: *
    Pin: release wheezy
    Pin-Priority: 900
    Package: *
    Pin: release wheezy-backports
    Pin-Priority: 90
    Package: *
    Pin: release jessie
    Pin-Priority: 60
    _EOF_

解决编译依赖：

    $ apt-get install build-essential autogen pkg-config -y
    $ apt-get install libtalloc-dev libreadline-dev libpam0g-dev libhttp-parser-dev libpcl1-dev -y
    $ apt-get -t wheezy-backports install libgnutls28-dev -y
    $ apt-get -t jessie install libprotobuf-c-dev libhttp-parser-dev -y

下载ocserv并编译：

    $ cd /tmp
    $ ocserv_version=$(wget -qO- http://www.infradead.org/ocserv/download.html | grep -o '[0-9]*\.[0-9]*\.[0-9]*')
    $ wget ftp://ftp.infradead.org/pub/ocserv/ocserv-$ocserv_version.tar.xz -O ocserv.tar.xz
    $ tar -Jxvf ocserv.tar.xz
    $ cd ./ocserv-*
    $ sed -i "s/#define MAX_CONFIG_ENTRIES 96/#define MAX_CONFIG_ENTRIES 200/g" ./src/vpn.h
    $ ./configure --prefix=/usr --sysconfdir=/etc --without-pam --without-radius
    $ make && make install
    $ cd ~
    $ rm -rf /tmp/ocserv*

## 配置说明

修改ocserv配置文件：

    $ mkdir /etc/ocserv
    $ vi /etc/ocserv/ocserv.conf

使用如下配置：

    auth = "certificate"
    isolate-workers = false
    max-clients = 0
    max-same-clients = 0
    tcp-port = 443
    udp-port = 443
    keepalive = 32400
    dpd = 90
    mobile-dpd = 1800
    try-mtu-discovery = true
    server-cert = /etc/ocserv/cert.pem
    server-key = /etc/ocserv/key.pem
    ca-cert = /etc/ocserv/ca.pem
    cert-user-oid = 2.5.4.3
    compression = true
    no-compress-limit = 256
    tls-priorities = "NORMAL:%SERVER_PRECEDENCE:%COMPAT:-VERS-SSL3.0:-ARCFOUR-128"
    auth-timeout = 40
    idle-timeout = 1200
    mobile-idle-timeout = 2400
    min-reauth-time = 1
    max-ban-score = 50
    ban-reset-time = 300
    cookie-timeout = 172800
    deny-roaming = false
    rekey-time = 172800
    rekey-method = ssl
    use-utmp = true
    use-occtl = true
    occtl-socket-file = /var/run/occtl.socket
    pid-file = /var/run/ocserv.pid
    socket-file = /var/run/ocserv-socket
    run-as-user = nobody
    run-as-group = nogroup
    net-priority = 5
    device = vpn
    predictable-ips = false
    default-domain = 你的域名
    dns = 8.8.8.8
    ipv4-network = 192.168.1.0/24
    ipv6-network = 你的IPv6池
    ping-leases = false
    output-buffer = 1000
    cisco-client-compat = true
    custom-header = "X-DTLS-MTU: 1420"
    custom-header = "X-CSTP-MTU: 1280"
    no-route = 0.0.0.0/255.0.0.0
    no-route = 1.0.0.0/255.128.0.0
    no-route = 1.160.0.0/255.224.0.0
    no-route = 1.192.0.0/255.224.0.0
    no-route = 10.0.0.0/255.0.0.0
    no-route = 14.0.0.0/255.224.0.0
    no-route = 14.96.0.0/255.224.0.0
    no-route = 14.128.0.0/255.224.0.0
    no-route = 14.192.0.0/255.224.0.0
    no-route = 27.0.0.0/255.192.0.0
    no-route = 27.96.0.0/255.224.0.0
    no-route = 27.128.0.0/255.128.0.0
    no-route = 36.0.0.0/255.192.0.0
    no-route = 36.96.0.0/255.224.0.0
    no-route = 36.128.0.0/255.128.0.0
    no-route = 39.0.0.0/255.224.0.0
    no-route = 39.64.0.0/255.192.0.0
    no-route = 39.128.0.0/255.192.0.0
    no-route = 42.0.0.0/255.0.0.0
    no-route = 43.224.0.0/255.224.0.0
    no-route = 45.64.0.0/255.192.0.0
    no-route = 47.64.0.0/255.192.0.0
    no-route = 49.0.0.0/255.128.0.0
    no-route = 49.128.0.0/255.224.0.0
    no-route = 49.192.0.0/255.192.0.0
    no-route = 54.192.0.0/255.224.0.0
    no-route = 58.0.0.0/255.128.0.0
    no-route = 58.128.0.0/255.224.0.0
    no-route = 58.192.0.0/255.192.0.0
    no-route = 59.32.0.0/255.224.0.0
    no-route = 59.64.0.0/255.192.0.0
    no-route = 59.128.0.0/255.128.0.0
    no-route = 60.0.0.0/255.192.0.0
    no-route = 60.160.0.0/255.224.0.0
    no-route = 60.192.0.0/255.192.0.0
    no-route = 61.0.0.0/255.192.0.0
    no-route = 61.64.0.0/255.224.0.0
    no-route = 61.128.0.0/255.192.0.0
    no-route = 61.224.0.0/255.224.0.0
    no-route = 100.64.0.0/255.192.0.0
    no-route = 101.0.0.0/255.128.0.0
    no-route = 101.128.0.0/255.224.0.0
    no-route = 101.192.0.0/255.192.0.0
    no-route = 103.0.0.0/255.192.0.0
    no-route = 103.224.0.0/255.224.0.0
    no-route = 106.0.0.0/255.128.0.0
    no-route = 106.224.0.0/255.224.0.0
    no-route = 110.0.0.0/254.0.0.0
    no-route = 112.0.0.0/255.128.0.0
    no-route = 112.128.0.0/255.224.0.0
    no-route = 112.192.0.0/255.192.0.0
    no-route = 113.0.0.0/255.128.0.0
    no-route = 113.128.0.0/255.224.0.0
    no-route = 113.192.0.0/255.192.0.0
    no-route = 114.0.0.0/255.128.0.0
    no-route = 114.128.0.0/255.224.0.0
    no-route = 114.192.0.0/255.192.0.0
    no-route = 115.0.0.0/255.0.0.0
    no-route = 116.0.0.0/255.0.0.0
    no-route = 117.0.0.0/255.128.0.0
    no-route = 117.128.0.0/255.192.0.0
    no-route = 118.0.0.0/255.224.0.0
    no-route = 118.64.0.0/255.192.0.0
    no-route = 118.128.0.0/255.128.0.0
    no-route = 119.0.0.0/255.128.0.0
    no-route = 119.128.0.0/255.192.0.0
    no-route = 119.224.0.0/255.224.0.0
    no-route = 120.0.0.0/255.192.0.0
    no-route = 120.64.0.0/255.224.0.0
    no-route = 120.128.0.0/255.224.0.0
    no-route = 120.192.0.0/255.192.0.0
    no-route = 121.0.0.0/255.128.0.0
    no-route = 121.192.0.0/255.192.0.0
    no-route = 122.0.0.0/254.0.0.0
    no-route = 124.0.0.0/255.0.0.0
    no-route = 125.0.0.0/255.128.0.0
    no-route = 125.160.0.0/255.224.0.0
    no-route = 125.192.0.0/255.192.0.0
    no-route = 127.0.0.0/255.0.0.0
    no-route = 139.0.0.0/255.224.0.0
    no-route = 139.128.0.0/255.128.0.0
    no-route = 140.64.0.0/255.224.0.0
    no-route = 140.128.0.0/255.224.0.0
    no-route = 140.192.0.0/255.192.0.0
    no-route = 144.0.0.0/255.192.0.0
    no-route = 144.96.0.0/255.224.0.0
    no-route = 144.224.0.0/255.224.0.0
    no-route = 150.0.0.0/255.224.0.0
    no-route = 150.96.0.0/255.224.0.0
    no-route = 150.128.0.0/255.224.0.0
    no-route = 150.192.0.0/255.192.0.0
    no-route = 152.96.0.0/255.224.0.0
    no-route = 153.0.0.0/255.192.0.0
    no-route = 153.96.0.0/255.224.0.0
    no-route = 157.0.0.0/255.192.0.0
    no-route = 157.96.0.0/255.224.0.0
    no-route = 157.128.0.0/255.224.0.0
    no-route = 157.224.0.0/255.224.0.0
    no-route = 159.224.0.0/255.224.0.0
    no-route = 161.192.0.0/255.224.0.0
    no-route = 162.96.0.0/255.224.0.0
    no-route = 163.0.0.0/255.192.0.0
    no-route = 163.96.0.0/255.224.0.0
    no-route = 163.128.0.0/255.192.0.0
    no-route = 163.192.0.0/255.224.0.0
    no-route = 166.96.0.0/255.224.0.0
    no-route = 167.128.0.0/255.192.0.0
    no-route = 168.160.0.0/255.224.0.0
    no-route = 169.254.0.0/255.255.0.0
    no-route = 171.0.0.0/255.128.0.0
    no-route = 171.192.0.0/255.224.0.0
    no-route = 172.16.0.0/255.240.0.0
    no-route = 175.0.0.0/255.128.0.0
    no-route = 175.128.0.0/255.192.0.0
    no-route = 180.64.0.0/255.192.0.0
    no-route = 180.128.0.0/255.128.0.0
    no-route = 182.0.0.0/255.0.0.0
    no-route = 183.0.0.0/255.192.0.0
    no-route = 183.64.0.0/255.224.0.0
    no-route = 183.128.0.0/255.128.0.0
    no-route = 192.0.0.0/255.255.255.0
    no-route = 192.0.2.0/255.255.255.0
    no-route = 192.88.99.0/255.255.255.0
    no-route = 192.96.0.0/255.224.0.0
    no-route = 192.160.0.0/255.248.0.0
    no-route = 192.168.0.0/255.255.0.0
    no-route = 192.169.0.0/255.255.0.0
    no-route = 192.170.0.0/255.254.0.0
    no-route = 192.172.0.0/255.252.0.0
    no-route = 192.176.0.0/255.240.0.0
    no-route = 198.18.0.0/255.254.0.0
    no-route = 198.51.100.0/255.255.255.0
    no-route = 202.0.0.0/255.128.0.0
    no-route = 202.128.0.0/255.192.0.0
    no-route = 202.192.0.0/255.224.0.0
    no-route = 203.0.0.0/255.128.0.0
    no-route = 203.128.0.0/255.192.0.0
    no-route = 203.192.0.0/255.224.0.0
    no-route = 210.0.0.0/255.192.0.0
    no-route = 210.64.0.0/255.224.0.0
    no-route = 210.160.0.0/255.224.0.0
    no-route = 210.192.0.0/255.224.0.0
    no-route = 211.64.0.0/255.192.0.0
    no-route = 211.128.0.0/255.192.0.0
    no-route = 218.0.0.0/255.128.0.0
    no-route = 218.160.0.0/255.224.0.0
    no-route = 218.192.0.0/255.192.0.0
    no-route = 219.64.0.0/255.224.0.0
    no-route = 219.128.0.0/255.224.0.0
    no-route = 219.192.0.0/255.192.0.0
    no-route = 220.96.0.0/255.224.0.0
    no-route = 220.128.0.0/255.128.0.0
    no-route = 221.0.0.0/255.224.0.0
    no-route = 221.96.0.0/255.224.0.0
    no-route = 221.128.0.0/255.128.0.0
    no-route = 222.0.0.0/255.0.0.0
    no-route = 223.0.0.0/255.224.0.0
    no-route = 223.64.0.0/255.192.0.0
    no-route = 223.128.0.0/255.128.0.0
    no-route = 224.0.0.0/224.0.0.0

为了方便有多个设备的用户的使用，设置`max-same-clients = 0`可使得无限个设备可以使用同一个账号密码进行连接，也可以为其指定一个特定的上限进行限制。

在复杂网络条件下，缩短`mobile-dpd`的数值可以提高移动设备上AnyConnect客户端检测连接是否中断的频率，从而减少锁屏时在基站间切换时造成的暂时连接中断。默认设置下AnyConnect客户端会每隔30分钟检测一次连接情况，并在发现连接中断后自动重新连接。缩短`mobile-dpd`的数值可能提高设备的唤醒频率，从而增加电池的消耗，请酌情取舍。

由于iOS系统在锁屏一段时间后会中断VPN连接，而屏幕解锁后AnyConnect会自动重新连接，如果中间连接中断的时间超过了`cookie-timeout`参数设置的数值，那么重新连接会失败。`cookie-timeout = 86400000`可以使AnyConnect在连接中断后的1000天内自动重新连接成功。

`cisco-client-compat = true`可以确保ocserv同Cisco AnyConnect for iOS的兼容性，若不启用会导致证书登陆报出`GnuTLS error: No certificate was found`这样的错误。

请将服务器SSL证书放置在`/etc/ocserv/cert.pem`，服务器SSL证书秘钥放置在`/etc/ocserv/cert.key`。

安装`gnutls`用于签发证书：

    $ apt-get install gnutls-bin

创建用户证书认证CA：

    $ certtool --generate-privkey --outfile ca-key.pem
    $ cat << _EOF_ > ca.tmpl
    cn = "VPN CA"
    organization = "DarkNode"
    serial = 1
    expiration_days = 3650
    ca
    signing_key
    cert_signing_key
    crl_signing_key
    _EOF_
    $ certtool --generate-self-signed --load-privkey ca-key.pem --template ca.tmpl --outfile ca-cert.pem

创建用户证书：

    $ certtool --generate-privkey --outfile user-key.pem
    $ cat << _EOF_ > user.tmpl
    cn = "VPN"
    unit = "VPN"
    expiration_days = 365
    signing_key
    tls_www_client
    _EOF_
    $ certtool --generate-certificate --load-privkey user-key.pem --load-ca-certificate ca-cert.pem --load-ca-privkey ca-key.pem --template user.tmpl --outfile user-cert.pem

使用openssl将用户证书转换为`.p12`格式，以避免使用certtool转换时与Cisco AnyConnect for iOS的兼容性问题：

    $ openssl pkcs12 -export -inkey user-key.pem -in user-cert.pem -certfile ca-cert.pem -out user.p12 -password pass:

将用户证书认证CA拷贝到ocserv配置文件中设置的位置：

    $ cp ./ca-cert.pem /etc/ocserv/ca.pem

将用户证书`user.p12`放置到某种Web服务器的网页目录下，使得用户证书可以通过URL访问得到，随后打开Cisco AnyConnect for iOS，依次点击“诊断”、“证书”、“导入用户证书”，输入`user.p12`的完整URL即可。

创建管理脚本：

    $ vi /etc/init.d/ocserv

保存如下脚本：

    #!/bin/sh
    ### BEGIN INIT INFO
    # Provides:          ocserv
    # Required-Start:    $remote_fs $syslog
    # Required-Stop:     $remote_fs $syslog
    # Default-Start:     2 3 4 5
    # Default-Stop:      0 1 6
    ### END INIT INFO
    # Copyright Rene Mayrhofer, Gibraltar, 1999
    # This script is distibuted under the GPL

    PATH=/bin:/usr/bin:/sbin:/usr/sbin
    DAEMON=/usr/sbin/ocserv
    PIDFILE=/var/run/ocserv.pid
    DAEMON_ARGS="-c /etc/ocserv/ocserv.conf"

    case "$1" in
    start)
    if [ ! -r $PIDFILE ]; then
    echo -n "Starting OpenConnect VPN Server Daemon: "
    start-stop-daemon --start --quiet --pidfile $PIDFILE --exec $DAEMON -- \
    $DAEMON_ARGS > /dev/null
    echo "ocserv."
    else
    echo -n "OpenConnect VPN Server is already running.\n\r"
    exit 0
    fi
    ;;
    stop)
    echo -n "Stopping OpenConnect VPN Server Daemon: "
    start-stop-daemon --stop --quiet --pidfile $PIDFILE --exec $DAEMON
    echo "ocserv."
    rm -f $PIDFILE
    ;;
    force-reload|restart)
    echo "Restarting OpenConnect VPN Server: "
    $0 stop
    sleep 1
    $0 start
    ;;
    status)
    if [ ! -r $PIDFILE ]; then
    # no pid file, process doesn't seem to be running correctly
    exit 3
    fi
    PID=`cat $PIDFILE | sed 's/ //g'`
    EXE=/proc/$PID/exe
    if [ -x "$EXE" ] &&
    [ "`ls -l \"$EXE\" | cut -d'>' -f2,2 | cut -d' ' -f2,2`" = \
    "$DAEMON" ]; then
    # ok, process seems to be running
    exit 0
    elif [ -r $PIDFILE ]; then
    # process not running, but pidfile exists
    exit 1
    else
    # no lock file to check for, so simply return the stopped status
    exit 3
    fi
    ;;
    *)
    echo "Usage: /etc/init.d/ocserv {start|stop|restart|force-reload|status}"
    exit 1
    ;;
    esac

    exit 0

修改脚本权限：

    $ chmod 755 /etc/init.d/ocserv

启用流量转发：

    $ vi /etc/sysctl.conf
    net.ipv4.ip_forward = 1
    net.ipv6.conf.all.forwarding=1
    $ sysctl -p

修改`rc.local`：

    $ vi /etc/rc.local

在`exit 0`前加入：

    iptables -I FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
    iptables -t nat -A POSTROUTING -o venet0 -j MASQUERADE
    iptables -I INPUT -p tcp --dport 443 -j ACCEPT
    iptables -I INPUT -p udp --dport 443 -j ACCEPT
    /etc/init.d/ocserv start

至此，服务端配置完毕。
