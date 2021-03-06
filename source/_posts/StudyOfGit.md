---
title: 必须要会的Git基本使用及常用命令操作
date: 2017-4-25 13:15:08
toc: true
author: tabris
# 图片推荐使用图床(腾讯云、七牛云、又拍云等)来做图片的路径.如:http://xxx.com/xxx.jpg
img:
# 如果top值为true,则会是首页推荐文章
top: false
# 如果要对文章设置阅读验证密码的话,就可以在设置password的值,该值必须是用SHA256加密后的密码,防止被他人识破
password:
# 本文章是否开启mathjax，且需要在主题的_config.yml文件中也需要开启才行
mathjax: false
summary:
categories:
  - 学习
  - 实用技能
tags:
  - git
---

之前一直用的svn，后来换了之后才发现git的强大功能，是svn不能比的。缺点就是可能上手比较难一点，刚开始自己研究了两天才摸索出来一些基本使用方法。
最近做项目需要建库等等，都用到了git，随着越来越多的使用，也越来有越多的认识。

<!-- more -->

一开始都是别人建好远程库，克隆下来就行了。
下面内容只是带你git入门，一些基础的东西，是开发过程中一些基本的操作，单单这些你会用了之后就能发现他的好处，以及使用命令行Enter敲击时的快感，还能提高逼格。
当然我们还是为了方便项目管理。

##### 安装
git工具下载地址，可以选择适合自己的操作系统：https://git-scm.com/downloads
安装完git，要配置环境变量，拷贝git安装目录下的bin文件目录，如D:\Program Files\Git\bin
,将目录拷贝添加到PATH变量后。
__注意：与前面的值要用“;”号隔开__
具体步骤：

> 右键计算机-属性-高级系统设置-环境变量-PATH将目录添加到后面，%JAVA_HOME%\bin;%JAVA_HOME%\jre\bin;D:\Program Files\Git\bin

