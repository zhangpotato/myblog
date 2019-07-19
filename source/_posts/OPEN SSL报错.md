---
title: 搭建HDP集群报OPEN SSL问题解决方法
date: 2019-03-27 09:27:00
author: Potato
top: false # 如果top值为true，则会是首页推荐文章
# 本文章是否开启mathjax，且需要在主题的_config.yml文件中也需要开启才行
mathjax: false
categories: HDP
tags:
  - Hadoop问题
  - BigData
---

报错：

```shell
ERROR 2015-09-23 09:47:07,402 NetUtil.py:77 - [Errno 8] _ssl.c:492: EOF occurred in violation of protocol
ERROR 2015-09-23 09:47:07,402 NetUtil.py:78 - SSLError: Failed to connect. Please check openssl library versions.
Refer to: https://bugzilla.redhat.com/show_bug.cgi?id=1022468 for more details.
WARNING 2015-09-23 09:47:07,402 NetUtil.py:105 - Server at https://test.org:8440is not reachable, sleeping for 10 seconds...
WARNING 2015-09-23 09:47:07,402 NetUtil.py:105 - Server at https://test.org:8440is not reachable, sleeping for 10 seconds...
INFO 2015-09-23 09:47:14,746 main.py:74 - loglevel=logging.INFO
INFO 2015-09-23 09:47:14,746 main.py:74 - loglevel=logging.INFO
INFO 2015-09-23 09:47:17,403 NetUtil.py:59 - Connecting to https://test.org:8440/ca
WARNING 2015-09-23 09:47:17,404 NetUtil.py:82 - Failed to connect to https://test.org:8440/ca due to [Errno 111] Connection refused
WARNING 2015-09-23 09:47:17,404 NetUtil.py:105 - Server at https://test.org:8440is not reachable, sleeping for 10 seconds...
WARNING 2015-09-23 09:47:17,404 NetUtil.py:105 - Server at https://test.org:8440is not reachable, sleeping for 10 seconds...
ERROR 2015-09-23 09:47:19,780 main.py:315 - Fatal exception occurred:
```

解决方法：

```shell
vim /etc/ambari-agent/conf/ambari-agent.ini
在[security]下添加
force_https_protocol=PROTOCOL_TLSv1_2
```

```shell
vi /etc/python/cert-verification.cfg 
在[https] 下添加
verify=disable
```

然后在Ambari界面点击retry进行重试操作