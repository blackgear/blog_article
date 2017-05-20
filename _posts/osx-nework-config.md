title: OS X下的科学上网完整配置方案
date: 2015-09-30T01:00:00+08:00
updated: 2016-01-21T14:35:00+08:00
categories: network
tags: [osx, shadowsocks, polipo]
description: 本文介绍了在OS X下进行从普通应用程序到命令行程序的完整科学上网配置方案
---

## 相关说明

本方案的基本架构如下：

使用[Pcap\_DNSProxy](https://github.com/chengr28/Pcap_DNSProxy)提供最佳的DNS查询结果，通过[shadowsocks-libev](https://github.com/shadowsocks/shadowsocks-libev)提供顺畅的网络连接，通过[polipo](https://github.com/jech/polipo)将socks5代理转换为兼容性更好的http代理并提供一个本地http服务器，利用[Mono\_Pac](https://github.com/blackgear/mono_pac)生成适宜的Pac文件并用[polipo](https://github.com/jech/polipo)的本地http服务器来host。

本方案使用的所有程序均为开源程序，均不对除本机外的设备提供网络服务，以避免各种潜在威胁。

## 本地DNS的配置

首先通过Homebrew安装[Pcap\_DNSProxy](https://github.com/chengr28/Pcap_DNSProxy)并修改其配置文件中的如下三项：

    $ brew install pcap_dnsproxy
    $ vi /usr/local/etc/pcap_DNSproxy/Config.ini
    [Listen]
    Operation Mode = Proxy
    [Local DNS]
    Local Main = 1
    Local Routing = 1

修改`Operation Mode`为`Proxy`以确保其只为本机提供代理域名解析请求服务，修改`Local Main`、`Local Routing`为`1`以确保国内网站获得更好的CDN解析结果。

设置开机启动：

    $ sudo cp -fv /usr/local/opt/pcap_dnsproxy/*.plist /Library/LaunchDaemons
    $ sudo chown root /Library/LaunchDaemons/homebrew.mxcl.pcap_dnsproxy.plist
    $ launchctl load /Library/LaunchDaemons/homebrew.mxcl.pcap_dnsproxy.plist

进入系统偏好设置-网络-高级-DNS，将DNS服务器设置为`127.0.0.1`即可。

## 本地网络连接的配置

首先通过Homebrew安装[shadowsocks-libev](https://github.com/shadowsocks/shadowsocks-libev)并修改配置文件，在其中加入`local`项，以确保其只为本机提供服务：

    $ brew install shadowsocks-libev
    $ vi /usr/local/etc/shadowsocks-libev.json
    {
        "server":"服务器地址",
        "server_port":服务器端口,
        "local":"localhost",
        "local_port":1080,
        "password":"服务器密码",
        "timeout":服务器超时时间,
        "method":"服务器加密方式"
    }

设置开机启动：

    $ ln -sfv /usr/local/opt/shadowsocks-libev/*.plist ~/Library/LaunchAgents
    $ launchctl load ~/Library/LaunchAgents/homebrew.mxcl.shadowsocks-libev.plist

随后通过Homebrew安装[polipo](https://github.com/jech/polipo)并在`~/.polipo`新建配置文件：

    $ brew install polipo
    $ vi ~/.polipo
    socksParentProxy = "127.0.0.1:1080"
    socksProxyType = "socks5"
    proxyAddress = "127.0.0.1"
    proxyPort = 8080
    localDocumentRoot = "~/.www/"
    dnsNameServer = "127.0.0.1"
    dnsUseGethostbyname = false
    dnsMaxTimeout = 1s
    dnsQueryIPv6 = false
    cacheIsShared = true
    diskCacheRoot = ""
    disableVia = true
    censorReferer = false
    tunnelAllowedPorts = 1-65535
    disableLocalInterface = true

设置开机启动：

    $ ln -sfv /usr/local/opt/polipo/*.plist ~/Library/LaunchAgents
    $ launchctl load ~/Library/LaunchAgents/homebrew.mxcl.polipo.plist

创建`~/.www`目录，并使用[Mono\_Pac](https://github.com/blackgear/mono_pac)项目生成符合自己需求的Pac文件：

    $ mkdir ~/.www
    $ git clone https://github.com/blackgear/mono_pac.git
    $ cd ./src
    $ python ./make.py -p "PROXY 127.0.0.1:8080" -o ~/.www/proxy.pac

进入系统偏好设置-网络-高级-代理，将自动代理配置URL设置为`http://127.0.0.1:8080/proxy.pac`使GUI程序通过Pac决定是否走代理。

修改`~/.bash_profile`，加入与代理有关的环境变量使得命令行程序通过代理连接网络：

    $ vi ~/.bash_profile
    export all_proxy="http://127.0.0.1:8080"
    export http_proxy=$all_proxy
    export https_proxy=$all_proxy
    export ftp_proxy=$all_proxy
    export rsync_proxy=$all_proxy

通过Homebrew安装[socat](http://www.dest-unreach.org/socat/)并修改`~/.ssh/config`，使ssh连接通过代理进行连接：

    $ brew install socat
    $ vi ~/.ssh/config
    Host *
        ServerAliveInterval 5
        ServerAliveCountMax 3
        ControlMaster auto
        ControlPath ~/.ssh/%h-%p-%r
        ControlPersist yes
        ProxyCommand /usr/local/bin/socat - proxy:localhost:%h:%p

修改`~/.gitconfig`使git通过代理进行连接：

    $ cat ~/.gitconfig
    [https]
        proxy = 127.0.0.1:8080
    [http]
        proxy = 127.0.0.1:8080
