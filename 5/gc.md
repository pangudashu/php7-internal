## 5.3 垃圾回收

### 5.3.1 介绍
前面已经介绍过PHP变量的内存管理，即引用计数机制，当变量赋值、传递时并不会直接硬拷贝，而是增加value的引用数，unset、return等释放变量时再减掉引用数，减掉后如果发现refcount变为0则直接释放value，这是变量的基本gc过程，PHP正是通过这个机制实现的自动垃圾回收，但是有一种情况是这个机制无法解决的，从而因变量无法回收导致内存始终得不到释放，这种情况就是循环引用，简单的描述就是变量的内部成员引用了变量自身，比如数组中的某个元素指向了数组，这样数组的引用计数中就有一个来自自身成员，试图释放数组时因为其refcount仍然大于0而得不到释放，而实际上已经没有任何外部引用了，这种变量不可能再被使用，所以PHP引入了另外一个机制用来处理变量循环引用的问题。

下面看一个数组循环引用的例子：
```php
$a = [1];
$a[] = &$a;

unset($a);
```
`unset($a)`之前引用关系：

![gc_1](../img/gc_1.png)

注意这里$a的类型在`&`操作后已经转为引用，`unset($a)`之后：

![gc_2](../img/gc_2.png)


可以看到，`unset($a)`之后由于数组中有子元素指向`$a`，所以`refcount = 1`，此时是无法通过正常的gc机制回收的，但是$a已经已经没有任何外部引用了，所以这种变量就是垃圾，垃圾回收器要处理的就是这种情况，这里明确两个准则：

>> 1) 如果一个变量value的refcount减少到0， 那么此value可以被释放掉，不属于垃圾

>> 2) 如果一个变量value的refcount减少之后大于0，那么此zval还不能被释放，此zval可能成为一个垃圾

针对第一个情况GC不会处理，只有第二种情况GC才会将变量收集起来。另外变量是否加入垃圾检查buffer并不是根据zval的类型判断的，而是与前面介绍的是否用到引用计数一样通过`zval.u1.type_flag`记录的，只有包含`IS_TYPE_COLLECTABLE`的变量才会被GC收集。

目前垃圾只会出现在array、object两种类型中，数组的情况上面已经介绍了，object的情况则是成员属性引用对象本身导致的，其它类型不会出现这种变量中的成员引用变量自身的情况，所以垃圾回收只会处理这两种类型的变量。
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
### 5.3.2 回收过程
如果当变量的refcount减少后大于0，PHP并不会立即进行对这个变量进行垃圾鉴定，而是放入一个缓冲buffer中，等这个buffer满了以后(10000个值)再统一进行处理，加入buffer的是变量zend_value的`zend_refcounted_h`:
```c
typedef struct _zend_refcounted_h {
    uint32_t         refcount; //记录zend_value的引用数
    union {
        struct {
            zend_uchar    type,  //zend_value的类型,与zval.u1.type一致
            zend_uchar    flags, 
            uint16_t      gc_info //GC信息，垃圾回收的过程会用到
        } v;
        uint32_t type_info;
    } u;
} zend_refcounted_h;
```

一个变量只能加入一次buffer，为了防止重复加入，变量加入后会把`zend_refcounted_h.gc_info`置为`GC_PURPLE`，即标为紫色，下次refcount减少时如果发现已经加入过了则不再重复插入。垃圾缓存区是一个双向链表，等到缓存区满了以后则启动垃圾检查过程：遍历缓存区，再对当前变量的所有成员进行遍历，然后把成员的refcount减1(如果成员还包含子成员则也进行递归遍历，其实就是深度优先的遍历)，最后再检查当前变量的引用，如果减为了0则为垃圾。这个算法的原理很简单，垃圾是由于成员引用自身导致的，那么就对所有的成员减一遍引用，结果如果发现变量本身refcount变为了0则就表明其引用全部来自自身成员。具体的过程如下：

(1) 从buffer链表的root开始遍历，把当前value标为灰色(zend_refcounted_h.gc_info置为GC_GREY)，然后对当前value的成员进行深度优先遍历，把成员value的refcount减1，并且也标为灰色；

(2) 再次遍历buffer链表，检查当前value引用是否为0，为0则表示确实是垃圾，把它标为白色(GC_WHITE)，如果不为0则排除了引用全部来自自身成员的可能，表示还有外部的引用，并不是垃圾，同时因为步骤(1)对成员进行了refcount减1操作，则需要还原回去，再对所有成员进行深度遍历，把成员refcount加1，同时标为黑色；

(3) 

垃圾收集器的全局数据结构：
```c
typedef struct _zend_gc_globals {
    zend_bool         gc_enabled; //是否启用gc
    zend_bool         gc_active;  //是否在垃圾检查过程中
    zend_bool         gc_full;    //缓存区是否已满

    gc_root_buffer   *buf;   //启动时分配的用于保存可能垃圾的缓存区
    gc_root_buffer    roots; //指向buf中最新加入的一个可能垃圾
    gc_root_buffer   *unused;//指向buf中没有使用的buffer
    gc_root_buffer   *first_unused; //指向buf中第一个没有使用的buffer
    gc_root_buffer   *last_unused; //指向buf中最后一个可用buffer

    gc_root_buffer    to_free;  //待释放的垃圾
    gc_root_buffer   *next_to_free;

    uint32_t gc_runs;   //统计gc运行次数
    uint32_t collected; //统计已回收的垃圾数
} zend_gc_globals;
```

