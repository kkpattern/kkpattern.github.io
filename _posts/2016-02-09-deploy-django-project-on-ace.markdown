---
layout: post
comments: true
title: 在阿里云ACE上部署Django应用
---

PaaS提供了一种方便快捷的web部署方式. 通过支持特定的语言, 用户只需简单的上传自己的代码就可以让自己的网站运行在云端. 然而PaaS在提供极大的方便性的同时又牺牲了一定的灵活性. 由于不能自己随意部署环境, 不能随意运行终端命令, 因此在PaaS上部署一些已有的web框架时显得不够直接, 尤其是在牵涉到数据库和静态文件时. 本文将介绍一下在阿里云引擎ACE上部署Django的流程.

本文使用的Django版本为1.9, 为了方便演示, 使用了[Django tutorial](https://docs.djangoproject.com/en/1.9/intro/tutorial01/)中的poll项目.

1. 在ACE上创建应用

   首先在[ACE](http://ace.console.aliyun.com/console#/home)上创建一个应用.

   应用创建成功以后创建我们应用的第一个版本, 具体的步骤可以参考ACE的[官方文档](https://help.aliyun.com/document_detail/ace/quick-start/python/creat.html?spm=5176.product8314992_ace.6.100.R9Ja2j).


1. 初始化代码仓库

   在ACE上创建好应用后我们需要在本地创建一个仓库用于管理和发布我们的项目代码. 本文中使用了git来管理项目的代码, 而ACE则使用了svn.

   1. 将阿里云上的应用svn checkout到本地, svn的地址可以在ACE的应用管理页面的版本管理一项中看到.

      ```bash
      svn co http://repo1.svn.ace.aliyun.com/svn/xxxx/xxx/1
      ```

   1. 在checkout下来的svn仓库中初始化git仓库.

      `git init`

      将poll项目的git仓库设置为此git仓库的origin remote.

      `git remote add origin ~/path/to/project/directory`

      这样我们就可以方便的将poll项目的代码更新到ACE的svn仓库中.

      `git pull origin master`

1. 设置ACE所需的环境

   1. 创建`index.py`文件
      为了将我们的django项目部署到ACE上, 我们还需要添加一些额外的代码和文件. 我们首先在git中创建一个`deployment`分支, 将这些文件添加到该分支中方便管理.

      `git checkout -b deployment`

      ACE上的python项目都需要在项目根目录中提供一个`index.py`文件, 这个文件其实是标准的[WSGI](https://en.wikipedia.org/wiki/Web_Server_Gateway_Interface) client, 用于将网络请求的相关信息转交到对应的python应用程序处理, 并将应用程序返回的回复转发给请求者. 在django 1.9项目中, 在项目初始化的时候已经提供了好了这个文件, 名叫`wsgi.py`放置在项目的主目录下, 我们的项目叫`ACEDjango`因此我们执行如下的命令将该文件复制到根目录中并重命名为`index.py`.

      `cp ACEDjango/wsgi.py index.py`

   1. 创建`requirements.txt`文件
      为了告诉ACE服务器安装必须的python库, 我们需要在项目根目录中创建一个`requirements.txt`文件. 对于我们的项目, 该文件的内容为:

      ```
      mysql-python
      django==1.9
      ```

      因为我们稍后需要访问ACE自带的MySQL协议的云数据库, 因为我们需要安装`mysql-python`这个库. 同时我们通过`django==1.9`告诉ACE我们需要安装1.9版本的django.

   1. 收集静态文件
      ACE提供高效, 简单的部署静态文件的功能, 在第一次checkoutACE项目的svn时, 项目自带的app.ymal文件中已经默认设置了一个`static`文件夹用于部署静态文件. 用户也可以设置自己的静态文件目录和相关参数, 具体文档见[这里](https://help.aliyun.com/document_detail/ace/quick-start/python/code.html?spm=5176.docace/ext-reference/python.6.101.MBcXPE). 本文将直接使用默认的`static`目录.

      在`ACEDjango/settings.py`中设置`STATIC_ROOT`的位置为项目目录下的`static`目录中, 注意需要使用绝对路径. 设置好路径后执行`python manage.py collectstatic`命令将项目内的静态文件都收集到该目录中.

      此时的目录结构

      ```bash
      ├── ACEDjango
      │   ├── __init__.py
      │   ├── settings.py
      │   ├── urls.py
      │   └── wsgi.py
      ├── app.yaml
      ├── index.py
      ├── manage.py
      ├── polls
      │   ├── __init__.py
      │   ├── admin.py
      │   ├── apps.py
      │   ├── migrations
      │   ├── models.py
      │   ├── static
      │   ├── templates
      │   ├── tests.py
      │   ├── urls.py
      │   └── views.py
      ├── requirements.txt
      └── static
      ```

   1. 设置数据库
      接下来需要设置数据库用于访问, 在`ACEDjango/settings.py`中修改数据库相关设置如下：

      ```
      DATABASES = {
          'default': {
              'ENGINE': 'django.db.backends.mysql',
              'NAME': 'XXXXX',
              'USER': 'XXXXX',
              'PASSWORD': 'XXXXXXXXXX',
              'HOST': 'XXXXX.mysql.rds.aliyuncs.com',
              'PORT': 3306,
          }
      }
      ```

      连接数据库所需的相关信息可以在ACE管理控制页面的扩展信息数据库页面中, 注意`NAME`一项不是数据库的实例名.
      在设置好数据库中参数后执行`python manage.py migrate`初始化数据库.

   1. 创建管理用户
      为了管理我们的poll应用, 我们需要创建一个管理用户, 运行`python manager.py createsuperuser`命令将在ACE的扩展数据库中创建一个管理用户.

1. 上传代码并发布
   首先通过`git commit`将新添加的代码提交到git仓库的`deployment`分支上.
   最后通过`svn commit`将项目代码上传到ACE中, 在ACE的应用管理页面中点击发布.
   现在进入应用的管理页面, 输入刚刚创建的管理用户账号和密码即可登录并管理我们的应用.

1. 更新代码并发布
   当我们的项目代码更新以后, 我们只需要在git仓库的master分支上更新最新的项目代码, 然后在`deployment`分支上更新并提交到svn分支上即可重新分布. 具体命令如下:

   ```
   git checkout master
   git pull origin master
   git checkout deployment
   git rebase master
   svn commit
   ```

至此我们便完成了django项目在阿里云ACE上的部署, 其中数据库和静态文件都使用了ACE附带的扩展应用.
