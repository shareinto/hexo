title: windows下git-flow的安装
date: 2015-09-10 16:38:06
categories:
  - git
tags:
  - git
  - git-flow
------
# windows下git-flow的安装
---
### 本教程只针对安装了[1.9.5](http://cdncs.101.com/v0.1/static/skin_manager/default/biz-comp-main/ios/Git_V1.9.5_preview20150319.1435310867.exe?&attachment=true)(高版本此方法不行)版本的MSysGit的git-flow安装，如果是Cygwin版的请参照[这里](https://github.com/nvie/gitflow/wiki/Windows)

安装流程如下：
>1. 下载git-flow所依赖的包
>2. clone git-flow安装包

## 1. 下载git-flow所依赖的包
  下载 [git-flow.zip](http://cdncs.101.com/v0.1/static/skin_manager/default/biz-comp-main/ios/git-flow.zip?&attachment=true)。解压出**getopt.exe,libintl3.dll和libiconv2.dll**,放到Git安装目录（一般是C:\Program Files (x86)\Git）的bin目录下
  
## 2. clone git-flow安装包
  在C盘（此处随便，什么地方都可以）根目录下右键，点击**Git Bash** 运行：
  
```bash
$ git clone --recursive git://github.com/nvie/gitflow.git
```
 找到cmd(windows命令行)，右键点击->以管理员身份运行,进入到c:\gitflow文件夹,运行：
```bash
C:\gitflow> contrib\msysgit-install.cmd "C:\Program Files (x86)\Git"
```

## 最后

运行下面命令检测是否成功
```bash
$ git flow help
```
![git-flow-success](/image/git-flow-success.jpg)