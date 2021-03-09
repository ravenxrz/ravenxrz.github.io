---
title: Clion远端调试开发环境部署
date: 2020-12-23 11:24:56
categories: 杂文
tags:
---

从来没用过JetBrains家的Remote Development, 今天体验了一波，效果非常好。这里说一下如何配置，以及一些问题的解决方式。

<!--more-->

## 1. 环境说明

我是开了虚拟机来做Linux服务器，实际上如果你有云服务器或者有多余的主机，甚至是树莓派都可以用这个方法。

1. Ubuntu  服务，ip 192.168.18.142.
2. Windows 本地机

## 2. 基本配置

打开设置，如下图添加Remote　Host:

<img src="https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20201223112912644.png" alt="image-20201223112912644" style="zoom:50%;" />![image-20201223112935496](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20201223112935496.png)


如果发现无法连接，多半是Linux没有安装ssh-server:

```shell
sudo apt-get install openssh-server
```

上面配置好后，再按下图配置：

<img src="https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20201223113115796.png" alt="image-20201223113115796" style="zoom:50%;" />

接着可以在项目主界面右上角选择采用本地编译还是远端编译：

<img src="https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20201223113152657.png" alt="image-20201223113152657" style="zoom:50%;" />

如果一切顺利，现在就可以运行起来了。

下面在说一下其他配置与问题。

## 3. 其他配置

首先第一个问题，你的项目文件被推送到远端的哪个地方了？

打开：

<img src="https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20201223113411916.png" alt="image-20201223113411916" style="zoom:50%;" />

关注Root Path（点击旁边的Autodect就变为home目录，默认是根目录）

<img src="https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20201223113638033.png" alt="image-20201223113638033" style="zoom:50%;" />

点击旁边的Mappings：

<img src="https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20201223113604112.png" alt="image-20201223113604112" style="zoom: 50%;" />

主要关注远端项目位置， 这里的 远端项目位置就是和前面的“Root path”合并的位置。如我这里最终的效果是：

将 本地 `E:\RemoteTest` 推送到 远端 `/home/raven/Projects/RemoteTest/` 下。

还有一个Excluded Paths：用于排除哪些目录是不需要推送到远端的。

<img src="https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20201223113809706.png" alt="image-20201223113809706" style="zoom:50%;" />



还有个问题是头文件解析相关, 比如你做linux开发，很多linux特定的头文件在windows下是没有的，这时候怎么办呢？Clion在 **第一次推送文件到远端时，会自动拉取头文件依赖（个人估计是包含了/usr/include和/usr/local/include)**, 后续将不再更新，如果你在开发过程中，更新了库依赖，比如安装了新的库等，需要重新同步头文件依赖。点击下图进行更新：

<img src="https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20201223114321453.png" alt="image-20201223114321453" style="zoom:50%;" />



## 4. 问题说明&解决

### 1. 问题1：修改的文件如何实时同步？

> 下面的解决方案需要你学过vim

说完了这些，再说一下优化，如果你仅仅是像上面这样配置，你会发现在本地修改代码后，直接运行代码是没有修改的，可能要到第二次运行才是真的推送到远端的。如何解决这个问题？

根据官网所描述的，远端开发是采用 rsync 命令实现的，所以这里的问题是，你修改的代码没有落盘到文件，也就没有推送到远端。那如何落盘到文件？

首先，开启自动上传：

<img src="https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20201223114643521.png" alt="image-20201223114643521" style="zoom:50%;" />

接着 **需要你学过vim，打开ideavim插件，添加**

```
" run
nnoremap <Leader>r :action SaveAll<CR>:action RunClass<CR>
```

我的 Leader 键是分号。上面的意思是，按分号+r，自动执行一次保存全部，然后执行运行。 这样当自动保存执行后，clion自动推送修改的代码到远端，然后执行自动运行。就能全自动了。

最后贴一份自己常用的ideavim配置：

