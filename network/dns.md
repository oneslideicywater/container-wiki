# CoreDNS



## Pod DNS in kubernetes

kubernetes会给每个Pod配置DNS,比如我的环境里nginx Pod的默认DNS配置：

```bash
$ kubectl exec -it  nginx-deployment-66b6c48dd5-tg5s6 -- /bin/bash
root@nginx-deployment-66b6c48dd5-tg5s6:/# cat /etc/resolv.conf
nameserver 10.96.0.10  # kube-dns Service ClusterIP 
search rsmis.svc.cluster.local svc.cluster.local cluster.local kubernetes.dev
options ndots:5
```

查看ClusterIP:

```bash
$ kubectl get svc -n kube-system
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   32d
```

在kubernetes里，Service对象和Deployment对象都通过Pod的Label筛选自己管理的Pod.

而在`kube-system` namespace中，Service `kube-dns`和 Deployment `coredns`管理的Pod是相同的。

```yaml
# kubectl get svc kube-dns -n kube-system -o yaml
apiVersion: v1
kind: Service
metadata:
  name: kube-dns
  namespace: kube-system
spec:
  # ...
# kubectl get deployment coredns -n kube-system -o yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    k8s-app: kube-dns
  name: coredns
  namespace: kube-system
spec:
  replicas: 2
  selector:
    matchLabels:
      k8s-app: kube-dns
  # ...
```

也就是下面两个Pod:

```bash
# kubectl get pod -l k8s-app=kube-dns -n kube-system
NAME                       READY   STATUS    RESTARTS   AGE
coredns-59d64cd4d4-64fhc   1/1     Running   1          32d
coredns-59d64cd4d4-jb9n6   1/1     Running   1          32d
```