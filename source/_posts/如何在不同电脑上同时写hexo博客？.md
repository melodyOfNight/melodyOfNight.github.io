---
title: 如何在不同电脑上同时写hexo博客？
date: 2018-10-08 16:12:04
tags:
---
相信很多人都有这样的困扰，我在家里搭了hexo博客，配置了很久主题、样式等等，到了公司，想在公司写博客的时候，发现无从下手。从github上pull下来的都是静态文件，无法在原有的基础上继续进行博客编辑。我查阅了网上的资料，但是严格按网上的步骤没有成功，遇到很多坑，自己摸索了一套做法。但是核心思想没有改变，还是利用github的多分支功能。

<!-- more -->

一般情况下，hexo部署到github上的都是静态文件，即生成好的html文件。也就是说，如果pull下来，只能得到一些静态页面，而博客的样式或md文件是无法pull下来的。那么在另外一台电脑上进行部署的话，原来的博客样式就会被清空。这是我们不想看到的。

也有一种比较原始的办法，就是把该项目的目录完整拷贝一份，在不同机器上都粘贴一份。然后通过手动备份，保持不同电脑上文章目录同步(否则在不同电脑上部署会导致原来写的文章没有了)。虽然可以实现需求，但是确实有点麻烦，也没有真正利用到github。

为了完美解决这个问题，可以**利用github的不同分支来分别保存网站静态文件与hexo源码（md原始文件及主题等）**，实现在不同电脑上都可以自由写博客。

本文是建立在你已有hexo博客的情况下，教你如何把它迁移到另外一台电脑上，使两台电脑同时可以进行博客部署。下面就具体讲解一下如何实现吧！

*注：假设你有工作PC和个人PC，你原有的博客搭建在个人PC上，以下首先介绍个人PC上的操作步骤。*

## 个人PC

### 在github上新建远程仓库

将原来的page项目删除，新建一个和原来名字一样的空项目。不用初始README.md
此时只有一个空的master分支。

本地的目录不要动，到时候需要用你原来博客的配置和文章替换该项目中的文件夹。你可以把你本地的博客目录复制一份作为备份，以免误操作将原来的内容进行改动。

### 本地初始化一个Hexo项目

新建一个空目录，作为你的博客目录。进入该目录，右击Git bash here，初始化一个Hexo项目：

```
hexo init
npm install
npm install hexo-deployer-git --save

```
然后用自己原来博客里的文件替换掉这里的``source\``, ``scaffolds\``, ``themes\``,`` _config.yml``替换成自己原来博客里的。注意，这里把``themes/next``中的``.git/``目录删除，否则下面推送到hexo分支后jacman为空。这也是众多坑中的一个，由于水平有限具体原因不是很清楚，只能瞎猫碰上死耗子啦。

### 将整个目录推送到master
要推送到master分支，首先要将该目录初始化为本地Git仓库：

```
git init
 
//把博客目录下所有文件推送到master分支
git remote add origin git@github:chown-jane-y/chown-jane-y.github.io.git
git add .
git commit -m "first add hexo source code"
git push origin master
```

###在github上新建一个分支

新建一个分支hexo(名字可以自定义)，这时候hexo分支和master分支的内容一样，都是hexo的源文件。

并把hexo设为默认分支，这样的话在另外一台机器上克隆下来就直接进入hexo分支，并且以后所有操作都是在hexo分支下完成。

为什么需要这个额外的分支呢？

因为hexo d只把静态网页文件部署到master分支上，所以你换了另外一台电脑，就无法pull下来继续写博客了。有了hexo分支的话，就可以把hexo分支中的源文件(配置文件、主题样式等)pull下来，再hexo g的话就可以生成一模一样的静态文件了

###部署博客

还是和以前一样：

```
hexo g -d
```

博客已经成功部署到master分支，这时候到github查看两个分支的内容，hexo分支里是源文件，master里是静态文件

**注意：根目录下的_config.yml配置文件中branch一定要填master，否则hexo d就会部署到hexo分支下。**

###关联到远程hexo分支

在本地新建一个hexo分支并与远程hexo分支关联：

```
git checkout -b hexo
git pull origin hexo

```

另外别忘了，如果有修改的话，要推送到hexo分支上去：

```
git add .
git commit -m  ""
git push origin hexo

```
这样才能在另外的机器上pull下来，保持同步

##工作PC

个人PC上的工作已经完成了，下面讲一下如果你换到了另外一台电脑上，应该如何操作。

###将博客项目克隆下来

```
git clone git@github.com:chown-jane-y/chown-jane-y.github.io.git

```

克隆下来的仓库和你在个人PC中的目录是一模一样的，所以可以在这基础上继续写博客了。但是由于``.gitignore``文件中过滤了``node_modules\``，所以克隆下来的目录里没有``node_modules\``，这是hexo所需要的组件，所以要在该目录中重新安装hexo，**但不需要hexo init**。

```
npm install hexo
npm install
npm install hexo-deployer-git --save

```

### 新建一篇文章测试

```
hexo new "work PC test"

```
###推送到hexo分支

```
git add .
git commit -m "add work PC test"
git push origin hexo

```

###部署到master分支

```
hexo g -d

```

------------------------------

##日常操作

如果上面的过程都操作无误的话，你就可以在任何能联网的电脑上写博客啦。一般写博客的流程是下面这样。

###写博客前
不管你本地的仓库是否是最新的，都先pull一下，以防万一：

```
	
git pull origin hexo

```

把最新的pull下来，再开始撰写新的博客。

### 写博客

```
hexo new "title"
```
然后打开source/_posts/title.md，撰写博文。

### 写完博客
先推送到hexo分支上：

```
git add .
git commit -m "add article xxx"
git push origin hexo
```

最后部署到master分支上

```
	
hexo g -d

```

整个流程大概就是这样。