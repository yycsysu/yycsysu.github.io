---
layout: post
title: Flume Source Learning (1)
description: "ReliableSpoolingFileEventReader.class"
modified: 2015-08-29
tags: [flume, source code]
---

## (1) ReliableSpoolingFileEventReader
-----

[TOC]

### 初始化

>下面代码块使用一个名为canary的文件。canary: 金丝雀。
>金丝雀曾在矿井中被用于早期预警，这段代码的意义也在于此。
>1. 创建一个canary文件： 检测创建功能。
>2. 然后写。再读。Check是否读出内容： 检测读写功能。
>3. 最后删除canary文件： 检测删除功能。
>预先检测在Spooling directory内的所有操作能否成功。

``` java
// Do a canary test to make sure we have access to spooling directory
try {
  File canary = File.createTempFile("flume-spooldir-perm-check-", ".canary",
      spoolDirectory);
  Files.write("testing flume file permissions\n", canary, Charsets.UTF_8);
  List<String> lines = Files.readLines(canary, Charsets.UTF_8);
  Preconditions.checkState(!lines.isEmpty(), "Empty canary file %s", canary);
  if (!canary.delete()) {
    throw new IOException("Unable to delete canary file " + canary);
  }
  logger.debug("Successfully created and deleted canary file: {}", canary);
} catch (IOException e) {
  throw new FlumeException("Unable to read and modify files" +
      " in the spooling directory: " + spoolDirectory, e);
}
```

>这块代码位于上面那块之下。
>trackerDirPath传自上一级的SpoolDirectorySource类。默认值为 ".flumespool"
if条件块一：File.isAbsolute() : 判断路径是否为绝对路径。
            因为下面使用的是new File(parent, child)。
if条件块二：若trackerDirectory不存在则自动创建该目录。
if条件块三：确定当前trackerDirectory为目录。
if条件块四：删除旧的this.metaFile

``` java
File trackerDirectory = new File(trackerDirPath);

// if relative path, treat as relative to spool directory
if (!trackerDirectory.isAbsolute()) {
  trackerDirectory = new File(spoolDirectory, trackerDirPath);
}

// ensure that meta directory exists
if (!trackerDirectory.exists()) {
  if (!trackerDirectory.mkdir()) {
    throw new IOException("Unable to mkdir nonexistent meta directory " +
        trackerDirectory);
  }
}

// ensure that the meta directory is a directory
if (!trackerDirectory.isDirectory()) {
  throw new IOException("Specified meta directory is not a directory" +
      trackerDirectory);
}

this.metaFile = new File(trackerDirectory, metaFileName);
if(metaFile.exists() && metaFile.length() == 0) {
  deleteMetaFile();
}
```
``` java
private void deleteMetaFile() throws IOException {
    if(this.metaFile.exists() && !this.metaFile.delete()) {
        throw new IOException("Unable to delete old meta file " + this.metaFile);
    }
}
```
