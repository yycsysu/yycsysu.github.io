---
layout: post
title: Flume Source Learning (1)
description: "ReliableSpoolingFileEventReader.class"
modified: 2015-08-29
tags: [flume, source code]
image:
  feature: abstract-12.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
---

## (1) ReliableSpoolingFileEventReader
-----


>下面代码块使用一个名为canary的文件。canary: 金丝雀。
>金丝雀曾在矿井中被用于早期预警，这段代码的意义也在于此。
>1. 创建一个canary文件： 检测创建功能。
>2. 然后写。再读。Check是否读出内容： 检测读写功能。
>3. 最后删除canary文件： 检测删除功能。
>预先检测在Spooling directory内的所有操作能否成功。

``` java
File trackerDirectory;
try {
    trackerDirectory = File.createTempFile("flume-spooldir-perm-check-", ".canary", spoolDirectory);
    Files.write("testing flume file permissions\n", trackerDirectory, Charsets.UTF_8);
    List lines = Files.readLines(trackerDirectory, Charsets.UTF_8);
    Preconditions.checkState(!lines.isEmpty(), "Empty canary file %s", new Object[]{trackerDirectory});
    if(!trackerDirectory.delete()) {
        throw new IOException("Unable to delete canary file " + trackerDirectory);
    }

    logger.debug("Successfully created and deleted canary file: {}", trackerDirectory);
} catch (IOException var17) {
    throw new FlumeException("Unable to read and modify files in the spooling directory: " + spoolDirectory, var17);
}
```

>这块代码位于上面那块之下。
>trackerDirPath传自上一级的SpoolDirectorySource类。默认值为 ".flumespool"
>File.isAbsolute() : 判断路径是否为绝对路径。因为下面使用的是new File(parent, child)。
>第二个if条件块则是创建trackerDirectory和this.metaFile了。
>其实创建trackerDirectory也是为了得到this.metaFile。
>最后一个else则是删除旧的this.metaFile

``` java
trackerDirectory = new File(trackerDirPath);
if(!trackerDirectory.isAbsolute()) {
    trackerDirectory = new File(spoolDirectory, trackerDirPath);
}

if(!trackerDirectory.exists() && !trackerDirectory.mkdir()) {
    throw new IOException("Unable to mkdir nonexistent meta directory " + trackerDirectory);
} else if(!trackerDirectory.isDirectory()) {
    throw new IOException("Specified meta directory is not a directory" + trackerDirectory);
} else {
    this.metaFile = new File(trackerDirectory, ".flumespool-main.meta");
    if(this.metaFile.exists() && this.metaFile.length() == 0L) {
        this.deleteMetaFile();
    }
}

...

private void deleteMetaFile() throws IOException {
    if(this.metaFile.exists() && !this.metaFile.delete()) {
        throw new IOException("Unable to delete old meta file " + this.metaFile);
    }
}
```

``` html
<link rel="stylesheet" href="/assets/js/plugins/prettify/prettify.css">
<script src="/assets/js/plugins/prettify/prettify.js"></script>
<script type="text/javascript">
  $(function(){
    $("pre").addClass("prettyprint");
    prettyPrint();
  });
</script>
```