---
title: YII框架全版本文件包含漏洞
date: 2019-05-15 21:48:03
tags: PHP代码审计
---


文章首先发布在[先知社区](https://xz.aliyun.com/t/5051),转载请注明出处。

### 框架介绍
Yii框架是一个通用的WEB编程框架， 其代码简洁优雅，具有性能高，易于扩展等优点，在国内国内均具有庞大的使用群体。

###  漏洞介绍
首先需要说明的时候，这个漏洞不具备黑盒测试通用性，只有开发者利用yii所编写的应用存在某种用法，才有可能导致触发，但是对代码安全审计人员是一个很好的漏洞挖掘点。

由于控制器（Controller）向模板（View）注入变量的时候，采取了`extract($_params_, EXTR_OVERWRITE)`的模式，导致后面包含模板文件操作的`$_file_`变量可以在某些条件下任意覆盖，从而导致任意本地文件包含漏洞，严重可以导致在某些低php版本下执行任意php命令和远程文件包含操作。

### 漏洞详情

问题出现在 `framework/base/View.php`:

```
public function renderPhpFile($_file_, $_params_ = [])
    {
        $_obInitialLevel_ = ob_get_level();
        ob_start();
        ob_implicit_flush(false);
        extract($_params_, EXTR_OVERWRITE);   //overwrite 直接覆盖变量  l4yn3
        try {
            require $_file_;       //直接require $_file_变量，造成文件包含  l4yn3
            return ob_get_clean();
        } catch (\Exception $e) {
            while (ob_get_level() > $_obInitialLevel_) {
                if (!@ob_end_clean()) {
                    ob_clean();
                }
            }
            throw $e;
        } catch (\Throwable $e) {
            while (ob_get_level() > $_obInitialLevel_) {
                if (!@ob_end_clean()) {
                    ob_clean();
                }
            }
            throw $e;
        }
    }
```

这个方法当中存在任意变量覆盖问题，如果`$_param_`这个变量我们能控制，就能覆盖掉下面的`$_file_`变量。

跟进这个方法的调用链，发现同一个文件的`renderFile($viewFile, $params = [], $context = null)`方法调用了这个方法：

```
public function renderFile($viewFile, $params = [], $context = null)
    {
        $viewFile = $requestedFile = Yii::getAlias($viewFile);

        if ($this->theme !== null) {
            $viewFile = $this->theme->applyTo($viewFile);
        }
        if (is_file($viewFile)) {
            $viewFile = FileHelper::localize($viewFile);
        } else {
            throw new ViewNotFoundException("The view file does not exist: $viewFile");
        }

        $oldContext = $this->context;
        if ($context !== null) {
            $this->context = $context;
        }
        $output = '';
        $this->_viewFiles[] = [
            'resolved' => $viewFile,
            'requested' => $requestedFile
        ];

        if ($this->beforeRender($viewFile, $params)) {
            Yii::debug("Rendering view file: $viewFile", __METHOD__);
            $ext = pathinfo($viewFile, PATHINFO_EXTENSION);
            if (isset($this->renderers[$ext])) {
                if (is_array($this->renderers[$ext]) || is_string($this->renderers[$ext])) {
                    $this->renderers[$ext] = Yii::createObject($this->renderers[$ext]);
                }
                /* @var $renderer ViewRenderer */
                $renderer = $this->renderers[$ext];
                $output = $renderer->render($this, $viewFile, $params);
            } else {
                $output = $this->renderPhpFile($viewFile, $params);   //这里调用了漏洞方法l4yn3
            }
            $this->afterRender($viewFile, $params, $output);
        }

        array_pop($this->_viewFiles);
        $this->context = $oldContext;

        return $output;
    }
```

继续跟进，发现同样文件`View.php`的`render()`方法调用了上面的`renderFile()`方法，就此漏洞调用链出现。

```
render($view, $params = [], $context = null)

调用了

renderFile($viewFile, $params = [], $context = null)

调用了

renderPhpFile($_file_, $_params_ = [])   //存在漏洞
```


`render($view, $params = [], $context = null)`这个方法是Yii的Controller用来渲染视图的方法，也就是说我们只要控制了`render()`方法的`$params`变量，就完成了漏洞利用。


到此这个漏洞发展成了一个和 [《codeigniter框架内核设计缺陷可能导致任意代码执行》](https://bugs.leavesongs.com/php/codeigniter%E6%A1%86%E6%9E%B6%E5%86%85%E6%A0%B8%E8%AE%BE%E8%AE%A1%E7%BC%BA%E9%99%B7%E5%8F%AF%E8%83%BD%E5%AF%BC%E8%87%B4%E4%BB%BB%E6%84%8F%E4%BB%A3%E7%A0%81%E6%89%A7%E8%A1%8C/#)一样的漏洞。

### 漏洞利用
存在漏洞的写法如下：

```
public function actionIndex()
    {
        $data = Yii::$app->request->get();
        return $this->render('index', $data);
    }
```

这种情况下我们可以传递`_file_=/etc/passwd`来覆盖掉`require $_file_; `从而造成任意文件包含漏洞。

![](/uploads/yiivul1.png)
###  最后

这个漏洞已经提交给了Yii官方，但是一直没有收到任何回复。希望这篇文章能够帮助甲方用到Yii框架的代码审计人员，避免由这个问题造成严重的安全漏洞。


