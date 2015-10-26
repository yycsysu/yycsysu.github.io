---
layout: post
title: Azkaban Quick Start
description: ""
modified: 2015-10-17
tags: [azkaban]
image:
---

# 在开始之前

这里使用 Web server + Executor 的模式，不使用Solo server
[下载地址](http://azkaban.github.io/downloads.html)

# 环境搭建

#### 配置数据库

>注：目前Azkaban2仅支持MySQL作为数据存储仓库。

1. 安装MySQL

可参考： [MySQL Documentation Site](http://dev.mysql.com/doc/index.html)。

2. 配置数据库

为Azkaban创建一个数据库，如：
```
mysql> CREATE DATABASE azkaban;
```

为Azkaban创建一个数据库用户，如：
```
mysql> CREATE USER '<username>'@'localhost' IDENTIFIED BY '<password>';
```

配置用户权限，为创建的用户添加对于Azkaban数据库的INSERT, SELECT, UPDATE, DELETE权限。

```
mysql> GRANT SELECT,INSERT,UPDATE,DELETE ON <database>.* to '<username>'@'%' WITH GRANT OPTION;
```

(可选)配置MySQL max_allowed_packet属性，提高Packet Size，目前尚未发现未配置有何不良影响。

修改/etc/my.cnf，如下：

```
[mysqld]
...
max_allowed_packet=1024M
```

修改之后重启MySQL（数据库不方便重启的可以暂时跳过这一步）
```
$ sudo service mysqld restart
```

3. 创建Azkaban表

从[下载页面](http://azkaban.github.io/downloads.html)下载 azkaban-sql-script 包，在新建的Azkaban数据库里面运行**create-all-sql**脚本即可。忽略**update**开头的脚本。

4. 获取JDBC Connector Jar包

由于某些原因，Azkaban不提供这个包，我们需要自己下载然后放到Web Server以及Executor Server的extlib目录下。[MySQL JDBC connector jar](http://www.mysql.com/downloads/connector/j/)

#### 启动Azkaban Web Server

将下载的Web Server包解压，进入Web Server目录下。

##### 生成KeyStore

执行以下命令，根据指示输入即可。
```
$ keytool -keystore keystore -alias jetty -genkey -keyalg RSA
```

##### 配置

1. 在**azkaban.properties**中配置keyStroe相关参数，例如：

```
jetty.keystore=keystore
jetty.password=password
jetty.keypassword=password
jetty.truststore=keystore
jetty.trustpassword=password
```

2. 在**azkaban.properties**中配置mysql相关参数，例如：

```
database.type=mysql
mysql.port=3306
mysql.host=localhost
mysql.database=azkaban
mysql.user=azkaban
mysql.password=azkaban
mysql.numconnections=100
```

3. 在**azkaban.properties**中配置UserManager相关参数，例如：

UserManager提供了用户认证和用户角色信息。根据以下配置，Azkaban会使用**XmlUserManager**获取**azkaban-users.xml**用的帐号/密码和角色信息。
```
user.manager.class=azkaban.user.XmlUserManager
user.manager.xml.file=conf/azkaban-users.xml
```

##### 运行Web Server

确保**azkaban.properties**中的配置如下：

```
jetty.maxThreads=25
jetty.ssl.port=8443
```
执行bin/start-web.sh启动web server
执行bin/azkaban-web-shutdown.sh停止

#### 启动Azkaban Executor Server

#####配置

在**azkaban.properties**中配置mysql相关参数，例如：

```
database.type=mysql
mysql.port=3306
mysql.host=localhost
mysql.database=azkaban
mysql.user=azkaban
mysql.password=azkaban
mysql.numconnections=100
```

##### 运行Executor Server

确保**azkaban.properties**中的配置如下：

```
# Azkaban Executor settings
executor.maxThreads=50
executor.port=12321
executor.flow.threads=30
```
执行bin/start-exec.sh启动executor server
执行bin/azkaban-exec-shutdown.sh停止

# 使用Azkaban

#### 设置邮件提醒

配置发送端
在Web Server **azkaban.properties** 中将如下配置填充。 
```
# mail settings
mail.sender=
mail.host=
mail.user=
mail.password=
```

接收端有两种配置方法，一种是在job的配置文件中配置，例如：

```
# hello 

type=command
command=echo "Hello "
command.1= sh hello.sh
success.emails=persian.huang@jinfuzi.com,daniel.deng@jinfuzi.com
failure.emails=persian.huang@jinfuzi.com,daniel.deng@jinfuzi.com
notify.emails=persian.huang@jinfuzi.com,daniel.deng@jinfuzi.com
```

另一种是在执行Job的时候重写邮箱列表，例如：

{F423}

#### 工作流依赖关系配置

例如：

hello.job
```
# hello 

type=command
command=echo "Hello "
command.1= sh hello.sh
failure.emails=persian.huang@jinfuzi.com,daniel.deng@jinfuzi.com
retries=3
retry.backoff=1000
```
world.job
```
# world.job

type=command
dependencies=hello
command=echo "world!"
```

world2.job
```
# world.job

type=command
dependencies=hello
command=echo "world2!"
```

end.job
```
# world.job

type=command
dependencies=world,world2
command=echo "world2!"
```

打包上传可以查看工作流信息如下：
{F425}

#### Job Types使用介绍
可参考： [Jobtypes](http://azkaban.github.io/azkaban/docs/latest/#job-types)