# 一起线上事故引发的对PHP超时控制的思考

几周以前我们的一个线上服务nginx请求日志里突然出现大量499、500、502的错误，于此同时发现php-fpm的worker进程不断的退出，新启动的worker几乎过几十秒就死掉了，在php-fpm.log里发现如下错误：

```
[28-Dec-2016 23:21:02] WARNING: [pool www] child 6528, script '/home/qinpeng/sofa/site/sofa/htdocs/test.php' (request: "GET /test.php") execution timed out (15.028107 sec), terminating
[28-Dec-2016 23:21:02] WARNING: [pool www] child 6528 exited on signal 15 (SIGTERM) after 53.265943 seconds from start
[28-Dec-2016 23:21:02] NOTICE: [pool www] child 26594 started
```
从日志里也可以看出fpm worker进程因为执行超时(超过15s)而被kill掉了。

最终经过排查确定是因为访问redis没有设置读写超时，后端redis实例挂了导致请求阻塞而引发的故障，事故造成的影响非常严重，在故障期间整个服务完全不可用。

事后我一直不解为什么超时会导致fpm的退出？于是看了下php及php-fpm的几个超时配置的实现，最终得到了答案。与这次事故相关的主要是两个配置：一个是php中的max_execution_time，另一个是fpm中的request_terminate_timeout，下面将根据这两个配置具体分析PHP内核是如何处理的。(源码版本:php-7.0.12)

## 1、PHP的超时配置

### 1.1 max_input_time
这个配置在php.ini中，含义是PHP解析请求数据的最大耗时，如解析GET、POST参数等，这个参数控制的PHP从解析请求到执行PHP脚本的超时，也就是从php_request_startup()到php_execute_script()之间的耗时。

此配置默认值为60s，cli模式下被强制设为-1，关于这个参数没有什么可说的，不再展开分析，下面重点分析`max_execution_time`。

### 1.2 max_execution_time
此配置也在php.ini中，也就是说它是php的配置而不是fpm的，从源码注释上看这个配置的含义是：每个PHP脚本的最长执行时间。

默认值为30s，cli模式下为0(即cli下此配置不生效)。

从字面意义上猜测这个配置控制的是整个PHP脚本的最大执行耗时，也就是超过这个值PHP就不再执行了，那么导致fpm退出的原因是不是它造成的呢？我们用下面的例子测试下(max_execution_time = 10s)：
```
//test.php
<?php
sleep(11);

echo "hello~";
?>
```
`max_execution_time`配置的是10s，按照上面的猜测，浏览器请求test.php将因为超时不会有任何输出，并可能返回500以上的错误，我们来实际操作下（不要用cli执行）：
```
curl http://127.0.0.1:8000/test.php
```
结果输出：
```
hello~
```


### 1.3 request_terminate_timeout

## 2、优化思路
