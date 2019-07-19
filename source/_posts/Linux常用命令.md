---
title: Linux常用命令
date: 2019-06-04 11:37:00
author: Potato
top: true # 如果top值为true，则会是首页推荐文章
# 本文章是否开启mathjax，且需要在主题的_config.yml文件中也需要开启才行
mathjax: false
categories: Linux
tags:
  - Linux基础
---

## 一、压缩命令

默认压缩级别情况下,1.8G数据 压缩大小: gzip (462M) > bzip2 (238M) > xz (44M)

### **1、bzip2、bunzip2**

```shell
#压缩单个文件命令 压缩后demo.txt消失，生成demo.txt.bz2文件
bzip2 demo.txt
#解压命令
bunzip2 demo.txt.bz2
#压缩多个文件命令
bzip2 demo1.txt demo2.txt
#解压多个文件命令
bunzip2 demo1.txt.bz2 demo2.txt.bz2
```

### **2、gzip**

gzip只能对普通文件进行压缩和解压

```shell
#压缩命令
gzip demo.txt
#压缩并保留源文件
gzip -c demo.txt > demo.txt.gz
#解压命令
gzip -d demo.txt.gz
```

### **3、tar**

tar是打包、拆包的命令

“打包/拆包”“压缩/解压缩”有什么区别？我们用一个生活中的例子来解释：

```
- 就像搬家时，我们把每一床棉被都抽成真空，这叫作压缩，然后把好几床抽真空的棉被用绳子捆绑起来，这就叫打包。
- 东西搬到新家后，把绳子解开，就是拆包，然后把每床棉被舒展开，让棉被松软起来，这就是解压缩。
- 如果不抽真空，只是把几床棉被简单地用绳子捆起来，那么就单独用tar就好了。
- 如果只有一床棉被，打算抽真空，那么就用gzip就好了。
- 如果有好多床棉被，既要抽真空，又要捆起来，那么就要将tar和gzip结合起来使用。
```

```shell
#拆包解压
tar -xvf demo.tar
tar -xvzf demo.tar.gz
tar -xvjf demo.tar.bz2
#打包压缩 将/var/log下的文件打包解压到名为demo.tar.gz的文件夹
tar -cvf demo.tar /var/log		#仅打包，不压缩
tar -cvzf demo.tar.gz /var/log	#打包后用gzip压缩
tar -cvjf demo.tar.bz2 /var/log	#打包后用bzip2压缩
#参数解释
-c 表示进行打包动作
-z 使用gzip进行压缩和解压
-v 显示整个过程
-f 指定要打包的文件
-x 表示进行拆包动作
-j 使用bzip2进行压缩和解压
-t 显示包中的内容
#显示压缩包中的内容，但是不解压
tar -tvzf demo.tar.gz
```

##  二、端口监听