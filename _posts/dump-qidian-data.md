title: 提取起点上的小说月票榜信息
date: 2015-03-06 01:30:00
categories: network
tags: [json, cdn, dig]
description: 本文介绍了如何寻找开放的AJAX接口，并使用命令行脚本批量下载数据的方法。
---

## 早期思路

最近在研究月票榜的时间序列数据，试图找出用统计学方法来鉴别刷票这种行为的方法，首先需要解决的就是如何提取数据。早在13年年底，我写过一个python脚本来批量读取小说信息，方法很简单，利用requests读取数据，用re库提取数据。

这种方法很快遇到了问题，起点全站使用了蓝讯的CDN服务，这个服务似乎能够检测出我批量读取数据的行为，在持续读取的过程中容易遇到回应一个空白页面的问题。

我猜测这种限制是由蓝讯实行的，那么接下来的问题就是如何绕过了。

思路有两条：使用一个代理服务器池，或者绕过蓝讯直接找到服务器的真实IP。

## 寻找CDN背后的服务器真实IP

首先尝试dig起点主站：

    $ dig www.qidian.com
    ;; ANSWER SECTION:
    www.qidian.com.     128 IN  CNAME   www.qidian.com.ccgslb.com.cn.
    www.qidian.com.ccgslb.com.cn. 3827 IN   CNAME   cc00037.h.cnc.ccgslb.com.cn.
    ...

想绕过CDN找到真实IP，最简单的方法就是再dig一下裸域了：

    $ dig qidian.com
    ;; ANSWER SECTION:
    qidian.com.     225 IN  A   202.102.67.78

202.102.67.78就是我们的目标了，直接访问会被302跳转到www.qidian.com，我们试试改掉header：

    $ curl "http://202.102.67.78/Default.aspx" -H "host: www.qidian.com"

返回了正常的网页源代码。

现在，我们直接访问IP，就能完全绕过蓝讯的限制了。

## 寻找子站的真实IP

不幸的是，我们只找到了起点主站的IP，各个子站的IP并不是这个。我推测子站的IP也在这个IP段内，我写了一个小脚本来处理这个问题：

    for (( x = 255; x > 0; x-- )); do
        URL="http://202.102.58.$x/Default.aspx"
        echo $URL
        echo $URL >> data.log
        curl $URL -H "Host: m.qidian.com" --connect-timeout 5 -s  >> data.log
    done

把脚本丢到VPS上慢慢跑，喝杯茶之后脚本就跑完了，打开data.log搜索关键字，立刻就找到了m.qidian.com的真实IP，即：202.102.67.87。

在这个过程中，我还找到了另外一个有趣的地址：202.102.67.85。

## 寻找网页背后的API

最开始，我是从top.qidian.com来批量读取信息的，由于服务器是直接返回渲染好的网页，我不得不用正则表达式来处理返回的信息。后来我发现了h5.qidian.com，这个网页通过API来获得json数据，并且在本地渲染，如果找到了这个API，我们就可以轻轻松松的处理数据了。

打开浏览器，用F12调出控制台，切换到Network界面，刷新页面，过滤XHR类型，然后就能找到这个关键的API：

    http://h5.qidian.com/book/Top.ashx?ajaxMethod=gettopbooks&TopType=3&PageIndex=1&_=1393749423231

最后一项参数是当前的UnixTimeStamp，不过精确度多了三位。

不过，在我寻找起点子站的真实IP的时候，我发现了一个有趣的地址：202.102.67.85。这个地址背后的网页提供了一个更好用的API：

    http://202.102.67.85/ajax/Top.ashx?ajaxMethod=getmonthpklist&pageindex=1&pagesize=100

这个API可以用pagesize参数精确指定返回的数量，而h5.qidian.com一次只能返回20条数据，效率差了太多。

## 用jq处理json数据

此jq非jquary，这个jq是一个强大的处理json数据的命令行工具，在其[官网](http://stedolan.github.io/jq/)地址可以找到安装说明与用户手册。在Mac与Debian下可以用包管理器来安装：

    $ brew install jq
    $ apt-get install jq

这个工具的具体使用方法我就不多介绍了，参考[用户手册](http://stedolan.github.io/jq/manual/)即可。

我们用一条命令来处理前10页数据：

    $ curl "http://202.102.67.85/ajax/Top.ashx?ajaxMethod=getmonthpklist&pageindex=[1-10]&pagesize=100" -s | jq '.[0][] | "\(.BookName),\(.VoteMonth)"' -r

由于起点服务器自带超时机制，在查询超时的情况下会返回空数据，我不得不写了一个稍微复杂的脚本来处理这个问题：

    #!/usr/bin/env bash
    # -*- coding: utf-8 -*-

    cd $(dirname $0)

    LOG=./data/$(date +%s).log
    ((PID=1))

    while true; do
        RES=$(curl "http://202.102.67.85/ajax/Top.ashx?ajaxMethod=getmonthpklist&pageindex=$PID&pagesize=100" -s | jq '.[0][] | "\(.BookName),\(.VoteMonth)"' -r)
        if [ "$RES" == "" ]; then
            ((ERR++))
            if [ $ERR == 2 ]; then
                break
            fi
        else
            echo "$RES" >> $LOG
            ((ERR=0))
            ((PID++))
        fi
    done

现在，我们就可以轻轻松松读取起点的月票榜数据啦！
