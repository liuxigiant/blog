---
title: "Octopress迁移到Hexo"
catalog: true
date: 2018-11-18 18:51:06
subtitle:
header-img:
tags:
- Personal
---

## 1 起因

某次上网查找问题，找到一个个人博客的文章，偶然发现这也是一个静态博客，用hexo生成挂到了github pages上

后面上网搜了下hexo相关的文章，发现Hexo搭建的博客整体样式比Octopress好看很多，受众和文档也比较多

然后开始折腾把Octopress迁移到Hexo

## 2 迁移

### 2.1 选择主题

迁移前，先去[hexo官网](https://hexo.io/zh-cn/)选一个主题，或者上网搜索选一下主题

本博客现在使用的主题其实就是上面个人博客作者创建的，后来发现在hexo官网的主题中也有，链接如下：
[https://github.com/huweihuang/hexo-theme-huweihuang/](https://github.com/huweihuang/hexo-theme-huweihuang/)

### 2.2 基础命令安装

主要需要安装一下三种命令：

- npm: 前端开发应该很熟悉，搜索下node安装即可
- git: 这里主要是提到构建好的博客到GitHub Pages仓库的，主流版本管理工具，安装方式就不废话了
- hexo: 用于构建、生成、发布静态博客，安装命令 npm install -g hexo-cli

### 2.3 初始化Hexo本地博客目录

上面选择好的主题，从github上git clone到本地之后，就可以当作本地的博客的目录了

> 若不选择主题，直接搭建hexo本地博客目录，则需要执行hexo init初始化；
> 
> 而从github上clone下来的的已有的主题不需要初始化，按照主题的使用说明，**修改自己的配置**即可

### 2.4 迁移

其实迁移很简单，就是将原来_post下面的markdown 文件复制到现在的_post目录下即可

> 若存在相对路径的images文件，也拷贝到相应的目录下即可

### 2.5 构建静态博客

其实和Octopress差不多，只不过在使用hexo命令生成静态博客之前，需要执行npm install 命令安装依赖

> 执行npm相关命令，可指定或配置国内镜像，例如：npm install --registry=https://registry.npm.taobao.org

hexo相关命令：

- hexo new post-title ：新建文章
- hexo clean ：清理本地构建临时文件
- hexo generate ：生成静态文件
- hexo server ：本地预览，可在浏览器查看生成的博客
- hexo deploy ：发布到github pages 

## 3 问题

最后记录下一些问题

### 3.1 新建文章的文件名

从Octopress迁移过来的，希望markdown源文件名和以前的风格保持一致

修改根目录下的`_config.yml`文件中的`new_post_name`配置下，根据以前的习惯即可，我的配置是

```
new_post_name: :year-:month-:day-:title.md # File name of new posts
```

### 3.2 博客文章的链接


从Octopress迁移过来的，希望生成的博客文章的链接和以前一致

修改根目录下的`_config.yml`文件中的`permalink`配置下，根据以前的习惯即可，我的配置是

```
permalink: blog/:year:month:day/:title.html
```

### 3.3 博客文章目录TOC链接跳转失效

本人使用的[Hexo主题](https://github.com/huweihuang/hexo-theme-huweihuang/)中会有TOC目录链接失效，点击不跳转的问题，从生成的静态博客html源码来看，超链接是undefined，根据其[github issue](https://github.com/huweihuang/hexo-theme-huweihuang/issues/1)说是音乐播放器的问题，但是我禁用了播放器，重新构建后仍然不行。

后来当前主题也是从另外一个主题[beantech](https://github.com/YenYuHsuan/hexo-theme-beantech)的基础上修改的，找到一个issues [Toc feature seems not working](https://github.com/YenYuHsuan/hexo-theme-beantech/issues/19)，其解决方式如下：

```
open node_modules/hexo-toc/lib/filter.js under your project root directory and modify line 28 to 31 as follows
```

至此，虽然解决了TOC链接失效的问题，但是开启音乐播放器配置，TOC仍然失效，目前只好关闭这个配置了（可关注主题作者关于音乐播放器的issue）
