---
title: GitHub+Hexo搭建个人博客
no_word_count: false
tags: [Hexo,搭建]
---

>看到一句话，**实践是最好的成长，发表是最好的记忆**。
作为一个程序员，我们需要学习的东西太多，每天不是在码代码就是在学习怎么码代码，很多东西学习完之后，可能一段时间不用就会忘记，下次在用可能又是一脸懵逼，找不到思路。
怎么办呢？我们需要有个地方把这些保存下来，你可以选择笔记，有道云笔记，印象笔记都很不错，但是这些笔记都是属于你自己的，别人看不到，就没法和你交流沟通，有些东西在我们和他人交流时才能理解的更加透彻，真正掌握。
那么博客是个不错的选择，你可以选择博客园，CSDN，简书等博客网站，巴特，在这些博客网站上我们只能按照他们的套路去操作，无法做一些个性化的操作，受限于人的滋味不好受，怎么办呢？
嗖,建一个自己的博客网站。
建网站需要有个云服务器，可是服务器那么贵，一个入门配置每年都要好几百，对于我们这种人要每年付那么高的费用显得太奢侈，那么代码圣地GitHub可以帮我们

## 首先

要准备的东西

- [GitHub](https://github.com)账号
- [node.js](https://nodejs.org/en/)
- [Hexo](https://hexo.io/)
- [git](https://git-scm.com/)
- markdown编辑器 [语法](https://www.appinn.com/markdown/)
- 域名(如果不想使用github的域名)*[万网](https://wanwang.aliyun.com/)*

<!-- more -->

### 一、[Hexo](https://hexo.io/)

安装

```bash
npm install hexo -g
```

建一个新的文件夹blog(名字随意)
切换到blog目录下执行初始化，这个过程有点慢，hexo会自动下载很多东西 

```
hexo init
```

如果初始化报什么包的问题，让你再次运行 hexo init 别急 清理一下在执行

```
npm cache clean -force
```

初始化成功以后，生成静态页面

```
hexo g
```

以上操作都成功执行完成以后，启动服务

```
hexo s
```

访问地址:http://localhost:4000

这时我们便看到我们的博客环境已经运行起来。在本地跑有什么用？别急，先看看我们的hello world 是怎么运行的，一切都是由简入繁。

在我们的blog目录下Hexo帮我们生成了很多文件

>node_modules： hexo需要的一些模块
source：  点开发现sourc\\_posts\下我们的hello-world.md正静静的躺在这里，打开一看正是我们页面显示的内容，那么我们是不是可以猜测，这个源文件夹就是我们以后写博客的位置，不信的话后面改改试试。
themes： 主题，hexo提供了很多的主题供我们选择，这个文件夹正是存放主题的地方。
_config.yml： 主配置文件，为什么说主配置，那么肯定不只一个配置文件，在themes下也有一个_config.yml这个是针对当前主题的配置。


继续，当我们执行了`hexo g` 后，发现blog目录下又多了一个public文件夹，我们知道 `hexo g` 是生成静态页面的，那么不用想这个public文件夹一定有我们的页面.

好了，现在我们环境有了，服务也起来了，也知道在哪写东西，写的东西生成到哪了。

现在想想，怎么让我们的博客页面在外网可以访问到吧？
下面将我们生成的静态文件上传到GitHub上


### 二、[GitHub](https://github.com)
	
- New repository 

>注意名称格式：**账号名.github.io**

- 修改blog下的_config.yml文件

>deploy:
  type: git
  repo: 刚创建的仓库地址
  branch: master

然后安装部署模块

```
npm install hexo-deployer-git --save
```

执行部署

```
hexo d
```

这一步如果一切正常，那么你的public目录下的文件都会被长传到GitHub上面，同时blog目录下将会有一个*.deploy_git*目录

如果不幸遇到问题，那么就要一步一步解决了。
我当时遇到以下几个问题：

- 找不到git (没配git的环境变量)
- 代理问题。试试执行这个
```npm --registry https://registry.npm.taobao.org info underscore```

- git凭证问题 更新这个证书管理 [下载地址](https://github.com/Microsoft/Git-Credential-Manager-for-Windows/releases/)

- 如果还有奇葩问题，试试万能的 ```npm cache clean -force``` 命令

- 再有其他的坑我就不知道了

当填完这一系列坑之后 访问：https://账号名.github.io

此时大功告成，我们的博客已经可以在外网访问了！

### 如果要绑定域名，往下看

- 万网申请域名 需要花点小钱
- 解析域名 (现在域名必须实名认证后才能解析)
	- 添加A记录 
		- ping 账号名.github.io 得到ip
		- 一般配置两个 一个www 和一个不需要www的，这样我们的域名不管输不输www就都能被解析了
- 在source目录下 创建CNAME文件*(注意不要有扩展名)*编辑文件，写上你刚申请的域名保存即可。
- 在执行一下 `hexo g` 和 `hexo d` 命令将CNAME文件上传到GitHub上
- 在你的GitHub上找到新创建账号名.github.io仓库，打开Settings，找到GitHub Pages配置项，Custom domain配置中填入你的域名保存。
- 如果出现绿条 恭喜你 域名绑定成功。


***

### 以上搞定以后 便要做一些个性化设置了。

- 仔细看看_config.yml 和 主题下的_config.yml 文件，猜猜各个配置都是干什么的，不停的试试，就会知道了，或者去官网看看配置文档。
- 主题 就是模板，我们的页面既然是通过模板静态化的，那么想要在页面里加些东西，自然要研究一下这些模板，在模板上做改动
- 列几个扩展的功能，今天没时间了，扩展部分就不写了
 - 加入评论功能
 - 添加背景音乐
 - 博客阅读量
 - 百度监测
 - 搭建相册功能


### 总结
>markdown 用了才发现，真的挺好用的
在自己搭建的博客上的处女作，有些啰嗦，内容不够简练
