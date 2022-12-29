# Calico-BGP网络分析

## Calico BGP


Calico BGP模式之间Pod通信是要求**二层互通**的。





## Calico BGP模式网络分析

在启用了BGP fullmesh模式的kubernetes集群下，部署2副本的nginx Deployment.


网络结构:

```bash
Node: 172.16.67.138   Pod: 10.8.112.2 
Node: 172.16.67.139   Pod: 10.8.152.200
```

目的是测试不同Node下Pod之间的网络通信。

### Pod `10.8.112.2`

- Pod `10.8.112.2`的网络结构


```bash
root@nginx-deployment-66b6c48dd5-dsllr:/# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
3: eth0@if192: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 26:b4:33:7c:aa:d5 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.8.112.2/32 brd 10.8.112.2 scope global eth0
       valid_lft forever preferred_lft forever
```

路由表:

```bash
root@nginx-deployment-66b6c48dd5-dsllr:/# ip route
default via 169.254.1.1 dev eth0 
169.254.1.1 dev eth0 scope link
```

Calico为容器创建了一个默认路由`169.254.1.1`, `169.254.1.1`这个IP并不存在，而是由ARP代理机制返回其对应的Veth设备的MAC地址。

ARP:

```bash
root@nginx-deployment-66b6c48dd5-dsllr:/# ip neigh
169.254.1.1 dev eth0 lladdr ee:ee:ee:ee:ee:ee STALE
```

- pod所在主机网络结构

主机上Veth设备`cali459253b413d@if3` 和Pod `10.8.112.2`中的网卡`eth0@if192`对应:

> 其中`@ifx`对应的网卡序号. 比如容器中的`eth0@if192`代表其对应veth设备序号为192，也就是`cali459253b413d@if3`

```bash
2: ens192: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:50:56:95:8d:20 brd ff:ff:ff:ff:ff:ff
    inet 172.16.67.138/24 brd 172.16.67.255 scope global noprefixroute ens192
       valid_lft forever preferred_lft forever
    inet6 fe80::250:56ff:fe95:8d20/64 scope link
       valid_lft forever preferred_lft forever
192: cali459253b413d@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 2
    inet6 fe80::ecee:eeff:feee:eeee/64 scope link
       valid_lft forever preferred_lft forever
```

路由表:

```bash
[root@node2 ~]# ip route

# 未分配ip
blackhole 10.8.112.0/26 proto bird

# Pod `10.8.112.2` eth0网卡对应的veth设备对
# 主机会把所有目的IP地址为10.8.112.2的报文都转发到此设备
10.8.112.2 dev cali459253b413d scope link

# 所有10.8.152.192/26网段的报文转发至另一个节点172.16.67.139
10.8.152.192/26 via 172.16.67.139 dev ens192 proto bird
172.16.67.0/24 dev ens192 proto kernel scope link src 172.16.67.138 metric 100
```

那么从Pod `10.8.112.2`中生成的报文的网络路径:

1. 通过默认路由`169.254.1.1`发送到veth peer `cali459253b413d`
2. 查看`172.16.67.138`主机路由表，将IP转发到另一个K8s节点`172.16.67.139`

> 后面通过tcpdump可以观察到:
>  - 源MAC地址为`172.16.67.138` ens192网卡mac, 目的MAC地址为`172.16.67.139` ens192网卡mac
>  - 源IP地址为Pod IP`10.8.112.2`,目的地址为另一个k8s节点上的Pod的IP地址`10.8.152.200`



### Pod `10.8.152.200`

Pod `10.8.152.200`的网卡和路由信息不再记录。只记录其运行节点`172.16.67.139`的IP地址。

- 路由表

```bash
[root@node3 ~]# ip route
default via 172.16.67.1 dev ens192 proto static metric 100
# BGP Peer
10.8.112.0/26 via 172.16.67.138 dev ens192 proto bird

# 未分配的IP地址
blackhole 10.8.152.192/26 proto bird

# 目的地址为`10.8.152.200`,转发到此设备
10.8.152.200 dev cali9e48d9de449 scope link

172.16.67.0/24 dev ens192 proto kernel scope link src 172.16.67.139 metric 100
```

- 网卡信息

```bash
[root@node3 ~]# ip addr
# ens192的mac地址为 00:50:56:95:58:0c
2: ens192: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:50:56:95:58:0c brd ff:ff:ff:ff:ff:ff
    inet 172.16.67.139/24 brd 172.16.67.255 scope global noprefixroute ens192
       valid_lft forever preferred_lft forever
    inet6 fe80::250:56ff:fe95:580c/64 scope link
       valid_lft forever preferred_lft forever
# pod `10.8.152.200` eth0对应的veth设备
434: cali9e48d9de449@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 5
    inet6 fe80::ecee:eeff:feee:eeee/64 scope link
       valid_lft forever preferred_lft forever

```

### tcpdump pod间通信

下面使用tcpdump捕获Pod `10.8.112.2` 访问Pod `10.8.152.200`时的报文:

- Pod `10.8.112.2`中运行

```bash
$ curl 10.8.152.200:80
```

- Pod `10.8.152.200`所在的节点`172.16.67.139`上运行:

```bash
[root@node3 ~]# tcpdump -ne -i ens192 host 10.8.152.200
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on ens192, link-type EN10MB (Ethernet), capture size 262144 bytes
18:07:40.661127 00:50:56:95:8d:20 > 00:50:56:95:58:0c, ethertype IPv4 (0x0800), length 74: 10.8.112.2.48854 > 10.8.152.200.http: Flags [S], seq 3431499015, win 29200, options [mss 1460,sackOK,TS val 2973023234 ecr 0,nop,wscale 7], length 0
18:07:40.661224 00:50:56:95:58:0c > 00:50:56:95:8d:20, ethertype IPv4 (0x0800), length 74: 10.8.152.200.http > 10.8.112.2.48854: Flags [S.], seq 1538769802, ack 3431499016, win 28960, options [mss 1460,sackOK,TS val 2972803051 ecr 2973023234,nop,wscale 7], length 0
```

`00:50:56:95:8d:20 > 00:50:56:95:58:0c`的含义是`172.16.67.138 ens192 mac` => `172.16.67.139 ens192 mac`

`10.8.112.2.48854 > 10.8.152.200.http`函数是源IP地址是Pod IP `10.8.112.2` => 目的IP地址 Pod IP `10.8.152.200`.

可见BGP模式是要求kubernetes节点间二层互通的:


- Pod IP `10.8.112.2`所在的节点`172.16.67.138` 必须知道`172.16.67.139`的MAC地址。

封装源MAC为`172.16.67.138` ens192 mac, 目的MAC地址为`172.16.67.139` ens192 mac

如果二层不互通，`172.16.67.139`的MAC地址将无法找到（将`172.16.67.139/24`想象成`172.16.66.139/24`）。而BGP模式下又没有IPIP那样的封装，源IP地址为Pod IP的报文，路由器是不知道怎么样去转发的。

二层互通的情况下，目的MAC地址为对端Node `172.16.67.139` ens192 mac地址的报文将会直接发送到`172.16.67.139`机器上。

```bash
[root@node2 ~]# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
10.8.152.192    172.16.67.139   255.255.255.192 UG    0      0        0 ens192

[root@node2 ~]# ip neigh
172.16.67.139 dev ens192 lladdr 00:50:56:95:58:0c REACHABLE
```