## unshare

unshare将当前进程加入新的命名空间，当前进程detach出当前命名空间。

```c
int unshare(int flags);
```
- flags: `CLONE_*`参数


同时，linux系统里也会有一个对应的unshare命令。

```bash
$ strace unshare -n ip a
execve("/usr/bin/unshare", ["unshare", "-n", "ip", "a"], 
unshare(CLONE_NEWNET)                   = 0
```

## 代码

下面例子使用unshare通过参数创建新的命名空间，并执行一条命令。

```c
/* unshare.c 

   Copyright 2013, Michael Kerrisk
   Licensed under GNU General Public License v2 or later

   A simple implementation of the unshare(1) command: unshare
   namespaces and execute a command.
*/

#define _GNU_SOURCE
#include <sched.h>
#include <unistd.h>
#include <stdlib.h>
#include <stdio.h>

/* A simple error-handling function: print an error message based
   on the value in 'errno' and terminate the calling process */

#define errExit(msg)    do { perror(msg); exit(EXIT_FAILURE); \
                        } while (0)

static void
usage(char *pname)
{
    fprintf(stderr, "Usage: %s [options] program [arg...]\n", pname);
    fprintf(stderr, "Options can be:\n");
    fprintf(stderr, "    -i   unshare IPC namespace\n");
    fprintf(stderr, "    -m   unshare mount namespace\n");
    fprintf(stderr, "    -n   unshare network namespace\n");
    fprintf(stderr, "    -p   unshare PID namespace\n");
    fprintf(stderr, "    -u   unshare UTS namespace\n");
    fprintf(stderr, "    -U   unshare user namespace\n");
    exit(EXIT_FAILURE);
}

int
main(int argc, char *argv[])
{
    int flags, opt;

    flags = 0;

    while ((opt = getopt(argc, argv, "imnpuU")) != -1) {
        switch (opt) {
        case 'i': flags |= CLONE_NEWIPC;        break;
        case 'm': flags |= CLONE_NEWNS;         break;
        case 'n': flags |= CLONE_NEWNET;        break;
        case 'p': flags |= CLONE_NEWPID;        break;
        case 'u': flags |= CLONE_NEWUTS;        break;
        case 'U': flags |= CLONE_NEWUSER;       break;
        default:  usage(argv[0]);
        }
    }

    if (optind >= argc)
        usage(argv[0]);

    if (unshare(flags) == -1)
        errExit("unshare");

    execvp(argv[optind], &argv[optind]);  
    errExit("execvp");
}
```

运行如下命令:

```bash
# 创建网络 & PID命名空间
$ ./unshare -np ps 
   PID TTY          TIME CMD
 14970 pts/2    00:00:00 bash
 16833 pts/2    00:00:00 ps
```