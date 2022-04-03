---
layout: post
comments: true
---

Bash Bootstrap
==============

需求
----
每次启动terminal开始某项工作的时候都要执行一串机械的操作。例如进入指定的文件夹，启动特定的python environment等：

```bash
$ cd ~/Projects/some/project/path
$ source pyenv/bin/activate
$ ...
```

对与一个程序员来说，任何重复执行的机械操作都应该让电脑来做。于是我尝试了如何通过bash来完成这些准备工作。

使用bash脚本
-----------
最简单的方法就是将这些代码放到一个bash脚本中：

```bash
#!/bin/bash
cd ~/Projects/some/project/path
source pyenv/bin/activate
...
```

然后在启动terminal之后执行`source some_project.sh`就可以了。

这个方法简单直接，如果在某段时间内手里只有一个或者很少的几个项目，那么这样的方法就足够了。但是假设我们有很多项目同时在工作，这个方法就有点过于简单了。不同项目的bootstrap脚本遍布在文件夹内很不好管理，而且从一个项目切换到另一个项目也不够方便。受到virtualenvwrapper的启发，我向bash添加了一个workon命令来管理所有的bootstrap脚本。

Bash Bootstrap
--------------
我们的目标是当我们想开始某个项目的工作时，只需要输入

```bash
workon some_project
```

就可以自动执行相应的bootstrap脚本。

首先是向bash环境添加workon命令，将如下代码添加到`~/.bashrc`文件内即可:

```bash
function workon {
}
export -f workon
```

当然现在的workon命令还不会执行任何动作。接下来我们要让workon命令根据参数去调用相关的bootstrap脚本。我选择在`home`目录下建立了一个存放bootstrap脚本的文件夹：`.bash_bootstrap`。然后将各个项目的bootstrap脚本放入这个文件夹即可。

接着我们要让workon根据参数去调用`~/.bash_bootstrap`文件夹内相应的脚本。

```bash
function workon {
  file=~/.bash_bootstrap/"$1".sh

  if [ -f $file ]; then
    source $file
  else
    echo "$file is not found"
  fi
}
export -f workon
```

这样，当我们输入`workon some_project`的时候，bash就会自动执行该项目的bootstrap脚本了。

除了这些简单的代码以外，我们还可以添加参数检查，根据参数来调整bootstrap执行的细节等，这些都可以根据大家实际工作的项目情况来完善。基本的Bootstrap工作便介绍完了，如果你有更好的方法也欢迎通过github或者邮箱告诉我。
