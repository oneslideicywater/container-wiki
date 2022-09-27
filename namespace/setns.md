## setns

将当前进程加入已有命名空间。

其函数原型为：

```c
int setns(int fd, int nstype);
```

- `fd`: 要加入的命名空间的文件描述符，可以是一个符号链接
- `nstype`: 允许用户检测fd指向的命名空间类型，可以设置为`CLONE_NEW*`常量值,0代表不检查。




## 使用unshare创建命名空间


setns无法创建新的命名空间，所以我们需要在命令行中新建一个网络命名空间用于实验验证：

```bash
# 创建一个网络命名空间
$ unshare -n

# $$指向当前进程，查看当前进程的命名空间
$ ls -al /proc/$$/ns
total 0
dr-x--x--x. 2 root root 0 Sep 27 17:05 .
dr-xr-xr-x. 9 root root 0 Sep 27 17:05 ..
lrwxrwxrwx. 1 root root 0 Sep 27 17:06 ipc -> ipc:[4026531839]
lrwxrwxrwx. 1 root root 0 Sep 27 17:06 mnt -> mnt:[4026531840]
lrwxrwxrwx. 1 root root 0 Sep 27 17:06 net -> net:[4026532504]
lrwxrwxrwx. 1 root root 0 Sep 27 17:06 pid -> pid:[4026531836]
lrwxrwxrwx. 1 root root 0 Sep 27 17:06 user -> user:[4026531837]
lrwxrwxrwx. 1 root root 0 Sep 27 17:06 uts -> uts:[4026531838]

$ echo $$
7851
```



### 代码

下面这个例子，将当前进程加入一个命名空间，并在命名空间中执行一条命令.

```c
/* ns_exec.c 

   Copyright 2013, Michael Kerrisk
   Licensed under GNU General Public License v2 or later

   Join a namespace and execute a command in the namespace
*/
#define _GNU_SOURCE
#include <fcntl.h>
#include <sched.h>
#include <unistd.h>
#include <stdlib.h>
#include <stdio.h>

/* A simple error-handling function: print an error message based
   on the value in 'errno' and terminate the calling process */

#define errExit(msg)    do { perror(msg); exit(EXIT_FAILURE); \
                        } while (0)

int
main(int argc, char *argv[])
{
    int fd;

    if (argc < 3) {
        fprintf(stderr, "%s /proc/PID/ns/FILE cmd [arg...]\n", argv[0]);
        exit(EXIT_FAILURE);
    }

    fd = open(argv[1], O_RDONLY);   /* Get the file descriptor of the namespace you want to join */
    if (fd == -1)
        errExit("open");

    if (setns(fd, 0) == -1)         /* Join the namespace */
        errExit("setns");

    execvp(argv[2], &argv[2]);      /* Execute commands in the joined namespace */
    errExit("execvp");
}
```



编译代码，并运行传入参数：

```bash
$ ./ns_exec /proc/7851/ns/net ip a

1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
```