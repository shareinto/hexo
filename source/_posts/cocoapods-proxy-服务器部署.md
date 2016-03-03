title: cocoapods-proxy服务器部署
date: 2016-02-26 16:38:06
categories:
  - ios
tags:
  - ios
------
# cocoapods-proxy服务器部署
---
## 1. 制作git-mirrors镜像
>Dockerfile:
>```bash
FROM centos

MAINTAINER <dockerfun>

ENV GITLAB_HOST="git.menethil.net" \
    GITLAB_USER="850428" \
    PRIVATE_TOKEN="xmyV3Q4TXz1WDzPvyhha" \
    GITLAB_NAMESPACE="Mirrors" \
    GITLAB_URL="http:\/\/git.menethil.net" \
    SYSTEM_USER="root"


WORKDIR /root

RUN yum install python-setuptools -y  && yum install git -y && git clone https://github.com/alexvh/python-gitlab3.git

WORKDIR /root/python-gitlab3

RUN git checkout v0.5.4 && python setup.py install

RUN mkdir -p /root/.ssh &&  echo 'Host ${GITLAB_HOST}' > /root/.ssh/config && echo 'User ${GITLAB_USER}' >> /root/.ssh/config \
    && echo '${PRIVATE_TOKEN}' > /root/private_token

WORKDIR /root

RUN mkdir /root/repositories && git clone https://github.com/samrocketman/gitlab-mirrors.git

WORKDIR /root/gitlab-mirrors

COPY config.sh /root/gitlab-mirrors/config.sh

COPY entrypoint.sh /sbin/entrypoint.sh

ENTRYPOINT ["/sbin/entrypoint.sh"]
```
>其中另外两个文件如下：
>config.sh:
>```bash
#Environment file

#
# gitlab-mirrors settings
#

#The user git-mirrors will run as.
system_user=${SYSTEM_USER}
#The home directory path of the $system_user
user_home="/home/${SYSTEM_USER}"
#The repository directory where gitlab-mirrors will contain copies of mirrored
#repositories before pushing them to gitlab.
repo_dir="${user_home}/repositories"
#colorize output of add_mirror.sh, update_mirror.sh, and git-mirrors.sh
#commands.
enable_colors=true
#These are additional options which should be passed to git-svn.  On the command
#line type "git help svn"
git_svn_additional_options="-s"
#Force gitlab-mirrors to not create the gitlab remote so a remote URL must be
#provided. (superceded by no_remote_set)
no_create_set=false
#Force gitlab-mirrors to only allow local remotes only.
no_remote_set=false
#Enable force fetching and pushing.  Will overwrite references if upstream
#forced pushed.  Applies to git projects only.
force_update=false
#This option is for pruning mirrors.  If a branch is deleted upstream then that
#change will propagate into your GitLab mirror.  Aplies to git projects only.
prune_mirrors=false

#
# Gitlab settings
#

#This is the Gitlab group where all project mirrors will be grouped.
gitlab_namespace="${GITLAB_NAMESPACE}"
#This is the base web url of your Gitlab server.
gitlab_url="${GITLAB_URL}"
#Special user you created in Gitlab whose only purpose is to update mirror sites
#and admin the $gitlab_namespace group.
gitlab_user="${GITLAB_USER}"
#Generate a token for your $gitlab_user and set it here.
gitlab_user_token_secret="${PRIVATE_TOKEN}"
#Verify signed SSL certificates?
ssl_verify=false
#Push to GitLab over http?  Otherwise will push projects via SSH.
http_remote=false

#
# Gitlab new project default settings.  If a project needs to be created by
# gitlab-mirrors then it will assign the following values as defaults.
#

#values must be true or false
issues_enabled=false
wall_enabled=false
wiki_enabled=false
snippets_enabled=false
merge_requests_enabled=false
public=false
```
>entrypoint.sh:
>```bash
#!/bin/bash

set -e

[[ -n $DEBUG_ENTRYPOINT ]] && set -x

sed 's/\${GITLAB_NAMESPACE}/'"${GITLAB_NAMESPACE}"'/' -i ~/gitlab-mirrors/config.sh

sed 's/\${GITLAB_URL}/'"${GITLAB_URL}"'/' -i ~/gitlab-mirrors/config.sh

sed 's/\${GITLAB_USER}/'"${GITLAB_USER}"'/' -i ~/gitlab-mirrors/config.sh

sed 's/\${PRIVATE_TOKEN}/'"${PRIVATE_TOKEN}"'/' -i ~/gitlab-mirrors/config.sh

sed 's/\${SYSTEM_USER}/'"${SYSTEM_USER}"'/' -i ~/gitlab-mirrors/config.sh

if [ $# -eq 0 ];then
  echo "no arguments"
else
  $*
fi

exit 0
```
>镜像完成后，记得先在宿主机上面生成ssh-keygen,添加到自己搭建的gitlab服务器上，并运行镜像一次
>```bash
docker run --name git-mirrors -v /root/.ssh:/root/.ssh gitlab-mirrors /bin/bash
```
>记得-v映射到宿主机上的/root/.ssh目录
>进入容器
>```bash
docker-enter git-mirrors
```
>手动运行一次
>```bash
./add_mirror.sh --git --project-name gitlab-mirrors --mirror https://github.com/samrocketman/gitlab-mirrors.git
```
>在提示是否接受host的时候，输入yes
>之后退出容器并删除容器
>```bash
exit
docker rm -f git-mirrors
```

## 2.拉取salt-master镜像
>```bash
docker pull 120.26.128.207:5000/shareinto/salt-master:0.2
```
>该镜像集成了salt-master，salt-api环境
>运行该镜像脚本如下
>```bash
#!/bin/bash
docker rm -f salt-master
docker run \
--name salt-master \
-p 1022:22 \
-p 4505:4505 \
-p 4506:4506 \
-p 8886:8886 \
-v /data/etc/salt/pki:/etc/salt/pki -v /data/var/cache/salt:/var/cache/salt -v /data/srv/salt:/srv/salt \
-d 120.26.128.207:5000/shareinto/salt-master:0.2
```
## 3.安装salt-minion
>```bash
sudo yum install salt-minion -y
```
>修改/etc/salt/minion文件
>```bash
master: 172.24.133.159
id: 172.24.133.159
```
>这里因为master机和minion机是同一台机器，所以master和id一样，使用时替换成你自己的ip
>完成后，启动salt-minion服务
>```bash
sudo service salt-minion start
```

## 4.安装git客户端
>```bash
sudo yum install git -y
```

## 5.创建几个目录 /data/original /data/temp/specs /data/formal/specs
>```bash
sudo mkdir -p /data/original
sudo mkdir -p /data/temp/specs
sudo mkdir -p /data/formal/specs
cd /data/original
sudo git clone https://github.com/cocoapods/specs.git
cd /data/formal/specs
sudo git init
sudo git add .
sudo git commit -m "first commit"
sudo git remote add origin git@git.sdp.nd:Mirrors/specs.git
sudo git push -u origin master
```
>在最后push到服务器之前，记得先生成ssh-keygen,添加到自己搭建的gitlab服务器上