title: sdp-java组件模块使用教程
date: 2015-09-10 16:38:06
tags: sdp java-component
------
## 0. 准备
 >* 阅读本文前，请先登录[gitlab](http://git.menethil.net),以保证你的帐号在代码仓库中存在（注：初始密码和工号一样，密码修改请到[这里](http://erp.menethil.net),若工号不存在,请qq:56471924通知我)
 
## 1. 登录
>* 打开我渴了[共享平台](http://sdp.menethil.net),点击右上角登录按钮进行登录 如下图  
 ![login](http://7xlovv.com1.z0.glb.clouddn.com/login.jpg)  
 >* 在弹出框中输入工号和密码。（注：初始密码和工号一样，密码修改请到[这里](http://erp.menethil.net),若工号不存在,请qq:56471924通知我)
 
## 2. 创建
 >* 登录完成后，点击我的应用，跳转到我的应用页面  
 >* 点击**创建组件->Web组件**  
 ![create](http://7xlovv.com1.z0.glb.clouddn.com/create.jpg)  
 >* 输入**组件名称**和**组件描述**  
![component-name](http://7xlovv.com1.z0.glb.clouddn.com/component-name.jpg)  
  >* 点击下一步，等待几秒钟，创建成功  
  ![create-success](http://7xlovv.com1.z0.glb.clouddn.com/create-success.jpg)
  
## 3. 管理
 >* 点击**我的应用**标签，在**组件**模块下面可以看到刚才创建的组件
![component-list](http://7xlovv.com1.z0.glb.clouddn.com/component-list.jpg)  
>* 点击模块名称或图标，进入到模块详情页
>* GIT 仓库管理：
  * 如果您的windows操作系统还没有安装git客户端，请到[这里](http://cdncs.101.com/v0.1/static/skin_manager/default/biz-comp-main/ios/Git_V1.9.5_preview20150319.1435310867.exe?&attachment=true)下载安装
  * 安装完毕后，在空白地方点击鼠标右键，可以看到git相关菜单  
  ![git-setup](http://7xlovv.com1.z0.glb.clouddn.com/git-setup.jpg)  
  * git sshkey生成： 请先登录[gitlab](http://git.menethil.net),然后点击[这里](http://git.menethil.net/help/ssh/README)查看key生成教程，将生成的key添加到[这里](http://git.menethil.net/profile/keys/new)
  * 克隆仓库到本地： 找到你的开发目录，在空白的地点点击鼠标右键，点击**Git Bash**，键入 git clone 详情页中的GIT地址  
 ![git-clone](http://7xlovv.com1.z0.glb.clouddn.com/git-clone.jpg)
 
## 4. 开发
>进入第3步clone下来的代码文件夹
```bash
$ cd test
```
>我们的项目都使用git flow进行开发，windows下git
>flow的安装请参照[这里](http://shareinto.github.io/2015/09/10/windows-git-flow-install/),git flow[简明教程](http://danielkummer.github.io/git-flow-cheatsheet/)
>git flow安装完毕以后，在项目根目下运行
```bash
$ git flow init
```
>接下来一路回车就可以了
![git-flow-init](http://7xlovv.com1.z0.glb.clouddn.com/git-flow-init.jpg)
>接下来添加一个加法函数，运行
```bash
$ git flow feature start function_plus
```
>在eclipse导入项目
![eclipse-import](http://7xlovv.com1.z0.glb.clouddn.com/eclipse-import.jpg)
>添加Plus类
![add-plus-class](http://7xlovv.com1.z0.glb.clouddn.com/add-plus-class.jpg)
>添加完成后我们可以完成提交该功能点
```bash
$ git add .
$ git commit -a
```
>完成该功能点开发
```bash
$ git flow feature finish function_plus
```
>将develop分支推至远程develop分支
```bash
$ git push origin develop:develop
```
>此时，在gitlab上可以看到我们刚才编写的代码了
![git-flow-develop](http://7xlovv.com1.z0.glb.clouddn.com/git-lab-develop.jpg)
>在共享平台的test项目详情页下，当前环境选择**开发**(开发对应的是develop分支，只进行package操作，不进行deploy操作)，然后点击发布，输入版本号和发布内容，点击确定
![sdp-develop](http://7xlovv.com1.z0.glb.clouddn.com/sdp-develop.jpg)

## 5. 生产环境发布
>将develop分支内容合并到master分支
```bash
$ git flow release start function_plus
$ git flow release finish function_plus
```
>内容合并到master分支以后，就可以将本地master推送至远程master了
```bash
$ git push origin master:master
```
>在gitlab上可以看到我们编写的代码了
![gitlab-master](http://7xlovv.com1.z0.glb.clouddn.com/gitlab-master.jpg)
>在共享平台的test项目详情页下，当前环境选择**生产**(生产对应的是master分支，进行打包操作并发布到nexus库上)，然后点击发布，输入版本号和发布内容，点击确定
![sdp-master](http://7xlovv.com1.z0.glb.clouddn.com/sdp-master.jpg)
>成功以后，我们就可以在[nexus仓库](http://nexus.menethil.net)上找到我们刚才发布的组件了
![nexus-success](http://7xlovv.com1.z0.glb.clouddn.com/nexus-success.jpg)