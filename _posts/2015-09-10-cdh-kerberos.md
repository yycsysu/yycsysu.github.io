---
layout: post
title: 通过Cloudera Management为CDH群集中的服务配置Kerberos认证
description: ""
modified: 2015-09-10
tags: [cdh, hadoop, derberos]
image:
---

# 简介
-----

Kerberoso为一种计算机网络认证协议，它允许某实体在非安全网络环境下通信，向另一个实体以一种安全的方式证明自己的身份。这里指由麻省理工实现此协议，并发布的一套免费软件。Kerberos协议基于对称密码学。Kerberos工作在用于证明用户身份的"票据(tickets)"的基础上。KDC持有一个密钥(principals)数据库；每个网络实体——无论是客户还是服务器——共享一套只有他自己和KDC知道的密钥。密钥的内容用于证明实体的身份。对于两个实体间的通信，KDC产生一个会话密钥，用来加密他们之间的交互信息。

[[ https://en.wikipedia.org/wiki/Kerberos_(protocol) | Kerberos Wiki ]]



# 系统环境
-----


* 系统：//CentOS 6.3 x64// X 2

* Cloudera Manager：//5.4.4//

* CDH: //5.4.4//

* JDK: //1.7.0_79//

* 群集节点规划如下


|IP|Hostname|Roles|

|10.1.2.51 |      hadoop001     | NameNode, DataNode, Kerberos client

|10.1.2.52  |     hadoop002     | DataNode, Kerberos Server, Kerberos client


>注意：hostname 需使用小写，以避免不必要的错误。


# 前置条件
-----


### 1. 安装Cloudera Management和CDH：

可参考[[ http://phabricator.jinfuzi.com/w/java/cdp/base/installenv/ | 通过Cloudera Management安装CDH中的Hadoop/Hive/Spark ]]



### 2. 配置Kerberos服务：

>以下操作使用root权限进行。

#### 1) 安装 Kerberos Server:

在hadoop002上安装Kerberos Server:

```
[root@hadoop002 ~]# yum install krb5-server -y

```


#### 2) 安装 Kerberos Client:

在hadoop001和hadoop002上安装Kerberos Client:

```
[root@hadoop001 ~]# yum install krb5-workstation krb5-libs -y

[root@hadoop002 ~]# yum install krb5-workstation krb5-libs -y

```


#### 3) 编辑 Kerberos Server 配置文件（kdc.conf）：

>以下仅列出需要的配置，详细配置参考：[[ http://web.mit.edu/~kerberos/krb5-devel/doc/admin/conf_files/kdc_conf.html | kdc.conf ]]：

```
[root@hadoop002 ~]# vim /var/kerberos/krb5kdc/kdc.conf

```

```
[kdcdefaults]

 kdc_ports = 88

 kdc_tcp_ports = 88


[realms]

 CDP.COM = {

  #master_key_type = aes256-cts

  master_key_type = aes128-cts

  acl_file = /var/kerberos/krb5kdc/kadm5.acl

  dict_file = /usr/share/dict/words

  admin_keytab = /var/kerberos/krb5kdc/kadm5.keytab

  max_life = 1d

  max_renewable_life = 7d

  supported_enctypes = aes128-cts:normal des3-hmac-sha1:normal arcfour-hmac:normal des-hmac-sha1:normal des-cbc-md5:normal des-cbc-crc:normal

  default_principal_flags = +renewable, +forwardable

 }

```


说明：

* **CDP.COM** ：设定的realms, 一般使用大写。

* **master_key_type**， **supported_enctypes** ： 默认使用aes256-cts，为了简便，这里不使用 aes256-cts 算法（原因见下面说明）。

* **max_renewable_life = 7d**， **max_life = 1d**， **default_principal_flags = +renewable, +forwardable** ： Cloudera Manager要求配置凭证生命周期非零且可更新。


>**关于AES-256加密：**

>如果你使用的是CentOS或Red Hat Enterprise Linux 5.5及以上的系统，则默认使用AES-256加密Kerberos的tickts。需要给群集里的所有节点安装[[ http://www.oracle.com/technetwork/java/javase/downloads/index.html | Java Cryptography Extension (JCE) Unlimited Strength Jurisdiction Policy File ]]。


>这里使用的是JDK1.7，对应的JCE下载地址[[ http://www.oracle.com/technetwork/java/embedded/embedded-se/downloads/jce-7-download-432124.html | Java Cryptography Extension (JCE) Unlimited Strength Jurisdiction Policy Files for JDK/JRE 7 ]] 。下载完后将zip包内的两个文件解压到 **$JAVA_HOME/jre/lib/security** 即可。


####4) 修改 Kerberos Client 配置文件（krb5.conf）：


```
[root@hadoop002 ~]# vim /etc/krb5.conf

[logging]

 default = FILE:/DATA/log/krb5libs.log

 kdc = FILE:/DATA/log/krb5kdc.log

 admin_server = FILE:/DATA/log/kadmind.log


[libdefaults]

 default_realm = CDP.COM

 dns_lookup_realm = false

 dns_lookup_kdc = false

 ticket_lifetime = 24h

 renew_lifetime = 7d

 forwardable = true

 clockskew = 120

 udp_preference_limit = 1


[realms]

 CDP.COM = {

  kdc = hadoop002

  admin_server = hadoop002

 }


[domain_realm]

 .cdp.com = CDP.COM

 cdp.com = CDP.COM


```
说明：

* 基本上是默认配置，修改一些hostname和realms配置即可。

* **clockskew = 120** ：表示tickets与服务器的时钟偏差可容忍在120秒之内。

* **udp_preference_limit = 1** ： 据说可以避免一个Hadoop的错误。


#### 5) 编辑 访问控制列表文件 （kadm5.acl）


```
该文件包含所有获许管理 KDC 的主体名称。

[root@hadoop002 ~]# cat /var/kerberos/krb5kdc/kadm5.acl

*/admin@CDP.COM *

```


#### 6) 同步配置文件

将 hadoop002 中的 /etc/krb5.conf 拷贝到其他主机（即hadoop001）

```
[root@hadoop001 ~]# scp hadoop002:/etc/krb5.conf /etc/krb5.conf

```


#### 7) 创建数据库

该数据库用于存储principals。其中 **-r** 指定对应 realm。用 **-d** 可指定数据库名字，默认为principal。

```
[root@hadoop002 ~]#  kdb5_util create -r CDP.COM -s

```

>如果提示数据库已经存在，则要把 /var/kerberos/krb5kdc/ 目录下的 principal(数据库名称) 的相关文件都干掉。


#### 8) 启动服务

在 hadoop002 节点上运行：

```
[root@hadoop002 ~]# chkconfig --level 35 krb5kdc on

[root@hadoop002 ~]# chkconfig --level 35 kadmin on

[root@hadoop002 ~]# service krb5kdc start

[root@hadoop002 ~]# service kadmin start

```


#### 9) 创建 kerberos 管理员

关于 kerberos 的管理，可以使用 kadmin.local 或 kadmin，区别如下：


* 本地机器（即kerberos server所在主机）使用 kadmin.local。不需要密码。

```
[root@hadoop002 ~]# kadmin.local

Authenticating as principal root/admin@CDP.COM with password.

kadmin:

```

* 远端机器（即kerberos client）使用 kadmin 远程连接kerberos server，需要有管理员权限的principal。需要认证。

```
[root@hadoop001 ~]# kadmin

Authenticating as principal cloudera-scm/admin@CDP.COM with password.

Password for cloudera-scm/admin@CDP.COM:

kadmin:

```

所以这里我们需要创建一个远程管理员用以远端登陆。同时Cloudera Manager也会用到。

```
[root@hadoop001 ~]# kadmin.local -q "addprinc cloudera-scm/admin"

系统会提示输入密码，密码不能为空。

```


#### 10) Kerberos 相关操作

至此kerberos服务搭建完成，可以用以下几个指令测试是否搭建成功：

```
#查看principals

kadmin.local: list_principals


#添加principal

kadmin.local:  addprinc test

WARNING: no policy specified for test@CDP.COM; defaulting to no policy

Enter password for principal "test@CDP.COM":

Re-enter password for principal "test@CDP.COM":

Principal "test@CDP.COM" created.


#删除principal

kadmin.local:  delete_principal test

Are you sure you want to delete the principal "test@CDP.COM"? (yes/no): yes

Principal "test@CDP.COM" deleted.

Make sure that you have removed this principal from all ACLs before reusing.

[root@hadoop001 ~]# kadmin.local -q "addprinc root/admin" //查看principals。

```

```
#获得ticket

[root@hadoop002 ~]# kinit test

Password for test@CDP.COM:


#查看持有的ticket

[root@hadoop002 ~]# klist -e

Ticket cache: FILE:/tmp/krb5cc_0

Default principal: test@CDP.COM


Valid starting     Expires            Service principal

09/10/15 17:53:55  09/11/15 17:53:55  krbtgt/CDP.COM@CDP.COM

        renew until 09/10/15 17:53:55, Etype (skey, tkt): aes128-cts-hmac-sha1-96, aes128-cts-hmac-sha1-96


#销毁持有的ticket

[root@hadoop002 ~]# kdestroy

[root@hadoop002 ~]# klist

klist: No credentials cache found (ticket cache FILE:/tmp/krb5cc_0)

```


# 在Cloudera Manager上启动Kerberos
-----


>Important:

>1. If you have enabled YARN Resource Manager HA in your non-secure cluster, you should clear the StateStore znode in ZooKeeper before enabling Kerberos. To do this:

>2. Go to the Cloudera Manager Admin Console home page, click to the right of the YARN service and select Stop.

>3. When you see a Finished status, the service has stopped.

>4. Go to the YARN service and select Actions > Format State Store.

>5. When the command completes, click Close.


1. 进入Cloudera Manager

2. **Administratio** > **Kerberos**

3. 点击**Enable Kerberos**


#### 首先

{F332}

有四个前置条件

1. 有正在运行的KDC（即Kerberos Server），前置条件已经搭建。

2. KDC配置为非零生命周期且可更新，前置条件已经配置。

3. OpenLdap，这里没有用到。

4. 已创建远端登陆管理员帐户。


#### 下一步1

{F334}

都很明显


#### 下一步2

{F336}

在搭建KDC时已经手动同步了krb5.conf，因此这里不打勾。

>实际上在这里扔给Cloudera Manager管理会好一点。


#### 下一步3

{F338}

远端principal的名称和密码


#### 下一步4

{F340}

导入principal


#### 下一步5

{F342}

配置Cloudera Manager为群集中的服务创建的Principals的名称


#### 下一步6

{F344}

默认配置即可


#### 下一步7

{F346}

这一步由于之前在测试的时候创建了同样的principals没有删除而出错了。只要确认KDC里面没有群集服务要使用的principals名称就不会出错。


#### 查看principals

{F348}


到这里基本上就算完工了。接下来就是使用hdfs，跑mapreduce测试权限认证有没有正常运作。

>注：运行MR2时，要清空yarn/nm/usercache下的用户缓存文件。在SIMPLE模式下和在KERBORES模式下建立的文件权限不一样。


# 验证权限认证系统正常运行


认证当前用户并获得ticket，principal格式为: username@YOUR-REALM.COM（需提前在KDC添加principal）

```
[jinfuzi@hadoop001 ~]$ kinit

Password for jinfuzi@CDP.COM:

[jinfuzi@hadoop001 ~]$ klist

Ticket cache: FILE:/tmp/krb5cc_500

Default principal: jinfuzi@CDP.COM


Valid starting     Expires            Service principal

09/10/15 19:24:51  09/11/15 19:24:51  krbtgt/CDP.COM@CDP.COM

        renew until 09/10/15 19:24:51


#### 运行hadoop mapreduce任务


```


结果

```
[jinfuzi@hadoop001 ~]$ hadoop jar /DATA/cloudera/parcels/CDH-5.4.4-1.cdh5.4.4.p0.4/jars/hadoop-examples.jar pi 10 10000

Number of Maps  = 10

Samples per Map = 10000

...

Job Finished in 49.05 seconds

Estimated value of Pi is 3.14120000000000000000

```


#### 运行hive beeline


```
[jinfuzi@hadoop001 ~]$ beeline

Beeline version 1.1.0-cdh5.4.4 by Apache Hive

beeline> !connect jdbc:hive2://localhost:10000/;principal=hive/hadoop001@CDP.COM

scan complete in 5ms

Connecting to jdbc:hive2://localhost:10000/;principal=hive/hadoop001@CDP.COM

Enter username for jdbc:hive2://localhost:10000/;principal=hive/hadoop001@CDP.COM:

Enter password for jdbc:hive2://localhost:10000/;principal=hive/hadoop001@CDP.COM:

Connected to: Apache Hive (version 1.1.0-cdh5.4.4)

Driver: Hive JDBC (version 1.1.0-cdh5.4.4)

Transaction isolation: TRANSACTION_REPEATABLE_READ

0: jdbc:hive2://localhost:10000/> show databases;

+-----------------+--+

|  database_name  |

+-----------------+--+

| default         |

| hive_auth_test  |

+-----------------+--+

4 rows selected (3.639 seconds)

0: jdbc:hive2://localhost:10000/> select * from hive_auth_test.t1;

+--------+--+

| t1.id  |

+--------+--+

| 0      |

| 1      |

+--------+--+

2 rows selected (1.635 seconds)

0: jdbc:hive2://localhost:10000/>


```