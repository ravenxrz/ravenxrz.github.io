---
title: Qt 配置+fakevim自定义
categories: 杂文
tags: vim
abbrlink: afbe8180
date: 2020-08-10 19:29:53
---

项目原因，要写点qt代码。记录下个人配置：

<!--more-->

1. 高分屏下字体过小问题

![](https://pic.downk.cc/item/5f1a50e514195aa594c27812.jpg)

2. 上下文帮助菜单（即快捷键F1）字体过小问题：

目前在manjaro-gnome上通过更改字体大小无效，只能更改渲染引擎，然后再更改字体大小，才能生效。

![](https://pic.downk.cc/item/5f1a512214195aa594c29044.jpg)

3. fakevim配置

![](https://pic.downk.cc/item/5f1a524414195aa594c3099c.jpg)

注意右边有个传递Control按键，开启这个选项后，能够使系统自带的一些快捷键（如Ctrl+R,Ctrl+O)和vim的一些特性共同使用。 但是就我个人而言，这样会失去一些vim的快捷键，所以未开启。

如果不开启传递Control， 那么Ctrl+R，Ctrl+F等常用qtcreator的快捷键又无法使用，好在fakevim支持vimrc，也支持定义ex command，所以可以自行配置vimrc来做键位mapping。下面以 “Run“命令为例，讲解如何配置vimrc。

打开Fakevim的ex command mapping:

![](https://pic.downk.cc/item/5f1a535914195aa594c3663f.jpg)

然后打开你的vimrc，（个人为qt新建了一个配置文件，放在~/.qtvimrc下）

写上:

```
" run operation
map ;r :run<CR>
```

这样，就将 " ;r " 快捷键绑定为 Run命令了。

其余命令同理，现在fake vim中的ex command命令中写上mapping命令，然后在vimrc中mapping键位。

下面是我个人的全部qtvimrc：

```
" 显示当前模式
set showmode
" 共享系统粘贴板
set clipborad=unamed
" 打开行号
set number
" 打开相对行号
" set relativenumber
" 设置命令历史记录条数
set history=2000
" 关闭兼容vi
set nocompatible
" 开启语法高亮功能
syntax enable
" 允许用指定语法高亮配色方案替换默认方案
syntax on
" 模式搜索实时预览,增量搜索
set incsearch
" 设置搜索高亮
set hlsearch
" 忽略大小写 (该命令配合smartcase使用较好，否则不要开启)
set ignorecase
" 模式查找时智能忽略大小写
set smartcase
" vim自身命令行模式智能补全
set wildmenu
" 总是显示状态栏
set laststatus=2
" 显示光标当前位置
set ruler
" 高亮显示当前行/列
set cursorline
"set cursorcolumn
" 禁止折行
set nowrap
" 将制表符扩展为空格
set expandtab
" 设置编辑时制表符占用空格数
set tabstop=4
" 设置格式化时制表符占用空格数
set shiftwidth=4
" 让 vim 把连续数量的空格视为一个制表符
set softtabstop=4
" 基于缩进或语法进行代码折叠
set foldmethod=indent
set foldmethod=syntax
" 启动 vim 时关闭折叠代码
set nofoldenable

" 设置前导键
" let mapleader=";"
" 暂时取消搜索高亮快捷键
nnoremap <silent> ;l :<C-u>nohlsearch<CR><C-l>

" 移动相关
" 前一个缓冲区
nnoremap <silent> [b :w<CR>:bprevious<CR>
" 后一个缓冲区
nnoremap <silent> ]b :w<CR>:bnext<CR>
" 定义快捷键到行首和行尾
map H ^
map L $
" 定义快速跳转
nmap ;t <C-]>
" 定义快速跳转回退
nmap ;T <C-t>
" 标签页后退 ---标签页前进是gt
nmap gn gt
nmap gp gT

" 文件操作相关
" 定义快捷键关闭当前分割窗口
nmap ;q :q<CR>
" 定义快捷键保存当前窗口内容
nmap ;w :w<CR>

" 窗口操作相关
map <C-j> <C-W>j
map <C-k> <C-W>k
map <C-h> <C-W>h
map <C-l> <C-W>l

" 使用 qt内部功能
" run operation
map ;r :run<CR>
" copy operation
map ;c :copy<CR>
" paste operation
map ;v :paste<CR>
" cut operation
map ;x :cut<CR>
" Select All
map ;a :selectall<CR>
" reformat code
map ;f :reformatcode<CR>
" 找到usage
map ;u :findusages<CR>
" 调用idea的replace操作
map ;; :replace<CR>
```

基本上是从以前的vimrc配置中copy过来的，所以有些设置在qt中是无效的，但是不影响使用。 关键在 "使用qt内部功能"项下的映射。有些可惜的是fakevim不支持leader键位，所以只能在每处mapping中都硬编码为；  。

4.反缩进问题

qtcreator有个反人类的地方在于，写代码换行，会自动缩进，但是退格却又需要退几次才能回到上一行，可以改为：

![](https://pic.downk.cc/item/5f1a556014195aa594c428c1.jpg)

剩下的就看个人配置了。