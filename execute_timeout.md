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
sleep(20);

echo "hello~";
?>
```
`max_execution_time`配置的是10s，按照上面的猜测，浏览器请求test.php将因为超时不会有任何输出，并可能返回某个500以上的错误，我们来实际操作下（不要用cli执行）：
```
curl http://127.0.0.1:8000/test.php
```
结果输出：
```
hello~
```
很遗憾，结果不是预期的那样，脚本执行的很顺利，并没有中断，难道`max_execution_time`配置对fpm无效？网上有些文章认为"如果php-fpm中设置了 request_terminate_timeout 的话，那么 max_execution_time 就不生效"，事实上这是错误的，这俩值是没有任何关联的，下面我们就深入PHP内核看`max_execution_time`究竟怎么用。

grep下发现`max_execution_time`在`php_execute_script()`函数中有一处使用：
```
//main/main.c #line:2400
PHPAPI int php_execute_script(zend_file_handle *primary_file)
{
    ...
    zend_try {
        ...
        if (PG(max_input_time) != -1) {
            ...
            zend_set_timeout(INI_INT("max_execution_time"), 0);
        }

        ...
        zend_execute_scripts(...);
    }zend_end_try();
}
```
之前的一篇画的一幅图已经介绍过`php_execute_script()`函数的先后调用顺序：`php_module_startup` -> `php_request_startup` -> `php_execute_script` -> `php_request_shutdown` -> `php_module_shutdown`，它是PHP脚本的具体解析、执行的入口，`max_execution_time`在这个位置设置的可以进一步确定它控制的是PHP的执行时长，我们再到`zend_set_timeout()`中看下（去除了一些windows的无关代码）：
```
//Zend/zend_execute_API.c #line:1222
void zend_set_timeout(zend_long seconds, int reset_signals)
{
    EG(timeout_seconds) = seconds;
    ...
    {
        struct itimerval t_r;       /* timeout requested */
        int signo;

        if(seconds) {
            t_r.it_value.tv_sec = seconds;
            t_r.it_value.tv_usec = t_r.it_interval.tv_sec = t_r.it_interval.tv_usec = 0;
            setitimer(ITIMER_PROF, &t_r, NULL); //设定一个定时器，seconds秒后触发，到达时间后将发出ITIMER_PROF信号
        }
        signo = SIGPROF;

        if (reset_signals) {
#   ifdef ZEND_SIGNALS
            zend_signal(signo, zend_timeout);
#   else
            sigset_t sigset;

            signal(signo, zend_timeout); //设置信号处理函数，这个例子中就是设置ITIMER_PROF信号由zend_timeout()处理
            sigemptyset(&sigset);
            sigaddset(&sigset, signo);
            sigprocmask(SIG_UNBLOCK, &sigset, NULL);
#   endif
        }
    }
}
```
如果你用过C语言里面的定时器看到这里应该明白`max_execution_time`的含义了吧？`zend_set_timeout`设定了一个间隔定时器(itimer)，类型为`ITIMER_PROF`，问题就出在这，这个类型计算的程序在用户态、内核态下的`执行`时长，下面简单说下linux下的定时器及程序执行耗时的计量。

#### 1.2.1 间隔定时器itimer
间隔定时器设定的接口setitimer定义如下，setitimer()为Linux的API，并非C语言的Standard Library，setitimer()有两个功能，一是指定一段时间后，才执行某个function，二是每间格一段时间就执行某个function。
```
int setitimer(int which, const struct itimerval *value, struct itimerval *ovalue));

struct itimerval {
    struct timeval it_interval; //it_value时间后每隔it_interval执行
　　struct timeval it_value; //it_value时间后将开始执行
};

struct timeval {
    long tv_sec;
　　long tv_usec;
};
```
which为定时器类型：
* ITIMER_REAL : 以系统真实的时间来计算，它送出SIGALRM信号

* ITIMER_VIRTUAL : 以该进程在用户态下花费的时间来计算，它送出SIGVTALRM信号

* ITIMER_PROF : 以该进程在用户态下和内核态下所费的时间来计算，它送出SIGPROF信号

#### 1.2.2 内核态、用户态

#### 1.2.3 linux下程序的执行耗时

### 1.3 request_terminate_timeout

## 2、优化思路
