## kubectl 命令演练

### kubectl api-versions

打印出服务端支持group/version

```bash
[root@k8s-master ~]# kubectl api-versions
admissionregistration.k8s.io/v1
admissionregistration.k8s.io/v1beta1
// ...
apps/v1
batch/v1
v1
```

也就是编排文件中`apiVersion`字段。

```yaml
apiVersion: apps/v1
kind: Deployment
```