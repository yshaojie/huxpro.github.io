---
layout:     post
title:      "存储型elasticsearch集群参数配置"
date:       2019-12-04 17:03:00
author:     "聼雨夜"
catalog: true
tags:
    - elasticsearch
    - 搜索引擎
---
>该配置是用作存储型elasticsearch(例如:日志存储查询)集群

#### /etc/security/limits.conf 
```bash
cat <<EOF>>  /etc/security/limits.conf 
#elsaticsearch config(start)
#nproc和nofile可以根据实际情况再进行调整
rd  -  nproc   40960
rd  -  nofile  655350
rd  -  fsize   unlimited
rd  -  as   unlimited
#elsaticsearch config(end)
EOF
```

#### /etc/sysctl.conf
```bash
cat <<EOF>>  /etc/sysctl.conf
#elsaticsearch config(start)
vm.swappiness=0
vm.max_map_count=262144
#elsaticsearch config(end)
EOF
```

#### elasticsearch.yml
```yaml
bootstrap.memory_lock: true
bootstrap.system_call_filter: false
indices.memory.index_buffer_size: 20%
indices.memory.min_index_buffer_size: 96mb
thread_pool.index.queue_size: 300
thread_pool.bulk.queue_size: 300
```

#### JVM OPTIONS
```bash
-Djna.tmpdir=dir
```