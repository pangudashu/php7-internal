## 4.2 选择结构
程序并不都是顺序执行的，选择结构用于判断给定的条件，根据判断的结果来控制程序的流程。PHP中通过if、elseif、else和switch语句实现条件控制。这一节我们具体分析下这几条语句的执行过程。

### 4.2.1 if语句
If语句用法：
```php
if(Condition1){
    Statement1;
}elseif(Condition2){
    Statement2;
}else{
    Statement3;
}
```
各语句的条件可以有多个。

IF的语句有两部分组成：condition(条件)、statement(声明)，多个else就会对应多组这样的组合，所以整体来看if语句编译后是这样的结构：

![](../img/if.png)


### 4.2.2 switch语句
