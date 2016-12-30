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

#### a. 间隔定时器itimer
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
* __ITIMER_REAL__ : 以__系统真实时间__来计算，它送出SIGALRM信号

* __ITIMER_VIRTUAL__ : 以该进程在__用户态__下花费的时间来计算，它送出SIGVTALRM信号

* __ITIMER_PROF__ : 以该进程在__用户态__下和__内核态__下所费的时间来计算，它送出SIGPROF信号

it_interval指定间隔时间，it_value指定初始定时时间。如果只指定it_value，就是实现一次定时；如果同时指定 it_interval，则超时后，系统会重新初始化it_value为it_interval，实现重复定时；两者都清零，则会清除定时器。

#### b. 内核态、用户态
操作系统的很多操作会消耗系统的物理资源，例如创建一个新进程时，要做很多底层的细致工作，如分配物理内存，从父进程拷贝相关信息，拷贝设置页目录、页表等，这些操作显然不能随便让任何程序都可以做，于是就产生了特权级别的概念，与系统相关的一些特别关键性的操作必须由高级别的程序来完成，这样可以做到集中管理，减少有限资源的访问和使用冲突。Intel的X86架构的CPU提供了0到3四个特权级，而在我们Linux操作系统中则主要采用了0和3两个特权级，也就是我们通常所说的内核态和用户态。

每个进程都有一个4G大小的虚拟地址空间，其中0~3G为用户空间，3~4G为内核空间，每个进程都有一各用户栈、内核栈，程序从用户空间开始执行，当发生`系统调用`、`发生异常`、`外设产生中断`时就从用户空间切换到内核空间，`系统调用`都有哪些呢？可以从kernal源码中查到：linux-4.9/arch/x86/entry/syscalls/syscall_xx.tbl，比如读写文件read/write、socket等。

PHP本质就是普通的C程序，所以我们直接按照C语言程序分析就行了，内核态、用户态的区分简单讲就是如果cpu当前执行在用户栈还是内核栈上，比如程序里写的if、for、+/-等都在用户态下执行，而读写文件、请求数据库则将切换到内核态。

#### c. linux IO模式
PHP中操作最多的就是IO，比如访问数据、rpc调用等等，因此这里单独分析下IO操作引起的进程挂起。

对于一次IO访问（以read举例），数据会先被拷贝到操作系统内核的缓冲区中，然后才会从操作系统内核的缓冲区拷贝到应用程序的地址空间，linux系统产生了下面五种网络模式：

* 阻塞 I/O（blocking IO）
* 非阻塞 I/O（nonblocking IO）
* I/O 多路复用（ IO multiplexing）
* 信号驱动 I/O（ signal driven IO）
* 异步 I/O（asynchronous IO）: linux下很少用

阻塞IO下当用户进程调用了recvfrom这个系统调用，kernel就开始了IO的第一个阶段：准备数据（对于网络IO来说，很多时候数据在一开始还没有到达。比如，还没有收到一个完整的UDP包。这个时候kernel就要等待足够的数据到来）。这个过程需要等待，也就是说数据被拷贝到操作系统内核的缓冲区中是需要一个过程的。而在用户进程这边，整个进程会被阻塞、休眠。当kernel一直等到数据准备好了，它就会将数据从kernel中拷贝到用户内存，然后kernel返回结果，用户进程才解除block的状态，重新运行起来。通过ps命令我们也可出fpm等待io响应时的状态:
```
xiaoju   26700  0.0  0.2 207812  5340 ?        S    Dec28   0:16 php-fpm: pool www
```
ps命令进程的状态：R 正在运行或可运行  S 可中断睡眠 (休眠中, 受阻, 在等待某个条件的形成或接受到信号)。


最后我们回到PHP，总结一下：

`ITIMER_VIRTUAL`定时器只会在`用户态`下倒计时，在内核态下将停止倒计时，`ITIMER_PROF`在两种状态下都倒计时，`ITIMER_REAL`则以系统实际时间倒计时，因为除了这两种状态，程序还有一种状态:`挂起`，也就是说`ITIMER_REAL`之外的两种定时器记录的都是进程的活跃状态，也就是cpu忙碌的状态，而读写文件、sleep、socket等操作因为等待时间发生而挂起的时间则不包括。到这里你应该已经明白我们上面例子`max_execution_time`为什么"不生效"的原因了吧？这个时间限制的是__执行__时间，不含io阻塞、sleep时长，所以PHP脚本的实际执行时间远远大于`max_execution_time`的设定。

所以如果PHP里的定时器`setitimer`用的是`ITIMER_REAL`或者用下面的代码测试，上面的例子结果就是我们预期了。
```
<?php
while(1){
}
?>
```
将返回: 500 Internal Server Error。

文章开始提到的故障就是因为PHP等待redis相应而引起fpm的worker进程挂起，而这段时间是不包含的`ITIMER_PROF`定时器的计时中的，所以可以确定fpm的退出并不是`max_execution_time`的原因，但是`max_execution_time`确实也是限制PHP执行的配置，我们接下来继续从源码看下`max_execution_time`超时时PHP是如何中断执行、返回错误的。

`zend_set_timeout()`函数中设定的`ITIMER_PROF`定时器超时信号处理函数为`zend_timeout()`：
```
//Zend/zend_execute_API.c #line:1181
ZEND_API void zend_timeout(int dummy)
{

    if (zend_on_timeout) {
        ...
        zend_on_timeout(EG(timeout_seconds));
    }

    zend_error_noreturn(E_ERROR, "Maximum execution time of %pd second%s exceeded", EG(timeout_seconds), EG(timeout_seconds) == 1 ? "" : "s");
}
```
不要着急去`zend_on_timeout`里看，注意这个函数__zend_error_noreturn()__，从函数名称可以猜测它抛出了一个error错误，实际这就是将PHP中断执行的操作：
```
//Zend/zend.c
ZEND_COLD void zend_error_noreturn(int type, const char *format, ...) __attribute__ ((alias("zend_error"),noreturn));

ZEND_API ZEND_COLD void zend_error(int type, const char *format, ...)
{
    ...

    /* if we don't have a user defined error handler */
    if (Z_TYPE(EG(user_error_handler)) == IS_UNDEF
            || !(EG(user_error_handler_error_reporting) & type)
            || EG(error_handling) != EH_NORMAL) {
        zend_error_cb(type, error_filename, error_lineno, format, args);
    } else switch (type) {
        ...
    }

    ...
}
```

### 1.3 request_terminate_timeout

## 2、优化思路
