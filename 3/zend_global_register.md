### 3.3.4 全局execute_data和opline
Zend执行器在opcode的执行过程中，会频繁的用到execute_data和opline两个变量，execute_data为zend_execute_data结构，opline为当前执行的指令。普通的处理方式在执行每条opcode指令的handler时，会把execute_data地址作为参数传给handler使用，使用时先从当前栈上获取execute_data地址，然后再从堆上获取变量的数据，这种方式下Zend执行器展开后是下面这样：
```c
ZEND_API void execute_ex(zend_execute_data *ex)
{
    zend_execute_data *execute_data = ex;

    while (1) {
        int ret;

        if (UNEXPECTED((ret = ((opcode_handler_t)execute_data->opline->handler)(execute_data)) != 0)) {
            if (EXPECTED(ret > 0)) {
                execute_data = EG(current_execute_data);
            } else {
                return;
            }
        }
    }
}
```
执行器实际是一个大循环，从第一条opcode开始执行，execute_data->opline指向当前执行的指令，执行完以后指向下一条指令，opline类似eip(或rip)寄存器的作用。通过这个循环，ZendVM完成opcode指令的执行。opcode执行完后以后指向下一条指令的操作是在当前handler中完成，也就是说每条执行执行完以后会主动更新opline，这里会有下面几个不同的动作：
```c
ZEND_VM_CONTINUE()
ZEND_VM_ENTER()
ZEND_VM_LEAVE()
ZEND_VM_RETURN()
```
ZEND_VM_CONTINUE()表示继续执行下一条opcode；ZEND_VM_ENTER()/ZEND_VM_LEAVE()是调用函数时的动作，普通模式下ZEND_VM_ENTER()实际就是return 1，然后execute_ex()中会将execute_data切换到被调函数的结构上，对应的，在函数调用完成后ZEND_VM_LEAVE()会return 2，再将execute_data切换至原来的结构；ZEND_VM_RETURN()表示执行完成，比如exit，这时候execute_ex()将退出执行。下面看一个具体的例子：
```php
$a = "hi~";
echo $a;
```
执行过程如下图所示：

![](../img/executor.png)

以ZEND_ASSIGN这条赋值指令为例，其handler展开前如下：
```c
static ZEND_OPCODE_HANDLER_RET ZEND_FASTCALL ZEND_ASSIGN_SPEC_CV_CONST_HANDLER(ZEND_OPCODE_HANDLER_ARGS)
{
    USE_OPLINE
    ...
    ZEND_VM_NEXT_OPCODE_CHECK_EXCEPTION();
}
```
所有opcode的handler定义格式都是相同的，其参数列表通过ZEND_OPCODE_HANDLER_ARGS宏定义，展开后实际只有一个execute_data；ZEND_FASTCALL这个宏是用于指定C语言函数调用方式的，这里指定的是fastcall方式，GNU C下就是__attribute__((fastcall))。去掉一些非关键操作展开后：
```c
static int  __attribute__((fastcall)) ZEND_ASSIGN_SPEC_CV_CONST_HANDLER(zend_execute_data *execute_data)
{
    //USE_OPLINE
    const zend_op *opline = execute_data->opline;
    ...

    //ZEND_VM_NEXT_OPCODE_CHECK_EXCEPTION()
    execute_data->opline = execute_data->opline + 1;
    return 0;
}
```
从这个例子可以很清楚的看到，执行完以后会将execute_data->opline加1，也就是指向下一条opcode，然后返回0给execute_ex()，接着执行器在下一次循环时执行下一条opcode，依次类推，直至所有的opcode执行完成。

全局寄存器变量(Global Register Variables)是数据保存在寄存器中的一种变量，与存储在内存中的变量相比，寄存器变量具有更快的访问速度，在计算机的存储层次中，寄存器的速度最快，其次是内存，最慢的是内存。寄存器变量
