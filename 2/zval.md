## 2.1 变量的内部实现

变量是一个语言实现的基础，变量有两个组成部分：变量名、变量值，PHP中可以将其对应为：zval、zend_value，这两个概念一定要区分开，PHP中变量的内存是通过引用计数进行管理的，而且PHP7中引用计数是在zend_value而不是zval上，变量之间的传递、赋值通常也是针对zend_value。

PHP中可以通过`$`关键词定义一个变量：`$a;`，在定义的同时可以进行初始化：`$a = "hi~";`，注意这实际是两步：定义、初始化，只定义一个变量也是可以的，可以不给它赋值，比如：
```php
$a;
$b = 1;
```
这段代码在执行时会分配两个zval。

接下来我们具体看下变量的结构以及不同类型的实现。

### 2.1.1 变量的基础结构
```c
//zend_types.h
typedef struct _zval_struct     zval;

typedef union _zend_value {
    zend_long         lval;    //int整形
    double            dval;    //浮点型
    zend_refcounted  *counted;
    zend_string      *str;     //string字符串
    zend_array       *arr;     //array数组
    zend_object      *obj;     //object对象
    zend_resource    *res;     //resource资源类型
    zend_reference   *ref;     //引用类型，通过&$var_name定义的
    zend_ast_ref     *ast;     //下面几个都是内核使用的value
    zval             *zv;
    void             *ptr;
    zend_class_entry *ce;
    zend_function    *func;
    struct {
        uint32_t w1;
        uint32_t w2;
    } ww;
} zend_value;

struct _zval_struct {
    zend_value        value; //变量实际的value
    union {
        struct {
            ZEND_ENDIAN_LOHI_4( //这个是为了兼容大小字节序，小字节序就是下面的顺序，大字节序则下面4个顺序翻转
                zend_uchar    type,         //变量类型
                zend_uchar    type_flags,  //类型掩码，不同的类型会有不同的几种属性，内存管理会用到
                zend_uchar    const_flags,
                zend_uchar    reserved)     //call info，zend执行流程会用到
        } v;
        uint32_t type_info; //上面4个值的组合值，可以直接根据type_info取到4个对应位置的值
    } u1;
    union {
        uint32_t     var_flags;
        uint32_t     next;                 //哈希表中解决哈希冲突时用到
        uint32_t     cache_slot;           /* literal cache slot */
        uint32_t     lineno;               /* line number (for ast nodes) */
        uint32_t     num_args;             /* arguments number for EX(This) */
        uint32_t     fe_pos;               /* foreach position */
        uint32_t     fe_iter_idx;          /* foreach iterator index */
    } u2; //一些辅助值
};
```
`zval`结构比较简单，内嵌一个union类型的`zend_value`保存具体变量类型的值或指针，`zval`中还有两个union：`u1`、`u2`:
* __u1：__ 它的意义比较直观，变量的类型就通过`u1.v.type`区分，另外一个值`type_flags`为类型掩码，在变量的内存管理、gc机制中会用到，第三部分会详细分析，至于后面两个`const_flags`、`reserved`暂且不管
* __u2：__ 这个值纯粹是个辅助值，假如`zval`只有:`value`、`u1`两个值，整个zval的大小也会对齐到16byte，既然不管有没有u2大小都是16byte，把多余的4byte拿出来用于一些特殊用途还是很划算的，比如next在哈希表解决哈希冲突时会用到，还有fe_pos在foreach会用到......

从`zend_value`可以看出，除`long`、`double`类型直接存储值外，其它类型都为指针，指向各自的结构。

### 2.1.2 类型
`zval.u1.type`类型：
```c
/* regular data types */
#define IS_UNDEF                    0
#define IS_NULL                     1
#define IS_FALSE                    2
#define IS_TRUE                     3
#define IS_LONG                     4
#define IS_DOUBLE                   5
#define IS_STRING                   6
#define IS_ARRAY                    7
#define IS_OBJECT                   8
#define IS_RESOURCE                 9
#define IS_REFERENCE                10

/* constant expressions */
#define IS_CONSTANT                 11
#define IS_CONSTANT_AST             12

/* fake types */
#define _IS_BOOL                    13
#define IS_CALLABLE                 14

/* internal types */
#define IS_INDIRECT                 15
#define IS_PTR                      17
```

#### 2.1.2.1 标量类型
最简单的类型是true、false、long、double、null，其中true、false、null没有value，直接根据type区分，而long、double的值则直接存在value中：zend_long、double，也就是标量类型不需要额外的value指针。

