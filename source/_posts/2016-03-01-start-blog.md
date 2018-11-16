---
layout: post
title: "github上搭建个人博客 -- WIN7"
description: octopress博客搭建问题及优化总结
date: 2016-03-01 19:37:50 +0800
catalog: true
categories:
- Personal
tags:
- Personal
---

## 博客搭建

**参考：** [http://www.cnblogs.com/oec2003/archive/2013/05/27/3100896.html](http://www.cnblogs.com/oec2003/archive/2013/05/27/3100896.html)

**以下整理遇到的问题：**

- Ruby下载地址异常  
	附上一个1.9.3版本的csdn的下载链接：  
	[http://download.csdn.net/detail/midsummer411/6760211](http://download.csdn.net/detail/midsummer411/6760211)
	
	**特别提示：** 最好不要上[官网](http://rubyinstaller.org/)下载2.0.0版本的，我一开始就使用2.0.0版本，然后按照文章中的步骤一步步进行，出了一系列错误，搞定不了，后来换了个1.9.3版本的就OK了 

- 安装DevKit  
	参考文章中说道DevKit下载下来的是一个自压缩文件，其实就是个.exe文件，执行就好
	执行完 ruby dk.rb init 后配置config.yml文件，执行ruby的安装目录

- 安装Python后安装easy_install  
	Python安装没问题，直接略过，主要说说easy_install的安装，文章中给的链接 https://pypi.python.org/pypi/setuptools 不是安装包的下载链接，你可以在打开的页面中找 `Windows(simplified)` 章节，然后点击 [ez_setup.py](https://bootstrap.pypa.io/ez_setup.py) 下载
	
	easy_install命令的执行也不是必须配置环境变量，直接cd到Python安装目录的script目录下执行就好

- gem更新源更换成淘宝镜像  
	文章中给出的淘宝的http链接需要更换成https的  
	执行以下两条命令：(*具体可参考[淘宝RubyGems镜像](https://ruby.taobao.org/)*)  
	gem sources --add https://ruby.taobao.org/ --remove https://rubygems.org/  
	gem sources -l
	
<!-- more -->

- bundle install失败  
	参照第一条

- rake setup_github_pages报错  
	Errno:: ENOENT: No such file or directory - git remote -v  
	在path环境变量中配置path的路径  
	参考自：  
	http://stackoverflow.com/questions/12400185/octopress-blog-deploying-error-rake-aborted-no-such-file-or-directory-git-re

- rake deploy失败  **! [rejected]        master -> master (non-fast-forward)**  
	cd octopress/_deploy  
	git pull origin master  
	cd ..  
	rake deploy

## 博客优化

**参考：** [http://shengmingzhiqing.com/blog/octopress-tutorials-toc.html/](http://shengmingzhiqing.com/blog/octopress-tutorials-toc.html/)  

**以下整理遇到的问题：** 
 
- 侧边栏添加分类报错  
	Liquid Exception: incompatible encoding regexp match (ASCII-8BIT regexp with UTF-8 string)in _layouts/page.html  
	在新建的category_list_tag.rb 文件第一行增加声明 # encoding: utf-8

- rake deploy后master分支下的CNAME文件被删除
	在使用Octopress的时候，每次 rake generate , rake deploy 后，master分支下面的CNAME文件消失了。  
	正确的做法是，把CNAME文件放到在 source 目录下，其余的都删掉， rake generate 会自动拷贝到public目录下， rake deploy 再拷贝public目录内容到_deploy目录，并提交到master分支。  

- 代码高亮问题  
  尝试代码高亮，各种报错： `jekyll 2.5.3 | Error:  Pygments can't parse unknown language: </p>.`  

  **解决方案：**可以在`plugins/pygments_code.rb`文件增加输出报错的文件位置，增加代码是`#{code}`，然后执行`rake generate`时候就能看出具体的报错，针对具体的位置修改（可以通过将错误日志输出到文件`rake generate > log`，方便查看我问题）  

```
rescue MentosError
  raise "Pygments can't parse unknown language: #{lang}#{code}."
end
```
  
 