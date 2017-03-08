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
写给那些不了解联合体概念的同学：一个联合体定义了多个不同类型的成员，但是在某一刻只能使用其中一个成员。比如：如果给`value.lval`成员赋值了，那么你就需要用`value.lval`的方式来访问这个值，而不能用其他成员（否则会侵犯“strict aliasing”规则，并导致未定义的行为 - undefined behaviour）。这是因为联合体将其所有成员存储再一个相同的内存地址中，并根据访问的是哪一个成员来访问值。联合体的大小由其中的最大的成员决定。

在使用zvals时，类型标签用来寻找正在使用的是联合体中的哪一个成员。在介绍一些相关的APIs之前，我们先来大致过一下PHP支持的不同类型及其存储方式。
最简单的类型是`IS_NULL`：事实上不需要存储任何值，因为它本身就保留一个`null`值。

为了存储数字，PHP提供了`IS_LONG`和`IS_DOUBLE`类型，它们分别用于表达`long lval`和`double dval`两种成员。前者用于存储整形，然而后者用于存储浮点型数字。

针对于`long`类型有一些应该注意的事情：首先，这是一个有符号的整形类型，比如：他可以存储正数和负数，但是通常来说不适用于做一些位操作。第二，`long`类型在不同的平台上有不同的大小：在32位系统上是32位/4个字节，但是在64位系统上，要么是4个字节要么是8个字节。其中在64位的Unix系统上时，是8个字节，然而在64位的Windows系统上时仅仅使用了4个字节。

所以，你不应该依赖于`long`类型的大小。可以用`LONG_MIN`和`LONG_MAX`宏来判断当前`long`类型可以存储的最小值和最大值，同时可以使用`SIZEOF_LONG`（和`sizeof(long)`不同的是，它可以用在`#if`指令中）来获取该类型当前的大小。由于只需要存储两种值，那么理论上应该使用一个占用空间更小的类型（像`zend_bool`这样的无符号字符型），但是由于`zvalue_value`联合体的大小取决于其成员中最大的一个，所以这么做并不会实际减少内容的使用。故此，复用了这个`lval`成员。

字符串（`IS_STRING`）存储在结构体中`struct { char * val; int len; } str`，也就是说由一个`char *`类型的字符串和`int`类型的长度值组成。为了在字符串中使用NUL字节（`'\0'`），PHP字符串需要存储一个显示表示长度的字段（保证二进制安全）。先不考虑这些，PHP的字符串还是要以NUL结尾的，这是为了让一些不接收长度参数并且期望以NUL结尾的字符串的库函数更易使用。当然，在这个案例中字符串就不再需要是二进制安全的了，且它会在第一个出现NUL字节的地方被截断。比如很多文件系统相关的函数和大多数libc库的字符串函数都是这样的。

字符串的长度是以字节为单位（并不是Unicode code points），同时不包含其中的结尾字符 - NUL：字符串`"foo"`长度是3，尽管他实际上需要4个字节的存储空间。如果你需要利用`sizeof`来获取一个常量字符串的长度，那么要保证减少一个字节：`strlen("foo") == sizeof("foo") - 1`。

除此之外，很关键的一点是字符串的长度信息是存储在`int`类型中，而不是`long`类型中，或者其他什么类型中。这是一个不行的历史遗留问题，因为这会导致字符串的最大长度为2147483647字节。如果在字符串变量中存储比这多的数据会导致内存溢出（这样会导致长度为负数）。

剩下的三种类型在这里仅作简单的介绍，会在相应的章节中进行详细的阐述：

数组使用的是`IS_ARRAY`标签，存储在`HashTable *ht`成员中。将会在<b>Hashtables</b>章节中讨论`Hashtable`结构体是如何工作的。

对象（`IS_OBJECT`）使用`zend_object_value obg`成员， 它是由一个“object handle”和一组“object handlers”构成，其中“object hanle”是一个整型ID，用于寻找当前对象实际的值；“object handlers”定义了对象如何运作。将会在<b>Classes and objects</b>章节中对PHP的类和对象系统进行阐述。

资源（`IS_RESOURCE`）和对象类似，因为他们同样都存储了一个唯一ID，这个ID都是用来去寻找真实的值。ID存储在`long lval`成员中。将会在资源章节中（目前还没有完成）进行阐述。

为了有个形象的概念，下表给出了所有可用类型的标签和相应的值的存储位置：

<table>
    <tr>
        <th>类型标签</th>
        <th>存储位置</th>
    </tr> 
    <tr>
        <td>IS_NULL</td>
        <td>none</td>
    </tr>
    <tr>
        <td>IS_BOOL</td>
        <td>long lval</td>
    </tr>
    <tr>
        <td>IS_LONG</td>
        <td>long lval</td>
    </tr>
    <tr>
        <td>IS_DOUBLE</td>
        <td>double dval</td>
    </tr>
    <tr>
        <td>IS_STRING</td>
        <td>struct { char *val; int len; } str</td>
    </tr>
    <tr>
        <td>IS_ARRAY</td>
        <td>Hashtable *ht</td>
    </tr>
    <tr>
        <td>IS_OBJECT</td>
        <td>zend_object_value obj</td>
    </tr>
    <tr>
        <td>IS_RESOURCE</td>
        <td>long lval</td>
    </tr>
</table>

