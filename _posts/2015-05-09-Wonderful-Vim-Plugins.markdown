---
layout: post
comments: true
title: 好用的Vim插件
---

分享三个最近在工作中使用较多, 能提高效率的Vim插件.

## 1. Ag

Ag本身(又叫[the silver searcher](https://github.com/ggreer/the_silver_searcher))并不是一个Vim插件, 而是一个代码搜索工具. 其特点是使用了非常高效的搜索算法, 同时可以通过配置文件对需要搜索的文件进行过滤以进一步提供搜索效率. 在实际使用中Ag比grep和ack快一个数量级. 另外还有一个被称为[the platinum searcher](https://github.com/monochromegane/the_platinum_searcher)的项目, 其搜索速度略微快过Ag. 考虑到Ag已经是一个非常成熟稳定的项目, 我还是选择了使用Ag.

在类Unix系统上安装Ag比较简单, 在Ag的[github首页](https://github.com/ggreer/the_silver_searcher)上有详细的介绍.
在Windows平台上需要自己从源码编译. 如果不想这么麻烦可以使用[Chocolatey](https://chocolatey.org/)这个支持Windows的Package Manager进行安装, 不过Chocolatey上的Ag版本可能会过时. 安装好Ag后在命令行中运行`ag some_code /path/to/project`就可以很快得到匹配的结果.

在安装好Ag后再安装上[ag.vim](https://github.com/rking/ag.vim)这个插件, 就可以很方便的在Vim中通过命令`:Ag [options] {pattern} [{directory}]`搜索想要的代码. 所有搜索到的代码都会按文件和行号排序并显示在Vim的quick-fix窗口中. ag.vim还提供若干命令方便用户快速的在新窗口(或者tab页)中打开在quick-fix窗口中所选择的代码位置.

为了进一步方便在Vim中使用Ag, 可以在vimrc文件中添加`nmap <c-t> :Ag! ""<left>`. `Ag!`使得你在搜索代码时Vim不会默认直接打开搜索的第一个结果.

根据所在的平台和Vim版本, 使用ag.vim时可能会遇到一个搜索结果被泄露到terminal中的bug. 具体表现是在搜索过程中所有搜索到的结果会在terminal中刷出来然后再切换回Vim显示正常结果. 为了解决这个问题可以将autoload/ag.vim中执行
`silent! execute a:cmd . " " . escape(l:grepargs, '|')`之前设置好对应变量:

```
  let l:grepprg_bak=&grepprg
  let l:grepformat_bak=&grepformat
  let l:t_ti_bak=&t_ti
  let l:t_te_bak=&t_te
  let l:shellpipe_bak=&shellpipe
  try
    let &shellpipe="&>"
    let &grepprg=g:ag_prg
    let &grepformat=g:ag_format
    set t_ti=
    set t_te=
    silent! execute a:cmd . " " . escape(l:grepargs, '|')
  finally
    let &grepprg=l:grepprg_bak
    let &grepformat=l:grepformat_bak
    let &t_ti=l:t_ti_bak
    let &t_te=l:t_te_bak
    let &shellpipe=l:shellpipe_bak
  endtry
```

## 2. CtrlP

Sublime的一个王牌功能就是通过ctrl+p快捷键进入文件名模糊搜索模式. 能快速找到项目内需要编辑的文件. 在Vim中有一个名叫[CtrlP](https://github.com/kien/ctrlp.vim)的插件可以提供相同的功能.

安装好Ctrlp后使用相当简单, 按下ctrl+p后CtrlP会列出当前项目内的所有文件(及路径)名. 同时提供一个命令行供用户输入要搜索的文件名. 如图:

![CtrlP]({{ site.url }}/assets/wvp_ctrlp.png)

用户可以通过`ctrl+j`和`ctrl+k`上下选择要打开的文件, 通过`enter`在当前window中打开文件, 通过`ctrl+o`选择打开文件的方式.
在开始使用CtrlP以后我就很少再使用Nerdtree了. 不过Nerdtree在浏览一个新的项目的代码时还是非常重要的, 能够方便我们快速熟悉该项目的代码结构.

## 3. Syntastic

[Syntastic](https://github.com/scrooloose/syntastic)是一个用来检查程序语法的Vim插件, 支持的语言非常之多, 其原因是Syntastic本身并不做语法检查, 而是根据文件类型调用对应的外部语法检查工具进行语法检查, 然后再将结果以统一的方式显示在Vim中. 举例来说Syntastic可以使用flake8来检查Python语法. 首先安装flake8:

```bash
pip install flake8
```

然后在使用Vim编辑python文件时, 每当用户保存源代码文件时Syntastic都会自动调用flake8对代码进行语法检查, 如图所示![flake8]({{ site.url }}/assets/wvp_flake8.png).

对于Python这样的动态语言, 语法检查能够在实际运行程序之前发现不少错误, 节省开发时间.
