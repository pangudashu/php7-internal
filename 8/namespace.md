## 8.1 概述
什么是命名空间？从广义上来说，命名空间是一种封装事物的方法。在很多地方都可以见到这种抽象概念。例如，在操作系统中目录用来将相关文件分组，对于目录中的文件来说，它就扮演了命名空间的角色。具体举个例子，文件 foo.txt 可以同时在目录/home/greg 和 /home/other 中存在，但在同一个目录中不能存在两个 foo.txt 文件。另外，在目录 /home/greg 外访问 foo.txt 文件时，我们必须将目录名以及目录分隔符放在文件名之前得到 /home/greg/foo.txt。这个原理应用到程序设计领域就是命名空间的概念。(引用自php.net)

命名空间主要用来解决两类问题：
* 用户编写的代码与PHP内部的或第三方的类、函数、常量、接口名字冲突
* 为很长的标识符名称创建一个别名的名称，提高源代码的可读性

PHP命名空间提供了一种将相关的类、函数、常量和接口组合到一起的途径，不同命名空间的类、函数、常量、接口相互隔离不会冲突，注意：PHP命名空间只能隔离类、函数、常量和接口，不包括全局变量。

接下来的两节将介绍下PHP命名空间的内部实现，主要从命名空间的定义及使用两个方面分析。

## 8.2 命名空间的定义
### 8.2.1 定义语法
命名空间通过关键字namespace 来声明，如果一个文件中包含命名空间，它必须在其它所有代码之前声明命名空间，除了declare关键字以外，也就是说除declare之外任何代码都不能在namespace之前声明。另外，命名空间并没有文件限制，可以在多个文件中声明同一个命名空间，也可以在同一文件中声明多个命名空间。
```php
namespace com\aa;

const MY_CONST = 1234;
function my_func(){ /* ... */ }
class my_class { /* ... */ }
```
另外也可以通过{}将类、函数、常量封装在一个命名空间下：
```php
namespace com\aa{
    const MY_CONST = 1234;
    function my_func(){ /* ... */ }
    class my_class { /* ... */ }
}
```
但是同一个文件中这两种定义方式不能混用，下面这样的定义将是非法的：
```php
namespace com\aa{
    /* ... */
}

namespace com\bb;
/* ... */
```
### 8.2.2 内部实现
命名空间的实现实际比较简单，当声明了一个命名空间后，接下来编译类、函数和常量时会把类名、函数名和常量名统一加上命名空间的名称作为前缀存储，也就是说声明在命名空间中的类、函数和常量的实际名称是被修改过的，这样来看他们与普通的定义方式是没有区别的，只是这个前缀是内核帮我们自动添加的，例如：
```php
namespace com\aa;

const MY_CONST = 1234;
function my_func(){ /* ... */ }
class my_class { /* ... */ }
```
最终MY_CONST、my_func、my_class在EG(zend_constants)、EG(function_table)、EG(class_table)中的实际存储名称被修改为：com\aa\MY_CONST、com\aa\my_func、com\aa\my_class。

下面具体看下编译过程，namespace语法被编译为ZEND_AST_NAMESPACE类型的语法树节点，它有两个子节点：child[0]为命名空间的名称、child[1]为通过{}方式定义时包裹的语句。

![](../img/ast_namespace.png)

此节点的编译函数为zend_compile_namespace()：
```c
void zend_compile_namespace(zend_ast *ast)
{
    zend_ast *name_ast = ast->child[0];
    zend_ast *stmt_ast = ast->child[1];
    zend_string *name;
    zend_bool with_bracket = stmt_ast != NULL;

    //检查声明方式，不允许{}与非{}混用
    ...

    if (FC(current_namespace)) {
        zend_string_release(FC(current_namespace));
    }

    if (name_ast) {
        name = zend_ast_get_str(name_ast);

        if (ZEND_FETCH_CLASS_DEFAULT != zend_get_class_fetch_type(name)) {
            zend_error_noreturn(E_COMPILE_ERROR, "Cannot use '%s' as namespace name", ZSTR_VAL(name));
        }
        //将命名空间名称保存到FC(current_namespace)
        FC(current_namespace) = zend_string_copy(name);
    } else {
        FC(current_namespace) = NULL;
    }

    //重置use导入的命名空间符号表
    zend_reset_import_tables();
    ...
    if (stmt_ast) {
        //如果是通过namespace xxx { ... }这种方式声明的则直接编译{}中的语句
        zend_compile_top_stmt(stmt_ast);
        zend_end_namespace();
    }
}
```
从上面的编译过程可以看出，命名空间定义的编译过程非常简单，最主要的操作是把FC(current_namespace)设置为当前定义的命名空间名称，FC()这个宏为:CG(file_context)，前面曾介绍过，file_context是在编译过程中使用的一个结构：
```c
typedef struct _zend_file_context {
    zend_declarables declarables;
    znode implementing_class;

    //当前所属namespace
    zend_string *current_namespace;
    //是否在namespace中
    zend_bool in_namespace;
    //当前namespace是否为{}定义
    zend_bool has_bracketed_namespaces;

    //下面这三个值在后面介绍use时再说明，这里忽略即可
    HashTable *imports;
    HashTable *imports_function;
    HashTable *imports_const;
} zend_file_context;
```
编译完namespace声明语句后接着编译下面的语句，此后定义的类、函数、常量均属于此命名空间，直到遇到下一个namespace的定义，接下来继续分析下这三种类型编译过程中有何不同之处。

