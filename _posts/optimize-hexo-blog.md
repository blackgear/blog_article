title: Hexo博客的访问速度相关配置优化
date: 2015-05-31 00:41:34
categories: network
tags: [hexo, nginx, ssl]
---

## 总体思路

作为一个搭在廉价VPS上的静态博客，我希望尽可能优化博客的HTTPS访问速度，为此，我在最近一个月对博客进行了一系列改进。

## 配置Gzip

在nginx配置文件中加入：

    gzip on;
    gzip_http_version 1.1;
    gzip_vary on;
    gzip_min_length 256;
    gzip_buffers 16 8k;
    gzip_comp_level 5;
    gzip_proxied any;
    gzip_types text/plain text/css application/json application/javascript application/x-javascript text/javascript text/xml application/xml application/rss+xml application/atom+xml application/rdf+xml;

经过我的研究，国内大部分地区访问本站的平均速度约为0.5kb/s，人类最短反应时间是150ms，这意味着对于仅仅75b的文件大小差距就能造成可以感知的速度差异。

既然我已经拥有生成好了的网页文件，接下来就是具体衡量可以使用的参数，方法很简单，确保这个参数产生的文件大小同最高参数的文件大小相差小于75。

首先测试参数9：

    $ gzip -c9 index.html | wc -c

随后测试参数5：

    $ gzip -c5 index.html | wc -c

经过一番研究，要保证每个文件的压缩后大小同最高参数压缩后的大小相差小于125，至少需要使用参数5。所以在上述gzip配置中需要使用`gzip_comp_level 5`

## SSL配置与优化

本站目前使用了如下的SSL

    ssl on;
    ssl_certificate /etc/nginx/ssl/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/cert.key;
    ssl_session_tickets on;
    ssl_stapling on;
    ssl_stapling_verify on;
    ssl_trusted_certificate /etc/nginx/ssl/ca.pem;
    resolver 8.8.4.4 8.8.8.8 valid=300s;
    resolver_timeout 10s;
    ssl_dhparam /etc/nginx/ssl/dhparam.pem;
    ssl_prefer_server_ciphers on;
    ssl_protocols TLSv1.1 TLSv1.2;
    ssl_ciphers 'ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:kEDH+AESGCM:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-DSS-AES128-SHA256:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA:DHE-RSA-AES256-SHA:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!3DES:!MD5:!PSK';
    ssl_session_timeout 1d;
    ssl_session_cache builtin:1000 shared:SSL:20m;
    ssl_buffer_size 1400;

上面用到的允许使用的算法参数来自与Mozilla的一个[项目](https://mozilla.github.io/server-side-tls/ssl-config-generator/)，由于我的网站本体大量使用HTML5元素与CSS3选择器，所以在SSL算法的浏览器兼容性上直接选择了modern，索性直接避免老浏览器的访问。

以下设置用于启动OCSP stapling的特性：

    ssl_stapling on;
    ssl_stapling_verify on;
    ssl_trusted_certificate /etc/nginx/ssl/ca.pem;
    resolver 8.8.4.4 8.8.8.8 valid=300s;
    resolver_timeout 10s;

值得注意的是ca.pem文件与一般的ssl证书文件合并顺序不同，一般的ssl证书文件要求按照“网站证书-中间证书-CA证书”的顺序合并，OCSP stapling使用ca.pem的顺序是“CA证书-中间证书”，不包含网站证书。

## 缓存优化

由于本站是一个静态博客，大部分文件都可以直接设置一个较长的缓存时间：

    location ~ .*\.(ico|gif|jpg|jpeg|png|bmp|swf|wav)$ {
        expires 30d;
    }

    location ~ .*\.(js|css)$ {
        expires 10d;
    }

    location / {
        expires 1d;
        try_files $uri $uri/ =404;
    }

设置较长的缓存时间后，用户体验有了很大的提高。

## 网页优化

使用[InstantClick](http://instantclick.io)项目，当用户鼠标指向一个链接时就预先加载链接本身，根据测量，从用户鼠标指向一个链接到点击链接，中间有约500ms的时间差，利用这个时间差可以节约访问时间。

尽可能简化网页文件结构，去除class或是id等属性转而使用CSS3选择器，或许能够有用。
