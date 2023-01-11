# Kubernetes network namespace位置在哪里？

## linux network namespace

通常情况下`/var/run/netns`存放network namespace，但是docker没有将数据同步到linux runtime.

因此`ip netns list`看不到docker容器的network namespace. 

### 如何手动同步至liunx runtime呢？

```bash
$ docker run --rm -it busybox /bin/sh
$ docker ps 
CONTAINER ID   IMAGE     COMMAND     CREATED       STATUS       PORTS     NAMES
6e0f6fb84c09   busybox   "/bin/sh"   2 hours ago   Up 2 hours             cranky_margulis
$ docker inspect 6e0
[
    {
        "Id": "6e0f6fb84c09e02a1b3ad5a17cfc5b26136f1f6e46fdfcbe05b4c0ff36d90410",
        "Created": "2022-12-23T01:49:43.823821345Z",
        "Path": "/bin/sh",
        "Args": [],
        "State": {
            "Status": "running",
            # 容器在os上的pid
            "Pid": 49155,
        },
```

下面接下来PID获取其network namespace的位置，通过软连接链接到`/var/run/netns`, `ip netns list`就可以获取到了:

```bash
# 通过pid 获取network namespace
$ ls /proc/49155/ns/net
/proc/49155/ns/net
$ ls -al /proc/49155/ns/net
lrwxrwxrwx. 1 root root 0 Dec 23 09:49 /proc/49155/ns/net -> net:[4026532576]

# 同步至/var/run/netns
$ ln -s /proc/49155/ns/net /var/run/netns/box

# 查看容器的网络设备
$ ip netns exec box ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
9: eth0@if10: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
       
# `ip netns list`就可以获取到了       
$ ip netns list
box (id: 0)
test
```


## /run/docker/netns

`/run/docker/netns`是docker存放网络命名空间的文件系统位置，类型为`tempfs`.

kubernetes Pod中的pause容器的network namepace被pod中其他容器共享。

```bash
$ docker ps
38695ecd7d1b   registry.aliyuncs.com/google_containers/pause:3.4.1   "/pause"                 24 hours ago   Up 24 hours             k8s_POD_coredns-59d64cd4d4-z5m8w_kube-system_319c5a05-4fc1-4dc3-9b7d-7652b083f4b4_0
```
当然容器ID并非network namepace的名字，而是SandBox:

```bash
$ docker inspect 38695ecd7d1b
[
    {
        "Id": "38695ecd7d1b4e1bbf40d7712b45a983037345b3cd57dd9b132cf80fda5d45ad",
        # ...
        "NetworkSettings": {
            "Bridge": "",
            # network namepsace 
            "SandboxID": "bf34b01e754603d472844af7ca53943f0f70b0cac23e2bde50981e2095d9e791",
            "HairpinMode": false,
            # ...
        }
    }
]

```

查看`/run/docker/netns/`，可以发现`SandboxID`对应的network namespace.

```bash
[root@node1 ~]# ls /run/docker/netns/ |grep bf34b01e75
bf34b01e7546
```


下面这个程序可以验证这一点，打印出此network namespace的网络设备(通过`ip a`).

```go
package main

import (
	"fmt"
	"github.com/containernetworking/plugins/pkg/ns"
	"log"
	"os/exec"
)

func main() {
	containerNs,err:=ns.GetNS("/run/docker/netns/1eaefed87449")
	if err != nil {
		log.Fatal(err.Error())
	}
	err = containerNs.Do(func(hostNS ns.NetNS) error {
		cmd:=exec.Command("/bin/sh","-c","ip a")
		outputBytes,err:=cmd.CombinedOutput()
		if err != nil {
			return err
		}
		fmt.Println(string(outputBytes))
		fmt.Println("-------------------host namespace---------------")
		hostNS.Do(func(ns.NetNS) error {
			cmd:=exec.Command("/bin/sh","-c","ip a")
			outputBytes,err:=cmd.CombinedOutput()
			if err != nil {
				return err
			}
			fmt.Println(string(outputBytes))
			return nil
		})
		return nil
	})
	if err != nil {
		log.Fatal(err.Error())
	}
}
```

输出：

```bash
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
inet 127.0.0.1/8 scope host lo
valid_lft forever preferred_lft forever
9: eth0@if10: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
valid_lft forever preferred_lft forever

-------------------host namespace---------------
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
inet 127.0.0.1/8 scope host lo
valid_lft forever preferred_lft forever
inet6 ::1/128 scope host
valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
link/ether 00:0c:29:a2:5f:57 brd ff:ff:ff:ff:ff:ff
# ...
```

## golang版 `ip netns list`

那基于上面的认识，我们可以写出一个golang版的`ip netns list`,列出节点的所有网络命名空间，并打印出其网卡。

```bash
package main

import (
	"fmt"
	"github.com/containernetworking/plugins/pkg/ns"
	"io/fs"
	"log"
	"os/exec"
	"path/filepath"
	"strings"
)

var NETNS_FS_PATH="/run/docker/netns"

func main() {
	var nslist []string
	err := filepath.Walk(NETNS_FS_PATH, func(path string, info fs.FileInfo, err error) error {
		// skip if dir
		if info.IsDir() {
			return nil
		}


		containerNs,err:=ns.GetNS(path)
		if err != nil {
			log.Fatal(err.Error())
		}
		nsName:=strings.TrimPrefix(path,NETNS_FS_PATH+"/")
		nslist=append(nslist,nsName)
		log.Println(fmt.Sprintf("namespace:%s, path: %s",nsName,path))
		log.Println("network devices:")
		err = containerNs.Do(func(hostNS ns.NetNS) error {
			cmd:=exec.Command("/bin/sh","-c","ip a")
			outputBytes,err:=cmd.CombinedOutput()
			if err != nil {
				return err
			}
			fmt.Println(string(outputBytes))
			return nil
		})
		if err != nil {
			log.Fatal(err.Error())
		}
		return nil
	})
	if err != nil {
		log.Fatal(err.Error())
		return
	}
	fmt.Println("network namespace list:")
	for _, nsName := range nslist {
		fmt.Println(nsName)
	}

}
```

查看输出:

```bash
2022/12/23 11:36:15 namespace:15012fbbbcca, path: /run/docker/netns/15012fbbbcca
2022/12/23 11:36:15 network devices:
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
3: eth0@if20: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether f2:c0:1c:a9:25:d3 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.8.92.194/32 brd 10.8.92.194 scope global eth0
       valid_lft forever preferred_lft forever

# ...


2022/12/23 11:36:15 namespace:default, path: /run/docker/netns/default
2022/12/23 11:36:15 network devices:
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: ens192: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:50:56:95:98:4d brd ff:ff:ff:ff:ff:ff

network namespace list:
15012fbbbcca
3b4f42da2e6b
5608268f8292
5f821446162b
7d87be0df957
bf34b01e7546
default
f22717441520
```

可以发现`default`是宿主机的网络命名空间，其他的是pod pause容器的网络命名空间。

## Reference List

1. [Container Namespaces – Deep Dive into Container Networking](https://platform9.com/blog/container-namespaces-deep-dive-container-networking/)
