# K8S使用

## 概念

![image-20201023161629004](D:\docker-k8s\k8s-learn\K8S使用.assets\image-20201023161629004.png)

![image-20201023165012313](D:\docker-k8s\k8s-learn\K8S使用.assets\image-20201023165012313.png)

![image-20201023171257688](D:\docker-k8s\k8s-learn\K8S使用.assets\image-20201023171257688.png)

![image-20201023171320623](D:\docker-k8s\k8s-learn\K8S使用.assets\image-20201023171320623.png)

![image-20201023170719886](D:\docker-k8s\k8s-learn\K8S使用.assets\image-20201023170719886.png)


## 创建集群

```bash
# 查看kubectl 命令说明
kubectl create -h
kubectl create deployment -h


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

## 配置WEB管理界面

### kubernate dashbord

参考文档`https://blog.csdn.net/networken/article/details/85607593`

```bash
# 安装UI界面 可以先把下面的文件下载到服务器再执行 `kubectl apply -f 文件.yaml`
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml
amespace/kubernetes-dashboard created
serviceaccount/kubernetes-dashboard created
service/kubernetes-dashboard created
secret/kubernetes-dashboard-certs created
secret/kubernetes-dashboard-csrf created
secret/kubernetes-dashboard-key-holder created
configmap/kubernetes-dashboard-settings created
role.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrole.rbac.authorization.k8s.io/kubernetes-dashboard created
rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
deployment.apps/kubernetes-dashboard created
service/dashboard-metrics-scraper created
deployment.apps/dashboard-metrics-scraper created

#查看pod的情况 要指定`kubernetes-dashboard`的命名空间
$ kubectl get pods -n kubernetes-dashboard
NAME                                         READY   STATUS    RESTARTS   AGE
dashboard-metrics-scraper-7b59f7d4df-jmsmz   1/1     Running   0          6m8s
kubernetes-dashboard-74d688b6bc-s2pmz        1/1     Running   0          6m10s
# 查看服务
$ kubectl get service -n kubernetes-dashboard
NAME                        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
dashboard-metrics-scraper   ClusterIP   10.97.209.183   <none>        8000/TCP   6m36s
kubernetes-dashboard        ClusterIP   10.105.44.74    <none>        443/TCP    6m39s
# 查看 deploment
$ bectl get deployment -n kubernetes-dashboard
NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
dashboard-metrics-scraper   1/1     1            1           8m6s
kubernetes-dashboard        1/1     1            1           8m7s

#使用nodeport方式将dashboard服务暴露在集群外，指定使用30443端口，可自定义：
$ kubectl patch svc kubernetes-dashboard -n kubernetes-dashboard \
-p '{"spec":{"type":"NodePort","ports":[{"port":443,"targetPort":8000,"nodePort":30443}]}}'

# 再次查看服务 443已经映射到30443端口
$ kubectl get service -n kubernetes-dashboard
NAME                        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE
dashboard-metrics-scraper   ClusterIP   10.97.209.183   <none>        8000/TCP        19m
kubernetes-dashboard        NodePort    10.105.44.74    <none>        443:30443/TCP   19m

```

**以上的我尝试失败了**

第二种方法修改下载下来的yaml文件

```yml
# ...
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  ports:
    - port: 443
      targetPort: 8443
      nodePort: 30443 # 暴露给主机30443端口
  selector:
    k8s-app: kubernetes-dashboard

---
# ...
```

```bash
# 然后重新执行 
$ kubectl apply -f dashboard.yaml
$kubectl get service -n kubernetes-dashboard
NAME                        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE
dashboard-metrics-scraper   ClusterIP   10.97.209.183   <none>        8000/TCP        26m
kubernetes-dashboard        NodePort    10.105.44.74    <none>        443:30443/TCP   26m

```

浏览器访问 `https://192.168.19.98:30443`

Dashboard 支持 Kubeconfig 和 Token 两种认证方式，我们这里选择Token认证方式登录。

官方参考文档:

https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md

创建dashboard-adminuser.yaml：

