---
title: PHP学习笔记
date: 2026-06-13 00:00:00
categories:
  - Web安全
tags:
  - PHP
---
# 变量

## 作用域

golbal

在所有函数外部定义的变量，拥有**==全局作用域==**，全局变量可以被脚本中的任何部分访问，要在一个**函数中**访问一个全局变量，需要使用 **==global 关键字==**。

static

当一个函数完成时，它的所有变量通常都会被删除。使用**==static关键字==**使局部变量不要被删除。

~~~php
<?php
function myTest()
{
    static $x=0;
    echo $x;
    $x++;
    echo PHP_EOL;    // 换行符
}
 
myTest();
myTest();
myTest();
?>
~~~

## echo 和print

- echo - 可以输出**一个或多个**字符串
- print - 只允许输出**一个字符串**，**返回值总为 1**

# EOF

~~~php
<?php
$name="runoob";
$a= <<<EOF
        "abc"$name
        "123"
EOF;
// 结束需要独立一行且前后不能空格
echo $a;
?>
~~~

作用是定义字符串，用**单引号和双引号**时，**==不用加转义字符==**

变量**不需要**用连接符 **.** 或 **,** 来拼接。（直接echo 需要）。

# 数据类型

## 全局变量和局部变量

```php
<?php
$x=5; // 全局变量

function myTest()
{
    $y=10; // 局部变量
    echo "<p>测试函数内变量:<p>";
    echo "变量 x 为: $x";
    echo "<br>";
    echo "变量 y 为: $y";
} 

myTest();

echo "<p>测试函数外变量:<p>";
echo "变量 x 为: $x";
echo "<br>";
echo "变量 y 为: $y";
?>
//当我们调用myTest()函数并输出两个变量的值, 函数将会输出局部变量 $y 的值，但是不能输出 $x 的值，因为 $x 变量在函数外定义，无法在函数内使用，如果要在一个函数中访问一个全局变量，需要使用 global 关键字。 然后我们在myTest()函数外输出两个变量的值，函数将会输出全局变量 $x 的值，但是不能输出 $y 的值，因为 $y 变量在函数中定义，属于局部变量。
```

### global()

```php
<?php
$x=5;
$y=10;
 
function myTest()
{
    global $x,$y;
    $y=$x+$y;
}
 
myTest();
echo $y; // 输出 15
?>
//关键字用于函数内访问全局变量。
//在函数内调用函数外定义的全局变量，我们需要在函数中的变量前加上 global 关键字
<?php
$x=5;
$y=10;
 
function myTest()
{
    $GLOBALS['y']=$GLOBALS['x']+$GLOBALS['y'];
} 
 
myTest();
echo $y;
?>
//PHP 将所有全局变量存储在一个名为 $GLOBALS[index] 的数组中。
//index 保存变量的名称。这个数组可以在函数内部访问，也可以直接用来更新全局变量。    
```

### static()

```php
<?php
function myTest()
{
    static $x=0;
    echo $x;
    $x++;
    echo PHP_EOL;    // 换行符
}
 
myTest();
myTest();
myTest();
?>
//输出结果为1，2，3
//当一个函数完成时，它的所有变量通常都会被删除。为了不让某些局部变量被删除，在第一次声明变量时用static
<?php
function myTest($x)
{
    echo $x;
}
myTest(5);
?>
//输出结果为5
//参数是通过调用代码将值传递给函数的局部变量。
//参数是在参数列表中声明的，作为函数声明的一部分.
```

## 整形

可以用3种格式来指定：十进制， 十六进制（ 以 **0x 为前缀**）或八进制（**前缀为 0**）。

### var_dum()

`var_dum()`可以返回**变量的类型和值**

```php
<?php 
$x = 5985;
var_dump($x);
echo "<br>"; 
$x = -345; // 负数 
var_dump($x);
echo "<br>"; 
$x = 0x8C; // 十六进制数
var_dump($x);
echo "<br>";
$x = 047; // 八进制数
var_dump($x);
?>
//int(5985)
//int(-345)
//int(140)
//int(39)
```

