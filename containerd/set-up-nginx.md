## Ctr Setup a nginx Container

这篇文章使用`ctr`创建一个nginx容器，并在宿主机使用`curl`进行访问。



## Prerequisite

按照[Containerd Quickstart](https://github.com/containerd/containerd/blob/main/docs/getting-started.md)完成:

- 安装cni
- 安装runc
- 安装containerd, 并以systemd service方式启动

## Steps

### 1. 拉取镜像:

```bash
ctr images pull docker.io/library/nginx:latest
```

### 2. 编译cnitool:

```bash
git clone https://github.com/containernetworking/cni.git
cd cni
go mod tidy
cd cnitool
GOOS=linux GOARCH=amd64 go build .
```

### 3. 创建容器网络:

```bash
cat << EOF | tee /etc/cni/net.d/nginx.conf
{
    "cniVersion": "0.4.0",
    "name": "nginx",
    "type": "bridge",
    "bridge": "cni0",
    "isDefaultGateway": true,
    "forceAddress": false,
    "ipMasq": true,
    "hairpinMode": true,
    "ipam": {
        "type": "host-local",
        "subnet": "10.88.0.0/16"
    }
}
EOF
```

### 4. 创建容器网络命名空间:

```bash
[root@localhost cni]# ip netns add nginx
[root@localhost cni]# ip netns list
nginx
[root@localhost cni]# ls /var/run/netns/
nginx
```

给network namespace `nginx`添加网络:

```bash
[root@localhost cni]# cnitool add nginx /var/run/netns/nginx 
{
    "cniVersion": "0.4.0",
    "interfaces": [
        {
            "name": "cni0",
            "mac": "3a:55:12:e0:49:7a"
        },
        {
            "name": "veth96cb8bbe",
            "mac": "16:e7:3a:db:5e:43"
        },
        {
            "name": "eth0",
            "mac": "a2:51:45:e3:4c:f0",
            "sandbox": "/var/run/netns/nginx"
        }
    ],
    "ips": [
        {
            "version": "4",
            "interface": 2,
            "address": "10.88.0.2/16", # 分配的ip地址
            "gateway": "10.88.0.1"
        }
    ],
    "routes": [
        {
            "dst": "0.0.0.0/0",
            "gw": "10.88.0.1"
        }
    ],
    "dns": {}
}
[root@localhost cni]#cnitool check nginx /var/run/netns/nginx 
```

检查network namespace `nginx` 的网卡信息:
```bash
[root@localhost cni]# ip -n nginx addr
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eth0@if7: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether a2:51:45:e3:4c:f0 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.88.0.2/16 brd 10.88.255.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::a051:45ff:fee3:4cf0/64 scope link 
       valid_lft forever preferred_lft forever
```

### 5. 启动容器:

```bash
$ ctr run --with-ns=network:/var/run/netns/nginx -d docker.io/library/nginx:latest nginx
```

进入容器:

```bash
[root@localhost cni]# ctr task exec -t --exec-id nginxbash nginx bash 
root@localhost:/# ls                                                  
```

- `--exec-id`: exec specific id for the process 这个名字可以随便起。
- `-t` allocate a TTY for the container

> 容器内curl命令无响应，暂时不知道为什么...


在容器外部访问nginx主页面:

```bash
[root@localhost cni]# curl http://10.88.0.2:80
<!DOCTYPE html>
<!--...-->
<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```


## Clean up

停止容器内Task

```bash
[root@localhost cni]# ctr task list
TASK     PID      STATUS    
nginx    29783    RUNNING
[root@localhost cni]# ctr task kill nginx
[root@localhost cni]# ctr container del nginx
[root@localhost cni]# ctr task list
TASK    PID    STATUS
```

删除network命名空间及cni生成的文件:

```bash
[root@localhost cni]# cnitool del nginx /var/run/netns/nginx
[root@localhost cni]# ip netns del nginx
[root@localhost cni]# rm -rf /var/lib/cni/
```