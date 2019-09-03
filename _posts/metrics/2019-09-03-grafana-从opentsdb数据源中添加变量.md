---
layout:     post
title:      "在grafana中使用opentsdb查询模板变量"
date:       2019-09-03 17:03:00
author:     "聼雨夜"
catalog: true
tags:
    - grafana
    - opentsdb
---
### 问题说明
**如果通过opentsdb查询变量,在语法都写正确的情况下,返回结果为空,<br/>**
**可能是opentsdb层面存在问题,从以下步骤检查问题**

### 解决问题步骤
```text
grafana官方原文:
If you do not see template variables being populated in Preview of values section, 
you need to enable tsd.core.meta.enable_realtime_ts in the OpenTSDB server settings.
. Also, to populate metadata of the existing time series data in OpenTSDB, 
you need to run tsdb uid metasync on the OpenTSDB server.
```
#### tsd.core.meta.enable_realtime_ts
opentsdb配置**tsd.core.meta.enable_realtime_ts=true**,然后重启opentsdb服务

#### uid metasync
保证上一步做完后,然后执行命令**${opentsdb_home}/build/tsdb  uid metasync**
注意:如果后边该变量有新的值,结果也还没查询出来,那么还是反复执行**${opentsdb_home}/build/tsdb  uid metasync**即可


