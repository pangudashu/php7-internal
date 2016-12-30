# 一起线上事故引发的对PHP超时控制的思考

几周以前我们的一个线上服务nginx请求日志里突然出现大量499、500、502的错误，于此同时发现php-fpm的worker进程不断的退出，新启动的worker几乎过几十秒就死掉了，在php-fpm.log里发现如下错误：

```
[28-Dec-2016 23:21:02] WARNING: [pool www] child 6528, script '/home/qinpeng/sofa/site/sofa/htdocs/test.php' (request: "GET /test.php") execution timed out (15.028107 sec), terminating
[28-Dec-2016 23:21:02] WARNING: [pool www] child 6528 exited on signal 15 (SIGTERM) after 53.265943 seconds from start
[28-Dec-2016 23:21:02] NOTICE: [pool www] child 26594 started
```
最终经过排查确定是因为访问redis没有设置读写超时，后端redis实例挂了导致请求阻塞而引发的故障，造成的影响非常严重，在故障期间整个服务完全不可用。

## 1、PHP的超时配置

### 1.1 max_input_time

### 1.2 max_execution_time

### 1.3 request_terminate_timeout

## 2、优化思路
