## 深秋之夜360面试有感
### 前言
刚刚面完360渗透岗，本着前辈们一面试一总结的原则，要完成这次记录，总结经验教训。面了不到20分钟，意料之外，他说大概了解我的能力了，这怕是时间越短反应能力越浅哇，伤心。之前周五下午把简历投出去，不一会收到收到简历回复。周六晚上刚HCTF题目到接近三点，周日上午去三教写项目报告，中午吃饭的时候收到短信，让我加他（一直以为是HR，后来发现不是）微信，说看了我的简历和博客，感觉还挺有好感（羞涩，会不会回来看到？），问了他什么时候会面试，他说就最近会安排，猜到周一。周日晚上HCTF结束，19名。晚上回去开始复习，主要看了SQL注入，把报错注入和宽字节注入重新复习了一遍，祈祷不要周一面试。今天上午收到微信，问我今天下午什么时候有空，下午满课，约在晚上7:30。7:00左右告诉我找了个人面我，8点前，7:35左右接到电话。全程态度超赞。

接到电话，说先问两个Web方面的漏洞，问我SQL比较熟悉那个数据库，我说MYSQL，问不同权限下注入有什么不同的地方，问查不出东西怎么注入，谈到Sqlserver注入，问有哪些权限，然后注入有什么不同，问Oracle可以带外吗，怎么带外。问SSRF，然后问Windows域环境下内网渗透，然后说看了我博客问代码审计和Python装饰器，然后说刚才内网渗透忘了问我Kerberos协议，两个票据中的一个在渗透中有什么作用。最后问我写的NDIS中间层驱动，还问了会不会逆向。

### MYSQL不同权限下注入
mysql下权限管理是通过grant语句进行的，mysql的角色很随意，建立角色和建用户没啥差别，我都懒得介绍，写两个语句举个例子吧。
```SQL
CREATE DATABASE crmdb;
CREATE ROLE IF NOT EXISTS 'crm_dev', 'crm_read', 'crm_write';
GRANT ALL ON crmdb.* TO crm_dev;
GRANT SELECT ON crmdb.* TO crm_read;
GRANT INSERT, UPDATE, DELETE ON crm.* TO crm_write;
CREATE USER crm_dev1@localhost IDENTIFIED BY 'passwd1990';
GRANT crm_dev TO crm_dev1@localhost;
```
与渗透相关的权限我想到的有两个，一个是查数据的权限，一个是进行更高层次提权的涉及文件的权限。对于数据查询的权限，如果没有赋予其他数据库的权限，就没法进行查询，这里有几个比较特殊的表，一个information_schema,一个是mysql。

那么关于infomation_schema曾经我以为我执行下面的语句，则这个用户不会有该表的权限
```SQL
CREATE USER 'ctf'@'192.168.16.3' IDENTIFIED BY '123456';
GRANT SELECT ON ctf.* TO ctf@192.168.16.3;
```

其实不然，我下面这么验证一下，新建一个用户并把所有权限都去了，看看会怎么样

![](img/20180315-1.jpg)

用这个用户登录上去，然后查询一下information_schema表

![](img/20180315-2.jpg)

但是对mysql表却报错了，想想也理所应当，mysql中有控制mysql用户及其权限的表，如果谁都能查能改还得了？这里我后来继续测试发现我们普通新建的用户是不具有mysql数据库任何权限的。除非被赋予权限。

第二个是涉及提权需要用的权限，mysql提权需要使用的权限，写文件，使用root用户新建的用户不具备FILE相关权限，甚至在mysql5.7中，root用户的权限都受到了限制，需要修改mysql配置文件来启用权限。

说了这么多，在普通权限的数据库注入上，可以查询information_schema表，root用户下，权限较大，未对文件读写权限做限制的条件下，再加上一些条件可以写shell的。同样的，如果要通过UDF或MOF提权也需要文件相关权限，普通用户在未被赋予相关权限的条件下是没法进行这些操作的。

补充一个写shell的前提条件：
1. 有文件权限
2. 知道web目录绝对路径
3. 对web目录可写
4. 可写的web目录有执行权限

### MSSQL呢
sqlserver是基于角色管理的啊？？？？当时我回答的啥玩意！

![](img/20180315-3.jpg)

#### 角色
sqlserver中角色分为数据库角色和服务器角色，管理员可以创建数据库角色，但是服务器角色是固定的。服务器角色是全局的，数据库角色是针对某一个数据库实例的。因此在权限管控上不应该给用户赋予服务器权限而应该服务数据库角色。

服务器角色：
角色名 | 说明
----|---
sysamdin | 就是那个sa,执行任何动作
serveradmin | 配置服务器设置,关闭服务器等
setupadmin | add and remove linked servers by using Transact-SQL statements
securityadmin | 管理登录和属性，使用grant/deny/revoke操作服务器级别、数据库级别的角色权限，重置登录密码
processadmin | 管理sql server进程
dbcreator | 创建、修改和删除数据库，还可以恢复数据库
diskadmin | 管理磁盘文件
bulkadmin | 使用bulk insert语句

