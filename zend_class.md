### 3.4.1 类
类是现实世界或思维世界中的实体在计算机中的反映，它将某些具有关联关系的数据以及这些数据上的操作封装在一起。在面向对象中类是对象的抽象，对象是类的具体实例。

在PHP中类编译阶段的产物，而对象是运行时产生的，它们归属于不同阶段。

PHP中我们这样定义一个类：
```php
class 类名 {
    常量;
    成员属性;
    成员方法;
}
```

一个类可以包含有属于自己的常量、变量（称为“属性”）以及函数（称为“方法”），本节将围绕这三部分具体弄清楚以下几个问题：

* a.类的存储及索引
* b.成员属性的存储结构
* c.成员方法的存储结构
* d.成员方法的调用过程及与普通function调用的差别

#### 3.4.1.1 类的结构及存储
首先我们看下类的数据结构：
```c
struct _zend_class_entry {
    char type;          //类的类型：内部类ZEND_INTERNAL_CLASS(1)、用户自定义类ZEND_USER_CLASS(2)
    zend_string *name;  //类名，PHP类不区分大小写，统一为小写
    struct _zend_class_entry *parent; //父类
    int refcount;
    uint32_t ce_flags;  //类掩码，如普通类、抽象类、接口，除了这还有别的含义，暂未弄清

    int default_properties_count;        //普通属性数，包括public、private
    int default_static_members_count;    //静态属性数，static
    zval *default_properties_table;      //普通属性值数组
    zval *default_static_members_table;  //静态属性值数组
    zval *static_members_table;
    HashTable function_table;  //成员方法哈希表
    HashTable properties_info; //成员属性基本信息哈希表，key为成员名，value为zend_property_info
    HashTable constants_table; //常量哈希表，通过constant定义的

    //以下是构造函授、析构函数、魔法函数的指针
    union _zend_function *constructor;
    union _zend_function *destructor;
    union _zend_function *clone;
    union _zend_function *__get;
    union _zend_function *__set;
    union _zend_function *__unset;
    union _zend_function *__isset;
    union _zend_function *__call;
    union _zend_function *__callstatic;
    union _zend_function *__tostring;
    union _zend_function *__debugInfo;
    union _zend_function *serialize_func;
    union _zend_function *unserialize_func;

    zend_class_iterator_funcs iterator_funcs;

    //下面这几个暂时忽略，后面碰到的时候再分析其作用
    /* handlers */
    zend_object* (*create_object)(zend_class_entry *class_type);
    zend_object_iterator *(*get_iterator)(zend_class_entry *ce, zval *object, int by_ref);
    int (*interface_gets_implemented)(zend_class_entry *iface, zend_class_entry *class_type); /* a class implements this interface */
    union _zend_function *(*get_static_method)(zend_class_entry *ce, zend_string* method);

    /* serializer callbacks */
    int (*serialize)(zval *object, unsigned char **buffer, size_t *buf_len, zend_serialize_data *data);
    int (*unserialize)(zval *object, zend_class_entry *ce, const unsigned char *buf, size_t buf_len, zend_unserialize_data *data);

    uint32_t num_interfaces; //实现的接口数
    uint32_t num_traits;
    zend_class_entry **interfaces; //实现的接口

    zend_class_entry **traits;
    zend_trait_alias **trait_aliases;
    zend_trait_precedence **trait_precedences;

    union {
        struct {
            zend_string *filename;
            uint32_t line_start;
            uint32_t line_end;
            zend_string *doc_comment;
        } user;
        struct {
            const struct _zend_function_entry *builtin_functions;
            struct _zend_module_entry *module; //所属扩展
        } internal;
    } info;
}
```
举个例子具体看下，定义一个User类，它继承了Human类，User类中有一个常量、一个静态属性、两个普通属性：
```php
//父类
class Human {}

class User extends Human
{
    const type = 110;

    static $name = "uuu";
    public $uid = 900;
    public $sex = 'w';

    public function __construct(){
    }

    public function getName(){
        return $this->name;
    }
}
```
其对应的zend_class_entry存储结构如下图。

![zend_class](img/zend_class.png)

开始的时候已经提到，类是编译阶段的产物，编译完成后我们定义的每个类都会生成一个zend_class_entry，它保存着类的全部信息，在执行阶段所有类相关的操作都是用的这个结构。

所有PHP脚本中定义的类以及内核、扩展中定义的内部类通过一个以"类名"作为索引的哈希表存储，这个哈希表保存在Zend引擎global变量中：__zend_executor_globals.class_table__(即：__EG(class_table)__)，与function的存储相同，关于这个global变量前面[《3.3.1.3 zend_executor_globals》](zend_executor.md#3313-zend_executor_globals)已经讲过。

![zend_eg_class](img/zend_eg_class.png)

在接下来的小节中我们将对类的常量、成员属性、成员方法的实现具体分析。

#### 3.4.1.2 类常量
PHP中可以把在类中始终保持不变的值定义为常量，在定义和使用常量的时候不需要使用 $ 符号，常量的值必须是一个定值，不能是变量、数学运算的结果或函数调用，也就是说它是只读的，无法进行赋值。

常量通过__const__定义：
```php
class my_class {
    const 常量名 = 常量值;
}
```
常量通过__class_name::常量名__访问，或在class内部通过__self::常量名__访问。

常量是类维度的数据(而不是对象的)，它们通过`zend_class_entry.constants_table`进行存储，这是一个哈希结构，通过__常量名__索引，value就是具体定义的常量值。

#### 3.4.1.3 成员属性
类的变量成员叫做“属性”。属性声明是由关键字 __public__，__protected__ 或者 __private__ 开头，然后跟一个普通的变量声明来组成，关于这三个关键字这里不作讨论，后面分析可见性的章节再作说明。

属性中的变量可以初始化，但是初始化的值必须是常数，这里的常数是指 PHP 脚本在编译阶段时就可以得到其值，而不依赖于运行时的信息才能求值，比如`public $time = time();`这样定义一个属性就会触发语法错误。

成员属性又分为两类：__普通属性__、__静态属性__。静态属性通过__static__声明，通过__self::$property__或__类名::$property__访问；普通属性通过__$this->property__或__$object->property__访问。

```php
class my_class {
    //普通属性
    public $property = 初始化值;

    //静态属性
    public static $property_2 = 初始化值;
}
```
与常量的存储方式不同，成员属性的__初始化值__并不是__直接__用以"属性名"作为索引的哈希表存储的，而是通过数组保存的，普通属性、静态属性各有一个数组分别存储。

![zend_class_property](img/zend_class_property.png)

看到这里你可能有个疑问：使用时成员属性是如果找到的呢？

实际只是成员属性的__VALUE__通过数组存储的，访问时仍然是根据以"属性名"为索引的散列表查找具体VALUE的，这个散列表并没有按照普通属性、静态属性分为两个，而是只用了一个：__HashTable properties_info__。此哈希表存储元素的value类型为__zend_property_info__。

```c
typedef struct _zend_property_info {
    uint32_t offset; //普通成员变量的内存偏移值
                     //静态成员变量的数组索引
    uint32_t flags;  //属性掩码，如public、private、protected及是否为静态变量
    zend_string *name; //属性名
    zend_string *doc_comment;
    zend_class_entry *ce; //所属类
} zend_property_info;
```

#### 3.4.1.4 成员方法
