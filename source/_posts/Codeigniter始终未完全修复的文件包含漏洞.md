---
title: Codeigniter始终未完全修复的文件包含漏洞
date: 2019-04-17 21:04:01
tags: PHP代码审计
---
### Codeigniter介绍
Codeigniter是一款非常流行的PHP框架。我也曾经用这个框架开发了不少项目，因为结构简单，没有Laravel等框架那么复杂，在PHP开发圈使用量也比较大。

目前最新稳定版本是3.1.10。

### 文件包含漏洞
偶然翻到2016年PHITHON牛发过的[《codeigniter框架内核设计缺陷可能导致任意代码执行》](https://bugs.leavesongs.com/php/codeigniter%E6%A1%86%E6%9E%B6%E5%86%85%E6%A0%B8%E8%AE%BE%E8%AE%A1%E7%BC%BA%E9%99%B7%E5%8F%AF%E8%83%BD%E5%AF%BC%E8%87%B4%E4%BB%BB%E6%84%8F%E4%BB%A3%E7%A0%81%E6%89%A7%E8%A1%8C/#)文章：这是一个因为`extract()`变量覆盖+`include()`文件包含的文件包含漏洞（某些条件下可以升级为任意代码执行）。

这个漏洞的原理想了解的人可以看一下PHITHON这篇文章，这里简单说几句。

在Codeigniter当中，Controller向模板传递变量有三种方式：

```
<?php
defined('BASEPATH') OR exit('No direct script access allowed');

class Welcome extends CI_Controller {
    
    public function Test1()
    {
        $data['a'] = "Nihao";
        $this->load->view('welcome_message', $data);
    }

    public function Test2()
    {
        $this->load->vars(["a" => "Nihao"]);
        $this->load->view('welcome_message');
    }

    public function Test3()
    {
        $a = 'a';
        $b = "hello";
        $this->load->vars($a, $b);
        $this->load->view('welcome_message');
    }
}

```

这3种传参方式当中，任何一个数组的key可控，或者第一个参数可控，都可以读取任意文件：

```
<?php
defined('BASEPATH') OR exit('No direct script access allowed');

class Welcome extends CI_Controller {
    
    public function Test1()
    {
        $data['_ci_path'] = "C:/windows/win.ini";
        $this->load->view('welcome_message', $data);
    }

    public function Test2()
    {
        $this->load->vars(["_ci_path" => "C:/windows/win.ini"]);
        $this->load->view('welcome_message');
    }

    public function Test3()
    {
        $a = '_ci_path';
        $b = "C:/windows/win.ini";
        $this->load->vars($a, $b);
        $this->load->view('welcome_message');
    }
}
```
问题就是如果用户能传入`_ci_path`这个参数，那么在system/core/Loader.php的代码：

```
protected function _ci_load($_ci_data)
{
	// Set the default data variables
	foreach (array('_ci_view', '_ci_vars', '_ci_path', '_ci_return') as $_ci_val)
	{
		$$_ci_val = isset($_ci_data[$_ci_val]) ? $_ci_data[$_ci_val] : FALSE;
	}
	//省略无关代码
	empty($_ci_vars) OR $this->_ci_cached_vars = array_merge($this->_ci_cached_vars, $_ci_vars);
	extract($this->_ci_cached_vars);
	ob_start();
	if ( ! is_php('5.4') && ! ini_get('short_open_tag') && config_item('rewrite_short_tags') === TRUE)
	{
		echo eval('?>'.preg_replace('/;*\s*\?>/', '; ?>', str_replace('<?=', '<?php echo ', file_get_contents($_ci_path))));
	}
	else
	{
		include($_ci_path); // include() vs include_once() allows for multiple views with the same name
	}
```
`extract()`就会造成$_ci_path被覆盖,从而`include($_ci_path)`变成任意文件包含漏洞。

**这里重点看一下官方对这个问题的处理方式。**

这个漏洞官方在版本`3.0.5`进行了修复，修复方式如下([查看](https://github.com/bcit-ci/CodeIgniter/blob/3.0.5/system/core/Loader.php#L930))：
```
if (is_array($_ci_vars))
{
	foreach (array_keys($_ci_vars) as $key)
	{
		if (strncmp($key, '_ci_', 4) === 0)
		{
			unset($_ci_vars[$key]);
		}
	}
	$this->_ci_cached_vars = array_merge($this->_ci_cached_vars, $_ci_vars);
}
extract($this->_ci_cached_vars);
```
添加了一个`$_ci_vars`的检测逻辑，如果`$_ci_vars`是数组，并且它的key开头为`_ci_`,就`unset`掉这个变量，这样我们提交的`_ci_path=/etc/passwd`就不能用了，从而解决了漏洞。

但是这个函数只是用来处理上面三中传递模板的方式的第一种的(`Test1`)。

`Test2`和`Test3`两种方式的传参方法：

```
public function vars($vars, $val = '')
{
	if (is_string($vars))
	{
		$vars = array($vars => $val);
	}
	$vars = $this->_ci_object_to_array($vars);
	if (is_array($vars) && count($vars) > 0)
	{
		foreach ($vars as $key => $val)
		{
			$this->_ci_cached_vars[$key] = $val;
		}
	}
	return $this;
}

```
并未做任何处理，也就是说`Test2`和`Test3`两种方式依然存在漏洞。


**官方的这个修复方式在版本`3.1.3`出现了变更**。

取消了上面的检测代码，新添加了一个`_ci_prepare_view_vars`方法:
```
protected function _ci_prepare_view_vars($vars)
{
	if ( ! is_array($vars))
	{
		$vars = is_object($vars)
			? get_object_vars($object)
			: array();
	}
	foreach (array_keys($vars) as $key)
	{
		if (strncmp($key, '_ci_', 4) === 0)
		{
			unset($vars[$key]);
		}
	}
	return $vars;
}
```
这个方法，在两个地方进行了调用，第一个是:

```
public function view($view, $vars = array(), $return = FALSE)
{
	return $this->_ci_load(array('_ci_view' => $view, '_ci_vars' => $this->_ci_prepare_view_vars($vars), '_ci_return' => $return));
}
```
上面对`Test1`的调用方式进行了修复。

```
public function vars($vars, $val = '')
{
	$vars = is_string($vars)
		? array($vars => $val)
		: $this->_ci_prepare_view_vars($vars);
	foreach ($vars as $key => $val)
	{
		$this->_ci_cached_vars[$key] = $val;
	}
	return $this;
}
```
上面对`Test2`和`Test3`的利用方式进行了修复。

这么看来问题是都修复了。

问题就在于上面的代码，用了一个三目运算符：

```
$vars = is_string($vars)
		? array($vars => $val)
		: $this->_ci_prepare_view_vars($vars);
```

这意思就是说:`$this->load->var($param)`这种方式，如果`$param`是数组才会进行`$this->_ci_prepare_view_vars($param)`处理。

**如果是字符串就直接放过了，也就以下情况不处。也就是说`Test3`的利用方式没做过滤。**

```
public function Test3()
{
    $a = $_GET['a'];  //_ci_path
    $b = $_GET['b']; // /etc/passwd
    $this->load->vars($a, $b);
    $this->load->view('welcome_message');
}
```

直到最新稳定版，这个问题依然存在。

当然直接这么写代码的情况比较少，但是审计的项目多了，你会发现，你所理解的少的写法，在项目代码当中都可能会遇到。

并且这个问题在某些复杂的业务逻辑问题下，依然可能会转化成这个代码模式。