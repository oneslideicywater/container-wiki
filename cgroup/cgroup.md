#  Cgroup Overview
<!-- TOC -->

- [Cgroup Overview](#cgroup-overview)
  - [概念](#概念)
    - [important](#important)
  - [查看子系统种类](#查看子系统种类)
    - [解释](#解释)
    - [Code](#code)
  - [kubernetes](#kubernetes)
  - [docker](#docker)
  - [Cgroup子系统下的文件](#cgroup子系统下的文件)
    - [tasks](#tasks)

<!-- /TOC -->
## 概念


cgroup(control groups) 是一个linux内核特性。用户可以使用cgroup控制分配给进程组的系统资源（cpu时间片，内存 etc.）. 使用cgroup，管理员可以更加细粒度的管理资源的分配和优先级，监控资源的使用情况。硬件资源更加智能的根据应用或用户来分割，提高整体使用效率。



### important

- 控制组 (Control Group):

一个控制组包含多个进程，而资源的限制也是定义在控制组上的。若一个进程加入到某一个控制组，则自动会受到定义在这个控制组上面的限制规则的影响。

- 层级(Layer):
  
一个子系统下面的控制组，可以进行嵌套，最终形成一个树形的结构。子节点控制组会继承父节点控制组上对于资源的限制规则。若在子节点的控制组重定义了和父节点中相同资源的规则，则会发生覆盖（子覆盖父）

- 子系统(subsystem):
 
在 CGroup 中，有很多子系统。一个子系统就代表一个资源控制器。`sys/fs/cgroup` 目录下的项目就是目前操作系统提供的全部子系统。



## 查看子系统种类

查看cgroup的子系统种类：

### 解释

column | description
-----| ---------
subsys_name| 表示subsystem名
hierarchy|  表示关联到的cgroup树的ID，如果多个subsystem关联到同一颗cgroup树，那么它们的这个字段将一样。比如图中的cpuset、cpu和cpuacct。
num_cgroups | 表示subsystem所关联的cgroup树中进程组的个数，即树上节点的个数。

### Code

```bash
[root@node1 ~]# cat /proc/cgroups
#subsys_name    hierarchy       num_cgroups     enabled
cpuset  5       44      1
cpu     4       187     1
cpuacct 4       187     1
memory  2       187     1
devices 11      188     1
freezer 3       44      1
net_cls 8       44      1
blkio   6       187     1
perf_event      9       44      1
hugetlb 7       44      1
pids    10      188     1
net_prio        8       44      1
```


cgroup在Linux中表现为一个文件系统,位于路径`/sys/fs/cgroup`:


```bash
[root@node1 ~]# mount -t cgroup
cgroup on /sys/fs/cgroup/systemd type cgroup (rw,nosuid,nodev,noexec,relatime,xattr,release_agent=/usr/lib/systemd/systemd-cgroups-agent,name=systemd)
cgroup on /sys/fs/cgroup/memory type cgroup (rw,nosuid,nodev,noexec,relatime,memory)
cgroup on /sys/fs/cgroup/freezer type cgroup (rw,nosuid,nodev,noexec,relatime,freezer)
cgroup on /sys/fs/cgroup/cpu,cpuacct type cgroup (rw,nosuid,nodev,noexec,relatime,cpuacct,cpu)
cgroup on /sys/fs/cgroup/cpuset type cgroup (rw,nosuid,nodev,noexec,relatime,cpuset)
cgroup on /sys/fs/cgroup/blkio type cgroup (rw,nosuid,nodev,noexec,relatime,blkio)
cgroup on /sys/fs/cgroup/hugetlb type cgroup (rw,nosuid,nodev,noexec,relatime,hugetlb)
cgroup on /sys/fs/cgroup/net_cls,net_prio type cgroup (rw,nosuid,nodev,noexec,relatime,net_prio,net_cls)
cgroup on /sys/fs/cgroup/perf_event type cgroup (rw,nosuid,nodev,noexec,relatime,perf_event)
cgroup on /sys/fs/cgroup/pids type cgroup (rw,nosuid,nodev,noexec,relatime,pids)
cgroup on /sys/fs/cgroup/devices type cgroup (rw,nosuid,nodev,noexec,relatime,devices)
```

默认的cgroup子系统列表:

subsystem | description
------|-------
blkio | sets limits on input/output access to and from block devices;
cpu | uses the CPU scheduler to provide cgroup tasks access to the CPU. It is mounted together with the cpuacct controller on the same mount;
cpuacct | creates automatic reports on CPU resources used by tasks in a cgroup. It is mounted together with the cpu controller on the same mount;
cpuset | assigns individual CPUs (on a multicore system) and memory nodes to tasks in a cgroup;
devices | allows or denies access to devices for tasks in a cgroup;
freezer | suspends or resumes tasks in a cgroup;
memory | sets limits on memory use by tasks in a cgroup and generates automatic reports on memory resources used by those tasks;
net_cls | tags network packets with a class identifier (classid) that allows the Linux traffic controller (the `tc` command) to identify packets originating from a particular cgroup task. A subsystem of `net_cls`, the `net_filter` (iptables) can also use this tag to perform actions on such packets. The `net_filter` tags network sockets with a firewall identifier (fwid) that allows the Linux firewall (the `iptables` command) to identify packets (skb->sk) originating from a particular cgroup task;
perf_event | enables monitoring cgroups with the perf tool;
hugetlb | allows to use virtual memory pages of large sizes and to enforce resource limits on these pages.


## kubernetes

kubernetes使用了给每个pod，pod里的每个容器设置了单独的命名空间。


```bash
[root@node1 ~]# ls /sys/fs/cgroup/cpu/kubepods/
besteffort burstable ...


[root@node1 ~]# ll /sys/fs/cgroup/cpu/kubepods/besteffort/
drwxr-xr-x 4 root root 0 10月 21 18:21 pod386ca65c-99ce-46c4-90aa-84f2f3631438
drwxr-xr-x 4 root root 0 10月 21 18:21 pod5677f971-4211-484f-8a60-d7a74a9d3b98
// ...
-rw-r--r-- 1 root root 0 10月  8 10:20 tasks
```

`/sys/fs/cgroup/cpu/kubepods/besteffort/`下`pod*`开头的目录，即是kubernetes给每个Pod设置的子控制组。


继续深入可以看到pod里运行容器对应的子控制组。

```bash
[root@node1 ~]# ll /sys/fs/cgroup/cpu/kubepods/besteffort/pod386ca65c-99ce-46c4-90aa-84f2f3631438/
总用量 0
drwxr-xr-x 2 root root 0 10月 21 18:22 75c00b62936300d6c2d744cea187abfcc531630f3ae038d5226bc34052cce08e
drwxr-xr-x 2 root root 0 10月 21 18:21 7c9442d204f9deb5ff2b27891ccc979de650e433b710867c070a4b274a69e5b9
// ...
```

查看pod容器和目录间的对应关系:

```bash
[root@node1 ~]# docker ps |grep 386c
75c00b629363  kuboard/kuboard-agent             "/start-kuboard-agen…"   13 days ago   Up 13 days             k8s_kuboard-agent_kuboard-agent-2-795bcb7fb9-l54pg_kuboard_386ca65c-99ce-46c4-90aa-84f2f3631438_4
7c9442d204f9   google_containers/pause:3.4.1   "/pause"                 13 days ago   Up 13 days             k8s_POD_kuboard-agent-2-795bcb7fb9-l54pg_kuboard_386ca65c-99ce-46c4-90aa-84f2f3631438_1
s
```

```bash
[root@node1 ~]# kubectl get pod kuboard-agent-2-795bcb7fb9-l54pg  -n kuboard -o yaml
apiVersion: v1
kind: Pod
metadata:
  // ...
  uid: 386ca65c-99ce-46c4-90aa-84f2f3631438

```

总结如下:
- `pod*`目录对应kubernetes pod 的`.metadata.uid`，也就是此pod的控制组
- `pod*`目录下子目录对应容器ID, 也就是此pod内容器的子控制组


## docker   

待续


## Cgroup子系统下的文件

`/sys/fs/cgroup/${sub_system}`下除了代表子控制组的目录，还有描述此控制组的一些信息文件。


### tasks

包含此控制组下的全部进程PID.每行一个。PID是属于主机PID namespace的.

