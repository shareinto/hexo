title: sdp-web使用教程
date: 2015-10-22
categories: sdp
tags: 
  - sdp
  - web
------
## 0. 准备
 >* 阅读本文前，请先登录[gitlab](http://git.menethil.net),以保证你的帐号在代码仓库中存在（注：初始密码和工号一样，密码修改请到[http://erp.menethil.net](http://erp.menethil.net),若工号不存在,请qq:56471924通知我)
 
## 1. 登录
>* 打开我渴了[共享平台](http://sdp.menethil.net),点击右上角登录按钮进行登录 如下图  
 ![login](http://7xlovv.com1.z0.glb.clouddn.com/login.jpg)  
 >* 在弹出框中输入工号和密码。（注：初始密码和工号一样，密码修改请到[http://erp.menethil.net](http://erp.menethil.net),若工号不存在,请qq:56471924通知我)
 
## 2. 创建
 >* 登录完成后，点击我的应用，跳转到我的应用页面  
 >* 点击**创建应用->Web应用**  
 ![create](http://7xlovv.com1.z0.glb.clouddn.com/sdp-web-create.jpg)  
 >* 输入**应用名称**和**应用描述**  
![component-name](http://7xlovv.com1.z0.glb.clouddn.com/sdp-web-name.jpg)  
  >* 点击下一步，等待几秒钟，创建成功  
  ![create-success](http://7xlovv.com1.z0.glb.clouddn.com/sdp-web-success.jpg)
  
## 3. 管理
 >* 点击**我的应用**标签，在**应用**模块下面可以看到刚才创建的web应用
![component-list](http://7xlovv.com1.z0.glb.clouddn.com/sdp-web-list.jpg)  
>* 点击应用名称或图标，进入到web应用详情页
>* 点击域名下的url，可以看到申请的web应用的默认首页=>***恭喜应用申请成功***
![url](http://7xlovv.com1.z0.glb.clouddn.com/sdp-web-domain.jpg)
![index](http://7xlovv.com1.z0.glb.clouddn.com/sdp-web-index.jpg)
>* GIT 仓库管理：
  * 如果您的windows操作系统还没有安装git客户端，请到[这里](http://cdncs.101.com/v0.1/static/skin_manager/default/biz-comp-main/ios/Git_V1.9.5_preview20150319.1435310867.exe?&attachment=true)下载安装
  * 安装完毕后，在空白地方点击鼠标右键，可以看到git相关菜单  
  ![git-setup](http://7xlovv.com1.z0.glb.clouddn.com/git-setup.jpg)  
  * git sshkey生成： 请先登录[gitlab](http://git.menethil.net),然后点击[http://git.menethil.net/help/ssh/README](http://git.menethil.net/help/ssh/README)查看key生成教程，将生成的key添加到[http://git.menethil.net/profile/keys/new](http://git.menethil.net/profile/keys/new)
  * 克隆仓库到本地： 找到你的开发目录，在空白的地点点击鼠标右键，点击**Git Bash**，键入 git clone 详情页中的GIT地址  
 ![git-clone](http://7xlovv.com1.z0.glb.clouddn.com/git-clone.jpg)
## 4. 开发
>进入第3步clone下来的代码文件夹
```bash
$ cd test
```
>我们的项目都使用git flow进行开发，windows下git
>flow的安装请参照[windows下git-flow的安装](http://shareinto.github.io/2015/09/10/windows-git-flow-install/),[git flow简明教程](http://danielkummer.github.io/git-flow-cheatsheet/)
>git flow安装完毕以后，在项目根目下运行
```bash
$ git flow init
```
>接下来一路回车就可以了
![git-flow-init](http://7xlovv.com1.z0.glb.clouddn.com/git-flow-init.jpg)
>接下来修改首页内容
```bash
$ git flow feature start update_index
```
>找到刚才clone的文件夹，进入src> main > webapp,编辑index.html文件
![index-folder](http://7xlovv.com1.z0.glb.clouddn.com/sdp-web-index-folder.jpg)
![index-edit](http://7xlovv.com1.z0.glb.clouddn.com/sdp-web-index-edit.jpg) 

>修改完成后我们可以完成提交该功能点
```bash
$ git add .
$ git commit -a
```
>完成该功能点开发
```bash
$ git flow feature finish update_index
```
>将develop分支推至远程develop分支
```bash
$ git push origin develop:develop
```
>在共享平台的test项目详情页下，当前环境选择**开发**(开发对应的是develop分支)，然后点击发布，输入版本号和发布内容，点击确定
![sdp-develop](http://7xlovv.com1.z0.glb.clouddn.com/sdp-web-develop.jpg)
>再次查看web应用的首页,发现更改的内容已经生效
![sdp-index](http://7xlovv.com1.z0.glb.clouddn.com/sdp-web-index-hello-world.jpg)

## 5. 生产环境发布
>将develop分支内容合并到master分支
```bash
$ git flow release start update_index
$ git flow release finish update_index
```
>内容合并到master分支以后，就可以将本地master推送至远程master了
```bash
$ git push origin master:master
```
>在共享平台的test项目详情页下，当前环境选择**生产**(生产对应的是master分支)，然后点击发布，输入版本号和发布内容，点击确定
![sdp-master](http://7xlovv.com1.z0.glb.clouddn.com/sdp-web-master.jpg)
>查看生产环境的首页，成功发布到生产环境
![nexus-success](http://7xlovv.com1.z0.glb.clouddn.com/sdp-web-index-master.jpg)