## 数组

~~~php
<?php
$cars=array("Volvo","BMW","TOtota");
var_dump($cars);
?>
array(3) {
  [0]=>
  string(5) "Volvo"
  [1]=>
  string(3) "BMW"
  [2]=>
  string(6) "TOtota"
}
~~~

## 对象

~~~php
<?php
class Car
{
  var $color;
  function __construct($color="green") {
    $this->color = $color;
  }
  function what_color() {
    return $this->color;
  }
}
?>
~~~

## 资源类型

```php
get_resource_type(resource $handle): string
```

`get_resource_type()`可以返回资源类型

## 类型比较

- 松散比较：使用两个等号 **==** 比较，**只比较值**，不比较类型。
- 严格比较：用三个等号 **===** 比较，**除了比较值**，**也比较类型**。

## 组合比较符

```php
$c = $a <=> $b;
```

- 如果 **$a > $b**, 则 **$c** 的值为 **1**。
- 如果 **$a == $b**, 则 **$c** 的值为 **0**。
- 如果 **$a < $b**, 则 **$c** 的值为 **-1**。

## 常量

通过`define()`定义

~~~php
define("GREETING", "欢迎访问 Runoob.com", true);
~~~

第一个参数是**变量名**，第二个参数是变量**值**，都是**必须的**。

第三个参数是控制**变量名大小写是否敏感**。**不是必须的**。

还可以通过const来定义

~~~php
const SITE_URL = "https://www.runoob.com";
echo SITE_URL; // 输出 "https://www.runoob.com"
~~~

### 常量数组

~~~php
define("FRUITS", [
    "Apple",
    "Banana",
    "Orange"
]);

echo FRUITS[0]; // 输出 "Apple"
~~~

或者用const

~~~php
const COLORS = [
    "Red",
    "Green",
    "Blue"
];

echo COLORS[1]; // 输出 "Green"
~~~

## 字符串

### **并置运算符**

`.`用于连接字符串

~~~php
<?php
$txt1="Hello world!";
$txt2="What a nice day!";
echo $txt1 . " " . $txt2;
?>
~~~

### PHP strlen() 函数

~~~php
<?php
echo strlen("Hello world!");
?>
~~~

返回字**符串的长度**

### PHP strpos() 函数

~~~php
<?php
echo strpos("Hello world!","world");
?>
~~~

返回要查找的字符串的**第一个位置**，若没有则返回**fasle**

# 运算符

| -x    | 设置负数 | 取 x 的相反符号                                              | `<?php $x = 2; echo -$x; ?>` | -2   |
| ----- | -------- | ------------------------------------------------------------ | ---------------------------- | ---- |
| ~x    | 取反     | x 取反，按二进制位进行"取反"运算。运算规则：`~1=-2;    ~0=-1;` | `<?php $x = 2; echo ~$x; ?>` | -3   |
| a . b | 并置     | 连接两个字符串                                               | "Hi" . "Ha"                  | HiHa |





## intdiv()

该函数返回值为**第一个参数除于第二个参数的值并取整**（向下取整）

## 不等于

可以用`!=`    也可以用`<>`

## 数组运算符

| 运算符  | 名称   | 描述                                                         |
| :------ | :----- | :----------------------------------------------------------- |
| x + y   | 集合   | x 和 y 的集合                                                |
| x == y  | 相等   | 如果 x 和 y 具有相同的键/值对，则返回 true                   |
| x === y | 恒等   | 如果 x 和 y 具有相同的键/值对，且顺序相同类型相同，则返回 true |
| x != y  | 不相等 | 如果 x 不等于 y，则返回 true                                 |
| x <> y  | 不相等 | 如果 x 不等于 y，则返回 true                                 |
| x !== y | 不恒等 | 如果 x 不等于 y，则返回 true                                 |