安装完成后使用 ``git --version``命令查看一下git版本，测试是否安装、配置成功。
##### 克隆远程库
使用cmd（安装过git直接可以右键文件夹使用``git bash here``）定位到要放置仓库的目录
``git clone [远程仓库地址]``
远程仓库就是托管到第三方平台上面的库。
常用的有github，这个私有是收费的，要用免费的只能是公共的。
目前国内用的比较多的[coding](https://coding.net/) ，和开源中国的[码云](http://git.oschina.net/)。原理都一样，只不过看起来会有点视觉上的差别，个人觉得coding比较简洁,适合学习，刚入门git的新手练习。而且视图更直观。
码云是我现在用的，功能要比coding多，包括直接下载上传文件以及打包好的apk文件。

在这里要说一下克隆的时候有的坑，__要克隆远程仓库必须是你在这个项目中，就是项目所有者（管理员）把你添加进这个项目成员__。输入克隆的指令后，如果是第一次使用会提示你输入用户名，和密码。

前面步骤如果无误，之后会显示克隆的进度，直到完成克隆。

##### 创建代码库
包括远程库（第三方平台）、本地库（存放代码信息）。

- 创建远程库：根据第三方平台提示进行创建，一般都有步骤说明，按照说明来就好。
  创建完成后建议初始化一下仓库，
  可以在远程上根据提示，创建使用README.md文件初始化项目。
  也可以使用git命令：

  ```
  git init
  echo "# HelloWorld" >> README.md
  git add README.md
  git commit -m "first commit"
  ```

  建议使用前者，直接在第三方上创建。

- 创建本地仓库：有两种方法
  1、使用git命令
  ``git init``
  2、android Studio中（这里使用AS为例，其它的IDEA、webStorm操作都一样）
  点击``VCS-import into version control-create git repository``
  会弹出选择仓库的路径，直接选当前项目就行，然后确定。
  创建完之后，找到项目路径会发现文件夹下多了个.git文件，这个就是存放代码的仓库。
  而且项目中的文件的名称都会变为红色，说明已经有仓库了，但是这些红色的项目文件，并没有加到本地仓库（.git仓库文件中）。
  __（关于颜色后面我会具体说，各种颜色代表的状态）__

- 关联本地和远程库
  关联就是把本地仓库的.git仓库文件，和远程（coding）创建的仓库联系起来，每次提交代码，将本地.git中代码，提交到远程库。
  使用命令：
  ``git remote add origin [远程仓库地址]``
  如果是首次使用，会提示输入用户名+密码，用户名一般是邮箱，输错是关联不成功的。
  关联成功则无提示，接着输入命令
  ``git push origin master``
  如果失败，很大可能是远程仓库已存在文件。可以执行
  ``git push -f origin master``强制提交。
  提交过程是能看到进度的。
  提交完成后可以去平台上查看有没有代码就知道是否成功。

__注意：所有命令行操作必须使用cmd或者git bash定位到项目目录下__

##### 仓库基本使用

提交过程：
``右键项目-git-add，弹框，选择是``；
这时候只是把代码添加到本地仓库，
``再右键项目选择commit  directory 在弹框的commit message中输入提交信息，选择commit and push``
然后会显示进度。
在多人协作开发一个项目的时候，提交之前一定要先pull一下（``VCS - pull``）,如果有冲突，选择合并或者是选择远程的，还是本地的，三者选一。
处理好这些再进行提交操作。

##### 分支管理

分支作为git一个重要的存在，可以进行版本回退，或者协作开发都是一个很便利的存在。
在创建仓库的时候，默认会有个master分支，如果是首次开发，则不需要创建分支。
但是在版本迭代的时候，特别是大版本迭代，就用到了分支，分支是相互独立存在的仓库。
互不影响，在克隆的时候切换一下分支，就会把不同分支下的仓库内容拷贝过来，就像1.0、2.0版本，是分开的，1.0在master分支，2.0版本在maste2分支，可以随时修改历史版本。

__常用命令

创建并切换到新建分支：
``git checkout -b master2``
切换分支：
``git checkout master2``
删除分支：
``git branch -d fmaster2``
将分支推送到远程仓库：
``git push origin <branch>``

##### 关于颜色

白色（正常色）：未改动或者没有仓库时的颜色。上
红色：未添加仓库的，在创建仓库时会出现。
绿色：已添加到本地仓库，没有进行commit push提交远程的。
蓝色：修改已经提交到本地仓库的代码。

##### 常见问题

> 有一种情况是提交/强制提交的时候出现

```
error: src refspec master does not match any.
error: failed to push some refs to 'https://github.com/wapchief/chat-room-JFrame.git'
```

说明是本地代码库为空
解决办法：
1、在项目中，如android studio，打开项目，右键->Git->+Add，然后重新右键->Git->commit Directory->commit and push->commit。执行之后会发现代码文件颜色都变成正常的白色，之后回到命令行执行提交操作
2、在本地仓库创建一个文件，推送到仓库
``touch README
git add README
git commit -m 'first commit'
git push origin master``
如果按照上面的步骤来的话是不会出现这种情况的，这种情况出现于已有现成的项目，并且本地项目的代码未提交到本地仓库。这时候提交到远程，就会判定你本地仓库为空。

> 补充一个在提交过程中出现无法解决问题的办法

如果在使用命令行操作时出现无法解决的错误，直接进入到项目文件，删除``.git``文件,然后右键该项目目录，或者使用cmd定位到该目录，重新执行

```
git init  #初始化本地仓库
git remote add origin [远程库地址]  #关联远程库
git add . #提交本地代码到本地仓库的暂存区
git commit -m '[提交说明]' #提交本地代码到本地仓库，并附上提交说明
git push -f origin master #强制推送到远程库
```

___

关于这些只是对于刚入门的学习者有些帮助，在我学习的时候也遇到了好多坑，至今有些问题还能遇到，但是不至于手忙脚乱，起码知道问题出在了哪个环节。
Git是一个很强大的版本控制工具，有很多功能，需要尝试去深入研究，希望学习者能够感受到他带来的便捷。

----

https方式每次都要输入密码，按照如下设置即可输入一次就不用再手输入密码的困扰而且又享受https带来的极速

设置记住密码（默认15分钟）：

```shell
git config --global credential.helper cache
```

如果想自己设置时间，可以这样做：

```shell
git config credential.helper 'cache --timeout=3600'
```

这样就设置一个小时之后失效

长期存储密码：

```shell
git config --global credential.helper store
```

增加远程地址的时候带上密码也是可以的。(推荐)

```shell
http://yourname:password@git.oschina.net/name/project.git
```

补充：使用客户端也可以存储密码的。

如果你正在使用ssh而且想体验https带来的高速，那么你可以这样做： 切换到项目目录下 ：

cd projectfile/
移除远程ssh方式的仓库地址

```shell
git remote rm origin
```

增加https远程仓库地址

```shell
git remote add origin http://yourname:password@git.oschina.net/name/project.git
```
