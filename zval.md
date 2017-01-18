# 变量的内部实现

PHP变量实现的核心结构是`zval`，各种类型的实现均基于此结构实现，是PHP中最基础的一个结构，下面就看下这个结构以及PHP变量的内存管理机制。

## zval结构
```
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
* u1：它的意义比较直观，变量的类型就通过`u1.type`区分，另外一个值`type_flags`为类型掩码，在变量的内存管理、gc机制中会用到，第三部分会详细分析，至于后面两个`const_flags`、`reserved`暂且不管
* u2：这个值纯粹是个辅助值，假如`zval`只有:`value`、`u1`两个值，整个zval的大小也会对齐到16byte，既然不管有没有u2大小都是16byte，把多余的4byte拿出来用于一些特殊用途还是很划算的，比如next在哈希表解决哈希冲突时会用到，还有fe_pos在foreach会用到......

从`zend_value`可以看出，除`long`、`double`类型直接存储值外，其它类型都为指针，指向各自的结构。

## 类型

## 内存管理(引用计数)

## 引用类型