__(1)编译类、函数__

前面章节曾详细介绍过函数、类的编译过程，总结下主要分为两步：第1步是编译函数、类，这个过程将分别生成一条ZEND_DECLARE_FUNCTION、ZEND_DECLARE_CLASS的opcode；第2步是在整个脚本编译的最后执行zend_do_early_binding()，这一步相当于执行ZEND_DECLARE_FUNCTION、ZEND_DECLARE_CLASS，函数、类正是在这一步注册到EG(function_table)、EG(class_table)中去的。

在生成ZEND_DECLARE_FUNCTION、ZEND_DECLARE_CLASS两条opcode时会把函数名、类名的存储位置通过操作数记录下来，然后在zend_do_early_binding()阶段直接获取函数名、类名作为key注册到EG(function_table)、EG(class_table)中，定义在命名空间中的函数、类的名称修改正是在生成ZEND_DECLARE_FUNCTION、ZEND_DECLARE_CLASS时完成的，下面以函数为例看下具体的处理：
```c
//函数的编译方法
void zend_compile_func_decl(znode *result, zend_ast *ast)
{
    ...
    //生成函数声明的opcode：ZEND_DECLARE_FUNCTION
    zend_begin_func_decl(result, op_array, decl);
    
    //编译参数、函数体
    ...
}
```
```c
static void zend_begin_func_decl(znode *result, zend_op_array *op_array, zend_ast_decl *decl)
{
    ...
    //获取函数名称
    op_array->function_name = name = zend_prefix_with_ns(unqualified_name);
    lcname = zend_string_tolower(name);

    if (FC(imports_function)) {
        //如果通过use导入了其他命名空间则检查函数名称是否已存在
    }
    ....
    //生成一条opcode：ZEND_DECLARE_FUNCTION
    opline = get_next_op(CG(active_op_array));
    opline->opcode = ZEND_DECLARE_FUNCTION;
    //函数名的存储位置记录在op2中
    opline->op2_type = IS_CONST;
    LITERAL_STR(opline->op2, zend_string_copy(lcname));
    ...
}
```
函数名称通过zend_prefix_with_ns()方法获取：
```c
zend_string *zend_prefix_with_ns(zend_string *name) {
    if (FC(current_namespace)) {
        //如果当前是在namespace下则拼上namespace名称作为前缀
        zend_string *ns = FC(current_namespace);
        return zend_concat_names(ZSTR_VAL(ns), ZSTR_LEN(ns), ZSTR_VAL(name), ZSTR_LEN(name));
    } else {
        return zend_string_copy(name);
    }
}
```
在zend_prefix_with_ns()方法中如果发现FC(current_namespace)不为空则将函数名加上FC(current_namespace)作为前缀，接下来向EG(function_table)注册时就使用修改后的函数名作为key，类的情况与函数的处理方式相同，不再赘述。

__(2)编译常量__

常量的编译过程与函数、类基本相同，也是在编译过程获取常量名时检查FC(current_namespace)是否为空，如果不为空表示常量声明在namespace下，则为常量名加上FC(current_namespace)前缀。

总结下命名空间的定义：编译时如果发现定义了一个namespace，则将命名空间名称保存到FC(current_namespace)，编译类、函数、常量时先判断FC(current_namespace)是否为空，如果为空则按正常名称编译，如果不为空则将类名、函数名、常量名加上FC(current_namespace)作为前缀，然后再以修改后的名称注册。整个过程相当于PHP帮我们补全了类名、函数名、常量名。

## 8.3 使用命名空间
