---
title: Trinity调度延迟解决方法
date: 2019-04-09 17:15:00
author: Potato
top: true # 如果top值为true，则会是首页推荐文章
# 本文章是否开启mathjax，且需要在主题的_config.yml文件中也需要开启才行
mathjax: false
categories: Trinity
tags:
  - ETL
  - Trinity
---

1、查询各表占用的空间

```sql
SELECT
   relname as "Table",
   pg_size_pretty(pg_total_relation_size(relid)) As "Size",
   pg_size_pretty(pg_total_relation_size(relid) - pg_relation_size(relid)) as "External Size"
FROM pg_catalog.pg_statio_user_tables ORDER BY pg_total_relation_size(relid) DESC;
)
```

2、需要维护的table

```sql
task, taskdependencyrule, jobexecution, jobexecutionschedule, jobtxdate
```

3、执行vacuum操作

![](/images/Trinity_vacuum/1.jpg)

右键需要维护的table->选择Maintenance->ANALYZE->ok

![](/images/Trinity_vacuum/2.jpg)

再选择Vacuum->选择full->ok->REINDEX->ok

![](/images/Trinity_vacuum/3.jpg)

