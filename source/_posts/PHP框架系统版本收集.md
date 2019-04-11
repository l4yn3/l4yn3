---
title: PHP框架系统版本收集
date: 2019-04-01 21:41:12
tags: 代码审计
---

#### 判断codeigniter框架和版本

```
grep -GR "define('CI_VERSION', '.*');"  ./
```

#### 判断yii框架和版本

```
 grep -GR -A3 "public static function getVersion()"  ./|grep "return '.*'"
```
和


```
grep -GR "\"yiisoft/yii2\": \"~.*\","
```

#### 判断thinkphp框架和版本

```
grep -GR "define('THINK_VERSION', '.*');"  ./
```

#### 判断Laravel以及版本

```
grep -GR "const VERSION = '.*'"  ./
```
如果没有数据则直接读取composer文件

```
 grep -GR "\"laravel/framework\": \".*.*\""  ./|grep composer.json
```

#### 判断Lumen框架和版本

```
grep -GR "public function version()"  ../|grep Lumen
```

#### 判断smarty模板引擎和版本

```
 grep -GR "@version .*" ./|grep Config_File
```

#### 判断是否使用了composer

```
find ./ -name="composer.json"
```

#### 判断是否使用了HDWIKI和版本信息

```
grep -GR "define('HDWIKI_VERSION', '.*');" ./
```



