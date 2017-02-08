# PHP7内核剖析

(更新中...)

* 第1章 PHP基本架构
   * [1.1 基本概念](base_introduction.md)
   * [1.2 PHP框架执行流程](base_process.md)
* 第2章 变量
   * [2.1 变量的内部实现](zval.md)
   * [2.2 数组](zend_ht.md)
   * [2.3 常量](var_common.md)
* 第3章 Zend虚拟机
   * [3.1 PHP代码的编译](zend_compile.md)
   * [3.2 函数实现](function_implement.md)
      * [3.2.1 内部函数](function_implement.md)
      * [3.2.2 用户函数的实现](function_implement.md#用户自定义函数的实现)
   * [3.3 Zend引擎执行流程](zend_executor.md)
   * 3.4 面向对象实现
* [第4章 PHP语法实现](php_language.md)
   * 4.1 变量
   * 4.2 运算符
   * 4.3 选择结构
   * 4.4 循环结构
   * 4.5 跳转语句
   * 4.6 函数
* 第5章 内存管理
   * 5.1 Zend内存池(zend_alloc.md)
   * 5.2 垃圾回收(gc.md)
* 第6章 扩展开发
