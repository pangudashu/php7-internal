# PHP7内核剖析
````
原创内容，转载请注明出处~

代码版本：php-7.0.12
````
## 反馈
[交流&吐槽](https://github.com/pangudashu/php7-internal/issues/3)  [错误反馈](https://github.com/pangudashu/php7-internal/issues/2)

![](img/my_wx2.png)

## 目录：
* 第1章 PHP基本架构
   * 1.1 PHP简介
   * 1.2 PHP7的改进
   * [1.3 FPM](1/fpm.md)
      * [1.3.1 概述](1/fpm.md)
      * [1.3.2 基本实现](1/fpm.md)
      * [1.3.3 FPM的初始化](1/fpm.md)
      * [1.3.4 请求处理](1/fpm.md)
      * [1.3.5 进程管理](1/fpm.md)
   * [1.4 PHP执行的几个阶段](1/base_process.md)
* 第2章 变量
   * [2.1 变量的内部实现](2/zval.md)
   * [2.2 数组](2/zend_ht.md)
   * [2.3 静态变量](2/static_var.md)
   * [2.4 全局变量](2/global_var.md)
   * [2.5 常量](2/zend_constant.md)
* 第3章 Zend虚拟机
   * [3.1 PHP代码的编译](3/zend_compile.md)
      * [3.1.1 词法解析、语法解析](3/zend_compile_parse.md)
      * [3.1.2 抽象语法树编译流程](3/zend_compile_opcode.md)
   * [3.2 函数实现](3/function_implement.md)
      * [3.2.1 内部函数](3/function_implement.md)
      * <a href="3/function_implement.md#用户自定义函数的实现">3.2.2 用户函数的实现</a>
   * [3.3 Zend引擎执行流程](3/zend_executor.md)
      * <a href="3/zend_executor.md#331-数据结构">3.3.1 基本结构</a>
      * <a href="3/zend_executor.md#332-执行流程">3.3.2 执行流程</a>
      * <a href="3/zend_executor.md#333-函数的执行流程">3.3.3 函数的执行流程</a>
      * [3.3.4 全局execute_data和opline](3/zend_global_register.md)
   * 3.4 面向对象实现
      * [3.4.1 类](3/zend_class.md)
      * [3.4.2 对象](3/zend_object.md)
      * [3.4.3 继承](3/zend_extends.md)
      * [3.4.4 动态属性](3/zend_prop.md)
      * [3.4.5 魔术方法](3/zend_magic_method.md)
      * [3.4.6 类的自动加载](3/zend_autoload.md)
   * [3.5 运行时缓存](3/zend_runtime_cache.md)
   * 3.6 Opcache
      * 3.6.1 opcode缓存
      * 3.6.2 opcode优化
      * 3.6.3 JIT
* 第4章 PHP基础语法实现
   * [4.1 类型转换](4/type.md)
   * [4.2 选择结构](4/if.md)
   * [4.3 循环结构](4/loop.md)
   * [4.4 中断及跳转](4/break.md)
   * [4.5 include/require](4/include.md)
   * [4.6 异常处理](4/exception.md)
* 第5章 内存管理
   * [5.1 Zend内存池](5/zend_alloc.md)
   * [5.2 垃圾回收](5/gc.md)
* 第6章 线程安全
   * [6.1 什么是线程安全](6/ts.md)
   * [6.2 线程安全资源管理器](6/ts.md)
* 第7章 扩展开发
   * [7.1 概述](7/intro.md)
   * [7.2 扩展的实现原理](7/implement.md)
   * [7.3 扩展的构成及编译](7/extension_intro.md)
      * [7.3.1 扩展的构成](7/extension_intro.md)
      * [7.3.2 编译工具](7/extension_intro.md)
      * [7.3.3 编写扩展的基本步骤](7/extension_intro.md)
      * [7.3.4 config.m4](7/extension_intro.md)
   * [7.4 钩子函数](7/hook.md)
   * [7.5 运行时配置](7/conf.md)
     * [7.5.1 全局变量](7/conf.md)
	 * [7.5.2 ini配置](7/conf.md)
   * [7.6 函数](7/func.md)
      * <a href="7/func.md#761-内部函数注册">7.6.1 内部函数注册</a>
      * <a href="7/func.md#762-函数参数解析">7.6.2 函数参数解析</a>
      * <a href="7/func.md#763-引用传参">7.6.3 引用传参</a>
      * <a href="7/func.md#764-函数返回值">7.6.4 函数返回值</a>
      * <a href="7/func.md#765-函数调用">7.6.5 函数调用</a>
   * [7.7 zval的操作](7/var.md)
      * [7.7.1 新生成各类型zval](7/var.md)
      * [7.7.2 获取zval的值及类型](7/var.md)
      * [7.7.3 类型转换](7/var.md)
      * [7.7.4 引用计数](7/var.md)
      * [7.7.5 字符串操作](7/var.md)
      * [7.7.6 数组操作](7/var.md)
   * [7.8 常量](7/constant.md)
   * 7.9 面向对象
      * 7.9.1 内部类注册
      * 7.9.2 定义成员属性
      * 7.9.3 定义成员方法
      * 7.9.4 定义常量
      * 7.9.5 类的实例化
   * 7.10 资源类型
   * 7.11 经典扩展解析
      * 7.8.1 Yaf
      * 7.8.2 Redis
* 第8章 命名空间
   * [8.1 概述](8/namespace.md)
   * [8.2 命名空间的定义](8/namespace.md)
      * [8.2.1 定义语法](8/namespace.md)
      * [8.2.2 内部实现](8/namespace.md)
   * [8.3 命名空间的使用](8/namespace.md)
      * [8.3.1 基本用法](8/namespace.md)
      * [8.3.2 use导入](8/namespace.md)
      * [8.3.3 动态用法](8/namespace.md)
 
## 附录
   * [附录1：break/continue按标签中断语法实现](try/break.md)
   * 附录2：defer推迟函数调用语法的实现

