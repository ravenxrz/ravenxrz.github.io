---
title: Ubuntu下OhMyZsh的快速安装教程
categories: Linux
toc: true
abbrlink: 6c55eca4
date: 2019-07-31 13:58:16
tags:
	- OhMyZsh
	- Ubuntu
---

> 依旧是常用工具补坑，每次重装系统或者更换设备都要安装oh-my-zsh，然后每次都会去百度教程，所以这里简单记录一下。
>

oh-my-zsh比起ubuntu自带的bash的优点，我这里就不细说了。**反正只用知道使用on-my-zsh可以提高生产力就行了。**

<!-- more -->
## 1. 安装oh-my-zsh

要使用on-my-zsh，首先得**安装zsh，并把shell环境切换为zsh**。

执行以下命令：

```shell
# 安装 Zsh
sudo apt install zsh
# 将 Zsh 设置为默认 Shell
chsh -s /bin/zsh
```

之后就可以**安装oh-my-zsh了**：

```shell
# 安装 Oh My Zsh
wget https://github.com/robbyrussell/oh-my-zsh/raw/master/tools/install.sh -O - | sh
```

## 2. 主题配置

打开~/.zshrc文件，更改ZSH_THEME字段,我这里选择的是agnoster主题。

```shell
ZSH_THEME="agnoster"
```

额外推荐 [Powerlevel9k](https://github.com/bhilburn/powerlevel9k)主题，样子如下图，如果要使用Powerlevel9k主题，需要手动安装它：

![](https://ae01.alicdn.com/kf/Hefe68429ca144cf6ba8031488499a0adJ.png)

```shell
git clone https://github.com/bhilburn/powerlevel9k.git ~/.oh-my-zsh/custom/themes/powerlevel9k
```

并修改.zshrc文件的ZSH_THEME字段为:

```shell
ZSH_THEME="powerlevel9k/powerlevel9k"
```

上述两个主题**都依赖powerline字体**，不然会有乱码现象，可通过以下命令安装powerline字体:

```shell
wget https://raw.githubusercontent.com/powerline/powerline/develop/font/10-powerline-symbols.conf
wget https://raw.githubusercontent.com/powerline/powerline/develop/font/PowerlineSymbols.otf
sudo mkdir /usr/share/fonts/OTF
sudo cp 10-powerline-symbols.conf /usr/share/fonts/OTF/
sudo mv 10-powerline-symbols.conf /etc/fonts/conf.d/
sudo mv PowerlineSymbols.otf /usr/share/fonts/OTF/
```

## 3.  常用插件配置

### 3.1 zsh-autosuggestions 历史命令提示插件

执行：
```shell
git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
```


### 3.2 zsh-syntax-highlighting 语法高亮

```shell
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
```

### 3.3 配置.zshrc文件使插件生效

打开~/.zshrc文件，找到plugin字段，修改为

```shell
plugins=( 
git  	
z 		# 快速跳转插件（oh-my-zsh自带，不用安装）
web-search 	# 命令行中使用搜索引擎 (自带)
zsh-syntax-highlighting 
zsh-autosuggestions 
) 
```

- z插件是快速实现目录跳转的，比如你现在在`/home/raven/Desktop`目录，曾经去过`/home/raven/Downloads`目录，那么可以直接输入`z Dow`然后tab一下，它会自动补全Downloads的绝对路径。

- web-search 是命令行中快速使用搜索引擎的插件，输入baidu xxx或者google xxx即可。个人觉得用处不算大。

## 4. 最后

最后source ~/.zshrc或者重启电脑就ok。
