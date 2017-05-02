## 4.3 循环结构
实际应用中有许多具有规律性的重复操作，因此在程序中就需要重复执行某些语句。循环结构是在一定条件下反复执行某段程序的流程结构，被反复执行的程序被称为循环体。循环语句是由循环体及循环的终止条件两部分组成的。

PHP中的循环结构有4种：while、for、foreach、do while，接下来我们分析下这几个结构的具体的实现。

### 4.3.1 while循环
while循环的语法：
```php
while(expression) 
{
    statement;//循环体
} 
```
while的结构比较简单，由两部分组成：expression、statement，其中expression为循环判断条件，当expression为true时重复执行statement，具体的语法规则：
```c
statement:
    ...
    |   T_WHILE '(' expr ')' while_statement { $$ = zend_ast_create(ZEND_AST_WHILE, $3, $5); }
    ...
;

while_statement:
        statement { $$ = $1; }
    |   ':' inner_statement_list T_ENDWHILE ';' { $$ = $2; }
;
```
从while语法规则可以看出，在解析时会创建一个`ZEND_AST_WHILE`节点，expression、statement分别保存在两个子节点中，其AST如下：

![](../img/ast_while.png)

while编译的过程也比较简单，比较特别的是while首先编译的是循环体，然后才是循环判断条件，更像是do while，编译过程大致如下：
* __(1)__ 首先编译一条ZEND_JMP的opcode，这条opcode用来跳到循环判断条件expression的位置，由于while是先编译循环体再编译循环条件，所以此时还无法确定具体的跳转值；
* __(2)__ 编译循环体statement；编译完成后更新步骤(1)中ZEND_JMP的跳转值；
* __(3)__ 编译循环判断条件expression；
* __(4)__ 编译一条ZEND_JMPNZ的opcode，这条opcode用于循环判断条件执行完以后跳到循环体的，如果循环条件成立则通过此opcode跳到循环体开始的位置，否则继续往下执行(即：跳出循环)。

具体的编译过程：
```c
void zend_compile_while(zend_ast *ast)
{   
    zend_ast *cond_ast = ast->child[0];
    zend_ast *stmt_ast = ast->child[1];
    znode cond_node;
    uint32_t opnum_start, opnum_jmp, opnum_cond;
    
    //(1)编译ZEND_JMP
    opnum_jmp = zend_emit_jump(0);
    
    //(2)编译循环体statement，opnum_start为循环体起始位置
    opnum_start = get_next_op_number(CG(active_op_array));
    zend_compile_stmt(stmt_ast);
    
    //设置ZEND_JMP opcode的跳转值
    opnum_cond = get_next_op_number(CG(active_op_array));
    zend_update_jump_target(opnum_jmp, opnum_cond);

    //(3)编译循环条件expression
    zend_compile_expr(&cond_node, cond_ast);
    
    //(4)编译ZEND_JMPNZ，用于循环条件成立时跳回循环体开始位置：opnum_start
    zend_emit_cond_jump(ZEND_JMPNZ, &cond_node, opnum_start);
}
```
编译后opcode整体如下：

![](../img/while_run.png)

运行时首先执行`ZEND_JMP`，跳到while条件expression处开始执行，然后由`ZEND_JMPNZ`对条件的执行结果进行判断，如果条件成立则跳到循环体statement起始位置开始执行，如果条件不成立则继续向下执行，跳出while，第一次循环执行以后将不再执行`ZEND_JMP`，后续循环只有靠`ZEND_JMPNZ`控制跳转，循环体执行完成后接着执行循环判断条件，进行下一轮循环的判断。

> __Note:__ 实际执行时可能会省略`ZEND_JMPNZ`这一步，这是因为很多while条件expression执行完以后会对下一条opcode进行判断，如果是`ZEND_JMPNZ`则直接根据条件成立与否进行快速跳转，不需要再由`ZEND_JMPNZ`判断，比如：
>
> $a = 123;
> while($a > 100){
>     echo "yes";
> }
> `$a > 100`对应的opcode：ZEND_IS_SMALLER，执行时发现$a与100类型可以直接比较(都是long)，则直接就能知道循环条件的判断结果，这种情况下将会判断下一条opcode是否为ZEND_JMPNZ，是的话直接设置下一条要执行的opcode，这样就不需要再单独执行依次ZEND_JMPNZ了。
> 
> 上面的例子如果`$a = '123';`就不会快速进行处理了，而是按照正常的逻辑调用ZEND_JMPNZ。



