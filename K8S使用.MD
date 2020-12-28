# K8S使用

## 概念

![image-20201023161629004](D:\docker-k8s\k8s-learn\K8S使用.assets\image-20201023161629004.png)

![image-20201023165012313](D:\docker-k8s\k8s-learn\K8S使用.assets\image-20201023165012313.png)

![image-20201023171257688](D:\docker-k8s\k8s-learn\K8S使用.assets\image-20201023171257688.png)

![image-20201023171320623](D:\docker-k8s\k8s-learn\K8S使用.assets\image-20201023171320623.png)

![image-20201023170719886](D:\docker-k8s\k8s-learn\K8S使用.assets\image-20201023170719886.png)


## 创建集群

```bash
# 使用`httpd`镜像（Apache）创建两个名词为`http-app`的`pod`应用
kubectl run http-app --image=httpd --replicas=2

kubectl get deployments

```

```bash
# 创建deployment，来管理Nginx的创建和管理
$ kubectl create deployment myweb --image=nginx:latest
deployment.apps/myweb created

# 查看 deployment
$ kubectl get deployments
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
myweb   1/1     1            1           2m28s

# 查看 pod
$ kubectl get pods
NAME                     READY   STATUS    RESTARTS   AGE
myweb-6d48cc849f-9zsrz   1/1     Running   0          5m42s

# 查看具体某个pod 工作情况
$ kubectl describe pod myweb-6d48cc849f-9zsrz
Name:         myweb-6d48cc849f-9zsrz
Namespace:    default
Priority:     0
Node:         centos7-2/192.168.19.101
Start Time:   Mon, 07 Dec 2020 16:59:46 +0800
Labels:       app=myweb
              pod-template-hash=6d48cc849f
Annotations:  <none>
Status:       Running
IP:           10.244.1.2
IPs:
  IP:           10.244.1.2
Controlled By:  ReplicaSet/myweb-6d48cc849f
Containers:
  nginx:
    Container ID:   docker://655317b6d3eaa254f341ac4b34fe2174062f0f57c524a1812ba7473a819098a4
    Image:          nginx:latest
    Image ID:       docker-pullable://nginx@sha256:6b1daa9462046581ac15be20277a7c75476283f969cb3a61c8725ec38d3b01c3
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Mon, 07 Dec 2020 17:01:21 +0800
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-vmfxr (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  default-token-vmfxr:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-vmfxr
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                 node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age    From               Message
  ----    ------     ----   ----               -------
  Normal  Scheduled  10m    default-scheduler  Successfully assigned default/myweb-6d48cc849f-9zsrz to centos7-2
  Normal  Pulling    9m51s  kubelet            Pulling image "nginx:latest"
  Normal  Pulled     8m29s  kubelet            Successfully pulled image "nginx:latest" in 1m21.839134248s
  Normal  Created    8m27s  kubelet            Created container nginx
  Normal  Started    8m26s  kubelet            Started container nginx

# 测试访问
$ curl 10.244.1.2
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>


```

```bash
# 把上面创建的deployment(`myweb`)的80端口 暴露给 物理机
$ kubectl expose deployment myweb --port=80 --type=NodePort
service/myweb exposed

# 查看暴露
$ kubectl get service (或者 kubectl get svc)
NAME         TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP   10.96.0.1     <none>        443/TCP        41d
myweb        NodePort    10.98.89.34   <none>        80:31892/TCP   48s 
# 这里表示myweb的80暴露给了物理机的31892端口
# 浏览器访问 http://192.168.19.98:31892 可以看到nginx主页（使用集群中的任意一个IP地址加这个端口号都可以访问）
```

