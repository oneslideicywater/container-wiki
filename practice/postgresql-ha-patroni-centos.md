## CentOS 7 patroni 搭建postgresql高可用

<!-- TOC -->

- [CentOS 7 patroni 搭建postgresql高可用](#centos-7-patroni-%E6%90%AD%E5%BB%BApostgresql%E9%AB%98%E5%8F%AF%E7%94%A8)
    - [Install Patroni](#install-patroni)
    - [Install etcd](#install-etcd)
    - [安装HAProxy](#%E5%AE%89%E8%A3%85haproxy)
- [高可用实验](#%E9%AB%98%E5%8F%AF%E7%94%A8%E5%AE%9E%E9%AA%8C)
    - [下载Patroni](#%E4%B8%8B%E8%BD%BDpatroni)
    - [创建用户](#%E5%88%9B%E5%BB%BA%E7%94%A8%E6%88%B7)
    - [启动实例A](#%E5%90%AF%E5%8A%A8%E5%AE%9E%E4%BE%8Ba)
    - [启动实例B](#%E5%90%AF%E5%8A%A8%E5%AE%9E%E4%BE%8Bb)
    - [主节点故障模拟](#%E4%B8%BB%E8%8A%82%E7%82%B9%E6%95%85%E9%9A%9C%E6%A8%A1%E6%8B%9F)
    - [主节点恢复模拟](#%E4%B8%BB%E8%8A%82%E7%82%B9%E6%81%A2%E5%A4%8D%E6%A8%A1%E6%8B%9F)

<!-- /TOC -->
### Install Patroni

1. 安装python等

```bash
#!/bin/bash

yum install python-psycopg2 -y 
yum install epel-release -y 

echo "install python3"
yum install -y python3

echo "install pip"
yum install  python3-pip -y

echo "install dep required by rust compiler" 
yum install gcc python3-devel -y
yum install bzip2-devel -y
yum install  xz-devel  -y 

pip install psycopg2-binary
pip install patroni[etcd]
pip3 install setuptools_rust
```
2. 设置python源

```bash
[root@localhost ~]# cat ~/.pip/pip.conf 
[global]
index-url =http://pypi.douban.com/simple/
[install]
trusted-host =pypi.douban.com
```

3. 安装postgresql 12

```bash
yum install https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm -y
yum install postgresql12 -y
yum install postgresql12-server -y
```


### Install etcd

下载[etcd](https://etcd.io/docs/v3.5/install/)的tar.gz包.

运行etcd:

```bash
$ ./etcd --data-dir=data/etcd --enable-v2=true
```

### 安装HAProxy


下载[HAProxy](https://www.haproxy.org/)的tar.gz包.

```bash
# 这里的3100是指内核版本号，查看uname -r
make TARGET=linux3100
make install PREFIX=/usr/local/haproxy 
```

启动HAProxy:
```bash
[root@localhost patroni-master]# /usr/local/haproxy/sbin/haproxy -f haproxy.cfg 
[WARNING]  (25698) : Server batman/postgresql_127.0.0.1_5432 is UP, reason: Layer7 check passed, code: 200, check duration: 2ms. 1 active and 0 backup servers online. 0 sessions requeued, 0 total in queue
```
## 高可用实验

### 下载Patroni

https://github.com/zalando/patroni.git

### 创建用户

patroni不能使用root用户启动。

因此先创建用户:

```bash
$ useradd patroni
$ chown -R patroni:patroni patroni-master
$ su - patroni
```
> 以下操作在patroni仓库项目根目录执行

### 启动实例A

可以发现实例A启动后变成Leader

```bash
[patroni@localhost patroni-master]$ ./patroni.py postgres0.yml
2022-11-14 05:48:58,818 INFO: Selected new etcd server http://localhost:2379
server promoting
2022-11-14 05:48:59,217 INFO: cleared rewind state after becoming the leader
2022-11-14 05:49:00,254 INFO: no action. I am (postgresql0), the leader with the lock
//...

```

### 启动实例B

实例B启动后变成从节点

```bash
[patroni@localhost patroni-master]$ ./patroni.py postgres1.yml 
2022-11-14 05:49:14,896 INFO: Selected new etcd server http://localhost:2379
2022-11-14 05:49:14,902 INFO: No PostgreSQL configuration items changed, nothing to reload.
2022-11-14 05:49:14,919 WARNING: Postgresql is not running.
2022-11-14 05:49:14,919 INFO: Lock owner: postgresql0; I am postgresql1
2022-11-14 05:49:15,293 INFO: establishing a new patroni connection to the postgres cluster
2022-11-14 05:49:15,313 INFO: no action. I am (postgresql1), a secondary, and following a leader (postgresql0)

```

通过HAProxy连接数据库：

```bash
psql --host 127.0.0.1 --port 5000 postgres
```

### 主节点故障模拟

按Ctrl+C停掉主节点, 查看从节点日志：

```bash
2022-11-14 05:59:52,593 WARNING: Request failed to postgresql0: GET http://127.0.0.1:8008/patroni (HTTPConnectionPool(host='127.0.0.1', port=8008): Max retries exceeded with url: /patroni (Caused by ProtocolError('Connection aborted.', ConnectionResetError(104, 'Connection reset by peer'))))
2022-11-14 05:59:52,685 WARNING: Could not activate Linux watchdog device: "Can't open watchdog device: [Errno 2] No such file or directory: '/dev/watchdog'"
2022-11-14 05:59:52,688 INFO: promoted self to leader by acquiring session lock
server promoting
2022-11-14 05:59:52,693 INFO: cleared rewind state after becoming the leader
2022-11-14 05:59:53,737 INFO: no action. I am (postgresql1), the leader with the lock
2022-11-14 06:00:03,725 INFO: no action. I am (postgresql1), the leader with the lock
```

可以发现从节点晋升为主节点，HAProxy连接数据库还是可以连上。


### 主节点恢复模拟

主节点故障恢复后变成从节点

```bash
[patroni@localhost patroni-master]$ ./patroni.py postgres0.yml
localhost:5432 - accepting connections
2022-11-14 06:01:37,607 INFO: Lock owner: postgresql1; I am postgresql0
2022-11-14 06:01:37,607 INFO: establishing a new patroni connection to the postgres cluster
2022-11-14 06:01:37,626 INFO: no action. I am (postgresql0), a secondary, and following a leader (postgresql1)
```