#### 2.1.2.2 字符串
PHP中字符串通过`zend_string`表示:
```c
struct _zend_string {
    zend_refcounted_h gc;
    zend_ulong        h;                /* hash value */
    size_t            len;
    char              val[1];
};
```
* __gc：__ 变量引用信息，比如当前value的引用数，所有用到引用计数的变量类型都会有这个结构，3.1节会详细分析
* __h：__ 哈希值，数组中计算索引时会用到
* __len：__ 字符串长度，通过这个值保证二进制安全
* __val：__ 字符串内容，变长struct，分配时按len长度申请内存

事实上字符串又可具体分为几类：IS_STR_PERSISTENT(通过malloc分配的)、IS_STR_INTERNED(php代码里写的一些字面量，比如函数名、变量值)、IS_STR_PERMANENT(永久值，生命周期大于request)、IS_STR_CONSTANT(常量)、IS_STR_CONSTANT_UNQUALIFIED，这个信息通过flag保存：zval.value->gc.u.flags，后面用到的时候再具体分析。

#### 2.1.2.3 数组
array是PHP中非常强大的一个数据结构，它的底层实现就是普通的有序HashTable，这里简单看下它的结构，下一节会单独分析数组的实现。

```c
typedef struct _zend_array HashTable;

struct _zend_array {
    zend_refcounted_h gc; //引用计数信息，与字符串相同
    union {
        struct {
            ZEND_ENDIAN_LOHI_4(
                zend_uchar    flags,
                zend_uchar    nApplyCount,
                zend_uchar    nIteratorsCount,
                zend_uchar    reserve)
        } v;
        uint32_t flags;
    } u;
    uint32_t          nTableMask; //计算bucket索引时的掩码
    Bucket           *arData; //bucket数组
    uint32_t          nNumUsed; //已用bucket数
    uint32_t          nNumOfElements; //已有元素数，nNumOfElements <= nNumUsed，因为删除的并不是直接从arData中移除
    uint32_t          nTableSize; //数组的大小，为2^n
    uint32_t          nInternalPointer; //数值索引
    zend_long         nNextFreeElement;
    dtor_func_t       pDestructor;
};
```
#### 2.1.2.4 对象/资源
```c
struct _zend_object {
    zend_refcounted_h gc;
    uint32_t          handle;
    zend_class_entry *ce; //对象对应的class类
    const zend_object_handlers *handlers;
    HashTable        *properties; //对象属性哈希表
    zval              properties_table[1];
};

struct _zend_resource {
    zend_refcounted_h gc;
    int               handle;
    int               type;
    void             *ptr;
};
```
对象比较常见，资源指的是tcp连接、文件句柄等等类型，这种类型比较灵活，可以随意定义struct，通过ptr指向，后面会单独分析这种类型，这里不再多说。

#### 2.1.2.5 引用
引用是PHP中比较特殊的一种类型，它实际是指向另外一个PHP变量，对它的修改会直接改动实际指向的zval，可以简单的理解为C中的指针，在PHP中通过`&`操作符产生一个引用变量，也就是说不管以前的类型是什么，`&`首先会创建一个`zend_reference`结构，其内嵌了一个zval，这个zval的value指向原来zval的value(如果是布尔、整形、浮点则直接复制原来的值)，然后将原zval的类型修改为IS_REFERENCE，原zval的value指向新创建的`zend_reference`结构。
```c
struct _zend_reference {
    zend_refcounted_h gc;
    zval              val;
};
```
结构非常简单，除了公共部分`zend_refcounted_h`外只有一个`val`，举个示例看下具体的结构关系：
```php
$a = "time:" . time();      //$a    -> zend_string_1(refcount=1)
$b = &$a;                   //$a,$b -> zend_reference_1(refcount=2) -> zend_string_1(refcount=1)
```
最终的结果如图：

![ref](../img/zend_ref.png)

注意：引用只能通过`&`产生，无法通过赋值传递，比如：
```php
$a = "time:" . time();      //$a    -> zend_string_1(refcount=1)
$b = &$a;                   //$a,$b -> zend_reference_1(refcount=2) -> zend_string_1(refcount=1)
$c = $b;                    //$a,$b -> zend_reference_1(refcount=2) -> zend_string_1(refcount=2)
                            //$c    ->                                 ---
```
`$b = &$a`这时候`$a`、`$b`的类型是引用，但是`$c = $b`并不会直接将`$b`赋值给`$c`，而是把`$b`实际指向的zval赋值给`$c`，如果想要`$c`也是一个引用则需要这么操作：
```php
$a = "time:" . time();      //$a       -> zend_string_1(refcount=1)
$b = &$a;                   //$a,$b    -> zend_reference_1(refcount=2) -> zend_string_1(refcount=1)
$c = &$b;/*或$c = &$a*/     //$a,$b,$c -> zend_reference_1(refcount=3) -> zend_string_1(refcount=1) 
```
这个也表示PHP中的 __引用只可能有一层__ ，__不会出现一个引用指向另外一个引用的情况__ ，也就是没有C语言中`指针的指针`的概念。

