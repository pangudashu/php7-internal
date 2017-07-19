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
执行器实际是一个大循环，从第一条opcode开始执行，execute_data->opline指向当前执行的指令，执行完以后指向下一条指令，opline类似eip(或rip)寄存器的作用。通过这个循环，ZendVM完成opcode指令的执行，如下图。

![](../img/executor.png)




全局寄存器变量(Global Register Variables)是数据保存在寄存器中的一种变量类型，与存储在内存中的变量相比，寄存器变量具有更快的访问速度，在计算机的存储层次中，寄存器的速度最快，其次是内存，最慢的是内存。
