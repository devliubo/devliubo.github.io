---
layout: post
title: "配置git多SSH-Key共存"
date: 2015-04-16 21:41
comments: true
author: devliubo
categories: git
tags: "git"

---

练习下写总结，整理下以前在配置git过程中的SSH-Key共存和移除submodule的方法~

<!-- more -->

#### 1.git的入门学习
推荐廖雪峰的网站: [http://www.liaoxuefeng.com](http://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000)

#### 2.关于多个SSH Key的共存
在使用git过程中，会遇到配置多个ssh-key的共存情况，比如一个连接公司的git，一个用来连接github，或者两个github账户。这里以github和oschina两个ssh-key共存举例。

首先配置github，生成ssh-key

{% blockquote %}
ssh-keygen -t rsa -C “aaa@gmail.com” -f ~/.ssh/github_id_rsa
{% endblockquote %}

过程中会要求设置密码，直接回车为空就可以了~

此时会生成两个文件github_id_rsa和github_id_rsa.pub
为了方便区分多个ssh-key，这里我们指定名为github_id_rsa，如果不指定会按照生成默认的id_rsa

由于在默认情况下，SSH agent只会去读取id_rsa，为了使新生成的github_id_rsa能被SSH agent读取，将github_id_rsa添加到SSH agent

{% blockquote %}
ssh-add ~/.ssh/github_id_rsa
{% endblockquote %}

可以查看生成的ssh-key，将ssh-key粘贴到github上

{% blockquote %}
vim ~/.ssh/github_id_rsa.pub
{% endblockquote %}

用相同的方法生成oschina的ssh-key，并粘贴到oschina上

之后为了让两个ssh-key共存，在~/.ssh下生成一个config文件

{% blockquote %}
sudo vim ~/.ssh/config
{% endblockquote %}

通过config文件指定不同的私钥对应的不同git服务器

{% codeblock %}
	#GitHub(aaa@gmail.com)
	Host github.com
	HostName github.com
	User git
	IdentityFile ~/.ssh/github_id_rsa
	
	#OSChina(bbb@gmail.com)
	Host git.oschina.net
	HostName git.oschina.net
	User git
	IdentityFile ~/.ssh/oschina_id_rsa
{% endcodeblock %}

然后可以测试下是否成功连接

{% blockquote %}
ssh -T github.com
ssh -T git.oschina.net
{% endblockquote %}

过程中会问你是否添加到knownhosts，yes即可，会在~/.ssh目录下生成一个known_hosts文件

**需要注意一点的是，git服务一般会根据配置文件的user.name和user.email来获取作者信息(比如上面的github的aaa@gmail.com和oschina的bbb@gmail.com)，如果多账户信息不同的话，需要注意在使用前修改配置。**

{% codeblock %}
	#查看配置信息
	git config --list
	
	#设置全局的name和email
	git config --global user.name "xxx"
	git config --global user.email "xxx.gmail.com"
{% endcodeblock %}

#### 3.git的submodule简单记录

增加一个submodule到项目比较简单，添加submodule到submodulePath目录下

{% blockquote %}
git submodule add git@github.com:submodule.git submodulePath
{% endblockquote %}

要特别记住的是：**git是根据父项目中保存的submodule的commit id来跟踪submodule项目变动的，在多人项目中，pull和push时一定要注意父项目的commit id和submodule的对应**

删除submodule要复杂点需要清除以下几个地方

{% codeblock %}
	#移除submodule项目及对应目录
	git rm --cached submodulePath
	git rm -rf submodulePath
	
	#删除掉下面两个文件中要移除的submodule相关信息
	vim .gitmodules
	vim .git/config
	
	#将删除submodule的更改提交
	git add --all
	git commit -m “remove submodule"
{% endcodeblock %}


*Ps:小索问我现在在北京觉得幸福么？被问得整个人都不好了，嗯，，，是个问题哈。。。。。。*