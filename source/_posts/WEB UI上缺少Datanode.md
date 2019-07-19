---
title: HDFS WEB UI上缺少datanode的问题解决方法
date: 2019-03-19 14:25:00
author: Potato
top: true # 如果top值为true，则会是首页推荐文章
# 本文章是否开启mathjax，且需要在主题的_config.yml文件中也需要开启才行
mathjax: false
categories: HDP
tags:
  - Hadoop问题
  - BigData
---

## 1、集群结构

```sh
192.168.0.60 host0
192.168.0.61 host1
192.168.0.62 host2
192.168.0.63 host3
192.168.0.64 host4
```

## 2、问题状况

```sh
#HDFS上显示存活的datanode个数为4个
#HDFS WEB UI上显示的datanode个数为三个
#缺少host4节点的datanode
```

## 3、问题解决思路

```sh
1、检查namenode与host4是否可以ping通
2、如果不可以ping通，检查host4上的hosts文件，以及网络文件配置
3、如果可以ping通，则检查host4当前的ip地址，有可能在免密配置完成后，重启主机，ip发生变化，此时192.168.168.0.64的地址仍然可以ping通，但是服务却不能连接，因此会导致在WEB UI上无法显示datanode，只需要修改网络配置文件，设置为静态IP即可，然后重启network。
4、停止Hadoop所有服务，然后重启Hadoop集群，即可在WEB UI上查看到host4的datanode信息。
```

