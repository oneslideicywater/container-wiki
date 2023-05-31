## nerdctl  

虽然 Docker 能干的事情，现在 Containerd 都能干，但 Containerd 还有一个非常明显的缺陷：CLI 不够友好。它无法像 Docker 和 Podman 一样通过一条简单的命令启动一个容器，它的两个 CLI 工具 `ctr` 和 `crictl` 都无法实现这么一件非常简单的需求，而这个需求是大多数人都需要的，我总不能为了在本地测试容器而专门部署一个 Kubernetes 集群吧？


ctr 的设计对人类不太友好，例如缺少以下这些和 Docker 类似的功能：

- `docker run -p <PORT>`
- `docker run --restart=always`
- 通过凭证文件 `~/.docker/config.json` 来拉取镜像
- `docker logs`


为了解决这个痛点，Containerd 官方推出了一个新的 CLI 叫 `nerdctl`。nerdctl 的使用体验和 docker 一样顺滑，例如：

```bash
nerdctl run -d -p 8080:80 --name=nginx --restart=always nginx
```

因为这是一个实战的帖子，所以下面我打算直接记录操作过程。

- 创建一个CentOS 7 虚拟机
- 安装containerd,buildkit等相关软件
- 测试容器运行，镜像构建和迁移

okay, 让我们开始吧！

## 安装containerd

[containerd](https://containerd.io/downloads/)官网下载containerd

按照[Getting started with containerd](https://github.com/containerd/containerd/blob/main/docs/getting-started.md)安装`runc`,`cni`等。

## 安装nerdctl

下载[nerdctl](https://github.com/containerd/nerdctl/releases/)，并放到PATH变量下。

## 安装buildkit

1. 下载[buildkit](https://github.com/moby/buildkit/releases/tag/v0.11.6), 并将全部二进制放到PATH变量下(e.g. `/usr/local/bin`)。


2. 将`examples`下的`buildkit.service`和`buildkit.socket`放在`/usr/lib/systemd/system`路径下。


### systemd配置解析

```bash
[Unit]
Description=BuildKit
Requires=buildkit.socket
After=buildkit.socket
Documentation=https://github.com/moby/buildkit

[Service]
Type=notify
ExecStart=/usr/local/bin/buildkitd --addr fd://

[Install]
WantedBy=multi-user.target
```
[Unit]部分:
- Description: 为这个unit添加描述,内容是BuildKit
- Requires: 此unit依赖于buildkit.socket
- After: 此unit应该在buildkit.socket之后启动
- Documentation: BuildKit的文档链接

[Service]部分:
- Type=notify: 此unit是一个notify类型,它会监听buildkit.socket文件并启动
- ExecStart: 启动BuildKit守护进程buildkitd,并使用--addr fd://

[Install]部分:
- WantedBy: 此unit属于multi-user.target,也就是说应该在系统启动后启动


这里的配置有个优化项：即buildkitd的按需启动，大致流程如下：

1. 安装buildkit.socket和buildkitd.service文件
2. 执行`sudo systemctl enable buildkit.socket`激活socket unit
3. 执行`sudo systemctl start buildkit.socket`启动监听
4. 此时buildkitd.service未启动,socket处于监听状态
5. 客户端连接socket文件描述符3,systemd自动启动buildkitd.service
6. buildkitd启动,接收并处理客户端请求
7. 无客户端连接一段时间,buildkitd自动停止


## 构建镜像

为了进行概念验证，使用go语言打一个最简单的镜像。

创建一个文件夹simple:

- main.go

```go
package main

import "fmt"

func main() {
    fmt.Println("Hello, world!")
}
```

- Dockerfile

```bash
FROM golang:1.16-alpine as builder
WORKDIR /app 
COPY main.go .
RUN go mod init example.com/hello && go build -o main .

FROM alpine:3.13  
WORKDIR /app
COPY --from=builder /app/main .  
CMD ["./main"]
```

构建镜像：

```bash
nerdctl build -t simple .
```

运行：

```bash
# nerdctl run simple
Hello, world!
```

导出：

```bash
# nerdctl load -i simple.tar 
unpacking docker.io/library/simple:latest (sha256:2c361eaee1badb8a7f075d6f70b286346320b4d91fd82f95e2737101db6d704f)...
Loaded image: simple:latest
```
另一台机器导入并运行:

> 这台机器是普通docker机器，可见镜像是通用的。nerdctl打出的镜像可以被docker使用。

```bash
[root@node4 ~]# docker load -i simple.tar
7df5bd7bd262: Loading layer  2.828MB/2.828MB
e630926500df: Loading layer      92B/92B
11e65e8935ee: Loading layer  1.074MB/1.074MB
Loaded image: simple:latest
[root@node4 ~]# docker run simple
Hello, world!
```


### Reference List

1. [终于可以像使用 Docker 一样丝滑地使用 Containerd 了](https://www.kubernetes.org.cn/9126.html)