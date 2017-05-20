title: 在Alpine Linux下静态链接并打包程序
date: 2016-12-05 00:00:00
updated: 2017-03-20T18:30:00+08:00
categories: linux
tags: [alpine, static, fpm]
description: 本文介绍了在Alpine下编译并静态链接，随后打包应用以便快速部署的方法
---

## 相关说明

很长一段时间内，我考虑过在VPS上使用Docker来快速初始化，但相对于我目前的使用方式来看，Docker从安装到使用都过于复杂了。

某一天我从Docker想到Golang，从Golang想到静态编译，又联想起fpm打包工具，一个新的方案产生了。

文中命令以Debian 8.0为基础。

## 安装Alpine Linux

将Alpine Linux安装到/var/rootfs下，再chroot进去，就能获得一个干净而隔离的编译环境，Alpine Linux下默认使用[musl-libc](https://www.musl-libc.org)，一个可以很方便地静态链接到程序里的libc库。

    #!/usr/bin/env bash
    # -*- coding: utf-8 -*-

    export SOURCE='http://dl-cdn.alpinelinux.org/alpine/latest-stable/main'
    export CHROOT='/var/rootfs'
    export VER_APK="2.6.8-r2"

    mkdir -p $CHROOT
    curl $SOURCE/x86_64/apk-tools-static-$VER_APK.apk | tar xz -C / sbin/apk.static
    apk.static -X $SOURCE -U --allow-untrusted --root $CHROOT --initdb add alpine-base

    echo 'nameserver 8.8.8.8' > $CHROOT/etc/resolv.conf
    echo $SOURCE > $CHROOT/etc/apk/repositories

    mount -t proc none $CHROOT/proc
    mount -o bind /sys $CHROOT/sys
    mount -o bind /dev $CHROOT/dev

    chroot $CHROOT /bin/sh -l

## 编译并静态链接Nginx

我使用如下的脚本编译并静态链接Nginx：

    #!/usr/bin/env bash
    # -*- coding: utf-8 -*-

    export VER_NGINX="1.11.13"
    export VER_LIBERSSL="2.5.2"
    export VER_ZLIB="1.2.11"
    export VER_PCRE="8.40"
    export PKG_BUILD="curl build-base linux-headers perl"

    apk --update add $PKG_BUILD
    curl -sSL http://nginx.org/download/nginx-$VER_NGINX.tar.gz | tar xz -C /tmp
    curl -sSL http://ftp.openbsd.org/pub/OpenBSD/LibreSSL/libressl-$VER_LIBERSSL.tar.gz | tar xz -C /tmp
    curl -sSL http://ftp.csx.cam.ac.uk/pub/software/programming/pcre/pcre-$VER_PCRE.tar.gz | tar xz -C /tmp
    curl -sSL http://zlib.net/zlib-$VER_ZLIB.tar.gz | tar xz -C /tmp

    cd /tmp/nginx-$VER_NGINX/
        ./configure \
            --prefix=/etc/nginx \
            --sbin-path=/usr/sbin/nginx \
            --conf-path=/etc/nginx/nginx.conf \
            --error-log-path=/var/log/nginx/error.log \
            --http-log-path=/var/log/nginx/access.log \
            --pid-path=/var/run/nginx.pid \
            --lock-path=/var/run/nginx.lock \
            --http-client-body-temp-path=/var/cache/nginx/client_temp \
            --http-proxy-temp-path=/var/cache/nginx/proxy_temp \
            --user=nobody \
            --group=nogroup \
            --with-file-aio \
            --with-threads \
            --with-http_ssl_module \
            --with-http_v2_module \
            --with-http_secure_link_module \
            --without-http_fastcgi_module \
            --without-http_uwsgi_module \
            --without-http_scgi_module \
            --without-http_memcached_module \
            --with-openssl=../libressl-$VER_LIBERSSL \
            --with-zlib=../zlib-$VER_ZLIB \
            --with-pcre=../pcre-$VER_PCRE \
            --with-pcre-jit \
            --with-cc=x86_64-alpine-linux-musl-gcc \
            --with-cc-opt="-Os" \
            --with-ld-opt="-static -s"
        make install

生成的程序将在`/var/rootfs/usr/sbin/nginx`处。

## 打包Nginx

首先参考[官方文档](https://fpm.readthedocs.io/en/latest/installing.html)安装fpm，接着准备如下的目录结构：

    nginx
    ├── etc
    │   └── nginx
    │       ├── ca.pem
    │       ├── cert.pem
    │       ├── dhparam.pem
    │       ├── key.pem
    │       ├── mime.conf
    │       ├── nginx.conf
    │       └── site.conf
    ├── script
    │   └── nginx
    ├── usr
    │   └── sbin
    │       └── nginx
    └── var
        ├── cache
        │   └── nginx
        └── log
            └── nginx

其中`nginx/script/nginx`是systemd配置文件：

    $ cat script/nginx
    [Unit]
    Description=Nginx Server Service
    After=network.target

    [Service]
    Type=forking
    PIDFile=/var/run/nginx.pid
    ExecStartPre=/bin/mkdir -p /var/www/
    ExecStartPre=/bin/chown Daniel:Daniel /var/www/
    ExecStartPre=/usr/sbin/nginx -t
    ExecStart=/usr/sbin/nginx
    ExecReload=/usr/sbin/nginx -s reload
    ExecStop=/sbin/start-stop-daemon --quiet --stop --retry QUIT/5 --pidfile /run/nginx.pid
    TimeoutStopSec=5
    KillMode=mixed

    [Install]
    WantedBy=multi-user.target

其中`nginx/etc/nginx`目录下是nginx配置文件，如此即可将nginx的配置文件一并打包，打包命令如下：

    fpm -s dir -C nginx -t deb -n nginx -v 1.11.13-0 -a amd64 -f \
        --exclude 'script' \
        --deb-systemd    nginx/script/nginx

随后即可生成`nginx_1.11.13-0_amd64.deb `，在目标机器上直接执行`dpkg -i nginx_1.11.13-0_amd64.deb`一键部署完成Nginx。

## 编译并静态链接shadowsocks

    #!/usr/bin/env bash
    # -*- coding: utf-8 -*-

    export VER_SSLIBEV="3.0.5"
    export VER_SIPOBFS="0.0.3"
    export VER_MBEDTLS="2.4.2"
    export VER_SODIUM="1.0.12"
    export VER_UDNS="0.4"
    export VER_PCRE="8.40"
    export VER_EV="4.24"
    export PKG_BUILD="curl build-base linux-headers autoconf automake libtool"
    export CFLAGS="-Os"
    export LDFLAGS="-static -s"

    apk --update upgrade
    apk --update add $PKG_BUILD
    curl -sSL https://github.com/shadowsocks/shadowsocks-libev/archive/v$VER_SSLIBEV.tar.gz | tar xz -C /tmp
    curl -sSL https://github.com/shadowsocks/simple-obfs/archive/v$VER_SIPOBFS.tar.gz | tar xz -C /tmp
    curl -sSL https://github.com/ARMmbed/mbedtls/archive/mbedtls-$VER_MBEDTLS.tar.gz | tar xz -C /tmp
    curl -sSL https://github.com/jedisct1/libsodium/releases/download/$VER_SODIUM/libsodium-$VER_SODIUM.tar.gz | tar xz -C /tmp
    curl -sSL http://www.corpit.ru/mjt/udns/udns-$VER_UDNS.tar.gz | tar xz -C /tmp
    curl -sSL http://ftp.csx.cam.ac.uk/pub/software/programming/pcre/pcre-$VER_PCRE.tar.gz | tar xz -C /tmp
    curl -sSL http://dist.schmorp.de/libev/Attic/libev-$VER_EV.tar.gz | tar xz -C /tmp

    curl -sSL https://github.com/shadowsocks/libbloom/archive/master.tar.gz | tar xz -C /tmp
    curl -sSL https://github.com/shadowsocks/libcork/archive/shadowsocks.tar.gz | tar xz -C /tmp
    curl -sSL https://github.com/shadowsocks/ipset/archive/shadowsocks.tar.gz | tar xz -C /tmp

    cd /tmp/libsodium-$VER_SODIUM/
        ./configure \
            --prefix=/usr \
            --enable-shared=false
        make install

    cd /tmp/mbedtls-mbedtls-$VER_MBEDTLS/
        make DESTDIR=/usr install

    cd /tmp/udns-$VER_UDNS/
        ./configure --with-ipv6
        make
        install -D -m0644 udns.h /usr/include/udns.h
        install -D -m0755 libudns.a /usr/lib/libudns.a

    cd /tmp/pcre-$VER_PCRE/
        ./configure \
            --prefix=/usr \
            --enable-jit \
            --enable-utf8 \
            --enable-unicode-properties \
            --enable-shared=false \
            --disable-cpp \
            --with-match-limit-recursion=8192
        make install

    cd /tmp/libev-$VER_EV/
        ./configure \
            --prefix=/usr \
            --enable-shared=false
        make install

    cd /tmp/shadowsocks-libev-$VER_SSLIBEV/
        cp /tmp/libbloom-master/* libbloom/
        cp /tmp/libcork-shadowsocks/* libcork/
        cp /tmp/ipset-shadowsocks/* libipset/
        ./autogen.sh
        ./configure --disable-documentation
        make install

    cd /tmp/simple-obfs-$VER_SIPOBFS/
        cp /tmp/libcork-shadowsocks/* libcork/
        ./autogen.sh
        ./configure --disable-documentation
        make install

生成的程序将在`/var/rootfs/usr/local/bin/ss-server`和`/var/rootfs/usr/local/bin/obfs-server`处。

## 打包shadowsocks

准备如下的目录结构：

    shadowsocks
    ├── etc
    │   └── shadowsocks.json
    ├── script
    │   ├── obfs
    │   └── shadowsocks
    └── usr
        └── bin
            ├── obfs-server
            └── ss-server

其中`script/obfs`和`script/shadowsocks`是systemd配置文件：

    $ cat shadowsocks/script/obfs
    [Unit]
    Description=Shadowsocks-Libev Obfs Service
    After=network.target

    [Service]
    Type=simple
    LimitNOFILE=32768
    ExecStart=/usr/bin/obfs-server -s 0.0.0.0 -p xxx --obfs http -r 127.0.0.1:xxx --failover 127.0.0.1:xxx
    Restart=on-failure

    [Install]
    WantedBy=multi-user.target

    $ cat shadowsocks/script/shadowsocks
    [Unit]
    Description=Shadowsocks-Libev Server Service
    After=network.target

    [Service]
    Type=simple
    LimitNOFILE=32768
    ExecStart=/usr/bin/ss-server -c /etc/shadowsocks.json
    Restart=on-failure

    [Install]
    WantedBy=multi-user.target

其中`shadowsocks/etc/shadowsocks.json`目录下是shadowsocks配置文件，如此即可将shadowsocks的配置文件一并打包，打包命令如下：

    fpm -s dir -C shadowsocks -t deb -n shadowsocks -v 3.0.5-0 -a amd64 -f \
        --exclude 'script' \
        --deb-systemd    shadowsocks/script/shadowsocks \
        --deb-systemd    shadowsocks/script/obfs

随后即可生成`shadowsocks_3.0.5-0_amd64.deb `，在目标机器上直接执行`dpkg -i shadowsocks_3.0.5-0_amd64.deb`一键部署完成shadowsocks。
