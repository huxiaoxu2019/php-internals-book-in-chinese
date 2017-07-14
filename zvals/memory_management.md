# 内存管理
`zval`结构体有两种作用：第一种，正如我们前文提到的那样，可以用来存储值及其类型。第二种，我们将会在本节讨论，它能够进行高效地管理变量的内存。

接下来我们来了解下引用计数（reference-counting）和复制写（copy-on-write）的概念，并弄清楚如何在扩展开发时使用它们。

## 值语义和引用语义
在PHP中，所有变量都是值语义的，除非显示地使用引用。也就是说，无论是将一个变量传给一个函数还是给另一个变量赋值，这总是通过创建一个独立的拷贝实现的。举个例子：

```php
<?php

$a = 1;
$b = $a;
$a++;

// Only $a was incremented, $b stays as is:
var_dump($a, $b); // int(2), int(1)

function inc($n) {
    $n++;
}

$c = 1;
inc($c);

// The $c value outside the function and the $n inside the function are distinct
var_dump($c); // int(1)
```

上面的例子显然说明了记住这一特性是非常关键的。这一特性同样适用于对象。

```php
<?php

$obj = (object) ['value' => 1];

function fnByVal($val) {
    $val = 100;
}

function fnByRef(&$ref) {
    $ref = 100;
}

// The by-value function does not modify $obj, the by-reference function does:

fnByVal($obj);
var_dump($obj); // stdClass(value => 1)
fnByRef($obj);
var_dump($obj); // int(100)
```

有人会说在PHP5以后，对象会自动地以引用的形式进行传递，但上面的例子并非如此：一个值传递（by-value）的函数是不可以直接修改参数它本身的值的，只有接收引用参数的函数可以。

当然，上面的对象却是和引用传递有类似之处：尽管你不可以给它赋值一个完全不同的值，但是仍然可以改变这个对象的一些属性。这是因为对象的值其实是一个ID，这个ID可以用来寻找对象真实存储的值。值引用只是不允许你改变这个ID成其它不同的对象的ID，或者改成其它完全不同的变量，但是并不是不可以通过这个ID来改变对象内部的真实的值。

资源类型也是类似，因为它们同样也是存储了一个用于寻找真实值的ID。所以，值语义同样是不允许改变这个ID值或者改成其它类型的值，但是可以允许改变资源的内容（比如向前移动指向文件的指针）。

## 引用计数和写时复制
如果你仔细想想上面的内容，你会得到一个结论，PHP必须做很多的复制操作。每一次给函数传递参数时，都需要复制。这也许对于整数或者双精度来说不是特别麻烦，但是假设是传递一个含有1千万个元素的数组给函数呢？复制1千万个元素效率会急剧下降。

为了避免上述问题，PHP采用了复制时写的机制：如果一个`zval`只被读取，不被修改的话，它可以同时被多个变量、函数等共用。如果其中的一个使用者想修改它，那么`zval`需要在改动前进行一次拷贝。

如果一个`zval`可以在多处使用，那么为了销毁（和释放）内存，PHP需要提供一些方式来判断`zval`何时不再被其它处引用。PHP通过跟踪`zval`引用频率简单地实现这一点。注意这里提到的“引用”和PHP中的引用（`&`）没有关系，它指的只是那些使用这个`zval`的一些变量或者函数等等。引用的次数叫做`refcount`，并存储在`zval`的`refcount__gc`成员中。

为了更好的理解，举个例子：

```php
<?php

$a = 1;    // $a =           zval_1(value=1, refcount=1)
$b = $a;   // $a = $b =      zval_1(value=1, refcount=2)
$c = $b;   // $a = $b = $c = zval_1(value=1, refcount=3)

$a++;      // $b = $c = zval_1(value=1, refcount=2)
           // $a =      zval_2(value=2, refcount=1)

unset($b); // $c = zval_1(value=1, refcount=1)
           // $a = zval_2(value=2, refcount=1)

unset($c); // zval_1 is destroyed, because refcount=0
           // $a = zval_2(value=2, refcount=1)
```

