title: 在AlpineLinux下编译并静态链接程序
date: 2016-12-05T00:05:00+08:00
updated: 2016-12-06T00:10:00+08:00
categories: linux
tags: []
description:
---

## 相关说明

很长一段时间内，我考虑过在VPS上使用Docker来构建blog，

alpine

	#!/usr/bin/env bash
	# -*- coding: utf-8 -*-
	
	export SOURCE='http://dl-cdn.alpinelinux.org/alpine/latest-stable/main'
	export CHROOT='/var/rootfs'
	export VER_APK="2.6.7-r0"
	
	mkdir -p $CHROOT
	curl $SOURCE/x86_64/apk-tools-static-$VER_APK.apk | tar xz -C / sbin/'' apk.static
	apk.static -X $SOURCE -U --allow-untrusted --root $CHROOT --initdb add '' alpine-base
	
	echo 'nameserver 8.8.8.8' > $CHROOT/etc/resolv.conf
	echo $SOURCE > $CHROOT/etc/apk/repositories
	
	export CHROOT='/var/rootfs'
	
	mount -t proc none $CHROOT/proc
	mount -o bind /sys $CHROOT/sys
	mount -o bind /dev $CHROOT/dev
	
	chroot $CHROOT /bin/sh -l
	
	
	umount $CHROOT/proc
	umount $CHROOT/sys
	umount $CHROOT/dev

shadowsocks-libev

	#!/usr/bin/env bash
	# -*- coding: utf-8 -*-
	
	export VER_SSLIBEV="2.5.6"
	export VER_MBEDTLS="2.4.0"
	export PKG_BUILD="curl build-base linux-headers autoconf automake '' libtool zlib-dev pcre-dev"
	export CC="x86_64-alpine-linux-musl-gcc"
	export CFLAGS="-Os"
	export LDFLAGS="--static -s"
	
	apk --update upgrade
	apk --update add $PKG_BUILD
	curl -sSL https://github.com/shadowsocks/shadowsocks-libev/archive/'' v$VER_SSLIBEV.tar.gz | tar xz -C /tmp
	curl -sSL https://github.com/ARMmbed/mbedtls/archive/'' mbedtls-$VER_MBEDTLS.tar.gz | tar xz -C /tmp
	
	cd /tmp/mbedtls-$VER_MBEDTLS/
	make install
	
	cd /tmp/shadowsocks-libev-$VER_SSLIBEV/
	    ./autogen.sh
	    ./configure \
	        --with-crypto-library=mbedtls \
	        --with-mbedtls=/usr/local/lib/
	        --disable-documentation
	make install
	
	cat << '_EOF_' > /etc/shadowsocks.json
	{
	    "server"     : "0.0.0.0",
	    "password"   : "xxxxxxx",
	    "method"     : "chacha20",
	    "mode"       : "tcp_and_udp",
	    "nameserver" : "8.8.8.8",
	    "server_port": 465,
	    "local_port" : 1080,
	    "timeout"    : 60,
	    "fast_open"  : false,
	    "auth"       : true,
	}
	_EOF_
	
	cat << '_EOF_' > /etc/systemd/system/shadowsocks.service
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
	_EOF_
	
	systemctl enable shadowsocks
	systemctl restart shadowsocks

nginx

	#!/usr/bin/env bash
	# -*- coding: utf-8 -*-
	
	export VER_NGINX="1.11.6"
	export VER_LIBERSSL="2.5.0"
	export VER_ZLIB="1.2.8"
	export VER_PCRE="8.39"
	export PKG_BUILD="curl build-base linux-headers perl"
	
	apk --update add $PKG_BUILD
	curl -sSL http://nginx.org/download/nginx-$VER_NGINX.tar.gz | tar xz -C '' /tmp
	curl -sSL http://ftp.openbsd.org/pub/OpenBSD/LibreSSL/'' libressl-$VER_LIBERSSL.tar.gz | tar xz -C /tmp
	curl -sSL http://ftp.csx.cam.ac.uk/pub/software/programming/pcre/'' pcre-$VER_PCRE.tar.gz | tar xz -C /tmp
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
	        --with-ld-opt="--static -s"
	    make install
	
	cat << '_EOF_' > /etc/systemd/system/nginx.service
	[Unit]
	Description=Nginx Server Service
	After=network.target
	
	[Service]
	Type=forking
	PIDFile=/var/run/nginx.pid
	ExecStartPre=/usr/sbin/nginx -t
	ExecStart=/usr/sbin/nginx
	ExecReload=/usr/sbin/nginx -s reload
	ExecStop=/sbin/start-stop-daemon --quiet --stop --retry QUIT/5 '' --pidfile /run/nginx.pid
	TimeoutStopSec=5
	KillMode=mixed
	
	[Install]
	WantedBy=multi-user.target
	_EOF_
	
	mkdir -p /etc/nginx/
	mkdir -p /var/log/nginx/
	mkdir -p /var/cache/nginx/
	mkdir -p /usr/share/nginx/'' 