## @运算符

`@eval()`  抑制代码的**错误信息**



# 数组

## 获取数组的长度

count函数

统计数组中**==元素的个数==**

~~~php
<?php
$cars=array("Volvo","BMW","Toyota");
echo count($cars);
?>
~~~

## 关联数组

带有**指定的键**的数组，**每个键关联一个值**

~~~php
$age=array("Peter"=>"35","Ben"=>"37","Joe"=>"43");

$age['Peter']="35";
$age['Ben']="37";
$age['Joe']="43";
~~~

两种方法都行

~~~php
<?php
$age=array("Peter"=>"35","Ben"=>"37","Joe"=>"43");
echo "Peter is " . $age['Peter'] . " years old.";
?>
~~~

使用键的方法



### 遍历关联数组

~~~php
<?php
$age=array("Peter"=>"35","Ben"=>"37","Joe"=>"43");
 
foreach($age as $x=>$x_value)
{
    echo "Key=" . $x . ", Value=" . $x_value;
    echo "<br>";
}
?>
~~~

用**foreach循环**

## 数组排序

- sort() - 对数组进行**升序**排列
- rsort() - 对数组进行**降序**排列
- asort() - 根据关联数组的**值**，对数组进行**升序**排列
- ksort() - 根据关联数组的**键**，对数组进行**升序**排列
- arsort() - 根据关联数组的**值**，对数组进行**降序**排列
- krsort() - 根据关联数组的**键**，对数组进行**降序**排列



~~~php
<?php
$cars=array("Volvo","BMW","Toyota");
sort($cars);
?>
~~~



## 超级全局变量

$GLOBALS 是一个包含了**全部变量**的全局组合数组。**变量的名字就是数组的键**。

~~~php
<?php 
$x = 75; 
$y = 25;
 
function addition() 
{ 
    $GLOBALS['z'] = $GLOBALS['x'] + $GLOBALS['y']; 
}
 
addition(); 
echo $z; 
?>
~~~

可以通过**==$GLOBALS['x']==**访问变量x



## PHP $_SERVER

$_SERVER 是一个包含了诸如**头信息**(header)、**路径**(path)、以及**脚本位置**(script locations)等等信息的数组。



## PHP $_REQUEST

PHP $_REQUEST 用于收集HTML表单提交的**数据**。

~~~php
<html>
<body>
 
<form method="post" action="<?php echo $_SERVER['PHP_SELF'];?>">
Name: <input type="text" name="fname">
<input type="submit">
</form>
 
<?php 
$name = $_REQUEST['fname']; 
echo $name; 
?>
 
</body>
</html>
~~~

收集表单数据

**$REQUEST** **是一个==超全局数组==**，它默认包含了 **$GET**, **$_POST** 和 **$_COOKIE** 中的数据

## PHP $_GET

## PHP $_POST

这两个和上面的那个效果是一样的，只是分别接受**==get方法==**，和**==post方法==**传来的参数。



# foreach 循环

foreach循环用于**遍历数组**

有一下两种形式

~~~php
foreach ($array as $value)
{
    要执行代码;
}
~~~

每进行一次循环，当前数组元素的值就会被赋值给 $value 变量（**数组指针会逐一地移动**），在进行下一次循环时，可以看到数组中的下一个值。

~~~php
foreach ($array as $key => $value)
{
    要执行代码;
}
~~~

每一次循环，当前数组元素的键与值就都会被赋值给 $key 和 $value 变量（数字指针会逐一地移动），在进行下一次循环时，可以看到数组中的下一个**键与值**。





示例

~~~php
<?php
$x=array(1=>"Google", 2=>"Runoob", 3=>"Taobao");
foreach ($x as $key => $value)
{
    echo "key  为 " . $key . "，对应的 value 为 ". $value . PHP_EOL;
}
?>
~~~



# 函数

## 变量函数

变量函数是指在 PHP 中，将一个**变量作为函数名**来调用的函数。

