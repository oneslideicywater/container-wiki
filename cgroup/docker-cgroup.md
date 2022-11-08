## Docker Cgroup Usage

### Limit Container IO

Docker允许CLI参数指定容器资源限制。

假如想要限制一个容器IO读速率到1MB/s:

```bash
docker run -it --device-read-bps /dev/sda:1mb centos
```

查看Docker容器对应的Cgroup组：

```bash
 $ cat /sys/fs/cgroup/blkio/docker/${container_id}/blkio.throttle.read_bps_device
8:0 1048576 // 2^20=1048576
```


