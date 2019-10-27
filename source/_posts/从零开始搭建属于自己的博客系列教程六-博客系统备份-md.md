---
title: 从零开始搭建属于自己的博客系列教程六-博客系统备份
date: 2019-05-04 15:58:34
categories: 博客搭建
toc: true
tags:
---


## 1.  写在最前面

> 折腾了整整一个五一，总算是完成了自己的博客系统。为了给想要搭建自己的博客，但是又和我一样是技术小白的人铺点路，所以准备开个博客搭建的系列教程，避免后来人像我一样走了一大堆弯路。

先贴上自己的博客网站：https://www.ravenxrz.ink   
<!-- more -->
系列教程目录：

- [从零开始搭建属于自己的博客系列教程零 -- 绪论](https://www.ravenxrz.ink/archives/ck27kp480001u4gvmay908xku/)
- [从零开始搭建属于自己的博客系列教程一 -- 博客系统搭建](https://www.ravenxrz.ink/archives/ck27kp47t001e4gvm3ztsc97x/)
- [从零开始搭建属于自己的博客系列教程二 -- 域名绑定](https://www.ravenxrz.ink/archives/ck27kp47u001h4gvm669q1u7p/)
- [从零开始搭建属于自己的博客系列教程三 -- 主题选择与CommentToMail插件介绍](https://www.ravenxrz.ink/archives/ck27kp48v003w4gvmdcd2d0fm/)
- [从零开始搭建属于自己的博客系列教程四 -- 去掉烦人的index.php后缀与常用插件介绍](https://www.ravenxrz.ink/archives/ck27kp47y001p4gvm3zsucst2/)
- [从零开始搭建属于自己的博客系列教程五 -- 图床选择与博客迁移](https://www.ravenxrz.ink/archives/ck27kp47w001j4gvm1ltvcpbg/)
- [从零开始搭建属于自己的博客系列教程六 -- 博客系统备份](https://www.ravenxrz.ink/archives/ck27kp47z001s4gvmbldsfb6h/)
- [从零开始搭建属于自己的博客系列教程七 -- 开启全站HTTPS](https://www.ravenxrz.ink/archives/ck27kp47x001m4gvmfyii1whr/)

## 2.  备份你的博客

很明显，个人搭建的博客万一down掉了可不像简书、csdn这种网站有人维护，所以我们需要自己定时来备份自己的博客。

注意，备份云盘是**google drive**。

### 2.1  安装gdrive

打开XShell，执行以下代码。

```
wget -O /usr/bin/gdrive "https://docs.google.com/uc?id=0B3X9GlR6EmbnQ0FtZmJJUXEyRTA&export=download"
chmod +x /usr/bin/gdrive
gdrive about
```

之后，将命令行给出的url复制到浏览器中，打开。（google drive，你懂需要什么的）

授权后，会给出token，复制粘贴到XShell中。即可

### 2.2 备份脚本

参考了https://www.moerats.com/archives/296/，所给的脚本。我们要知道，备份的东西主要有两个，一个是文章的数据库，这个很小，但是需要经常备份。另一个是整个网站的设置，插件、主题等等，这个很大，却不需要经常改动。所需我将脚本改为了两个，分别设置定期备份时间。

执行命令

```
mkdir /home/wwwbackups
cd /home/wwwbackups
```

`vi googledrive-website.sh` 输入以下内容

```sh
#!/bin/sh
##-----------------------Database Access--------------------------##
DB_NAME="typecho"
DB_USER="root"
DB_PASSWORD=""
##-----------------------Folder Web or Folder you want to backup--------------------------##
# NameOfFolder=("default") # If you have multiple folders you need to back up
NameOfFolder="default"
SourceOfFolder="/home/wwwroot/"
BackupLocation="/home/wwwbackups"
date=$(date +"%Y-%m-%d")
##That mean, you will Backup the folder /home/wwwroot/yourdomain.com and will save into Folder /backups
## ----------------------LOG FILE---------------------------------##
LOG_FILE=backups.log
## ----------------------run code-------------------------------##

# Create Backup Floder
if [ ! -d $BackupLocation ]; then
mkdir -p $BackupLocation
fi

# Delete All backed files
find $BackupLocation/*.zip -mtime +10 -exec rm {} \;

# Name of the Backup File
for fd in $NameOfFolder; do
file=$fd-$date.zip

# Zip the Folder you will want to Backup
echo "Starting to zip the folder and files"
cd $SourceOfFolder
zip -r $BackupLocation/$file $fd
sleep 5s

##Process Upload Files to Google Drive
gdrive upload --delete $BackupLocation/$file

# Check upload success or not and mail to user
if test $? = 0
then
echo "Your Data Successfully Uploaded to the Google Drive!"
echo -e "Your Data Successfully created and uploaded to the Google Drive!" | mail -s "Your VPS Backup from $date" zhang.xingrui@foxmail.com
else
echo "Error in Your Data Upload to Google Drive" > $LOG_FILE
fi
done
```

再建立` vi googledrive-mysql.sh`
```sh
#!/bin/sh
##-----------------------Database Access--------------------------##
DB_NAME="typecho"
DB_USER="root"
DB_PASSWORD=""
##-----------------------Folder Web or Folder you want to backup--------------------------##
# NameOfFolder=("default") # If you have multiple folders you need to back up
NameOfFolder="default"
SourceOfFolder="/home/wwwroot/"
BackupLocation="/home/wwwbackups"
date=$(date +"%Y-%m-%d")
##That mean, you will Backup the folder /home/wwwroot/yourdomain.com and will save into Folder /backups

## ----------------------LOG FILE---------------------------------##
LOG_FILE=backups.log

if [ ! -d $BackupLocation ]; then
mkdir -p $BackupLocation
fi
find $BackupLocation/*.zip -mtime +10 -exec rm {} \;
for fd in $NameOfFolder; do
# Name of the Backup File
file=$fd-$date.zip

# Tackle databases
mysqldump -u $DB_USER -p$DB_PASSWORD $DB_NAME > $BackupLocation/$date-$DB_NAME.sql
sleep 5s

##Process Upload Databases file to Google Drive
gdrive upload --delete $BackupLocation/$date-$DB_NAME.sql
if test $? = 0
then
echo "Your Data Successfully Uploaded to the Google Drive!"
echo -e "Your Data Successfully created and uploaded to the Google Drive!" | mail -s "Your VPS Backup from $date" zhang.xingrui@foxmail.com
else
echo "Error in Your Data Upload to Google Drive" > $LOG_FILE
fi
done
```
最后给两个文件执行权限:

```
chmod +x googledrive-mysql.sh
chmod +x googledrive-website.sh
```

现在你可以通过`bash googledrive-mysql.sh`和` bash googledrive-website.sh`来测试。

### 2.3 设置自动备份

这里要使用到crontab来进行备份设置，执行以下命令

```
vi /etc/crontab
```

添加如下两行

```
  0  2  *  *  * root  bash /home/wwwbackups/googledrive-mysql.sh		# 每天凌晨2点备份数据库文件
  0  2  *  *  1 root bash /home/wwwbackups/googledrive-website.sh		# 每个星期1凌晨2点备份整个网站文件
```

最后重启crond服务即可，可能并不需要重启：

```
service crond restart
```



## 3. 恢复备份

有时候需要更换主机，那么就需要重新搭建一下博客系统。具体步骤如下:

1. 如[本系列教程一](https://www.ravenxrz.ink/archives/build-your-own-blog-series-of-tutorials-from-scratch-blog-system-building.html)，购买`vps`，安装 [lnmp](https://www.ravenxrz.ink/go/aHR0cHM6Ly9sbm1wLm9yZy8=)，**不用安装typecho**。

2. 安装本篇教程所说的`gdrive`，并授权。

3. 通过`gdrive`，下载两个备份文件，一个是数据库`sql`，一个是整个网站文件。

4. 将网站文件解压并放置到`/home/wwwroot/`目录下，尝试在浏览器中打开`http://youip`，查看是否已恢复相关主题和插件。**如果遇到404情况，请参考本系列[教程一的FAQ小结](https://www.ravenxrz.ink/archives/build-your-own-blog-series-of-tutorials-from-scratch-blog-system-building.html)**。

5. 执行以下命令，恢复数据库。

   ```
    mysql -u数据库用户名 -p数据库密码 数据库名（一般为typecho) < 第3步中下载的sql文件路径
   ```

   **注意，执行上述步骤后，所有设置都会还原，包括你的博客系统的登陆密码。**

6.再次打开http://youip，查看是否已经恢复相关设置。
7.域名更换，打开你购买域名的控制面板，更改解析目标。

额外恢复问题：

1. 去掉`index`后缀？参考[本系列教程四](https://www.ravenxrz.ink/archives/build-your-own-blog-tutorial-series-from-scratch-4-remove-annoying-index-php-suffixes-and-introductions-to-common-plugins-1.html)
2. `CommentToMail`插件失效？重新配置STMP密码，参考[本系列教程四](https://www.ravenxrz.ink/archives/build-your-own-blog-tutorial-series-from-scratch-4-remove-annoying-index-php-suffixes-and-introductions-to-common-plugins-1.html)。
3. 重建博客系统备份？参考本篇教程第2小结。

### 参考

[Gdrive：Linux下谷歌网盘同步工具、自动备份VPS文件到Google Drive](https://www.moerats.com/archives/296/)

