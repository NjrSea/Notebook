# C语言中的##__VA_ARGS__宏

https://gcc.gnu.org/onlinedocs/cpp/Variadic-Macros.html

宏可以被声明成像函数一样接受多个参数的形式。如：

```
#define eprintf(...) fprintf(stderr, __VA_ARGS__)
```

这种宏被称为```variadic```。...中的内容被替换成```__VA_ARGS__```。因此我们有了这样的展开：

```
eprintf("%s:%d:", input_file, lineno)
    -> fprintf(stderr, "%s:%d:", input_file, lineno)
```

可变参数在它替换进宏前 就被完全```展开(macro-expanded)```了。你可以用'#'和'##'操作符来将可变参数变成字符串或者把它的开头(leading token)或者结尾(trailing token) 和其他的内容(token)结合。（注意：下面还有一个关于'##'的重要的例子）

如果你的宏很复杂，你需要一种比```__VA_ARGS__```更加具有描述性的可变参数。作为c++的扩展，你可以在一个参数名后加上`...`；上面的```eprintf```宏可以写成：

```
#define eprintf(arg...) fprintf(stderr, args)
```

```__VA_ARGS__```和这个扩展不能在同一个宏里使用。

当然，固定的参数和可变参数可以在同一个```variadic```宏里使用，如：

```
#define eprintf(format, ...) fprintf(stderr, format, __VA_ARGS__)
```

上面的表述方式看起来可描述性更前些，但是缺少灵活性：你必须立即提供至少一个参数。在标准C里，你不能省略逗号分隔的固定参数。此外，如果你不输入参数，则会报语法错误，因为format后边跟了一个逗号

```
eprintf("success!\n", )
    -> fprintf(stderr, "success!\n", );
```

GNU CPP 有一对扩展可以解决这个问题。首先，没有可变参数完全OK：

```
eprintf("success!\n")
    -> fprintf(stderr, "success!\n")
```

其次，当'##'符号被放在逗号和一个变量参数之间的时候，它就具有了特殊的意义。如果你写

```
#define eprintf(format, ...) fprintf (stderr, formt, ##__VA_ARGS__)
```

并且可变参数被忽略（left out）的时候，那么'##'前面的逗号会被删除。但是如果一个参数都没传，或者'##'前面不是逗号，这一规则是不奏效的。

```
eprintf("success!\n")
    -> fprintf(stderr, "success!\n");
```

上述并没有详细去说只有可变参数的情况，因为区分有没有参数、缺少参数等情况是没有意义的。CPP当符合特殊的C标准的时候保留逗号。否则的话，遵循标准扩展，逗号会被去掉。


标准C里```__VA_ARGS```只能存在可变宏```variadic macro```的后边作为可变参数列表的保留字。不要作为其他的用途。*

> We recommend you avoid using it except for its defined purpose.

```Variadic macros```从C99开始成为C语言的标准。