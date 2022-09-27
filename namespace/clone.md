`clone()`方法创建一个子进程，子进程运行在新的命名空间中。使用`CLONE_*` flag(包括 `CLONE_NEWIPC`, `CLONE_NEWNS`, `CLONE_NEWNET`, `CLONE_NEWPID`, `CLONE_NEWUSER`, `CLONE_NEWUTS`)创建新的命名空间，


`clone()`方法的参数如下：


```c
int clone(int (*child_func)(void *), void *child_stack, int flags, void *arg);
```

- `child_func`: The main function of the program that passes in the child process to run.
- `child_stack`: The stack space used by the incoming child process.
- `flags`: Indicates which `CLONE_*` flags to use.
- `args`: Used to pass in user parameters.



下面这个例子就是父进程创建一个子进程，然后各自输出自己命名空间的主机名。

```c
#define _GNU_SOURCE
#include <sys/wait.h>
#include <sys/utsname.h>
#include <sched.h>
#include <string.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

/* A simple error-handling function: print an error message based
   on the value in 'errno' and terminate the calling process */

#define errExit(msg)    do { perror(msg); exit(EXIT_FAILURE); \
                        } while (0)

static int              /* Start function for cloned child */
childFunc(void *arg)
{
    struct utsname uts;

    /* Modify host name in new UTS namespace */

    if (sethostname(arg, strlen(arg)) == -1)
        errExit("sethostname");

    /* Get and display hostname */

    if (uname(&uts) == -1)
        errExit("uname");
    printf("uts.nodename in child:  %s\n", uts.nodename);

    /* Keep the namespace open for a while, by sleeping.
       This allows some experimentation--for example, another
       process might join the namespace. */
     
    sleep(100);

    return 0;           /* Terminates child */
}

/* Define a stack for clone, stack size 1M */
#define STACK_SIZE (1024 * 1024) 

static char child_stack[STACK_SIZE];

int
main(int argc, char *argv[])
{
    pid_t child_pid;
    struct utsname uts;

    if (argc < 2) {
        fprintf(stderr, "Usage: %s <child-hostname>\n", argv[0]);
        exit(EXIT_FAILURE);
    }

    /* Call the clone function to create a new UTS namespace, where a function comes out and there is also a stack space (why the trailing pointer is because the stack is the opposite);
       The new process will execute in the user-defined function childFunc() */

    child_pid = clone(childFunc, 
                    child_stack + STACK_SIZE,   /* Because the stack is reversed, 
                                                   So the trailing pointer */ 
                    CLONE_NEWUTS | SIGCHLD, argv[1]);
    if (child_pid == -1)
        errExit("clone");
    printf("PID of child created by clone() is %ld\n", (long) child_pid);

    /* Parent falls through to here */

    sleep(1);           /* Allow time for child processes to change hostname */

    /* Displays the host name in the current UTS namespace, and 
       The host name in the UTS namespace where the child process resides is different */

    if (uname(&uts) == -1)
        errExit("uname");
    printf("uts.nodename in parent: %s\n", uts.nodename);

    if (waitpid(child_pid, NULL, 0) == -1)      /* Waiting for child process to end */
        errExit("waitpid");
    printf("child has terminated\n");

    exit(EXIT_SUCCESS);
}
```

运行结果：

```bash
# ./hello domain
PID of child created by clone() is 5723
uts.nodename in child:  domain
uts.nodename in parent: localhost.localdomain
```