```
" 显示当前模式
set showmode
" 共享系统粘贴板
set clipboard+=unnamed
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
set tabstop=8
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
let mapleader=";"
" 暂时取消搜索高亮快捷键
nnoremap <silent> <Leader>l :<C-u>nohlsearch<CR><C-l>

" 移动相关
" 前一个缓冲区
nnoremap <silent> [b :w<CR>:bprevious<CR>
" 后一个缓冲区
nnoremap <silent> ]b :w<CR>:bnext<CR>
" 定义快捷键到行首和行尾
map H ^
map L $
" 定义快速跳转
nmap <Leader>t <C-]>
" 定义快速跳转回退
nmap <Leader>T <C-t>
" 标签页后退 ---标签页前进是gt
nmap gn gt
nmap gp gT

" 文件操作相关
" 定义快捷键关闭当前分割窗口
nmap <Leader>q :q<CR>
" 定义快捷键保存当前窗口内容
nmap <Leader>w :w<CR>

" 窗口操作相关
map <C-j> <C-W>j
map <C-k> <C-W>k
map <C-h> <C-W>h
map <C-l> <C-W>l

" 使用idea内部功能
" copy operation
nnoremap <Leader>c :action $Copy<CR>
" paste operation
nnoremap <Leader>v :action $Paste<CR>
" cut operation
nnoremap <Leader>x :action $Cut<CR>
" Select All
nnoremap <Leader>a :action $SelectAll<CR>
" reformat code
nnoremap <Leader>f :action ReformatCode<CR>
" New File
nnoremap <Leader>n :action NewFile<CR>
" 找到usage
nnoremap <Leader>u :action FindUsages<CR>
" 调用idea的replace操作
nnoremap <Leader>; :action Replace<CR>
" go to class
nnoremap <Leader>gc :action GotoClass<CR>
" go to action
nnoremap <Leader>ga :action GotoAction<CR>
" run  SaveAll + Run
nnoremap <Leader>r :action SaveAll<CR>:action RunClass<CR>
" 显示当前文件的文件路径
nnoremap <Leader>fp :action ShowFilePath<CR>
" 隐藏激活窗口
nnoremap <Leader>h :action HideActiveWindow<CR>

" 中英文自动切换
set keep-english-in-normal

" other vim plugins
" comment plugin
set commentary
" surround plugin
set surround
" easymotion
set easymotion

" some useful shortcuts in ide, but doesn't remap those.
" open project file tree ---------- alt + 1
" open terminal window   ---------- alt + F12
```

### 2. 问题2：时间校准

windows和ubuntu的时间尽量保证一致，否则可能出现`warning: modification time in the future`类似的警告。 

具体修复同步问题，请参考：https://windowsreport.com/error-occurred-windows-synchronizing-time-windows-com/

> 修复这个花了好长好长时间。。。

时间服务器：

ntp.sjtu.edu.cn 上海交通大学网络中心 NTP 服务器地址

s1a.time.edu.cn 北京邮电大学

s1b.time.edu.cn 清华大学

s1c.time.edu.cn 北京大学

s1d.time.edu.cn 东南大学

s1e.time.edu.cn 清华大学

s2a.time.edu.cn 清华大学

s2b.time.edu.cn 清华大学

s2c.time.edu.cn 北京邮电大学

s2d.time.edu.cn 西南地区网络中心

s2e.time.edu.cn 西北地区网络中心

s2f.time.edu.cn 东北地区网络中心

s2g.time.edu.cn 华东南地区网络中心

s2h.time.edu.cn 四川大学网络管理中心

s2j.time.edu.cn 大连理工大学网络中心

s2k.time.edu.cn CERNET 桂林主节点

s2m.time.edu.cn 北京大学

## 5. 2021-01-10 WSL配置

前面介绍的是基于虚拟机的配置，但是对于性能较差的电脑来说，开虚拟机有点“重”了，而且虚拟机也不是随着windows的开机而启动的。 于是我们可以改用wsl。wsl的具体安装可以参考微软官方给的链接。

我的wsl选择的是ubuntu20. 默认安装后是没有sshd的，所以需要手动安装一下：

```shell
sudo apt install openssh-server
```

安装后，需要进行一点配置，改用root用户：

```shell
ssh-keygen -t dsa -f /etc/ssh/ssh_host_dsa_key
ssh-keygen -t rsa -f /etc/ssh/ssh_host_rsa_key
chmod 600 /etc/ssh/ssh_host_dsa_key
chmod 600 /etc/ssh/ssh_host_rsa_key
```

同时修改 `/etc/ssh/sshd_config`配置：

```shell
PasswordAuthentication yes
```

最后启动服务：

```shell
service ssh start
```

查看是否启动成功：

```shell
service ssh status
```

如果出现 `ssh is running`则启动成功。

然后打开clion，按照如下图配置“


![image-20210309221552707](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20210309221552707.png)