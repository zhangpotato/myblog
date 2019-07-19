---
title: Yarn Register DNS 端口占用
date: 2019-01-29 16:39:00
author: Potato
top: true # 如果top值为true，则会是首页推荐文章
# 本文章是否开启mathjax，且需要在主题的_config.yml文件中也需要开启才行
mathjax: false
categories: HDP
tags:
  - Hadoop问题
  - BigData
---

如果在安装时看到与YARN Registry DNS Bind Port相关的错误。可以参考以下步骤操作：

## 1、打开Ambari管理界面并进入YARN管理界面

![](/images/Yarn_Register_DNS/1.png)

## 2、点击CONFIGS，进入配置界面

![](/images/Yarn_Register_DNS/2.png)

## 3、点击ADVANCED进入高级设置中修改DNS端口号

![](/images/Yarn_Register_DNS/3.png)

**将53修改成其他不被占用的端口即可**

## 4、重启集群，YARN正常启动

![](/images/Yarn_Register_DNS/4.png)

参考文献：[Hortonworks](https://community.hortonworks.com/articles/225841/yarn-registry-dns-port-conflict-issue.html)

