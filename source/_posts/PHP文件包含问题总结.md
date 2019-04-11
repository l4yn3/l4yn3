---
title: PHP安全漏洞所需条件总结
date: 2019-04-05 16:47:17
tags: PHP安全
---
PHP的漏洞当中，各种利用方式所需要的条件是不一样的。有的需要指定版本，有的需要指定的php.ini条件，做此记录，防止时间久了模糊。

### 1).截断问题

Method | PHP.INI| PHP VERSION
---|---|---
%00 | `magic_quotes_gpc = Off`| < php 5.3.4
`iconv` | 无限制| 无要求

*注:`magic_quotes_gpc = Off` 表示没有开启自动转义或者没有手动使用addslashes()方法手动转义*

### 2).注入问题

Method | PHP.INI | PHP VERSION
---|---|---
宽字节| 无限制|无要求
PDO参数绑定宽字节注入| 无限制| < php 5.3.6
iconv | `magic_quotes_gpc = On`|无要求
sprintf | `magic_quotes_gpc = On`|无要求
substr | `magic_quotes_gpc = On`|无要求
urldecode | 无限制|无要求

*注:1.`magic_quotes_gpc = On` 表示开启了自动转义或者手动使用了addslashes()方法手动转义*

*注:2.PDO参数绑定宽字节注入指即使采用了PDO参数绑定的形式，也没有阻止双字节编码注入问题。原因是PHP在 5.3.6 版本下的PDO参数绑定采用了本地模拟参数绑定的方式。修复需要额外设置`setAttribute(PDO::ATTR_EMULATE_PREPARES, false)`*

### 3).文件包含问题

Method | PHP.INI | PHP VERSION
---|---|---
LFI(Local File Inclusion) | 无限制|无要求
RFI(Remote File Inclusion) | `allow_url_fopen = On` <br>`allow_url_include = On`|无要求
php://input| `allow_url_include = On`|无要求
php://filter | 无限制|无要求
phar:// | 无限制| >=php5.3.0
zip:// | 无限制| >=php5.3.0
data:URI schema | 无限制| >=php5.2
zip:// | 无限制| >=php5.3.0
长度截断| 无限制| < php5.3.4

*注:长度截断问题，在linux下4096字节时会达到最大值，在window下是256字节。只要不断的重复`./`*
### 参考
[https://chybeta.github.io](https://chybeta.github.io/2017/10/08/php%E6%96%87%E4%BB%B6%E5%8C%85%E5%90%AB%E6%BC%8F%E6%B4%9E/)

