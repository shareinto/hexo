title: centos7-kubernetes-deploy
date: 2017-03-20 15:58:17
tags:
categories: docker kubernetes
tags:
  - docker
  - kubernetes
---

# master
```bash
#!/bin/bash

set -e

kube_master="--master=http:\/\/127\.0\.0\.1:8080"

etcd_listen_client_urls="http:\/\/localhost:2379"

etcd_advertise_client_urls="http:\/\/localhost:2379"

kube_api_address="--insecure-bind-address=127\.0\.0\.1"

kube_api_port="--port=8080"

kubelet_port="--kubelet-port=10250"

kube_etcd_servers="--etcd-servers=http:\/\/127.0.0.1:2379"

kube_service_addresses="--service-cluster-ip-range=10\.254\.0\.0\/16"

kube_api_args=""

etcd_config="{ \"Network\": \"172.30.0.0/16\", \"SubnetLen\": 24, \"Backend\": { \"Type\": \"vxlan\" } }"

flannel_etcd_endpoints="http:\/\/127\.0\.0\.1:2379"

flannel_etcd_prefix="\/atomic\.io\/network"

TEMP=`getopt -o ab:c:: --long kube-master:,kube-master-port:,etcd-listen-client-urls:,etcd-advertise-client-urls:,kube-api-address:,kube-etcd-servers:,kube-service-addresses:,flannel-etcd-endpoints:,flannel-etcd-prefix: -- "$@"`

eval set -- "$TEMP"

while true ; do
        case "$1" in
                --kube-master ) kube_master=$2;shift 2;;
                --kube-master-port ) kube_master_port=$2;shift 2;;
                --etcd-listen-client-urls ) etcd_listen_client_urls=$2;shift 2;;
                --etcd-advertise-client-urls ) etcd_advertise_client_urls=$2;shift 2;;
                --kube-api-address ) kube_api_address=$2;shift 2;;
                --kube-etcd-servers ) kube_etcd_servers=$2;shift 2;;
                --kube-service-addresses ) kube_service_addresses=$2;shift 2;;
                --flannel-etcd-endpoints ) flannel_etcd_endpoints=$2;shift 2;;
                --flannel-etcd-prefix ) flannel_etcd_prefix=$2;shift 2;;
                --)shift;break;;
        esac
done


echo "[virt7-docker-common-release]
name=virt7-docker-common-release
baseurl=http://cbs.centos.org/repos/virt7-docker-common-release/x86_64/os/
gpgcheck=0" > /etc/yum.repos.d/virt7-docker-common-release.repo

yum -y install --enablerepo=virt7-docker-common-release kubernetes etcd flannel

sed -i "s/KUBE_MASTER=\".*\"/KUBE_MASTER=\"$kube_master\"/g" /etc/kubernetes/config

setenforce=0

systemctl disable iptables-services firewalld || true

systemctl stop iptables-services firewalld || true

sed -i "s/ETCD_LISTEN_CLIENT_URLS=\".*\"/ETCD_LISTEN_CLIENT_URLS=\"$etcd_listen_client_urls\"/g" /etc/etcd/etcd.conf

sed -i "s/ETCD_ADVERTISE_CLIENT_URLS=\".*\"/ETCD_ADVERTISE_CLIENT_URLS=\"$etcd_advertise_client_urls\"/g" /etc/etcd/etcd.conf

sed -i "s/KUBE_API_ADDRESS=\".*\"/KUBE_API_ADDRESS=\"$kube_api_address\"/g" /etc/kubernetes/apiserver

sed -i "s/KUBE_ETCD_SERVERS=\".*\"/KUBE_ETCD_SERVERS=\"$kube_etcd_servers\"/g" /etc/kubernetes/apiserver

sed -i "s/KUBE_SERVICE_ADDRESSES=\".*\"/KUBE_SERVICE_ADDRESSES=\"$kube_service_addresses\"/g" /etc/kubernetes/apiserver

sed -i "s/#\sKUBE_API_PORT=\".*\"/KUBE_API_PORT=\"$kube_api_port\"/g" /etc/kubernetes/apiserver

sed -i "s/#\sKUBELET_PORT=\".*\"/KUBELET_PORT=\"$kubelet_port\"/g" /etc/kubernetes/apiserver

sed -i "s/KUBE_API_ARGS=\".*\"/KUBE_API_ARGS=\"$kube_api_args\"/g" /etc/kubernetes/apiserver

sed -i "s/^KUBE_ADMISSION_CONTROL/# KUBE_ADMISSION_CONTROL/g" /etc/kubernetes/apiserver

sed -i "s/FLANNEL_ETCD_ENDPOINTS=\".*\"/FLANNEL_ETCD_ENDPOINTS=\"$flannel_etcd_endpoints\"/g" /etc/sysconfig/flanneld

sed -i "s/FLANNEL_ETCD_PREFIX=\".*\"/FLANNEL_ETCD_PREFIX=\"$flannel_etcd_prefix\"/g" /etc/sysconfig/flanneld

flannel_etcd_prefix_unescape=$(echo $flannel_etcd_prefix | sed 's/\\//g')

systemctl start etcd

etcdctl rm -r $flannel_etcd_prefix_unescape/config||true
etcdctl rm -r $flannel_etcd_prefix_unescape||true

etcdctl mkdir $flannel_etcd_prefix_unescape
etcdctl mk $flannel_etcd_prefix_unescape/config "$etcd_config"

for SERVICES in etcd kube-apiserver kube-controller-manager kube-scheduler flanneld; do
	systemctl restart $SERVICES
	systemctl enable $SERVICES
	systemctl status $SERVICES
done
```
部署命令