### 2.1.3 内存管理
接下来分析下变量的分配、销毁。

在分析变量内存管理之前我们先自己想一下可能的实现方案，最简单的处理方式：定义变量时alloc一个zval及对应的value结构(ref/arr/str/res...)，赋值、函数传参时硬拷贝一个副本，这样各变量最终的值完全都是独立的，不会出现多个变量同时共用一个value的情况，在执行完以后直接将各变量及value结构free掉。

这种方式是可行的，而且内存管理也很简单，但是，硬拷贝带来的一个问题是效率低，比如我们定义了一个变量然后赋值给另外一个变量，可能后面都只是只读操作，假如硬拷贝的话就会有多余的一份数据，这个问题的解决方案是： __引用计数+写时复制__ 。PHP变量的管理正是基于这两点实现的。

#### 2.1.3.1 引用计数
引用计数是指在value中增加一个字段`refcount`记录指向当前value的数量，变量复制、函数传参时并不直接硬拷贝一份value数据，而是将`refcount++`，变量销毁时将`refcount--`，等到`refcount`减为0时表示已经没有变量引用这个value，将它销毁即可。
```php
$a = "time:" . time();   //$a       ->  zend_string_1(refcount=1)
$b = $a;                 //$a,$b    ->  zend_string_1(refcount=2)
$c = $b;                 //$a,$b,$c ->  zend_string_1(refcount=3)

unset($b);               //$b = IS_UNDEF  $a,$c ->  zend_string_1(refcount=2)
```
引用计数的信息位于给具体value结构的gc中：
```c
typedef struct _zend_refcounted_h {
    uint32_t         refcount;          /* reference counter 32-bit */
    union {
        struct {
            ZEND_ENDIAN_LOHI_3(
                zend_uchar    type,
                zend_uchar    flags,    /* used for strings & objects */
                uint16_t      gc_info)  /* keeps GC root number (or 0) and color */
        } v;
        uint32_t type_info;
    } u;
} zend_refcounted_h;
```
从上面的zend_value结构可以看出并不是所有的数据类型都会用到引用计数，`long`、`double`直接都是硬拷贝，只有value是指针的那几种类型才__可能__会用到引用计数。

下面再看一个例子：
```php
$a = "hi~";
$b = $a;
```
猜测一下变量`$a/$b`的引用情况。

这个不跟上面的例子一样吗？字符串`"hi~"`有`$a/$b`两个引用，所以`zend_string1(refcount=2)`。但是这是错的，gdb调试发现上面例子zend_string的引用计数为0。这是为什么呢？
```c
$a,$b -> zend_string_1(refcount=0,val="hi~")
```

事实上并不是所有的PHP变量都会用到引用计数，标量：true/false/double/long/null是硬拷贝自然不需要这种机制，但是除了这几个还有两个特殊的类型也不会用到：interned string(内部字符串，就是上面提到的字符串flag：IS_STR_INTERNED)、immutable array，它们的type是`IS_STRING`、`IS_ARRAY`，与普通string、array类型相同，那怎么区分一个value是否支持引用计数呢？还记得`zval.u1`中那个类型掩码`type_flag`吗？正是通过这个字段标识的，这个字段除了标识value是否支持引用计数外还有其它几个标识位，按位分割，注意：`type_flag`与`zval.value->gc.u.flag`不是一个值。

