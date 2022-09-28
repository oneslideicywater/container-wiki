## PID 命名空间

## 什么是 process IDs?

在Linux中进程创建时会得到一个PID唯一标识，即使两个进程的启动命令一样，照样可以通过PID分辨两个进程。

进程的信息存储在一个特殊的文件系统-`procfs`, 如果查看/proc路径，会发现所有系统下运行的进程。


## PID命名空间



PID namespace的主要作用是进程隔离，引用man page的话说：

> PID namespace隔离了PID数字空间，即处于不同PID命名空间下的进程可以有相同的PID.

这个功能猛一看似乎没多大用，处于同一Linux系统下的进程PID当然不会冲突, 因为Linux维护了一份可用PID列表。但是在容器迁移场景下却大有用处。

PID命名空间允许用户停止/恢复容器内的进程组，在将容器迁移到其他机器时，容器内进程组的每个进程还可以保持原来的PID.


另外，PID命名空间也需要一个PID=1的init进程来维持存活，如果init进程挂掉，内核会发送SIGKILL信号到PID命名空间下所有进程，随后关闭命名空间。


## PID命名空间测试

```bash
# 创建pid namespace, /bin/bash作为unshare fork出的子进程加入pid namespace
[user@localhost ~] sudo unshare -fp /bin/bash

# pid namespace下执行一条新的命令
[root@localhost ~] sleep 90000 &

# 还可以看见其他进程
[root@localhost ~] ps -ef 
[truncated ]
.....

UID          PID    PPID  C STIME TTY          TIME CMD
root       11650   11634  0 09:17 pts/0    00:00:00 sudo unshare -fp /bin/bash
root       11654   11650  0 09:17 pts/0    00:00:00 unshare -fp /bin/bash
root       11655   11654  0 09:17 pts/0    00:00:00 /bin/bash
root       11661   11655  0 09:17 pts/0    00:00:00 sleep 8000
root       11671   11655  0 09:17 pts/0    00:00:00 ps -ef
```

你可以发现还可以发现其他进程，因为ps工具不支持命名空间,它只不过读取的是`/proc`，其实你已经在新的命名空间了：

```bash
## 杀掉pid，发现没有这个pid，因为新的命名空间中，sleep进程不是这个pid.
[root@localhost ~] kill -9 11361
bash: kill: (11361) - No such process
```

那如何实现ps仅查看当前命名空间的进程组呢？

ps既然是读取`/proc`路径，我们就需要挂载`/proc`路径到当前命名空间。但是，PID命名空间不是mount类型命名空间，所以还需要创建一个mount命名空间。所幸的是有一个方便的参数`--mount-proc`解决这个问题。

`--mount-proc`参数的man page:
```bash
      --mount-proc[=mountpoint]
              Just  before  running the program, mount the proc filesystem at mountpoint (default is /proc).  This is useful when creating a new pid namespace.  It also
              implies creating a new mount namespace since the /proc mount would otherwise mess up existing programs on the system.  The new proc filesystem is  explic‐
              itly mounted as private (by MS_PRIVATE|MS_REC).
```

```bash
[root@localhost oneslide]# unshare -fp --mount-proc /bin/bash
[root@localhost oneslide]# ps -ef
UID         PID   PPID  C STIME TTY          TIME CMD
root          1      0  1 10:46 pts/3    00:00:00 /bin/bash
root         35      1  0 10:46 pts/3    00:00:00 ps -ef
```