数据库角色（可以看到数据库角色比服务器角色名多一个db前缀，表示的是数据库角色）
角色名 | 说明
----|---
db_owner | 执行数据库的所有配置和维护活动，包括删除数据库
db_securityadmin | 修改角色成员身份和管理权限
db_accessadmin | 可以添加、删除可以访问数据库的各类型用户
db_backupoperator | 备份数据库
db_ddladmin | run any Data Definition Language (DDL) command in a database,不知道DDL就得看数据库，哈哈哈哈
db_datawriter | run DML language
db_datareader | can read all data from all user tables.
db_denydatawriter | can not use DML language within a database
db_denydatareader | cannot read any data in the user tables within a database

然后我从官方文档找到一张不错的图

![](img/20180315-4.jpg)

#### 用户
用户是数据库级的概念，数据库用户必须绑定具体的登入名，也可以在新建登入名的时候绑定此登入名拥有的数据库，当你绑定登入名数据库后，数据库默认就创建了此登入名同名的数据库用户，登入名与数据库用户之间就存在关联关系，数据库用户是架构和数据库角色的拥有者，即你可以将某个架构分配给用户那么该用户就拥有了该架构所包含的对象，也可以将某个数据库角色分配给用户，此用户就拥有该数据库角色的权限。


#### 架构
架构是对象的拥有者，架构本身无权限，架构包含数据库对象：如表、视图、存储过程和函数等，平时最常见的默认架构dbo.,如果没指定架构默认创建数据库对象都是以dbo.开头，架构的拥有者是数据库用户、数据库角色、应用程序角色。用户创建的架构和角色只能作用于当前库。


#### 用户名和登录名（要有耐心看哇）
qlserver登录名和用户名之间是一种映射关系，同一个登录名可以对应多个用户名。用登录名登录SQLSERVER后，在访问各个数据库时，SQLSERVER会自动查询此数据库中是否存在与此登录名关联的用户名，若存在就使用此用户的权限访问此数据库，若不存在就是用guest用户访问此数据库。一个登录名可以被授权访问多个数据库，但一个登录名在每个数据库中只能映射一次。即一个登录可对应多个用户，一个用户也可以被多个登录使用。好比SQLSERVER就象一栋大楼,里面的每个房间都是一个数据库.登录名只是进入大楼的钥匙,而用户名则是进入房间的钥匙.一个登录名可以有多个房间的钥匙，但一个登录名在一个房间只能拥有此房间的一把钥匙。

我们常见的dbo（用户名）是指以sa(登录名)或windows/administration(Windows集成验证登录方式)登录的用户,也就是说数据库管理员在SQLSERVER中的用户名就叫dbo,而不叫 sa,这一点看起来有点蹊跷,因为通常用户名与登录名相同(不是强制相同,但为了一目了然通常都在创建用户名时使用与登录名相同的名字),例如创建了一个登录名称为me,那么可以为该登录名me在指定的数据库中添加一个同名用户,使登录名me能够访问该数据库中的数据.当在数据库中添加了一个用户me后,之后以me登录名登录时在该数据库中创建的一切对象(表,函数,存储过程等)的所有者都为me,如me.table1,me.fn_test(),而不是dbo.table1,dbo.fn_test()。

搞清楚了SQL server的角色管理，再分析一下在SQL注入中我们可能遇到什么情况。本来想直接写权限相关的，但是发现自己对sql server注入了解太少了，补一下sql server注入，这里要感谢网络上各类分析，以下很多内容来自这些参考，深表感谢。

一步一步来，先模拟一下一个数据库建立的过程，sqlserver服务器管理员(sysadmin或dbcreator权限用户，记不清了往上看)建立一个数据库，然后在server的安全中建立一个登录名（拥有某种server权限，一般public），映射到创建的数据库，然后再在创建的数据库的安全中建立用户，赋予db前缀的某些权限，就完成了。所以我们关注的权限问题涉及到两个部分，一个是server权限，如果server权限很大，比如sa那么可以胡作非为了，如果只是public那么就得关注一下相应数据库的权限，如果这个权限也很低，比如只有db_datareader权限，那么就只能读数据了。

#### sql server系统表
数据库 | 说明
----|---
master | 记录SQLServer 系统的所有系统级别信息（表sysobjects）。他记录所有的登录账号（表sysusers）和系统配置。记录所有其他的数据库（表sysdatabases），包括数据库文件的位置。渗透中主要会用到它
Model | 作为在系统上创建数据库的模板，新创建的数据库的第一部分内容从Model 数据库复制过来，剩余部分由空页填充，新数据库的最小的大小是Model库的大小，往往比它大
Tempdb | 保存系统运行过程中产生的临时表和存储过程
Msdb | 代理程序调度警报和作业以及记录操作员时使用。比如，我们备份了一个数据库，会在表backupfile中插入一条记录，以记录相关的备份信息

这里特别提一下一个视图sysobjects，在数据库内创建的每个对象（约束、默认值、日志、规则、存储过程等）在这个视图表中占一行。我们可以使用这个表查询数据库中有哪些数据库、表、列、存储过程是否可用等等信息，类似于在mysql中我们通过Infomation_schema来查库表列。