```bash
$ cat > dashboard-adminuser.yaml << EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard  
EOF

# 再执行
$ kubectl apply -f dashboard-adminuser.yaml
serviceaccount/admin-user created
clusterrolebinding.rbac.authorization.k8s.io/admin-user created

```

说明：上面创建了一个叫admin-user的服务账号，并放在kubernetes-dashboard  命名空间下，并将cluster-admin角色绑定到admin-user账户，这样admin-user账户就有了管理员的权限。默认情况下，kubeadm创建集群时已经创建了cluster-admin角色，我们直接绑定即可。

查看admin-user账户的token

```shell
$ kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')
Name:         admin-user-token-tdtrx
Namespace:    kubernetes-dashboard
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: admin-user
              kubernetes.io/service-account.uid: 3cdddebe-31cf-48f4-8f2c-0a45ef1848fe

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1066 bytes
namespace:  20 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6InNPLTBFcDVGaWFGMXQ3Q240d003cnJyaVgtZUY1N1ZDenhoTUxpamhiNmcifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLXRkdHJ4Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiIzY2RkZGViZS0zMWNmLTQ4ZjQtOGYyYy0wYTQ1ZWYxODQ4ZmUiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZXJuZXRlcy1kYXNoYm9hcmQ6YWRtaW4tdXNlciJ9.NASgk1d_mRodISqb5jN8_RLicrpgu8esfWfro0ShFUalR6SOGt9nu9109ElWyZroGpKOsnYi26gNmpcEHheRqPdppBjCqP1yJJ36wvNcqSaCFITLlTLva5AzQvaC3YFBRmp4s3mBSkW2SwIR_VZuBXZ5hk8-Cmcl8D03l3KN3rCf4uzgiibBQIYc-y6neZns7Obz7KTeuhbEGo9ue_e3EeMzXX_4poHkfRZt8W1YfmqB9MZqjCJwUmylQvqH85GOEbGSGVDSwQSRHObTiv_2GxJx_dlqf2RSEVb3KhXKmDwTFla2wvYIkj-fu2ZIVHNNvzNbotVyRhTtIvZ3aBwWcg

```

把获取到的Token复制到登录界面的Token输入框中，成功登陆dashboard

### kuboard

官方文档`https://www.kuboard.cn`

## 命令使用

### 管理使用deployment

- 创建指定数量的pod
- 管理pod的状态和数量

1. 创建指定名称的deployment

```bash
# 使用run的方式创建（基于deployment 使用nginx:latest镜像创建名称为my-dep的部署，副本为1个）
kubectl run my-dep --image=nginx:latest --replicas=1
# 查看
kubectl get deployment
# 查看pod
kubectl get pod -o wide
```

2. 使用kuboard图像化界面创建

3. 使用yaml文件创建

   ```yaml
   apiVersion: apps/v1 # api版本
   kind: Deployment # 类型
   metadata: # 元数据
     name: my-dep # 名称
     labels: # 标签用来过滤筛选容器，标签名:值 的方式
       app: nginx
       env: dep
   spec: # 规格，配置该deployment的详情信息，即pod信息
     replicas: 1 # 副本数量
     selector: # 选择器
       matchLabels: # 标签选择器
         app: nginx
     template: # 使用模板创建pod
       metadata: # pod的原数据
           labels: # pod的标签
             app: nginx
             env: dep
       spec: # pod的规格
         containers: # pod的容器
           - name: nginx-dep # 容器名称
             image: nginx:latest # 使用的镜像		
   ```

   保存成`my-dep.yaml` 然后执行 `kubectl apply -f my-dep.yaml`

   命令行查看 deployment 和 pod 是否创建

### 查看k8s对象状态

#### 命令：`kubectl get 资源类型`

资源类型：node、deployment、pod、service、namespace 等等

选项：

- -n 名称空间 （所有名称空间 通过命令`kubectl get namespace`查看）

  `kubectl get pod -n kube-public` ，不加`-n` 查看 `default`名称空间

- -A 显示所有名称空间的

  `kubectl get pod -A` 

  `kubectl get pod -A -o wide`  多一些显示信息

#### 命令：`kubectl describe 资源类型`

`kubectl describe pod myweb-6d48cc849f-9zsrz`