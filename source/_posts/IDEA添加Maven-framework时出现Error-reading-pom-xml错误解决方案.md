---
title: IDEA添加Maven framework时出现Error reading /pom.xml错误解决方案
categories:
  - Maven
tags: Maven
abbrlink: 36f93f50
date: 2020-04-30 20:22:11
---

今天给Idea添加Maven框架的时候，出现了`Error reading pom.xml`错误，解决方案：

打开系统hosts文件（C:\Windows\System32\drivers\etc路径下），添加

```
127.0.0.1 localhost
```

即可。

