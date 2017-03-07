# 基础结构
 一个zval（“Zend value”的缩写）代表一个任意类型的PHP变量。所以，它很可能是PHP中最重要的数据结构，同时你将会频繁地使用它。本章节讲述zvals的基础概念及其使用方法。

## 类型和值
 每一个zval都会存储某个值和其对应的类型。这点非常重要，因为PHP是一门动态类型语言，所以变量的类型只有当运行时才会确定，并不是在编译时就能够确定。此外，zval的类型在其生命周期是可以改变的，所以如果这个zval在最初存储了一个整形，那么在之后的某个时间点他也可能会存储了一个字符串。
 类型是存储在一个整形的标签中（一个 unsigned char 类型的变量）。它有8中类型的值，分别对应着PHP中的8中变量类型。这些值可以用诸如IS_TYPE形式的常量来使用。比如：`IS_NULL`对应null类型，`IS_STRING`对应字符串类型。
 
 真实的值是存储在一个联合体中，如下所示：
 ```c
 typedef union _zvalue_value {
    long lval;
    double dval;
    struct {
        char *val;
        int len;
    } str;
    HashTable *ht;
    zend_object_value obj;
} zvalue_value;
```
