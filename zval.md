# 变量的内部实现

PHP变量实现的基础结构是`zval`，各种类型的实现均基于此结构实现，是PHP中最基础的一个结构，每个PHP变量都对应一个`zval`，下面就看下这个结构以及PHP变量的内存管理机制。

## 1.zval结构
```c
//zend_type.h
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
            ZEND_ENDIAN_LOHI_4( //忽略这个宏，直接分析下面的结构
                zend_uchar    type,         //变量类型
                zend_uchar    type_flags,  //类型掩码，不同的类型会有不同的几种属性，内存管理会用到
                zend_uchar    const_flags,
                zend_uchar    reserved)     /* call info for EX(This) */
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
* __u1：__它的意义比较直观，变量的类型就通过`u1.type`区分，另外一个值`type_flags`为类型掩码，在变量的内存管理、gc机制中会用到，第三部分会详细分析，至于后面两个`const_flags`、`reserved`暂且不管
* __u2：__这个值纯粹是个辅助值，假如`zval`只有:`value`、`u1`两个值，整个zval的大小也会对齐到16byte，既然不管有没有u2大小都是16byte，把多余的4byte拿出来用于一些特殊用途还是很划算的，比如next在哈希表解决哈希冲突时会用到，还有fe_pos在foreach会用到......

从`zend_value`可以看出，除`long`、`double`类型直接存储值外，其它类型都为指针，指向各自的结构。

## 2.类型
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

### 2.1 基本类型
最简单的类型是true、false、long、double、null，其中true、false、null没有value，直接根据type区分，而long、double的值则直接存在value中：zend_long、double。

### 2.2 字符串
PHP中字符串通过`zend_string`表示:
```c
struct _zend_string {
    zend_refcounted_h gc;
    zend_ulong        h;                /* hash value */
    size_t            len;
    char              val[1];
};
```
* __gc：__变量引用信息，比如当前value的引用数，所有用到引用计数的变量类型都会有这个结构，3.1节会详细分析
* __h：__哈希值，数组中计算索引时会用到
* __len：__字符串长度，通过这个值保证二进制安全
* __val：__字符串内容，变长struct，分配时按len长度申请内存

### 2.3 数组
array是PHP中非常强大的一个数据结构，它的底层实现就是普通的有序HashTable，这里简单看下它的结构，下一节会单独分析数组的实现。

```c
typedef struct _zend_array HashTable;

struct _zend_array {
    zend_refcounted_h gc;
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
### 2.4 对象/资源

### 2.5 引用

## 3.内存管理

### 3.1 引用计数

### 3.2 写时复制

### 3.3 垃圾回收


[参考：https://nikic.github.io/2015/05/05/Internal-value-representation-in-PHP-7-part-1.html]