~~~php
<?php
function foo() {
    echo "In foo()<br />\n";
}

function bar($arg = '')		//$arg = '' ，这是一个可选参数，如果没有传入实参，它默认被赋值为空字符串 ''
{
    echo "In bar(); argument was '$arg'.<br />\n";
}

// 使用 echo 的包装函数
function echoit($string)
{
    echo $string;
}

$func = 'foo';
$func();        // 调用 foo()

$func = 'bar';
$func('test');  // 调用 bar()

$func = 'echoit';
$func('test');  // 调用 echoit()
?>
~~~

## highlight_file() 

highlight_file() 是一个函数，用于对文件进行语法**==高亮显示==**。

`highlight_file(__FILE__);` 语法**高亮显示**当前代码。



## system()

`system()` 是 PHP 中用于执行外部程序的一个函数。即让操作系统去运行一个**==可执行文件==**或 **==shell 命令==**（例如 `ls`、`dir`、`cat`、`ping` 等），就像你在命令行终端输入命令一样。



`system('ls ')` 列出当前目录下的文件。

~~~php
system(string $command, int &$result_code = null): string|false
~~~



括号内的命令是字符串，因此要用引号

~~~
$filename = "test.txt";
system("cat $filename");   // 双引号，变量被解析 → 执行 cat test.txt
system('cat $filename');   // 单引号，变量不解析 → 执行 cat $filename（会报错，因为 shell 没有 $filename 变量）
system("cat " . $filename); // 用单引号涵盖固定部分，变量拼接，同样有效


~~~

## Shell中的引号

### 双引号

双引号是**弱引用**

