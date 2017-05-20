title: 用ACL代替Pac进行更底层的智能分流
date: 2017-04-23T21:10:00+08:00
updated: 2017-04-26T22:40:00+08:00
categories: network
tags: [acl, shadowsocks, geoip]
description: 本文介绍了如何使用shadowsocks-libev自带的ACL功能实现智能分流的方案
---

## 相关说明

之前我曾经使用Pac进行[智能分流](https://darknode.in/network/write-efficient-pac/)，但这一方案存在几个缺陷：

1. Pac运作于浏览器的js引擎中，而各个系统支持的js版本不同，例如Windows下就不支持reduce函数。
2. Pac只能控制HTTP协议相关的代理，比如Mail.app就不会受Pac的控制。

后来发现[shadowsocks-libev](https://github.com/shadowsocks/shadowsocks-libev)原生提供了ACL控制功能，可以直接使用ACL代替Pac完成智能分流。

## ACL相关配置

可以直接参考项目中的ACL[配置范例](https://github.com/shadowsocks/shadowsocks-libev/tree/master/acl)，目前可以使用白名单模式或者是黑名单模式。

白名单模式的基本语法如下：

    [proxy_all]

    [bypass_list]
    1.0.1.0/24
    1.0.2.0/23
    1.0.8.0/21
    (^|\.)baidu\.com$

黑名单模式的基本语法如下：

    [bypass_all]

    [proxy_list]
    91.108.4.0/22
    91.108.56.0/22
    (^|\.)google(\.(?!-)[a-zA-Z0-9-]{1,63}(?<!-)){1,2}$
    (^|\.)googleapis(\.(?!-)[a-zA-Z0-9-]{1,63}(?<!-)){1,2}$

支持使用正则匹配域名或者是使用IP段匹配IP，看起来很美好，但是socks代理有一个特性：DNS解析是由客户端程序控制的：

1. 客户端程序可以通过socks5代理连接一个域名，在socks5服务器上解析域名再连接IP，
2. 客户端程序也可以在本地进行DNS解析，然后通过socks5代理直接连接一个IP。

而目前的[shadowsocks-libev](https://github.com/shadowsocks/shadowsocks-libev)只能直接处理第一种方式下的域名，或者第二种方式下的IP，也就是说，我们必须指明域名在白名单中还是黑名单中，如果域名不在名单里，无法根据域名解析出来的IP进行进一步GEOIP分流。

## 增加DNS本地解析

ACL相关的处理代码在`local.c`中的如下段落：

    int bypass = 0;
    if (host_match > 0)
        bypass = 1;                 // bypass hostnames in black list
    else if (host_match < 0)
        bypass = 0;                 // proxy hostnames in white list
    else {
        int ip_match = acl_match_host(ip);
        switch (get_acl_mode()) {
            case BLACK_LIST:
                if (ip_match > 0)
                    bypass = 1;               // bypass IPs in black list
                break;
            case WHITE_LIST:
                bypass = 1;
                if (ip_match < 0)
                    bypass = 0;               // proxy IPs in white list
                break;
        }
    }

在处理完对域名的匹配之后，会直接进行对IP的匹配，但是直接连接域名的时候IP是空的，所以需要追加DNS解析的代码：

    int bypass = 0;
    int resolved = 0;
    struct sockaddr_storage storage;
    memset(&storage, 0, sizeof(struct sockaddr_storage));
    int err;

    if (host_match > 0)
        bypass = 1;                 // bypass hostnames in black list
    else if (host_match < 0)
        bypass = 0;                 // proxy hostnames in white list
    else {
    #ifndef ANDROID
        if (atyp == 3) {            // resolve domain so we can bypass domain with geoip
            err = get_sockaddr(host, port, &storage, 0, ipv6first);
            if ( err != -1) {
                resolved = 1;
                switch(((struct sockaddr*)&storage)->sa_family) {
                    case AF_INET: {
                        struct sockaddr_in *addr_in = (struct sockaddr_in *)&storage;
                        dns_ntop(AF_INET, &(addr_in->sin_addr), ip, INET_ADDRSTRLEN);
                        break;
                    }
                    case AF_INET6: {
                        struct sockaddr_in6 *addr_in6 = (struct sockaddr_in6 *)&storage;
                        dns_ntop(AF_INET6, &(addr_in6->sin6_addr), ip, INET6_ADDRSTRLEN);
                        break;
                    }
                    default:
                        break;
                }
            }
        }
    #endif
        int ip_match = acl_match_host(ip);
        switch (get_acl_mode()) {
            case BLACK_LIST:
                if (ip_match > 0)
                    bypass = 1;               // bypass IPs in black list
                break;
            case WHITE_LIST:
                bypass = 1;
                if (ip_match < 0)
                    bypass = 0;               // proxy IPs in white list
                break;
        }
    }

这样就能进行一次本地DNS解析了，之后进入对IP的匹配处理，就能实现当域名匹配失败之后继续对IP进行匹配的功能了。

目前这一改动已经被[merge](https://github.com/shadowsocks/shadowsocks-libev/commit/81006d4690cfbd04afabc176d2c1a3bb69a12808)进主分支了，所以下一个版本3.0.6就无需再改动代码并重新编译了。

## DNS污染问题

在目前的逻辑下，优先匹配域名，随后匹配IP，所以只要使用黑名单模式，将被污染的域名加入黑名单，不进行解析就直接走代理：

    [bypass_all]

    [proxy_list]
    google
    (^|\.)twitter\.com$
    (^|\.)twimg\.com$
    (^|\.)facebook\.com$

参考现在流行的[surge](https://nssurge.com)中的`force-remote-resolve`配置项，一般来说只有少数几个域名会享受这个待遇，比如Google、Facebook、Twitter家的几个域名遭遇了这一问题，比如`google`这一关键字，`twimg.com`这一域名等。

随后我们需要将国外IP也放入`proxy_list`中，从[APNIC](https://ftp.apnic.net/stats/apnic/delegated-apnic-latest)获得IP表再进行处理就好，首先提取出中国的IP段，随后求其补集，再去除保留IP段即可。若直接提取非中国的IP段，那么很多分配不明的IP段会被排除，导致无法连接telegram等情况发生。建议直接使用[此项目](https://github.com/x1angli/regional-ip-addresses)生成的`outwall.txt`。
