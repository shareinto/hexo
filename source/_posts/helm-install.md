title: helm安装
date: 2019-02-12 16:07:10
tags:
  - kubernetes
---
## 安装Helm
>Helm的安装分为两部分：客户端（helm）和服务端（Tiller）
# 客户端安装
> 1. [在此选择你需要的版本进行下载](https://github.com/helm/helm/releases)
2. 解压：tar -zxvf  helm-v2.0.0-linux-amd64.tgz
3. 将二进制文件移至目标路径： mv linux-amd64/helm /usr/local/bin/helm
# 服务端安装
>```sh
$ helm init --upgrade
```
>由于默认情况下，helm会拉取gcr.io/kubernetes-helm/tiller镜像，并且会以https://kubernetes-charts.storage.googleapis.com 作为stable的repository，国内无法直接访问，可用阿里云代替
```sh
$ helm init --upgrade -i registry.cn-hangzhou.aliyuncs.com/google_containers/tiller:v2.10.0 --stable-repo-url https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
```
# 给Tiller授权
>因为 Helm 的服务端 Tiller 是一个部署在 Kubernetes 中 Kube-System Namespace 下 的 Deployment，它会去连接 Kube-Api 在 Kubernetes 里创建和删除应用。
>而从 Kubernetes 1.6 版本开始，API Server 启用了 RBAC 授权。目前的 Tiller 部署时默认没有定义授权的 ServiceAccount，这会导致访问 API Server 时被拒绝。所以我们需要明确为 Tiller 部署添加授权。
```sh
$ kubectl create serviceaccount --namespace kube-system tiller
$ kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
$ kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'
```