sqlserver注入的简单过程如下：
1. 判断是mssql，存在sysobjects表，说明是mssql
    1. `exists (select * from sysobjects) --`
    2. `exists (select count(*) from sysobjects) --`
2. 判断是不是站库分离
    查@@servername和host_name()是不是一样
3. 判断XP_CMDSHELL是否存在
    1. `Select count(*) FROM master..sysobjects Where xtype = 'X' AND name = 'xp_cmdshell' xtype='X'`表示是扩展存储类型
4. 判断权限
    1. `Select IS_SRVROLEMEMBER('sysadmin');`返回1 则是sa权限
    2. `Select IS_MEMBER('db_owner')`返回1 表示是db_owner权限
    3. `select IS_srvrpreemember('public')` public权限
    4. `Select HAS_DBACCESS('数据库名')` 判断是否有数据库访问权限

我觉得对sqlserver的权限控制搞明白后，自然也就明白了不同权限下注入的限制，举一个小例子，server创建的登录名赋予服务器角色public，具体隐射到的数据库用户的数据库角色是db_owner，那么可以查询本数据的所有东西，想使用xp_cmdshell基本上就不可能了，会没有执行权限。分析几个主要权限下的注入攻击。

#### 常见的存储过程
* xp_cmdshell———利用此存储过程可以直接执行系统命令
* xp_regread ———- 利用此存储过程可以进行注册表读取
* xp_regwrite———-利用此存储过程可以写入注册表
* xp_dirtree————利用此存储过程可以进行列目录操作
* xp_enumdsn——–利用此存储过程可以进行odbc连接
* xp_loginconfig—–利用此存储过程可以进行配置服务器安全模式信息

#### 检测存储过程是否启动
```SQL
and 1=(Select count(*) from master..sysobjects where xtype='X' and name='存储过程名称')--
```

#### 启用xp_cmdshell（要求数据库支持堆叠查询）：
```SQL
EXEC sp_addextendedproc xp_cmdshell ,@dllname ='xplog70.dll' --
EXEC sp_configure 'show advanced options', 1;RECONFIGURE;EXEC sp_configure 'xp_cmdshell', 1;RECONFIGURE--
//1为开启，0为关闭.
```
如提示：拒绝了对对象 'sp_addextendedproc' (数据库 'mssqlsystemresource'，架构 'sys')的 EXECUTE 权限。或提示'用户没有执行此操作的权限。'说明权限不够。如果执行不成功，那么可能xplog70.dll文件被删除了，我们可以上传一个xplog70.dll文件，然后进行恢复，语句如下：
```SQL
exec sp_addextendedproc 'xp_cmdshell','c:xplog70.dll'
```

#### 利用xp_cmdshell执行系统命令：
```SQL
exec master..xp_cmdshell "系统命令" --
```
1. sa权限
使用xp_cmdshell创建windows用户，远程登录。
没开远程桌面使用注册表修改存储过程
```SQL
exec master..xp_regwrite 'HKEY_LOCAL_MACHINE','SYSTEM\CurrentContrpreSet\Contrpre\Terminal Server','fDenyTSConnections','REG_DWORD',0;--
```

2. db_owner权限
* 获取web目录，利用xp_dirtree存储过程
* 差异备份写入shell
* 日志备份写shell

3. public权限
* 查数据，登后台突破
* 遍历服务器目录及文件

### Oracle
oracle能不能带外？可以
1. 使用UTL_HTTP.REQUEST accesslog/一个脚本接受数据
2. UTL_INADDR.get_host_addr dnslog
3. SYS.DBMS_LDAP.INIT dnslog
linux环境下数据库能不能带外,当时我说了原理用的ipc那个共享，想想windows下才有这个，最后测试linux下不行，当时回答问题太急了，想都没想说可以。。

### SSRF
有哪几种攻击方式
1. 内网探测
2. 使用一些协议可以带payload打内网的服务
3. dos攻击

这个建议看看猪猪侠的《一个只影响有钱人的漏洞》

### Windows域环境下内网渗透
具体的不写了，这里我下次单独总结一下，提一下一个地方，抓取hash后无法破解可以使用msf或者psexec进行hash注入

Kerberous协议,看着个吧[敞开的地狱之门：Kerberos协议的滥用](http://www.freebuf.com/articles/system/45631.html)

问了两个票据中的第一个在渗透中有什么特别的用处？第一个是TGT嘛，包含有关用户名、域名、时间和组成员资格等信息，拿着它去向票据授予服务器请求ST?

在微软活动目录中颁发的TGT是可移植的。由于Kerberos的无状态特性，TGT中并没有关于票据来源的标识信息。这意味着可以从某台计算机上导出一个有效的TGT，然后导入到该环境中其他的计算机上。新导入的票据可以用于域的身份认证，并拥有票据中指定用户的权限来访问网络资源。这种特别的攻击方法被称为"pass-the-ticket"攻击。查到相关资料，mimikatz可以完成这件事。

### 其他
代码审计，举了最近的Typecho反序列的例子，问Python的装饰器会不会，应用场景，当时应该答路由哇。。。其他的涉及我个人的问题，就不赘述了。