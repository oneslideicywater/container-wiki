## mount命名空间

### mount命名空间测试

下载alpine系统的文件目录：

```bash
[root@localhost ~] export CONTAINER_ROOT_FOLDER=/container_practice
[root@localhost ~] mkdir -p ${CONTAINER_ROOT_FOLDER}/fakeroot
[root@localhost ~] cd ${CONTAINER_ROOT_FOLDER}
[root@localhost ~] wget https://dl-cdn.alpinelinux.org/alpine/v3.13/releases/x86_64/alpine-minirootfs-3.13.1-x86_64.tar.gz
[root@localhost ~] tar xvf alpine-minirootfs-3.13.1-x86_64.tar.gz -C fakeroot
```

创建一个mount命名空间：

```bash
unshare -m 
```

查看挂载点，发现mount命名空间内和宿主机是一样的。

```bash
$ df -h
Filesystem           Size  Used Avail Use% Mounted on
/dev/mapper/cs-root   36G  5.2G   31G  15% /
tmpfs                737M     0  737M   0% /sys/fs/cgroup
devtmpfs             720M     0  720M   0% /dev
tmpfs                737M     0  737M   0% /dev/shm
tmpfs                737M  8.6M  728M   2% /run
tmpfs                148M     0  148M   0% /run/user/0
/dev/vda1            976M  197M  713M  22% /boot


$ ls /
bin   container_practice  etc   lib    media  opt   root  sbin  sys  usr
boot  dev                 home  lib64  mnt    proc  run   srv   tmp  var
```

产生这种现象的原因是systemd默认递归式分享挂载点给所有命名空间。

如果在命名空间里新挂载一个位置，宿主机是看不到的。

- 命名空间

```bash
$ mount -t tmpfs tmpfs /mnt

$ findmnt |grep mnt
└─/mnt     tmpfs               tmpfs      rw,relatime,seclabel,uid=1000,gid=1000
```

- 宿主机

```bash
$ findmnt |grep mnt
```

另外容器内还需要有个特点，就是根目录。

```bash
# 看似前后位置一样，实际上是namespace里新建了一个挂载点
$ mount --bind ${CONTAINER_ROOT_FOLDER}/fakeroot ${CONTAINER_ROOT_FOLDER}/fakeroot
$ cd ${CONTAINER_ROOT_FOLDER}/fakeroot
```

改变根目录：

```bash
$ mkdir old_root
$ pivot_root . old_root
$ export PATH=/bin:/sbin:$PATH
$ umount -l /old_root
```
现在再去查看根目录，发现和宿主机根目录不一样了。

```bash
root@new-mnt$ ls -l /
total 12
drwxr-xr-x    2 root     root          4096 Jan 28 21:51 bin
drwxr-xr-x    2 root     root            18 Feb 17 22:53 dev
drwxr-xr-x   15 root     root          4096 Jan 28 21:51 etc
drwxr-xr-x    2 root     root             6 Jan 28 21:51 home
drwxr-xr-x    7 root     root           247 Jan 28 21:51 lib
drwxr-xr-x    5 root     root            44 Jan 28 21:51 media
drwxr-xr-x    2 root     root             6 Jan 28 21:51 mnt
drwxrwxr-x    2 root     root             6 Feb 17 23:09 old_root
drwxr-xr-x    2 root     root             6 Jan 28 21:51 opt
drwxr-xr-x    2 root     root             6 Jan 28 21:51 proc
drwxr-xr-x    2 root     root             6 Feb 17 22:53 put_old
drwx------    2 root     root            27 Feb 17 22:53 root
drwxr-xr-x    2 root     root             6 Jan 28 21:51 run
drwxr-xr-x    2 root     root          4096 Jan 28 21:51 sbin
drwxr-xr-x    2 root     root             6 Jan 28 21:51 srv
drwxr-xr-x    2 root     root             6 Jan 28 21:51 sys
drwxrwxrwt    2 root     root             6 Feb 19 16:38 tmp
drwxr-xr-x    7 root     root            66 Jan 28 21:51 usr
drwxr-xr-x   12 root     root           137 Jan 28 21:51 var
```


### pivot_root和chroot的区别

The main difference is that *pivot_root* is intended to switch the complete system over to a new root directory and remove dependencies on the old one, so that you would be able to unmount the original root directory and proceed as if it had never been in use.



*chroot* is intended to apply for
the lifetime of a single process, with the rest of the system continuing to run in the old root dir, and the system being unchanged when the chrooted process exits.

The difference is shrinking now that Linux has per-process namespaces anyway. If you "chroot" a single init-type process that never exits and don't care about unmounting the original root, it's the same for practical purposes.
