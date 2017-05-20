title: 内核隧道的介绍与校园网内的应用
date: 2016-03-04T22:00:00+08:00
updated: 2016-12-01T00:00:00+08:00
categories: network
tags: [kernel, tunnel, debian]
description: 本文介绍了使用IP隧道、SIT隧道、GRE隧道的命令与操作，并给出了校园网内的实例
---

## 相关说明

服务器信息：

- ConoHa 1GB KVM Tokyo
- 2 Core CPU
- 50GB SSD
- Debian 8.0 64bit
- IPv4 addr x1
- IPv6 addr x17

路由器信息：

- EdgeRouter X v1.8
- Dual-Core 880 MHz MIPS1004Kc
- 256 MB DDR3 RAM
- 256 MB NAND

## 隧道基础知识

Linux内核支持的隧道主要有IP隧道、SIT隧道、GRE隧道三种，L2TPv3隧道比较少见，本文将不做介绍。

IP隧道是一种比较简单的隧道，其原理就是直接在原始的IP包之前加入一个新的IP报文头指向对端，将原有的整个IP包作为新的IP包的Payload。对端收到这个IP包之后取出Payload部分进一步处理。

由于存在IPv4和IPv6两种协议，所以IP隧道也就对应地产生了四种类型：

- IPIP隧道：将IPv4包封装在IPv4包中
- SIT隧道：将IPv6包封装在IPv4包中
- IPIP6隧道：将IPv4包封装在IPv6包中
- IP6IP6隧道：将IPv6包封装在IPv6包中

SIT隧道的功能与原理与IP隧道比较接近，这里就姑且把它们归类在一起。GRE隧道可以视为对IP隧道的增强，加入了一些额外的字节以提供GRE Key等高级功能，对隧道内传输的报文没有限制，根据外层使用的包的类型可以分为两类：

- GRE隧道：将IPv4或者IPv6包封装在IPv4包中
- IP6GRE隧道：将IPv4或者IPv6包封装在IPv6包中

建立这些隧道需要你拥有能够互相直接Ping通的IP地址，运营商做了NAT，没有公网IP的用户就没有办法建立一个通往VPS的隧道了。由于这些隧道与IP地址是绑定的，所以通过PPPOE上网，没有固定IP的用户会比较麻烦，需要写脚本定时通知远端本次获得的IP是什么并创建隧道等等。

## 创建一个隧道

创建一个用IPv4进行封装的隧道需要使用如下命令：

    $ ip tunnel add tun0 mode 隧道类型 remote 远端IPv4地址 local 本地IPv4地址

比如：

    $ ip tunnel add tun0 mode ipip remote 1.2.3.4 local 6.7.8.9
    $ ip tunnel add tun0 mode sit remote 1.2.3.4 local 6.7.8.9
    $ ip tunnel add tun0 mode gre remote 1.2.3.4 local 6.7.8.9

创建一个用IPv6进行封装的隧道需要使用如下命令：

    $ ip -6 tunnel add tun0 mode 隧道类型 remote 远端IPv6地址 local 本地IPv6地址

比如：

    $ ip tunnel add tun0 mode ipip6 remote 2001::1 local 2400::1
    $ ip tunnel add tun0 mode ip6ip6 remote 2001::1 local 2400::1
    $ ip tunnel add tun0 mode ip6gre remote remote 2001::1 local 2400::1

创建隧道的命令需要在本地和远端分别执行，远端执行时注意要交换IP地址。

## 通过隧道互访

创建隧道接口tun0之后，我们需要为其分配IP地址，需要按照隧道类型分配对应类型的IP。

实际上隧道不存在服务器和客户端的概念，为了方便讲解，我们约定远端的机器采取尾数为1的IP地址，本地的机器采取尾数为2的IP地址。

为内层是IPv4包的ipip隧道、ipip6隧道分配IPv4地址可以在本地执行：

    $ ip addr add 10.24.0.2 peer 10.24.0.1 dev tun0

在远端执行同样的命令，注意交换IP地址：

    $ ip addr add 10.24.0.1 peer 10.24.0.2 dev tun0

内层IPv4使用peer选项可以节省添加路由表的步骤，而内层IPv6需要手动添加路由表。

为内层是IPv6包的sit隧道、ip6ip6隧道分配IPv6地址可以在本地执行：

    $ ip -6 addr add fe80:abcd::2/128 dev tun0
    $ ip -6 route add fe80:abcd::1/128 via fe80:abcd::1/128

在远端执行同样的命令，注意交换IP地址：

    $ ip -6 addr add fe80:abcd::1/128 dev tun0
    $ ip -6 route add fe80:abcd::2/128 via fe80:abcd::1/128

## 正式建立连接

使用如下命令即可正式建立连接，所有隧道通用：

    $ ip link set tun0 up

执行到这一步的时候，你应该可以从本地Ping或者Ping6远端了。

内层是IPv4包的ipip隧道、ipip6隧道可以使用：

    $ ping 10.24.0.1