- **保留空格** 和大部分特殊字符的字面含义（比如 `*`、`?`、`[ ]` 不再作为通配符）。
- **允许变量扩展**（`$var` 或 `${var}`）和**命令替换**（``cmd`` 或 `$(cmd)`）。
- 允许转义某些字符（如 `\$` 表示字面 `$`，`\"` 表示字面 `"`，`\\` 表示字面 `\`）。

~~~shell
name="world"
echo "Hello $name"      # 输出 Hello world
echo "Today is $(date)" # 执行 date 命令并输出结果
echo "*"                # 输出 *，而不是当前目录的文件列表
~~~

### 单引号

单引号是**强引用**（所有字符原样保留）

~~~php
echo 'Hello $name'      # 输出 Hello $name，变量不扩展
~~~





## eval

`eval()` 是一个特殊的函数，它能够将字符串作为 **PHP 代码**来执行。

`eval ( string $code )`



$code **不能包含**php的打开和关闭标签（如<?php   ?>）

且必须用**==分号==** ;正确终止



如果代码执行成功，返回 `NULL`，除非在执行的代码中调用了 `return` 语句，则该 `return` 的值就是 `eval()` 的**返回值**。



若执行的代码存在**语法错误**，会直接导致**==脚本终止运行==**。



## fputs()和fwrite()

作用 `fputs()` 是 `fwrite()` 的别名，向文件句柄中写入内容。

`fputs(resource $handle, string $string, int $length = ?)`

向一个文件中写入内容

第一个参数是文件的句柄，第二个参数是要写入的字符串。最后一个参数可无。

**返回值**：写入的**字节数**，失败返回 `false`。



## fopen()

**作用**：打开文件或 URL。

**语法**：

```php
fopen(string $filename, string $mode)
```

`$filename`：文件路径，这里是 `'mooyuan.php'`。

`$mode`：打开模式，`'w'` 表示 **只写方式写入**，从文件开头开始，若文件不存在则尝试创建，若存在则清空内容。

**返回值**：

成功返回文件句柄（resource 类型），供 `fputs` / `fwrite` 等函数使用。

失败返回 `false`。





## strpos()

~~~php
strpos(string $haystack, string $needle, int $offset = 0): int|false
~~~

- **`$haystack`**：被搜索的字符串
- **`$needle`**：要查找的子串
- **`$offset`**：可选，从哪个位置开始搜索（整数）
- **返回值**：
  - 找到时返回**首次出现的位置索引（从 0 开始）**
  - 未找到时返回 **`false`**



## include()

include()就相当于把**包含的文件内容**加载到了**当前文件**里

实际上include不是函数，而是**语言构造器**，后面可以不加括号



## phpinfo()

输出当前php环境的配置信息



## isset()

是 PHP 中非常常用的内置函数，用于检测变量是否已设置并且不是 `null`

**返回值**

- **`true`**：变量存在且其值不为 `null`
- **`false`**：变量不存在，或者存在但值为 `null`



## null和空字符

- **`null`** 表示“空值”或“无值”，它不属于字符串类型，而是独立的数据类型（`NULL`）。
- **`""`** 表示“空字符串”，是一个长度为 0 的字符串，属于字符串类型。



## exec()

exec()函数用于**==执行外部命令==**    

第一个参数是要**执行的命令**

第二个参数是用于存储命令输出结果的数组（每行输出作为**数组的一个元素**）



- **如果使用 `exec()`、`passthru()`**：它们默认**不经过 shell**，所以无论反引号还是 `$()` 都不会被解析，命令替换确实无效。
- **如果使用 `shell_exec()`、`system()`、反引号运算符**：它们会调用 shell，此时反引号和 `$()` 是**有效**的命令分隔/替换方式，除非额外做了转义或过滤。



## preg_match_all()

~~~php
preg_match_all(string $pattern, string $subject, array &$matches = null): int|false
~~~

- **`$pattern`**：正则表达式模式（需包含定界符，如 `/.../`）。
- **`$subject`**：被搜索的字符串。
- **`$matches`**（引用传递）：存储匹配结果的数组。结构由 `$flags` 决定。

~~~php
preg_match_all("/ /", $ip, $m)
~~~

**返回值**

- 返回**完整匹配的次数**（即使子组也可能匹配，但计数是完整匹配的次数）。
- 如果发生错误（如正则编译失败），返回 `false`。





# trait

**Trait** 是 PHP 提供的一种**==代码复用机制==**，用于解决 PHP 单继承（一个类只能继承一个父类）的限制。

~~~php
trait Logger {
    public function log($msg) {
        echo "Log: $msg";
    }
}

trait Timestamp {
    public function getTime() {
        return date('Y-m-d H:i:s');
    }
}

class User {
    use Logger, Timestamp;  // 同时引入两个 trait
}

$user = new User();
$user->log("User created");    // Log: User created
echo $user->getTime();          // 2026-04-25 ... 
~~~



# 魔数常量

## __TRAIT__

`__TRAIT__` 是 PHP 的一个**魔术常量**，它返回当前 **trait** 的完整名称。

| 名称              | 内容                   |
| ----------------- | ---------------------- |
| ____LINE____      | php文件中当前行号      |
| ____FILE____      | 文件的完整路径和文件名 |
| ____DIR____       | 目录                   |
| ____FUNCTION____  | 返回函数被定义时的名字 |
| ____CLASS__       | 返回类的名字           |
| ____TRAIT____     | 返回TRAIT的名字        |
| ____METHOD____    | 返回方法名             |
| ____NAMESPACE____ | 返回命名空间的名字     |



## `__FILE__`

`__FILE__` 是一个魔法常量，它代表当前文件的**完整路径和文件名**。



# 命名空间

命名空间解决了重名问题

命名空间相当于给代码加上了**==路径前缀==**，让不同模块的同名类可以共存。



文件的第一个代码必须是`namespace()`

~~~php
<?php
namespace MyProject;
// ... 其他代码

// 错误（前面有 HTML 或空行）
<html>
<?php
namespace MyProject;  // 致命错误
~~~

一个文件中可以定义**多个命名空间**（但**不推荐**）。推荐**一个文件只定义==一个命名空间==**，方便自动加载。



## 三种名称解析方式

假设当前命名空间为 `App\Controller`，我们看三种写法的解析结果

| 写法         | 示例                    | 解析结果                                   |
| :----------- | :---------------------- | :----------------------------------------- |
| 非限定名称   | `new User()`            | `App\Controller\User`                      |
| 限定名称     | `new Admin\User()`      | `App\Controller\Admin\User`                |
| 完全限定名称 | `new \App\Model\User()` | `App\Model\User`（前面反斜杠表示绝对路径） |





函数和变量会回**退到全局**

~~~php
namespace App\Util;

function strlen($str) {
    return \strlen($str) - 1;
}

echo strlen('hi');   // 调用当前空间的 strlen，输出 1
echo \strlen('hi');  // 调用全局 strlen，输出 2
echo PHP_VERSION;    // 当前空间没有此常量，自动回退到全局常量
~~~



但是**类名不会**

~~~php
namespace App\Util;
$arr = new ArrayObject();  // 致命错误：找不到 App\Util\ArrayObject
$arr = new \ArrayObject(); // 正确
~~~



## use用法

**基本用法**

~~~php
use App\Model\User;   //导入类
~~~

**一行导入多个**

~~~php
use App\Model\{User, Article, Comment};  // PHP 7+ 语法
~~~

**导入函数和常量**

~~~php
use function App\Helper\formatDate;
use const App\Config\APP_NAME;
~~~

**导入对动态类名无效**

~~~php
use App\Model\User;

$className = 'User';
$obj = new $className();   // 错误！解析为当前空间下的 \App\Controller\User
// 必须使用完全限定名的字符串：
$className = '\App\Model\User';
$obj = new $className();   // 正确
~~~



## namespace

类似于this

~~~php
namespace App\Util;

namespace\foo();              // 调用 App\Util\foo()
new namespace\MyClass();      // 实例化 App\Util\MyClass
~~~



# 析构函数

析构函数(destructor) 与构造函数相反，当对象**结束其生命周期时**（例如对象所在的函数已调用完毕），系统**==自动执行==析构函数**。

~~~php

   function __destruct() {
       print "销毁 " . $this->name . "\n";
   }
~~~



# 继承

PHP 使用关键字 **extends** 来**==继承一个类==**，PHP 不支持多继承

~~~php
<?php 
// 子类扩展站点类别
class Child_Site extends Site {
   var $category;

    function setCate($par){
        $this->category = $par;
    }
  
    function getCate(){
        echo $this->category . PHP_EOL;
    }
}
~~~



# 方法重写

如果**从父类继承的方法**不能满足子类的需求，可以**对其进行改写**，这个过程叫**方法的覆盖**（override），也称为**==方法的重写==**。

~~~php
function getUrl() {
   echo $this->url . PHP_EOL;
   return $this->url;
}
~~~

**重写方法**



# 访问控制

PHP 对属性或方法的访问控制，是通过在前面添加关键字 public（公有），protected（受保护）或 private（私有）来实现的。

- **public（公有）：**公有的类成员可以在**任何地方**被访问。
- **protected（受保护）：**受保护的类成员则可以被**其自身**以及**其子类和父类**访问。
- **private（私有）：**私有的类成员则只能被**其定义所在的类**访问。



类中的方法可以被定义为公有，私有或受保护。如果**没有设置这些关键字**，则该方法默认为**==公有==**。

类属性必须定义为公有，受保护，私有之一。如果用 **var** 定义，则被视为**==公有==**。



# 接口

使用**接口**（interface），可以指定某个类**==必须实现哪些方法==**，但**不需要定义**这些方法的**具体内容**。

接口是通过 **interface** 关键字来定义的，就像定义一个标准的类一样，但其中定义所有的**==方法都是空的==**。

要实现一个接口，使用 **implements** 操作符。

~~~php4
<?php

// 声明一个'iTemplate'接口
interface iTemplate
{
    public function setVariable($name, $var);
    public function getHtml($template);
}


// 实现接口
class Template implements iTemplate
{
    private $vars = array();
  
    public function setVariable($name, $var)
    {
        $this->vars[$name] = $var;
    }
  
    public function getHtml($template)
    {
        foreach($this->vars as $name => $value) {
            $template = str_replace('{' . $name . '}', $value, $template);
        }
 
        return $template;
    }
}
~~~



# 常量

在定义和使用常量的时候**不需要使用** $ 符号。

~~~php
<?php
class MyClass
{
    const constant = '常量值';

    function showConstant() {
        echo  self::constant . PHP_EOL;    //使用常量
    }
}

~~~

`::` 是范围解析操作符，用来引用类的静态成员（常量、静态属性、静态方法）

`self` 指向类自身（类似 `MyClass`），而不是某个实例。



# 抽象类

任何一个类，如果它里面至少有一个**方法是被声明为抽象的**，那么这个类就必须被声明为**抽象的**。

主要作用：作为基类，**强制子类==必须实现==其中的抽象方法**。

**==不能被实例化==**

## 抽象方法

只声明了其==**调用方式（参数）**==，没定义其**具体的功能实现**。

(需要在子类中自己写**具体的方法**)



在子类中这些方法的访问控制必须**和父类中一样**（**或者更为宽松**）

~~~php
<?php
abstract class AbstractClass
{
 // 强制要求子类定义这些方法
    abstract protected function getValue();
    abstract protected function prefixValue($prefix);

    // 普通方法（非抽象方法）
    public function printOut() {
        print $this->getValue() . PHP_EOL;
    }
}
~~~



# Static

声明类属性或方法为 static(静态)，就可以**不实例化类**而直接访问。

静态属性**不能通过一个类已实例化的对象来访问**（但**静态方法可以**）。



~~~php
<?php
class Foo {
  public static $my_static = 'foo';
  
  public function staticValue() {
     return self::$my_static;
  }
}

print Foo::$my_static . PHP_EOL;
$foo = new Foo();

print $foo->staticValue() . PHP_EOL;
?>    
~~~



# Final

PHP 5 新增了一个 final 关键字。如果父类中的方法被声明为 final，则**子类无法覆盖该方法**。如果一个类被声明为 final，则**==不能被继承==**。

~~~php
<?php
class BaseClass {
   public function test() {
       echo "BaseClass::test() called" . PHP_EOL;
   }
   
   final public function moreTesting() {
       echo "BaseClass::moreTesting() called"  . PHP_EOL;
   }
}

class ChildClass extends BaseClass {
   public function moreTesting() {
       echo "ChildClass::moreTesting() called"  . PHP_EOL;
   }
}
// 报错信息 Fatal error: Cannot override final method BaseClass::moreTesting()
?>
~~~

上述代码会报错



# 调用父类构造方法

PHP 不会在子类的构造方法中自动的调用父类的构造方法。要执行父类的构造方法，需要在子类的构造方法中调用 **parent::__construct()** 。

~~~php
<?php
class BaseClass {
   function __construct() {
       print "BaseClass 类中构造方法" . PHP_EOL;
   }
}
class SubClass extends BaseClass {
   function __construct() {
       parent::__construct();  // 子类构造方法不能自动调用父类的构造方法
       print "SubClass 类中构造方法" . PHP_EOL;
   }
}
~~~



# allow_url_open = On

`allow_url_open = On`

- **含义**：允许PHP的文件系统函数（如 `fopen()`, `file_get_contents()`, `copy()` 等）处理**远程资源**（如HTTP、FTP URL）。
- **开启后的能力**：正常的程序可以更方便地获取远程内容。



# allow_url_include = On

- **含义**：允许 `include`, `require`, `include_once`, `require_once` 等语句包含**远程资源**（如HTTP、FTP URL）作为代码执行。