很容易看出：当引用增加一次，`refcount`就加1，当引用减少一次，`refcount`就减1.如果`refcount`减少到0，那么`zval`就销毁。

当循环引用时上述的方法将无法正常运作：

```php
<?php

$a = []; // $a = zval_1(value=[], refcount=1)
$b = []; // $b = zval_2(value=[], refcount=1)

$a[0] = $b; // $a = zval_1(value=[0 => zval_2], refcount=1)
            // $b = zval_2(value=[], refcount=2)
            // The refcount of zval_2 is incremented because it
            // is used in the array of zval_1

$b[0] = $a; // $a = zval_1(value=[0 => zval_2], refcount=2)
            // $b = zval_2(value=[0 => zval_1], refcount=2)
            // The refcount of zval_1 is incremented because it
            // is used in the array of zval_2

unset($a);  //      zval_1(value=[0 => zval_2], refcount=1)
            // $b = zval_2(value=[0 => zval_1], refcount=2)
            // The refcount of zval_1 is decremented, but the zval has
            // to stay alive because it's still referenced by zval_2

unset($b);  //      zval_1(value=[0 => zval_2], refcount=1)
            //      zval_2(value=[0 => zval_1], refcount=1)
            // The refcount of zval_2 is decremented, but the zval has
            // to stay alive because it's still referenced by zval_1
```

在上述代码执行后，我们就无法再拿到这两个`zval`变量了，但是它们依然还存在，因为他们互相引用着。这是一个引用计数无法正常工作的典型的例子。

