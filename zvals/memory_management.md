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

