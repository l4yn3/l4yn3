---
title: Confluence 漏洞分析环境搭建
date: 2019-04-20 11:37:07
tags: Java漏洞分析
---

## 下载方式

https://product-downloads.atlassian.com/software/confluence/downloads/atlassian-confluence-6.9.0.zip

输入指定的版本进行下载。

## 目录结构

![](/uploads/confluence1.png)

`confluence`目录是程序目录，是我们分析漏洞的重点

其它文件夹都是tomcat容器相关的文件，比如`bin`目录放着tomcat和各个系统下的管理脚本，我们可以不管。

## 漏洞分析环境准备
1.用IDEA新建一个项目，然后用文件夹`confluence`作为WEB根目录。

2.将WEB-INF目录下的`lib`,`atlassian-bundled-plugins`和`atlassian-bundled-plugins`都添加成Library.

![](/uploads/confluence2.png)

![](/uploads/confluence3.png)

源码都是jar文件，添加成Library之后就可以直接用IDEA打开查看源代码了。

## WEB访问
#### 1. 配置修改
配置WEB访问，需要打开文件 `confluence/WEB-INF/classes/confluence-init.properties`,然后添加如下代码到最后：

`confluence.home=/anycode/atlassian-confluence-6.6.11/data`

后面的路径确保是存在的。

如果不设置·confluence.home·启动的时候会提示错误`setting your Confluence home`。

然后执行

```
./bin/startup.sh
```

完成启动。

#### 2.  访问 `http://localhost:8090`进行配置。
![](/uploads/confluence4.png)

![](/uploads/confluence5.png)

跳转到官网网站，如果没有账号就注册一个，接着跳转到`Generate License` 页面。

![](/uploads/confluence6.png)

![](/uploads/confluence7.png)

点no，复制 license key 到刚才的页面继续。


![](/uploads/confluence8.png)


![](/uploads/confluence9.png)

![](/uploads/confluence10.png)

## 静态调试

## 动态调试

## 参考资料
https://paper.seebug.org/884/
https://chybeta.github.io/2019/04/06/Analysis-for-%E3%80%90CVE-2019-3396%E3%80%91-SSTI-and-RCE-in-Confluence-Server-via-Widget-Connector/