为了解决这个问题，PHP提供了第二种垃圾回收机制：一个循环回收者。我们可以先忽略它，因为循环回收者（不像引用计数机制）是对于扩展开发者透明的。如果你希望深入地了解这个话题，你可以参阅PHP手册[description of the algorithm](http://php.net/manual/en/features.gc.collecting-cycles.php)。

另外一个需要考虑到的情况是“真实”的PHP引用（也就是`&`，而不是上面提到的仅仅是内部的相互“引用”）。为了表示这个状态`zval`使用了一个PHP引用布尔值`is_ref`标记，这个标记存储在`zval`的`is_ref__gc`成员变量中。

如果`zval`的`is_ref`值为1，那么就说明这个`zval`在修改前不应该进行拷贝。而是应该直接修改它的值：

```php
<?php

$a = 1;   // $a =      zval_1(value=1, refcount=1, is_ref=0)
$b =& $a; // $a = $b = zval_1(value=1, refcount=2, is_ref=1)

$b++;     // $a = $b = zval_1(value=2, refcount=2, is_ref=1)
          // Due to the is_ref=1 PHP directly changes the zval
          // rather than making a copy
```

在上面的例子中，在`zval`类型的变量`$a`被引用之前，它的refcount值为1。现在我们来看一个非常相似的例子，不同的是refcount的值大于1：

```php
<?php

$a = 1;   // $a =           zval_1(value=1, refcount=1, is_ref=0)
$b = $a;  // $a = $b =      zval_1(value=1, refcount=2, is_ref=0)
$c = $b   // $a = $b = $c = zval_1(value=1, refcount=3, is_ref=0)

$d =& $c; // $a = $b = zval_1(value=1, refcount=2, is_ref=0)
          // $c = $d = zval_2(value=1, refcount=2, is_ref=1)
          // $d is a reference of $c, but *not* of $a and $b, so
          // the zval needs to be copied here. Now we have the
          // same zval once with is_ref=0 and once with is_ref=1.

$d++;     // $a = $b = zval_1(value=1, refcount=2, is_ref=0)
          // $c = $d = zval_2(value=2, refcount=2, is_ref=1)
          // Because there are two separate zvals $d++ does
          // not modify $a and $b (as expected).
```

正如你看到的那样，在使用`&`操作符引用一个`is_ref=0`且`refcount>1`的`zval`类型的变量时，需要进行拷贝。同样地在传值的上下文中，试着去使用一个`is_ref=1`且`refcount>1`的变量时也需要进行拷贝。所以使用PHP的引用通常会降低代码的效率：几乎PHP中所有的函数都使用传值语义，所以当传递给函数一个`is_ref=1`的`zval`变量时将要触发拷贝。

## 分配并初始化zvals

目前为止你应该对`zval`的内存管理有所了解了，接下来我们来看看如何去实践。首先我们讨论下`zval`的分配：

```c
zval *zv_ptr;
ALLOC_ZVAL(zv_ptr);
```

这段代码分配了一个`zval`类型的变量，但是并没有对其成员进行初始化。这里有很多用来分配永驻的`zval`变量的宏，直到请求结束也不会被销毁。

```c
zval *zv_ptr;
ALLOC_PERMANENT_ZVAL(zv_ptr);
```

这两个宏之间的差别在于前者使用了`emalloc()`，后者使用了`malloc()`函数。还有关键的一点是直接尝试去分配`zval`变量并不起作用：

```c
/* This code is WRONG */
zval *zv_ptr = emalloc()sizeof(zval);
```

原因是垃圾回收器需要在`zval`变量中存储一些额外的信息，所以实际上需要被分配的结构体不是`zval`而是`zval_gc_info`：

```c
typedef struct _zval_gc_info {
    zval z;
    union {
        gc_root_buffer       *buffered;
        struct _zval_gc_info *next;
    } u;
} zval_gc_info;
```

这个`ALLOC_*`宏会分配一个`zval_gc_info`结构体，并初始化一些额外的成员，但是这个值可以直接透明地当成`zval`类型的变量使用(因为这个结构体的第一个成员就是`zval`类型的变量)。

在分配`zval`后，需要进行初始化。有两种方式：第一种就是使用`INIT_PZVAL`，它会对进行相应的赋值`refcount=1`，`is_ref=0`，但是不会对存储值的成员赋值：

```c
zval *zv_ptr;
ALLOC_ZVAL(zv_ptr);
INIT_PZVAL(zv_ptr);
/* zv_ptr has garbage type+value here */
```

第二种是`INIT_ZVAL`，它同样会进行赋值操作，`refcount=1`,`is_ref=0`，但是会将其类型设置为`IS_NULL`类型。

```c
zval *zv_ptr;
ALLOC_ZVAL(zv_ptr);
INIT_ZVAL(*zv_ptr);
/* zv_ptr has type=IS_NULL here */
```

`INIT_PZVAL`接受一个`zval*`类型的值（所以会有一个`P`），，然而`INIT_ZVAL`接受一个`zval`类型的。当将`zval*`类型的值传给后面的宏时，首先要解除引用。

因为一步进行`zval`的分配和初始化是非常频繁的，所以有两个宏来一步完成这些操作：

```c
zval *zv_ptr;
MAKE_STD_ZVAL(zv_ptr);
/* zv_ptr has garbage type+value here */

zval *zv_ptr;
ALLOC_INIT_ZVAL(zv_ptr);
/* zv_ptr has type=IS_NULL here */
```

`MAKE_STD_ZVAL()`宏使用了`INIT_PZVAL()`宏进行初始化和分配，然而`ALLOC_INIT_ZVAL()`宏使用了`INIT_ZVAL()`宏。

## 管理引用计数和zval销毁

在你分配并初始化`zval`后，就可以使用前文介绍的引用计数机制了。PHP提供了一些用于管理`refcount`的宏：

```c
Z_REFCOUNT_P(zv_ptr)      /* Get refcount */
Z_ADDREF_P(zv_ptr)        /* Increment refcount */
Z_DELREF_P(zv_ptr)        /* Decrement refcount */
Z_SET_REFCOUNT(zv_ptr, 1) /* Set refcount to some particular value (here 1) */
```

和`Z_`开头的宏类似，同样会有不同数量的`_P`后缀，没有，一个或者两个，分别用于处理`zval`，`zval*`或者`zval**`类型的变量。

`Z_ADDREF_P()`将是你使用频繁的宏。举个简单的例子：

```c
zval *zv_ptr;
MAKE_STD_ZVAL(zv_ptr);
ZVAL_LONG(zv_ptr, 42);

add_index_zval(some_array, 0, zv_ptr);
add_assoc_zval(some_array, "num", zv_ptr);
Z_ADDREF_P(zv_ptr);
```

上述代码将一个存有整型数字42的`zval`变量分别以索引`0`和`'num'`加入到一个数组内。所以这个`zval`变量被两处地方使用。在使用`MAKE_STD_ZVAL()`宏分配并初始化后，`zval`变量的`refcount`初始值为1。为了在两处使用这个`zval`变量需要`refcount`的值为2，所以需要使用`Z_ADDREF_P()`宏来操作。

另外一个与之对应的宏`Z_DELPREF_P()`很少用得到：通常来说仅仅减少`refcount`的值是不够的，因为你要去判断`refcount`是否等于0的情况。因为当`refcount`等于0的情况下，需要销毁并释放`zval`变量。

```c
Z_DELREF_P(zv_ptr);
if (Z_REFCOUNT_P(zv_ptr) == 0) {
    zval_dtor(zv_ptr);
    efree(zv_ptr);
}
```
`zval_dtor()`宏接收一个`zval*`类型的参数并销毁它的值：如果是字符串，那么就销毁这个字符串，如果是数组，那么就销毁并释放HashTable，如果是对象或是资源，那么就将其refcount值递减（这样会导致销毁和释放）。

通常来说不会使用上述代码来检查refcount，可以使用这个`zval_ptr_dtor()`宏：

```c
zval_ptr_dtor(&zv_ptr);
```

这个宏接收一个`zval**`类型的参数（历史原因，他也接收`zval*`类型的参数），递减refcount同时检查它是否需要销毁和释放。但并不想我们上面手动写的代码那样，它也包含了对循环垃圾回收的支持。下面是它相应的一部分实现代码：

```c
static zend_always_inline void i_zval_ptr_dtor(zval *zval_ptr ZEND_FILE_LINE_DC TSRMLS_DC)
{
    if (!Z_DELREF_P(zval_ptr)) {
        ZEND_ASSERT(zval_ptr != &EG(uninitialized_zval));
        GC_REMOVE_ZVAL_FROM_BUFFER(zval_ptr);
        zval_dtor(zval_ptr);
        efree_rel(zval_ptr);
    } else {
        if (Z_REFCOUNT_P(zval_ptr) == 1) {
            Z_UNSET_ISREF_P(zval_ptr);
        }

        GC_ZVAL_CHECK_POSSIBLE_ROOT(zval_ptr);
    }
}
```

`Z_DELREF_P()`宏返回了它递减后的refcount值，所以`!Z_DELREF_P(zval_ptr)`和`Z_DELPREF_P(zval_ptr); Z_REFCOUNT_P(zval_ptr) == 0;`写法等同。

除了执行了预期的`zval_dtor()`和`efree()`操作外，同时也调用了两个形如`GC_*`的宏处理循环垃圾回收，并断言`&EG(uninitialized_zval))`永远不被释放（这个是引擎使用的魔术zval）。

此外，如果这个`zval`是有一处引用，那么会进行`is_ref=0`的设置。在这种情况下，如果`is_ref=1`是没有意义的，因为PHP的引用只有在zval变量被两个或者更多的持有者共享时才有意义。

对于这些宏使用的一些提示：你不应该使用`Z_DELREF_P()`宏(它仅适用于在你确定`zval`既不会被释放，也不会是一个可能的根圆的情况)。取而代之的是你应该使用`zval_ptr_dtor()`宏来递减`refcount`。`zval_dtor()`宏是特别为临时的，栈存储的`zvals`设计的：

```c
zval zv;
INIT_ZVAL(zv);

/* Do something with zv here */

zval_dtor(&zv);
```

分配于栈上的临时`zval`不能被分享的原因在于它将会在其所在语句块结束时被释放掉，所以它不能使用引用计数，可以无区别地使用`zval_dtor()`宏来销毁。

## 复制zvals

虽然写时复制机制能够节省大量的zval副本，但是它们也需要在一定情况下触发，比如：当你想要改变`zval`的值或者把它传递给另一个存储位置时。

PHP提供了大量在不同使用场景下的复制宏，最简单的就是`ZVAL_COPY_VALUE()`，它仅仅拷贝`value`和`type`两个`zval`的成员：

```c
zval *zv_src;
MAKE_STD_ZVAL(zv_src);
ZVAL_STRING(zv_src, "test", 1);

zval *zv_dest;
ALLOC_ZVAL(zv_dest);
ZVAL_COPY_VALUE(zv_dest, zv_src);
```

此时`zv_dest`会拥有和`zv_src`一样的类型和值。注意这里的“相同的值”指的是它们持有了同一个字符串变量（`char *`），即如果`zv_src`变量销毁了，那么其持有的字符串也将会被释放，同时`zv_dest`将会留下一个指向被释放的字符串的悬空指针（dangling pointer）。为了避免这种情况，可以使用`zval_copy_ctor()`宏。

```c
zval *zv_dest;
ALLOC_ZVAL(zv_dest);
ZVAL_COPY_VALUE(zv_dest, zv_src);
zval_copy_ctor(zv_dest);
```

`zval_copy_ctor()`宏会对`zval`进行一个完全的拷贝，也就是说如果是一个字符串，那么`char*`将会被拷贝，如果是一个数组，那么`HashTable*`将会被拷贝，如果是一个对象或者资源，那么他们的内部引用计数将会自增。

目前为止唯一一件未提及的事情就是`refcount`和`is_ref`标签的初始化问题。这些事情可以使用`INIT_PZVAL()`宏来搞定，也可以使用`MAKE_STD_ZVAL()`宏。另外一种方式就是使用`INIT_PZVAL_COPY()`来代替`ZVAL_COPY_VALUE()`，`ZVAL_COPY_VALUE()`会在拷贝时对`refcount`和`is_ref`进行初始化。

由于`INIT_PZVAL_COPY()`和`zval_copy_ctor()`的组合使用非常频繁，因此两者已被融合在`MAKE_COPY_VALUE()`宏中：
```c
zval *zv_dest;
ALLOC_ZVAL(zv_dest);
MAKE_COPY_ZVAL(&zv_src, zv_dest);
```

这个宏有一些怪异（tricky signature），因为参数的位置进行了调换（目标地址是第二个参数，而不是第一个了），同时需要原始变量是一个`zval**`类型的。再次声明这是一个历史遗留问题，并没有任何技术上的意义。

除了这些基础的复制宏以外，还有一些更复杂的。其中最重要的就是`ZVAL_ZVAL`，在函数中返回`zval`的场景下使用频率非常高。如下：

```c
ZVAL_ZVAL(zv_dest, zv_src, copy, dtor)
```

其中`copy`参数指定是否需要调用`zval_copy_ctor()`宏来对目标`zval`进行操作，`dtor`参数指定是否需要调用`zval_ptr_dtor()`宏来对原始`zval`进行操作。我们来看看这所有的四种可能性组合，并分析下其相应的行为。最简单的情况就是同时将`copy`和`dtor`设置为0：

```c
ZVAL_ZVAL(zv_dest, zv_src, 0, 0);
/* equivalent to: */
ZVAL_COPY_VALUE(zv_dest, zv_src)
```

上述例子中`ZVAL_ZVAL()`可以看成一个简单的`ZVAL_COPY_VALUE()`调用。因为使用0，0作为参数并没有实际意义。一种更有用的变形：copy=1，dtor=0：

```c
ZVAL_ZVAL(zv_dest, zv_src, 1, 0);
/* equivalent to: */
ZVAL_COPY_VALUE(zv_dest, zv_src);
zval_copy_ctor(&zv_src);
```

这基本上是一种类似于`MAKE_COPY_ZVAL()`的正常zval赋值，只是没有`INIT_PZVAL()`这一步。这在将其拷贝到已经初始化的`zvals`时非常有用。此外，设置dtor=1仅增加`zval_ptr_dtor()`调用：

```c
ZVAL_ZVAL(zv_dest, zv_src, 1, 1);
/* equivalent to: */
ZVAL_COPY_VALUE(zv_dest, zv_src);
zval_copy_ctor(zv_dest);
zval_ptr_dtor(&zv_src);
```

最有趣的场景是copy=0，dtor=1的组合：

```c
ZVAL_ZVAL(zv_dest, zv_src, 0, 1);
/* equivalent to: */
ZVAL_COPY_VALUE(zv_dest, zv_src);
ZVAL_NULL(zv_src);
zval_ptr_dtor(&zv_src);
```

这构成了一种`zval`的“运动” - 在不调用复制构造器的情况下，将`zv_src`的值转移到`zv_dest`变量中。当`zv_src`的引用计数refcount=1时，这种`zval`的“运动”就会发生。如果`zv_src`的引用计数refcount=1，那么在调用`zval_ptr_dtor()`后`zv_src`会被销毁掉。如果`refcount`的值大于1，那么`zval`将会保留一个`NULL`值。

还有另外两种用来复制`zval`的宏：`COPY_PZVAL_TO_ZVAL()`和`REPLACE_ZVAL_VALUE()`。很少会用到，所以不在此讨论了。

## 分离zvals

上述提到的一些宏主要用在拷贝一个`zval`到另一个储存地址的场景。一个典型的例子就是拷贝一个值到`return_value` zval中。还有另一系列的宏“zval 分离”，主用在写时复制的上下文场景中。下面的例子利于理解其功能 ：

```c
#define SEPARATE_ZVAL(ppzv)                     \
    do {                                        \
        if (Z_REFCOUNT_PP((ppzv)) > 1) {        \
            zval *new_zv;                       \
            Z_DELREF_PP(ppzv);                  \
            ALLOC_ZVAL(new_zv);                 \
            INIT_PZVAL_COPY(new_zv, *(ppzv));   \
            *(ppzv) = new_zv;                   \
            zval_copy_ctor(new_zv);             \
        }                                       \
    } while (0)
```

如果引用计数（`refcount`）为1，那么`SEPARATE_ZVAL()`将不会有任何操作。如果引用计数大于1，那么它会从老的`zval`中移除一个引用计数，将它拷贝到一个新的`zval`中，并将新的`zval`赋值给`*ppzv`。注意改宏接受一个`zval**`类型的值，并会将`zval*`所指向的内容更改到一个新的内容。

实际操作是怎么样的呢？假设，你想修改一个数组下标所对应的值，比如`$array[42]`。为了进行修改操作，你首先要拿到指向存储`zval*`值的指针`zval**`（译者注：这里面`zval*`指的是前面提到的`$array`变量，即代码中的ppzv）。由于引用计数的关系，你不能直接修改它（因为它可能在被其他地方所共享着），而是需要先分离。分离操作中，如果引用计数是1的话，那么就会使用当前“老的`zval`"，否则会进行赋值操作。在后者的情况中（引用计数大于1的场景），新的`zval`会被赋值给`*ppzv`，在这种场景下，`*ppzv`就是数组`$array`的存储地址。



