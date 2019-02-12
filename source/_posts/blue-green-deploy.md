title: 基于Ingress的BlueGreenDeployment
date: 2017-04-24 18:12:40
categories: kubernetes
tags:
  - kubernetes
  - linux
  - docker
------
## Blue-Green Deployment
[Blue-Green](https://martinfowler.com/bliki/BlueGreenDeployment.html)是一种无宕机的升级技术，和滚动升级不同，蓝绿部署是启动一个运行着新版应用的副本的集群，旧版的应用依旧提供服务，直到新的应用真正启动并配置好负载均衡器。这种方式的一个好处是任何时候都只有一个版本的应用在运行，减少了处理多个并发版本带来的复杂性。当副本个数很少时，蓝绿部署也能很好地工作。

在传统的发布中，新版本的服务只有上线以后（此时新版本的软件已经暴露给了用户）测试人员才能够进行线上测试，实际上这个时候的测试的意义并不是太大，因为如果存在bug的话，那么这个bug已经暴露给了最终用户，要解决bug要么继续发布更新的版本，要么进行线上回滚，这对于用户来说是一种非常不友好的体验，对于开发和测试人员也产生了一定的压力。

蓝绿发布将新版本的服务发布到一个新的生产环境中，该环境和旧版本的环境完全一致，唯一的区别是最终用户是访问不到新版本的服务，这时候只有测试人员可以访问。这样，就有办法保证测试人员有足够的时间进行系统测试。
![](/image/canary-release-1.png)
当测试人员完成测试后，再将流量切换至新版本服务。
![](/image/canary-release-3.png)
切换成功以后，再将旧版本的环境进行删除。

## 在kubernetes中实现蓝绿发布
kubernetes本身并不提供蓝绿发布的功能，包括在deployment中，它的发布策略只包含滚动发布（rolling update）和重建发布(recreate)。要实现蓝绿发布，我们必须将其业务提取到自己的管理层来。

### 准备
1. 一个kubernetes集群环境
2. 至少有一个Ingress Controller。（我们将使用Ingress来进行host和service的绑定）

###  blue-deployment
第一步，我们创建一个verion 为 1 的deployment。

deployment-blue.yaml:
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: demo-deployment-blue
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: demo
        track: blue
    spec:
      volumes:
      - name: shared-data
        emptyDir: {}
      restartPolicy: Always

      containers:

      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
        volumeMounts:
        - name: shared-data
          mountPath: /usr/share/nginx/html

      - name: debian-container
        image: debian
        volumeMounts:
        - name: shared-data
          mountPath: /pod-data
        command: ["/bin/sh"]
        args: ["-c", "echo hello this is version 1 > /pod-data/index.html && sleep 1000000000"]
```
```bash

kubectl create -f deployment-blue.yaml

kubectl get pod -l app=demo --show-labels

demo-deployment-blue-2523566789-641sj    2/2       Running   0          1h        app=demo,pod-template-hash=2523566789,track=blue
demo-deployment-blue-2523566789-h88ch    2/2       Running   0          1h        app=demo,pod-template-hash=2523566789,track=blue
demo-deployment-blue-2523566789-kqsdg    2/2       Running   0          1h        app=demo,pod-template-hash=2523566789,track=blue
```

### blue-service

创建blue-service

service-blue.yaml:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: demo-service-blue
  labels:
    app: demo-service-blue
spec:
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
  selector:
    app: demo
    track: blue
```
```bash
kubectl create -f service-blue.yaml

kubectl describe svc demo-service-blue

Name:                   demo-service-blue
Namespace:              default
Labels:                 app=demo-service-blue
Selector:               app=demo,track=blue
Type:                   ClusterIP
IP:                     10.254.1.251
Port:                   <unset> 80/TCP
Endpoints:              172.30.40.26:80,172.30.40.28:80,172.30.56.18:80
Session Affinity:       None
No events.
```
该Service通过app=demo,track=blue的标签找到了三个终节点(我们在deployment中指定了复本数为3)

### stable-ingress
最后我们通过stable-ingress将该服务暴露给最终用户。

ingress-stable.yaml:
```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: demo-ingress-stable
  annotations:
    kubernetes.io/ingress.class: "nginx"
    ingress.kubernetes.io/force-ssl-redirect: "false"
spec:
  rules:
  - host: demo-stable
    http:
      paths:
      - backend:
          serviceName: demo-service-blue
          servicePort: 80
        path: /
```
验证：
```bash
kubectl create -f ingress-stable.yaml

kubectl get ing demo-ingress-stable

NAME                  HOSTS         ADDRESS         PORTS     AGE
demo-ingress-stable   demo-stable   172.24.133.92   80        1h
```

最后我们访问一下该host
```bash
curl 172.24.133.92 -H "HOST:demo-stable"
hello this is version 1
```

### green-deployment
接下来，我们打算发布一个新的服务版本（version 2）。我们先不删除旧版本的服务。而是直接发布一个新的deployment,我们称它为deployment-green:

deployment-green.yaml:
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: demo-deployment-green
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: demo
        track: green
    spec:
      volumes:
      - name: shared-data
        emptyDir: {}
      restartPolicy: Always

      containers:

      - name: nginx
        image: 172.24.133.2:5000/nginx:1.7.9
        ports:
        - containerPort: 80
        volumeMounts:
        - name: shared-data
          mountPath: /usr/share/nginx/html

      - name: debian-container
        image: 172.24.133.2:5000/debian
        volumeMounts:
        - name: shared-data
          mountPath: /pod-data
        command: ["/bin/sh"]
        args: ["-c", "echo hello this is version 2 > /pod-data/index.html && sleep 1000000000"]
```
```bash
kubectl create -f deployment-green.yaml

kubectl get pod -l app=demo --show-labels

NAME                                     READY     STATUS    RESTARTS   AGE       LABELS
demo-deployment-blue-2523566789-641sj    2/2       Running   0          1h        app=demo,pod-template-hash=2523566789,track=blue
demo-deployment-blue-2523566789-h88ch    2/2       Running   0          1h        app=demo,pod-template-hash=2523566789,track=blue
demo-deployment-blue-2523566789-kqsdg    2/2       Running   0          1h        app=demo,pod-template-hash=2523566789,track=blue
demo-deployment-green-3779826479-7nfx3   2/2       Running   0          1h        app=demo,pod-template-hash=3779826479,track=green
demo-deployment-green-3779826479-ck8v0   2/2       Running   0          1h        app=demo,pod-template-hash=3779826479,track=green
demo-deployment-green-3779826479-n40kj   2/2       Running   0          1h        app=demo,pod-template-hash=3779826479,track=green
```
这里可以看到新旧版本一共六个pod，我们能过标签track将它们区分开来。

### green-service
接下来创建新版本的service

service-green.yaml:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: demo-service-green
  labels:
    app: demo-service-green
spec:
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
  selector:
    app: demo
    track: green
```

```bash
kubectl create -f service-green.yaml

kubectl describe svc demo-service-green

Name:                   demo-service-green
Namespace:              default
Labels:                 app=demo-service-green
Selector:               app=demo,track=green
Type:                   ClusterIP
IP:                     10.254.125.14
Port:                   <unset> 80/TCP
Endpoints:              172.30.40.29:80,172.30.56.20:80,172.30.56.21:80
Session Affinity:       None
No events.
```
### canary-ingress
接下来创建一个专门针对测试人员的ingress。它的host为demo-canary。

ingress-canary.yaml:
```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: demo-ingress-canary
  annotations:
    kubernetes.io/ingress.class: "nginx"
    ingress.kubernetes.io/force-ssl-redirect: "false"
spec:
  rules:
  - host: demo-canary
    http:
      paths:
      - backend:
          serviceName: demo-service-green
          servicePort: 80
        path: /
```

```bash
kubectl create -f ingress-canary.yaml

kubectl get ing demo-ingress-canary

NAME                  HOSTS         ADDRESS         PORTS     AGE
demo-ingress-canary   demo-canary   172.24.133.92   80        2h
```

这时候，分别能过两个不同的host进行访问
```bash
curl 172.24.133.92 -H "HOST:demo-stable"
hello this is version 1

curl 172.24.133.92 -H "HOST:demo-canary"                         
hello this is version 2
```
这个时候，最终用户还是能过host:demo-stable来进行访问，它所访问到的服务版本为1，而测试人员可以通过host:demo-canary访问版本2。

### 切换
当测试人员完成测试，这个时候就可以将流量引到版本2
修改 ingress-stable.yaml:
```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: demo-ingress-stable
  annotations:
    kubernetes.io/ingress.class: "nginx"
    ingress.kubernetes.io/force-ssl-redirect: "false"
spec:
  rules:
  - host: demo-stable
    http:
      paths:
      - backend:
          serviceName: demo-service-green
          servicePort: 80
        path: /
```

```bash
kubectl apply -f ingress-stable.yaml

curl 172.24.133.92 -H "HOST:demo-stable"
hello this is version 2
```

这时候，用户流量被引到版本2，发布成功。

### 清理
```bash
kubectl delete ing demo-canary
kubectl delete svc demo-service-blue
kubectl delete deployment demo-deployment-blue
```
按顺序清理以上资源。

## 缺陷
关于蓝绿发布的缺陷，目前主要有两点：
1. 在发布期间要比原来多占用一倍的服务器资源。
2. 需要维护一份当前用户流量所至的环境的数据（例如current:blue）。

但相比旧的发布流程，蓝绿发布所带来的系统可用性的提升和用户体验的提升是非常巨大的。
（全文完）

