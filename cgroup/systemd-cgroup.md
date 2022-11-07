## CentOS 7 Systemd Cgroup 层级
<!-- TOC -->

- [CentOS 7 Systemd Cgroup 层级](#centos-7-systemd-cgroup-层级)
  - [Systemd单元类型](#systemd单元类型)
  - [默认slice类型](#默认slice类型)
  - [systemd-cgls](#systemd-cgls)
  - [Reference List](#reference-list)

<!-- /TOC -->
Cgroup控制组可以进行嵌套,但是为了能够更加规范的去组织.

Systemd推出了一个规范, 文件系统路径: `/sys/fs/cgroup/systemd`


### Systemd单元类型

Systemd的单元类型(System Unit Type), "单元"的概念对应cgroup里层级, 但是按照其划分逻辑上不同分成下面三种:

Unit Type | Description
-----|------
Service | systemd 读取unit配置文件一起绑定启动/停止的一组进程. 使用`${name}.service`命名方式. 
Scope | 任意进程通过`fork()`调用,并在运行时注册到systemd. 比如用户Session,容器,虚拟机通常是Scope类型.使用`${name}.scope`命名方式.
Slice | 盛放Scope类型和Service类型的盒子,不包含具体进程,具体进程都在其下面的Scope和Service中.命名方式为`${name}.slice`. 
 Slice嵌套 | Slice类型也可以嵌套, 使用`-`作为路径分隔符. 比如`parent-0.slice`的父级对象是`parent.slice`


### 默认slice类型

Slice一般是由系统管理员创建, 当然也可以由程序动态生成.

系统默认也内置了一些Slice:

- `.slice` — the root slice;
- `system.slice` — the default place for all system services;
- `user.slice` — the default place for all user sessions;
- `machine.slice` — the default place for all virtual machines and Linux containers.


### systemd-cgls

systemd-cgls命令可以较友好的方式展示本机的cgroup组织结构.

```bash
[root@node1 ~]# systemd-cgls
├─1 /usr/lib/systemd/systemd --switched-root --system --deserialize 22  # .slice - 根slice
├─kubepods  # kubernetes自定义层级划分, 不存在systemd的单元类型
│ ├─besteffort
│ │ ├─pod386ca65c-99ce-46c4-90aa-84f2f3631438
│ │ │ ├─75c00b62936300d6c2d744cea187abfcc531630f3ae038d5226bc34052cce08e
│ │ │ │ ├─ 7037 /bin/sh
│ │ │ │ ├─25282 nginx: master process nginx -g daemon off
│ │ │ │ └─25293 nginx: worker proces
│ │ │ └─7c9442d204f9deb5ff2b27891ccc979de650e433b710867c070a4b274a69e5b9
│ │ │   └─22623 /pause
├─user.slice   # 用户slice
│ ├─user-0.slice # 用户子slice, 父亲的名字叫user, 孩子的名字叫user-0
│ │ ├─session-5989.scope
│ │ │ ├─26572 sshd: root@notty
│ │ ├─session-5988.scope
│ │ │ ├─ 8042 systemd-cgls
│ └─user-42.slice
│   └─session-c1.scope  # slice下scope, 运行时注册入systemd
│     ├─1906 gdm-session-worker [pam/gdm-launch-environment]
│     ├─2026 /usr/libexec/gnome-session-binary --autostart /usr/share/gdm/greeter/autostart
│     ├─2129 dbus-launch --exit-with-session /usr/libexec/gnome-session-binary --autostart /usr/share/gdm/greeter/autostart
└─system.slice
  ├─docker.service  # slice下的service 
  │ └─21237 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock # service下的进程
  ├─kubelet.service
  │ └─8259 /usr/local/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --
  ├─containerd.service
  │ ├─10815 /usr/bin/containerd
│     ├─3097 /usr/libexec/gsd-screensaver-proxy
│     ├─3100 /usr/libexec/gsd-sharing
│     ├─3105 /usr/libexec/gsd-smartcard
└─system.slice
  ├─docker.service
  │ └─21237 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
  ├─kubelet.service
  │ └─8259 /usr/local/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --
  ├─containerd.service  # containerd.service
  │ ├─10815 /usr/bin/containerd   
  │ ├─21658 /usr/bin/containerd-shim-runc-v2 -namespace moby -id cd4f5f588b0d70a63b9945309135a372f4b5ae94af4dadbd2bf794eb94d24225 -address / # 每个容器进程, 模式是"PID: 启动命令"
  │ ├─21694 /usr/bin/containerd-shim-runc-v2 -namespace moby -id fef8e0c5e2d745dd736f788aba0170bb85cc0e40e293331b194d60599f7d028f -address /
  ├─colord.service
  │ └─3142 /usr/libexec/colord

```

那以`contained.service`为例子看下其文件树结构:

```bash
[root@node1 ~]# ll /sys/fs/cgroup/systemd/system.slice/containerd.service/
-rw-r--r-- 1 root root 0 9月  30 18:13 cgroup.clone_children
--w--w--w- 1 root root 0 9月  30 18:13 cgroup.event_control
-rw-r--r-- 1 root root 0 9月  30 18:13 cgroup.procs
-rw-r--r-- 1 root root 0 9月  30 18:13 notify_on_release
-rw-r--r-- 1 root root 0 9月  30 18:13 tasks

```

### Reference List 

1. [Default Cgroup Hierarchies](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/resource_management_guide/sec-default_cgroup_hierarchies)