```bash
./kubernetes-deploy-master --etcd-listen-client-urls "http:\/\/0.0.0.0:2379" --etcd-advertise-client-urls "http:\/\/0.0.0.0:2379" --kube-api-address "--address=0.0.0.0" --flannel-etcd-prefix "\/kube-centos\/network" --kube-etcd-servers "--etcd-servers=http:\/\/<master-host>:2379" --flannel-etcd-endpoints "http:\/\/<master-host>:2379" --kube-master "--master=http:\/\/<master-host>:8080" --kube-etcd-servers "--etcd-servers=http:\/\/<master-host>:2379"
```

# minion
```bash
#!/bin/bash

set -e

kube_master="--master=http:\/\/127\.0\.0\.1:8080"

flannel_etcd_endpoints="http:\/\/127\.0\.0\.1:2379"

flannel_etcd_prefix="\/atomic\.io\/network"

kubelet_address="--address=127.0.0.1"

kubelet_port="--port=10250"

kubelet_hostname="--hostname-override=127.0.0.1"

kubelet_api_server="--api-servers=http:\/\/127.0.0.1:8080"

kubelet_args=""

cluster_server="http:\/\/127.0.0.1:8080"

cluster="default-culster"

context="default-context"

context_user="default-admin"


TEMP=`getopt -o a: --long kube-master:,flannel-etcd-endpoints:,flannel-etcd-prefix:,kubelet-address:,kubelet-port:,kubelet-hostname:,kubelet-api-server:,kubelet-args:,cluster-server:,cluster:,context:,context-user: -- "$@"`

eval set -- "$TEMP"

while true ; do
        case "$1" in
                --kube-master ) kube_master=$2;shift 2;;
                --flannel-etcd-endpoints ) flannel_etcd_endpoints=$2;shift 2;;
                --flannel-etcd-prefix ) flannel_etcd_prefix=$2;shift 2;;
		--kubelet-address ) kubelet_address=$2;shift 2;;
		--kubelet-port ) kubelet_port=$2;shift 2;;
		--kubelet-hostname) kubelet_hostname=$2;shift 2;;
		--kubelet-api-server ) kubelet_api_server=$2;shift 2;;
		--kubelet-args ) kubelet_args=$2;shift 2;;
		--cluster ) cluster=$2;shift 2;;
		--cluster-server ) cluster_server=$2;shift 2;;
		--context ) context=$2;shift 2;;
		--context-user ) context_user=$2;shift 2;;

                --)shift;break;;
        esac
done


echo "[virt7-docker-common-release]
name=virt7-docker-common-release
baseurl=http://cbs.centos.org/repos/virt7-docker-common-release/x86_64/os/
gpgcheck=0" > /etc/yum.repos.d/virt7-docker-common-release.repo

yum -y install --enablerepo=virt7-docker-common-release kubernetes etcd flannel

sed -i "s/KUBE_MASTER=\".*\"/KUBE_MASTER=\"$kube_master\"/g" /etc/kubernetes/config

setenforce=0

systemctl disable iptables-services firewalld || true

systemctl stop iptables-services firewalld || true



sed -i "s/KUBELET_ADDRESS=\".*\"/KUBELET_ADDRESS=\"$kubelet_address\"/g" /etc/kubernetes/kubelet

sed -i "s/#\sKUBELET_PORT=\".*\"/KUBELET_PORT=\"$kubelet_port\"/g" /etc/kubernetes/kubelet

sed -i "s/KUBELET_HOSTNAME=\".*\"/KUBELET_HOSTNAME=\"$kubelet_hostname\"/g" /etc/kubernetes/kubelet

sed -i "s/KUBELET_API_SERVER=\".*\"/KUBELET_API_SERVER=\"$kubelet_api_server\"/g" /etc/kubernetes/kubelet

sed -i "s/#\sKUBE_API_PORT=\".*\"/KUBE_API_PORT=\"$kube_api_port\"/g" /etc/kubernetes/apiserver

sed -i "s/#\sKUBELET_PORT=\".*\"/KUBELET_PORT=\"$kubelet_port\"/g" /etc/kubernetes/apiserver

sed -i "s/KUBELET_ARGS=\".*\"/KUBELET_ARGS=\"$kubelet_args\"/g" /etc/kubernetes/kubelet

sed -i "s/^KUBELET_POD_INFRA_CONTAINER/# KUBELET_POD_INFRA_CONTAINER/g" /etc/kubernetes/kubelet

sed -i "s/FLANNEL_ETCD_ENDPOINTS=\".*\"/FLANNEL_ETCD_ENDPOINTS=\"$flannel_etcd_endpoints\"/g" /etc/sysconfig/flanneld

sed -i "s/FLANNEL_ETCD_PREFIX=\".*\"/FLANNEL_ETCD_PREFIX=\"$flannel_etcd_prefix\"/g" /etc/sysconfig/flanneld

for SERVICES in kube-proxy kubelet flanneld docker; do
    systemctl restart $SERVICES
    systemctl enable $SERVICES
    systemctl status $SERVICES
done

kubectl config set-cluster $cluster --server=$cluster_server
kubectl config set-context $context --cluster=$cluster --user=$context_user
kubectl config use-context $context
```

部署命令
```bash
./kubernetes-deploy-minion --kubelet-address "--address=0.0.0.0" --kubelet-hostname "--hostname-override=centos-minion-1" --kubelet-api-server "--api-servers=http:\/\/<master-host>:8080" --flannel-etcd-endpoints "http:\/\/<master-host>:2379" --flannel-etcd-prefix "\/kube-centos\/network" --cluster-server "http:\/\/<master-host>:8080" --kube-master "--master=http:\/\/<master-host>:8080"
```