内层是IPv6包的sit隧道、ip6ip6隧道可以使用：

    $ ping6 fe80:abcd::1

## PKU校内网络实例

为了搭建一个无需支付国际网关费用即可访问所有网站的网络环境，可以使用IPIP6隧道，利用学校提供的IPv6网络承载IPv4流量。

PKU校内`静态IPv6地址`规则如下：

 - `静态IPv6地址`=`校园网地址:本网段地址::本机地址`
    - `校园网地址`为`2001:da8:201`
    - 如果本机的IPv4地址为`162.105`开头，则`本网段地址`为IPv4地址的第3组数字加1000，如本机IPv4地址为`162.105.10.100`，则`本网段地址`为1010。
    - 如果本机的IPv4地址为`222.29`开头，则`本网段地址`为IPv4地址的第3组数字加1300，如本机IPv4地址为`222.29.10.100`，则`本网段地址`为1310。
    - `本机地址`为任意1-4位十六进制数，但不得为1。
- `IPv6网关地址`=`校园网地址:本网段地址::1`

某楼的通过网线直连可以获得的IPv4地址为`162.105.105.xx`，根据上述规则，该楼的用户可以随意选择`2001:da8:201:1105::`开头的任意的某个IP作为自己的静态IPv6的IP，注意不要与其他用户已有的IP冲突。而IPv6网关地址则是`2001:da8:201:1105::1`

在本案例中，我选择`2001:da8:201:1105::2048`作为路由器的静态IPv6地址，eth0插外网网线，eht1-4组成switch0，IP段192.168.1.1/24。另外假设VPS的IPv6地址是`2400:8500:1301::99 `，

为路由器外网网口添加静态IPv6地址可以使用命令：

    $ ip -6 addr add 2001:da8:201:1105::2048/64 dev eth0
    $ ip -6 route add ::/0 via 2001:da8:201:1105::1

在路由器上创建IPIP6隧道并绑定IP：

    $ ip -6 tunnel add tun0 mode ipip6 remote 2400:8500:1301::99 local 2001:da8:201:1105::2048
    $ ip addr add 10.24.0.2 peer 10.24.0.1 dev tun0
    $ ip link set tun0 up

在VPS上创建IPIP6隧道并绑定IP：

    $ ip -6 tunnel add tun0 mode ipip6 remote 2001:da8:201:1105::2048 local 2400:8500:1301::99
    $ ip addr add 10.24.0.1 peer 10.24.0.2 dev tun0
    $ ip link set tun0 up

此时`10.24.0.1`与`10.24.0.2`直接已经可以互相ping通了。

接下来在路由器上添加路由规则，使路由器上的所有流量从IPIP6隧道出去：

    $ ip route replace 0.0.0.0/0 via 10.24.0.1

在VPS上添加路由规则，使路由器上的其他IP段在192.168.1.1/24的设备的数据包的回复通过IPIP6隧道回到路由器上：

    $ ip route add 192.168.1.0/24 via 10.24.0.2

最后在VPS上修复mss、mtu等问题并设置NAT：

    $ iptables -A FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS  --clamp-mss-to-pmtu
    $ iptables -t nat -A POSTROUTING -o eth0 -s 10.24.0.2 -j MASQUERADE
    $ iptables -t nat -A POSTROUTING -o eth0 -s 192.168.1.0/24 -j MASQUERADE

至此，无需支付国际网关费用即可自由访问全部网站的网络环境就配置好了。

最后是网络结构拓扑图：

    2001:da8:201:1105::2048      2400:8500:1301::99
            ┌────────┐                      ┌─────┐
            │ Router ◆──────────────────────◆ VPS │
            └────◆───┘10.24.0.2    10.24.0.1└─────┘
     192.168.1.1 │
                 │
     192.168.1.2 │
            ┌────◆────┐
            │ Macbook │
            └─────────┘

在路由器上执行的全部命令：

    $ ip -6 addr add 2001:da8:201:1105::2048/64 dev eth0
    $ ip -6 route add ::/0 via 2001:da8:201:1105::1
    $ ip -6 tunnel add tun0 mode ipip6 remote 2400:8500:1301::99 local 2001:da8:201:1105::2048
    $ ip addr add 10.24.0.2 peer 10.24.0.1 dev tun0
    $ ip link set tun0 up
    $ ip route replace 0/0 via 10.24.0.1

在VPS上执行的全部命令：

    $ ip -6 tunnel add tun0 mode ipip6 remote 2001:da8:201:1105::2048 local 2400:8500:1301::99
    $ ip addr add 10.24.0.1 peer 10.24.0.2 dev tun0
    $ ip link set tun0 up
    $ ip route add 192.168.1.0/24 via 10.24.0.2
    $ iptables -A FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS  --clamp-mss-to-pmtu
    $ iptables -t nat -A POSTROUTING -o eth0 -s 10.24.0.2 -j MASQUERADE
    $ iptables -t nat -A POSTROUTING -o eth0 -s 192.168.1.0/24 -j MASQUERADE