支持引用计数的value类型其`zval.u1.type_flag` __包含__ (注意是&，不是等于)`IS_TYPE_REFCOUNTED`：
```c
#define IS_TYPE_REFCOUNTED          (1<<2)
```
下面具体列下哪些类型会有这个标识：
```c
|     type       | refcounted |
+----------------+------------+
|simple types    |            |
|string          |      Y     |
|interned string |            |
|array           |      Y     |
|immutable array |            |
|object          |      Y     |
|resource        |      Y     |
|reference       |      Y     |
```
simple types很显然用不到，不再解释，string、array、object、resource、reference有引用计数机制也很容易理解，下面具体解释下另外两个特殊的类型：
* __interned string：__ 内部字符串，这是种什么类型？我们在PHP中写的所有字符都可以认为是这种类型，比如function name、class name、variable name、静态字符串等等，我们这样定义:`$a = "hi~";`后面的字符串内容是唯一不变的，这些字符串等同于C语言中定义在静态变量区的字符串：`char *a = "hi~";`，这些字符串的生命周期为request期间，request完成后会统一销毁释放，自然也就无需在运行期间通过引用计数管理内存。

* __immutable array：__ 只有在用opcache的时候才会用到这种类型，不清楚具体实现，暂时忽略。

#### 2.1.3.2 写时复制
上一小节介绍了引用计数，多个变量可能指向同一个value，然后通过refcount统计引用数，这时候如果其中一个变量试图更改value的内容则会重新拷贝一份value修改，同时断开旧的指向，写时复制的机制在计算机系统中有非常广的应用，它只有在必要的时候(写)才会发生硬拷贝，可以很好的提高效率，下面从示例看下：

```php
$a = array(1,2);
$b = &$a;
$c = $a;

//发生分离
$b[] = 3;
```
最终的结果：

![zval_sep](../img/zval_sep.png)

不是所有类型都可以copy的，比如对象、资源，事实上只有string、array两种支持，与引用计数相同，也是通过`zval.u1.type_flag`标识value是否可复制的：
```c
#define IS_TYPE_COPYABLE         (1<<4)
```
```c
|     type       |  copyable  |
+----------------+------------+
|simple types    |            |
|string          |      Y     |
|interned string |            |
|array           |      Y     |
|immutable array |            |
|object          |            |
|resource        |            |
|reference       |            |
```
__copyable__ 的意思是当value发生duplication时是否需要或者能够copy，这个具体有两种情形下会发生：
* a.从 __literal变量区__ 复制到 __局部变量区__ ，比如：`$a = [];`实际会有两个数组，而`$a = "hi~";//interned string`则只有一个string
* b.局部变量区分离时(写时复制)：如改变变量内容时引用计数大于1则需要分离，`$a = [];$b = $a; $b[] = 1;`这里会分离，类型是array所以可以复制，如果是对象：`$a = new user;$b = $a;$a->name = "dd";`这种情况是不会复制object的，$a、$b指向的对象还是同一个

具体literal、局部变量区变量的初始化、赋值后面编译、执行两篇文章会具体分析，这里知道变量有个`copyable`的属性就行了。

#### 2.1.3.3 变量回收
PHP变量的回收主要有两种：主动销毁、自动销毁。主动销毁指的就是 __unset__ ，而自动销毁就是PHP的自动管理机制，在return时减掉局部变量的refcount，即使没有显式的return，PHP也会自动给加上这个操作，另外一个就是写时复制时会断开原来value的指向，这时候也会检查断开后旧value的refcount。

#### 2.1.3.4 垃圾回收
PHP变量的回收是根据refcount实现的，当unset、return时会将变量的引用计数减掉，如果refcount减到0则直接释放value，这是变量的简单gc过程，但是实际过程中出现gc无法回收导致内存泄漏的bug，先看下一个例子：

```php
$a = [1];
$a[] = &$a;

unset($a);
```
`unset($a)`之前引用关系：

![gc_1](../img/gc_1.png)

`unset($a)`之后：

![gc_2](../img/gc_2.png)

可以看到，`unset($a)`之后由于数组中有子元素指向`$a`，所以`refcount > 0`，无法通过简单的gc机制回收，这种变量就是垃圾，垃圾回收器要处理的就是这种情况，目前垃圾只会出现在array、object两种类型中，所以只会针对这两种情况作特殊处理：当销毁一个变量时，如果发现减掉refcount后仍然大于0，且类型是IS_ARRAY、IS_OBJECT则将此value放入gc可能垃圾双向链表中，等这个链表达到一定数量后启动检查程序将所有变量检查一遍，如果确定是垃圾则销毁释放。

标识变量是否需要回收也是通过`u1.type_flag`区分的：
```c
#define IS_TYPE_COLLECTABLE
```
```c
|     type       | collectable |
+----------------+-------------+
|simple types    |             |
|string          |             |
|interned string |             |
|array           |      Y      |
|immutable array |             |
|object          |      Y      |
|resource        |             |
|reference       |             |
```
具体的垃圾回收过程这里不再介绍，后面会单独分析。

