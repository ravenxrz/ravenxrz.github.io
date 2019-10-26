---
title: Ubuntu包管理的小知识
date: 2019-08-07 13:58:17
categories: Linux
toc: true
tags:
---

## Ubuntu PPA小知识

PPA,表示Personal Package Archives,也就是个人软件包集。再说白点就是非官方源。

<!-- more -->

### 为什么要用PPA？

有些包吧，不能进入到官方源去。或者说要进入到官方源中的周期过长，用户很难得到及时的更新，于是需要使用PPA。PPA由launchpad.net提供，个人用户可将包发布在上面。

## 1. 添加PPA

```shell
sudo add-apt-repository ppa:ppa-user/ppa-name
```

如：

```
sudo add-apt-repository ppa:atareao/atareao
```

添加ppa源后，需要依次执行：

```shell
sudo apt update # 更新源数据库，会在/etc/apt/sources.list.d目录下生成一个源文件，用于存放包地址。
# 为什么不放在/etc/apt/sources.list 下，是因为不要和主源混淆了
sudo apt install xxx
```

## 2. 删除PPA

### 2.1 方法一

```shell
sudo add-apt-repository -r ppa:atareao/atareao
```

这个方法是ubuntu自带的，但是需要你记得ppa源的具体名称，比较麻烦，不推荐。

### 2.2 方法二

暴力删除，其实添加ppa,说白了就是在`/etc/apt/sources.list.d`目录下生成一个对应的源文件，我们可以找到该文件，暴力删除即可。

暴力方法嘛，也不太推荐。

### 2.2 方法三

安装`ppa-purge`工具，然后删除。

```shell
sudo apt-get install ppa-purge 
sudo ppa-purge ppa:/atareao/atarea
```

注意，这里的ppa是可以通过tab补全的，所以不需要记住全称。

## 参考：

- [LinuxTool](https://github.com/guodongxiaren/LinuxTool/blob/master/APT.md)

## 3. 你知道APT和APT-Get的区别吗

- apt其实是apt-get和apt-cache的一个子集，它提供了更方便的options给用户，方便用户统一调用。
- apt-get不会过时，因为它可以更细颗度的管理包，但是对于常规linux用户，使用apt管理更方便。

这里简单列一下apt与apt-get在选项上的差别：

|   apt command    | the command it replaces |               function of the command                |
| :--------------: | :---------------------: | :--------------------------------------------------: |
|   apt install    |     apt-get install     |                  Installs a package                  |
|    apt remove    |     apt-get remove      |                  Removes a package                   |
|    apt purge     |      apt-get purge      |          Removes package with configuration          |
|    apt update    |     apt-get update      |              Refreshes repository index              |
|   apt upgrade    |     apt-get upgrade     |           Upgrades all upgradable packages           |
|  apt autoremove  |   apt-get autoremove    |              Removes unwanted packages               |
| apt full-upgrade |  apt-get dist-upgrade   | Upgrades packages with auto-handling of dependencies |
|    apt search    |    apt-cache search     |               Searches for the program               |
|     apt show     |     apt-cache show      |                Shows package details                 |

以上解释摘自：https://itsfoss.com/apt-vs-apt-get-difference/
