---
date: 2018-8-6 14:03:02
title: idea差点带走我可怜的C盘
no_word_count: false
tags: [技术,工具,idea]
---

>昨天下午搭环境，在idea里点了下更新maven`updating maven repository indices...`等了好久还没好，于是电脑没关机，下班走人了，万万没想到，今天早上来了之后。。。
<!--more-->
![](1.png "惊恐....")
又试下了下昨天的操作
![](2.png "吓死....")
还从没试过C盘爆了的情况...
害怕，赶紧终止了！！

>将IDEA的一些文件转移到D盘，修改idea的bin目录下有个一个idea.properties配置文件，默认是在用户目录下，改到D盘
```
#---------------------------------------------------------------------
# Uncomment this option if you want to customize path to IDE config folder. Make sure you're using forward slashes.
#---------------------------------------------------------------------
# idea.config.path=${user.home}/.IntelliJIdea/config
idea.config.path=D:/IDEA/.IntelliJIdea/config

#---------------------------------------------------------------------
# Uncomment this option if you want to customize path to IDE system folder. Make sure you're using forward slashes.
#---------------------------------------------------------------------
# idea.system.path=${user.home}/.IntelliJIdea/system
idea.system.path=D:/IDEA/.IntelliJIdea/system

```
>重启IDEA ，这时IDEA的配置目录是空的，选择导入已有的配置，也就是原来C盘用户目录下的`.IntelliJIdea/config`即可恢复原有的配置。

>maven本地仓库顺手也挪了一下，实在是怕了。
>修改maven /conf/setting.xml
```xml
<localRepository>D:/java/apache-maven-3.5.0/repository</localRepository>
```
>IDEA `contrl+atr+s` 搜maven ，maven home directory:选择使用自己安装的maven，use setting file:选择刚改的setting.xml





