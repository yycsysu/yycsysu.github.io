---
layout: post
title: Hive数据集群间迁移流程
description: ""
modified: 2015-09-13
tags: [hive,hadoop]
image:
---

# 简介
基于全表导出和全表导入。流程为Source Hive -> Source HDFS Cluster -> Destination HDFS Cluster -> Destination Hive

# （Source端）群集操作

#### 1. 创建导出临时目录
这里定为hdfs://tmp/hive-export/<database name>
假设这里导出的数据库名为cdp_data

```
$ sudo -u hdfs dfs -mkdir -p /tmp/hive-export/cdp_data
```

#### 2. 生成导出数据脚本
```
$ sudo -u hdfs hive -e "use cdp_data; show tables;" | \
awk '{printf "export table %s to @/tmp/hive-export/cdp_data%s@;\n",$1,$1}' | \
sed "s/@/'/g" > export.hql
```

#### 3. 执行导出数据脚本
```
$ sudo -u hdfs hive -e "use cdp_data; source export.hql"
```

#### 4. 数据导出完成

# （Destination端）群集操作

#### 1 创建导入临时目录
这里定为hdfs://tmp/hive-import/<database name>

```
$ sudo -u hdfs dfs -mkdir -p /tmp/hive-import/cdp_data
```

#### 2. 从Source端复制导出到HDFS的数据
这里用DistCp，该步只能在Destination端进行。并且需要用hftp连接Source端的hdfs文件系统。这是为了避免因Cluster版本不同产生的问题。
```
$ sudo -u hdfs hadoop distcp hftp://<source host>:50070/tmp/hive-export/cdp_data \
hdfs://<destination host>:8020/tmp/hive-import/cdp_data
```

#### 3. 生成导入数据脚本

```
$ sudo -u hdfs hdfs dfs -ls /tmp/hive-import/cdp_data/ | \
awk '{print $8}' | awk -F '/' '{print $5}' | grep -v "^$"  > table.list

$ cat table.list  | \
awk '{printf "import table %s from @/tmp/hive-import/cdp_data/%s@;\n",$1,$1}' | \
sed "s/@/'/g" > import.hql
```

#### 4. 执行导入数据脚本

```
$ sudo -u hdfs hive -e "use cdp_data; source import.hql"
```