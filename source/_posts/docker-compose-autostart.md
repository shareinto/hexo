title: docker-compose开机自动重启
date: 2016-03-09 15:09:07
tags:
  - docker
  - docker-compose
  - linux
---
## docker-compose开机自动重启
>  在生产环境中，整个团队需要发布的容器数量很可能极其庞大，而容器之间的联系和拓扑结构也很可能非常复杂，如果依赖人工记录和维护这样复杂的容器关系，并保障集群正常运行、监控、迁移、高可用等常规运维需求，实在是力不从心。
>因此，我们需要一种像Dockerfile定义Docker容器一样能够定义容器集群的编排和部署工具，Compose就是来解决这种问题的工具。

## docker-compose的安装
>docker-compose的安装非常简单：
>```bash
$ curl -L https://github.com/docker/compose/releases/download/1.2.0/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose
$ chmod +x /usr/local/bin/docker-compose
```

## docker-compose.yml的编写
>这是sdp环境的一个例子
>```bash
gitlab: 
    image: 'registry.menethil.net:5000/sameersbn/gitlab' 
    ports: 
        - "8082:80" 
        - "10022:22" 
    links: 
        - redis:redisio 
        - postgresql:postgresql 
        - ldap:ldap1 
    volumes: 
        - /data/srv/docker/gitlab/gitlab:/home/git/data 
    environment: 
        - GITLAB_PORT=80 
        - GITLAB_SSH_PORT=10022 
        - GITLAB_HOST=git.menethil.neti 
        - LDAP_ENABLED=true 
        - LDAP_HOST=10.117.105.140 
        - LDAP_UID=uid 
        - LDAP_BIND_DN=cn=Manager,dc=tuleap,dc=local 
        - LDAP_PASS=123456 
        - LDAP_ALLOW_USERNAME_OR_EMAIL_LOGIN=false 
        - LDAP_BLOCK_AUTO_CREATED_USERS=false 
        - LDAP_BASE=ou=people,dc=tuleap,dc=local 
        - LDAP_LABEL=LDAP 
        - UNICORN_TIMEOUT=72000 
        - NGINX_MAX_UPLOAD_SIZE=10000M 
redis: 
    image: 'registry.menethil.net:5000/sameersbn/redis:latest' 
    volumes: 
        - /data/var/lib/redis:/var/lib/redis 
    expose: 
        - "6379" 
postgresql: 
    image: 'registry.menethil.net:5000/sameersbn/postgresql:9.4-3' 
    volumes: 
        - /data/var/lib/postgresql:/var/lib/postgresql 
    environment: 
        - DB_NAME=gitlabhq_production
        - DB_USER=gitlab
        - DB_PASS=password
    ports:
        - "5432:5432"
    expose:
        - "5432"

nginx:
    image: 'registry.menethil.net:5000/shareinto/nginx:0.1'
    ports:
        - '80:80'
    volumes:
        - /root/nginx/nginx-conf:/etc/nginx/conf.d
        - /data/etc/salt/pki/minion:/etc/salt/pki/minion
ldap:
    image: 'registry.menethil.net:5000/enalean/ldap'
    ports:
        - '389:389'
        - '636:636'
    expose:
        - "389"
        - "636"
    volumes:
        - /data/ldap/data:/data
    environment:
        - LDAP_ROOT_PASSWORD=123456
        - LDAP_MANAGER_PASSWORD=123456
jenkins:
    image: 'registry.menethil.net:5000/shareinto/jenkins:0.1'
    ports:
        - "8080:8080"
    volumes:
        - /data/home/.m2/repository:/home/.m2/repository
        - /data/data/jenkins/jobs:/data/jenkins/jobs
        - /data/data/jenkins:/data/jenkins
    environment:
        - JENKINS_HOME=/data/jenkins
        - NEXUS_HOST=http://10.117.105.140:8081
    user: root
    links:
        - ldap:ldap1
nexus:
    image: 'registry.menethil.net:5000/sonatype/nexus'
    ports:
        - "8081:8081"
    volumes:
        - /data/sonatype-work:/sonatype-work
    user: root
mongo:
    image: 'registry.menethil.net:5000/tutum/mongodb'
    ports:
        - "27017:27017"
        - "28017:28017"
    volumes:
        - AUTH=no
    ports:
        - "4506:4506"
        - /data/etc/salt/pki:/etc/salt/pki
        - /data/var/cache/salt:/var/cache/salt
        - /data/etc/salt/master.d:/etc/salt/master.d
        - /data/srv/salt:/srv/salt
erp:
    image: 'registry.menethil.net:5000/shareinto/omco.erp.web:1.0.0'
    ports:
        - "5004:5004"
    links:
        - ldap:ldap1
sdp:
    image: 'shareinto/portal'
    ports:
        - "8889:8080"
    links:
        - saltmaster
        - jenkins
        - nexus
        - gitlab
```
>文件的说明文档可以查看[这里](https://docs.docker.com/v1.8/compose/yml/)

## 开机自动运行
>找到/etc/rc.d/rc.local文件,添加以下脚本
>```bash
/usr/local/bin/docker-compose -f /root/gitlab-compose/docker-compose.yml up -d
```
>其中-f参数是指定docker-compose.yml文件的参数
>设置完以后，重启操作系统，耐心等待一会，就可以看到对应的docker容器都启动起来了