---
layout: post
cover: false
navigation: true
title: C指针与内存
date: 2020-09-28 07:50:25
tags: ['Language']
subclass: 'post tag-fiction'
author: smartkeyerror
categories: smartkeyerror
---

C 语言指针真正精髓的地方在于**指针可以进行加减法**，这一点极大的提升了程序的对指针使用的灵活性，同时也带来了不小的学习负担。正是因为 C 语言指针可运算，才奠定了如今 C 语言的地位。

<!---more--->


### 1. 指针

#### 1.1 指针的基本概念

对于内存，我们可以简单地认为它就是大小相同、连续排布的格子，每一个格子的大小为一字节。为了更方便地找到某一个格子，我们通过对内存进行编号，通过编号来找到某一个具体的内存格子。

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/Snorlax/pointer/simple-memory.png)

这样的编号通常称之为内存地址，如果程序想要获取某一块内存存放的数据，必须通过内存地址定位，再取出对应内存的数据。

一个指针变量存储着另一块内存的起始地址，相较于直接寻址的方式，如果想要通过一个指针获取指向的内存变量的话，首先需要获取到指针变量存储的内存地址，再通过这个地址来获取变量，所以这种方式又称为间接寻址。

在 C 函数实现中，所传入的参数均为原有变量的一个拷贝，在函数中对参数进行修改是无法影响到原有变量的值的，若需要对参数进行修改，可向函数传递该变量的内存地址。如此一来，即使对参数（地址）进行了拷贝，也可以通过地址对其进行修改，而变量的地址就保存在指针当中。

```cpp
void main() {
    int number;
    int *pointer;
}
```

如上述代码所示，语句 `int number` 声明了一个变量，且其类型为整型。语句 `int *pointer` 同样声明了一个变量，只不过 `*pointer` 的类型为整型而已。`*` 作为一元操作符时表示对间接寻址或者是间接引用，所以变量 `pointer` 是一个指针，其中保存了变量的地址。

实际上，`int *pointer` 也可以写作为 `int* pointer`，对于编译器而言，这两种方式最终的结果都是一样的: 声明一个指针变量。但是，`int*` 这一写法会让人产生歧义，认为 `pointer` 是一个 `int*` 类型，然而 C 语言中并无该类型。**应当将 `*pointer` 作为一个整体看待，其类型为 `int`**。

```cpp
/*
 * 所以，当我们将 *pointer 看作是一个整体之后，对于多变量声明就不会再产生疑惑 
 * *pointer 与 number 的类型均为 int
 */
void main() {
    int *pointer, number;
}
```

指针变量中既然存放的是内存单元的地址，那么其大小就与所指向的类型无关了。在 64 位系统指针变量占用 8 个字节，在 32 位系统下占用 4 个字节。


#### 1.2 指针的初始化

```cpp
void main() {
    int number = 100;       // ①
    int *pointer;           // ②

    *pointer = number;      // ③
}
```

上述代码在前两行中声明了两个变量: 整型变量 `number` 与指向整型的指针 `pointer`。在第三行中，将 `pointer` 指向的内存内容更新为 `number` 的值（100）。在运行时却抛出了 `Segmentation fault` 错误，这是为什么？

原因在于 `pointer` 中保存的值是不确定的，可能是上一个程序中的某一个变量值，也可能恰好是一个地址，我们无法得知变量 `pointer` 中的内容到底是什么。这种已经声明但为正确初始化的指针通常称之为“野指针”。

所谓“指针初始化”就是指使得当前指针变量中保存的地址是正确的、合法的，当前进程有权限访问的。因此，对上述代码进行稍加修改即可正确运行：

```cpp
void main() {
    int number = 100;
    int *pointer;

    pointer = &number;
}
```

操作符 `&` 表示取一个变量的地址，并将该地址赋给变量 `pointer`，由此一来指针中的地址就是正确且合法的。

#### 1.3 指针的运算

C 语言中的指针支持运算，不过仅支持加减法，并不支持乘除法。在前面我们已经提到过了，一元运算符 `*` 表示间接寻址，即取出指针的地址，根据该地址找到指针指向的变量。而对于指针的加减法而言，与指针指向的类型密切相关。

```cpp
#include <stdio.h>

void main() {
    int *pointer, number = 10;
    pointer = &number;

    printf("pointer's value is: %p \n", pointer);

    pointer = pointer + 1;
    printf("pointer's value is: %p \n", pointer);
}

// pointer's value is: 0x7ffd0df7f61c 
// pointer's value is: 0x7ffd0df7f620
```

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/Snorlax/pointer/pointer-increase.png)

可以看到，对一个指向 `int` 的指针加 1，其地址增加了 4，正好是一个整型的大小。其原因在于如果仅仅是增加 1 的话，那么指针将会指向该整型的第二个字节，那么此时指针所指向的“整型”值将会变得非常奇怪，因为最后一个字节是其它数据的。

所以，对指针进行自增时，指针将会指向原来指向元素的下一个元素，这就使得我们可以通过指针的加减法来访问数组中的元素了。

#### 1.4 指针与数组

我们经常听到数组名称其实就是一个指针，数组名表示数组的首地址。但是这并不正确，数组名称和指针绝不等价，我们可以认为数组名称具有指针的特性，但不能说数组和指针等价。一个最为典型的例子就是使用 `sizeof` 求数组的长度。

```cpp
#include <stdio.h>

void main() {
    int number[] = {1, 2, 3};
    int *pointer = number;

    printf("number's size: %ld \n", sizeof(number));
    printf("pointer's size: %ld \n", sizeof(pointer));
}

// number's size: 12 
// pointer's size: 8 
```

在上述代码中我们明确地表示了 `number` 是一个整型数组，表示整型的集合，所以 `sizeof` 的结果为数组占用内存的总大小。而 `pointer` 仅仅只是一个整型指针，编译器并不知道它到底是指向一个整型，还是一坨整型，那么 `sizeof` 的结果自然而然的是指针占用内存空间的大小。

所以，**数组名称所代表的含义要高于指针，但是这并不改变数组名是一个指针的事实**。数组在内存中连续分布，再结合前面所提到的指针的运算，使得我们可以通过指针的方式访问数组中的元素。

```cpp
#include <stdio.h>

void main() {
    int number[] = {1, 2, 3};
    int *pointer = number;

    for (int i = 0; i < sizeof(number) / sizeof(int); i++) {
        printf("%d ", *(pointer + i));
    }
}
```

实际上，`array[n]` 就相当于 `*(array + n)`，所以 `number[2]` 也可以用 `[2]number` 来表示，这不过后者除了让人感到迷惑以外，没有任何实际价值。

> `[2]number` 使用指针的方法展开结果为 `*(2 + number)`，仅调换了位置而已

另外一点需要注意的就是，当数组作为函数参数时，将会“退化”成指针，失去其原有的数组特性：即无法在函数内部获得数组大小。

```cpp
#include <stdio.h>

void bar(int number[]) {
    long int size = sizeof(number);
    printf("number size: %ld", size);
}

void main() {
    int number[] = {1, 2, 3};
    bar(number);
    
}
```

此时编译器将会给出警告：

```bash
warning: ‘sizeof’ on array function parameter ‘number’ will return size of ‘int *’
```

也就是说，尽管我们在参数中声明了 `number` 是一整型数组，在函数内部仍然将其看作为指针。因此，如果想要在函数中对参数中的数组进行遍历的话，必须传入数组的原有大小。

保存着指针的数组称之为指针数组，一个最常见的例子就是字符串数组。字符串在底层由数组实现，并在尾部添加 `\0` 表示结尾，而在前面也提到过，数组可以使用指针进行表示，只不过会有所“牺牲”。

```cpp
#include <stdio.h>
#include <string.h>

void main() {
    char *name[] = {"Mark Twain", "Victor Marie Hugo"};

    for (int i = 0; i < sizeof(name)/sizeof(char *); i++) {
        printf("name: %s, length: %ld \n", name[i], strlen(name[i]));
    }
}
```

当数组作为函数参数时将会“退化”为指针，那么 `char *name[]` 又该如何仅使用指针的形式表示呢？ `char **name`，也就是传说中的二级指针，理解起来稍微有一点点绕。由于 `char name[]` 在形式上与 `char *name` 等价，那么 `char *name[]` 就可以写成 `char *(*name)`，去掉括号就是 `char **name`，二级指针。

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/Snorlax/pointer/pointer-to-pointer.png)

```cpp
#include <stdio.h>

void main() {
    char a = 'A';
    char *a_pointer = &a;
    char **a_pointer_pointer = &a_pointer;
    
    printf("%c", **a_pointer_pointer);
}
```

把二级指针想象成套娃就行，至于二级指针指向的是一个指针还是一个数组，一级指针指向的是一个字符串还是一个字符，由运行时确定，不过在大多数的函数定义中，`char **` 表示一个字符串数组。

#### 1.5 函数指针

在函数式编程中我们经常需要将一个函数作为参数传递给另外一个函数，其中比较经典的例子线程的创建与执行。不同于 `fork` 或者 `vfork` 调用，`pthread_create` 调用以一个函数指针作为参数，创建的线程转而执行该函数。

```cpp
int filterArray(int *array, int size);
```

上述代码是一个简单的函数原型，接收一个整型数组和其长度，并返回一个整型。一个函数总是占用一段连续的内存区域，函数名称在某些情况下会被转换成该函数所在内存区域的首地址，这一点和数组非常的相似。也就是说，函数名称 `filterArray` 可以被替换成 `(*pointerName)`，这样一来就得到了一个指向函数入口的指针：

```cpp
int (*pointerName)(int *array, int size);
```

所以，函数指针其实就是对函数原型的稍加修改而已。

#### 1.6 对复杂指针的解释

有时候我们会看到诸如 `char *(*name[124])(int **p)` 这种看起来非常复杂的指针，对于此类复杂指针，我们只需要牢记两个关键点即可：
- 对于一个符号定义而言，找到其名称，然后再按照优先级顺序进行解析。
- 牢记 C 语言中操作符的优先级。

首先来看 C 语言中的操作符优先级，这个非常非常重要：

| 操作符 | 描述                       | 
| :----: | :------------------------- |
| ()     | 聚组，例如 `5 / (2 + 3)`，和数学中的`()`语义相同   |
| ()     | 函数调用，例如 `3 + foo()`，首先调用函数，而后执行加法操作 |
| []     | 下标引用，例如 `foo()[2]`，首先调用函数，根据函数返回的结果进行下标引用  |
| .      | 访问结构体成员，如 `student[1].name` 首先获取数组索引为1的元素，而后访问`name`成员 |
| ->     | 访问结构指针成员，和 `.` 作用相同 |
| ++     | **后缀**自增，如 `struct.name++`，首先获取结构体成员再对其进行自增 |
| --     | **后缀**自减，同后缀自增              |
| !      | 逻辑反                                |
| ~      | 按位取反                              |
| ++     | **前缀**自增                          |
| --     | **前缀**自减                          |
| *      | 间接访问，如 `*p++` 首先对指针 `p` 进行自增，然后对自增后的指针进行间接访问         |
| &      | 取变量地址                            |


可以看到，`*` 间接访问操作的优先级要远远低于 `()`、`[]`、`++` 以及 `.` 等常用操作符，也就是说，`*` 在复杂指针中就是个弟弟。

接下来就对 ``char *(*name[124])(int **p)`` 这个指针做一个具体的分析。首先得找到变量名称，上面有两个： `name` 以及 `p`，结合外面两对括号可知这是个函数指针，`p` 为函数参数。再来看 `*name[124]`，`[]` 的优先级高于 `*`，所以 `name` 是一个数组，数组中保存了某种类型的指针。将所有的信息整合起来，就可以得到：`name` 是一个指针数组，保存了原型为 `char *funcName(int **p)` 的函数指针。

复杂指针的解析其实就是小学时代做的 `()` 类计算题，牢记运算符优先级，再仔细的一层一层解析。

#### 1.7 指针与字符串

C 语言中字符串并非像其它语言一样，将其设置为基本数据类型，而是构建于数组之上，并在数组末尾添加 `\0` 表示结尾，这也是为什么数组和字符串经常成对出现的原因。

```cpp
int main() {
    char *hello = {"Hello"};
    char world[] = {"World"};

    printf("hello: %s \n", hello);
    printf("world: %s \n", world);
    
}
```

如上述代码所示，我们既可以使用数组初始化一个字符串，也可以使用字符指针来初始化一个字符串，只不过**使用字符指针初始化的字符串具有只读特性，使用数组初始化的字符串具有读写特性**。

```cpp
int main() {
    char *hello = {"Hello"};
    char world[] = {"World"};

    printf("hello: %s \n", hello);
    printf("world: %s \n", world);
    
    world[0] = 'w';     // 正确执行
    hello[0] = 'h';     // Segmentation fault
}
```

### 2. 内存

本节并不会介绍虚拟内存、MMU 以及 TLB 等内容，因为它们更贴近于操作系统。对于 C 程序语言而言，并不需要知道物理内存是个什么样子。

#### 2.1 内存对齐

内存对齐是指一个数据类型在内存中存放时，对其地址的要求。简单来说内存对齐就是使得其内存地址是该类型大小的整数倍，例如 `double` 类型的变量，其内存地址需是 8 的倍数（`double` 大小为 8 字节）。

内存对齐的主要目的就是满足部分 CPU 对内存读写的要求以及优化 CPU 读取内存的效率。在 ARM、Alpha 等架构下，如果读取的内存是非对齐的（例如一个 4 字节的 `int` 落在一个奇数的内存地址上）则会直接抛出异常，当然，Intel 没有这样的限制。

那么为什么说 Intel 等不对内存对齐有要求的 CPU 来说使用内存对齐能优化读取效率呢？ 这是因为 CPU 在进行**偶数**地址读取时将更有效率，其中涉及到 RAM 的电路设计故不再深入展开，只需要知道**奇数的物理内存地址不大可能出现在 CPU 中**

#### 2.2 结构体与内存对齐

对于零散定义的变量，在编程时完全可以忽略内存对齐的影响，因为我们关注的是变量本身，而不是地址。但是，结构体中变量由于其紧凑的内存分布，开发人员不得不将内存对齐的因素考虑其中。

```cpp
#include <stdio.h>

struct Node {
    char mark;
    int size;
    char flag;
};


int main() {
    struct Node node;

    printf("node size: %ld \n", sizeof(node));
    
    printf("mark address: %p \n", &node.mark);
    printf("size address: %p \n", &node.size);
    printf("flag address: %p \n", &node.flag);
}

// node size: 12 
// mark address: 0x7ffeef36125c 
// size address: 0x7ffeef361260 
// flag address: 0x7ffeef361264 
```

上述代码的运行结果 `Node` 大小并非预料的 6，而是 12，因为内存对齐的原因，`Node` 结构体额外花费了一倍的内存空间。再来看结构体变量的地址分配：

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/Snorlax/pointer/struct.png)

由于 `int` 在本机器上占用 4 字节，所以其起始地址必须是 4 的倍数，也就是 `0x7ffeef361260`。所以第一个 `char` 类型后面会有 3 字节的内存空隙。而最后一个 `char` 类型后会有 3 字节空隙的原因则是遵循另一个原则：结构体的大小为对齐系数的整数倍。在 Linux 平台下，该值通常为 4，所以 `Node` 的大小必须为 4 的整数倍，故填充了 3 字节的空隙。

不过，我们可以通过调整结构体成员变量的定义顺序，来减少内存空隙，以节省内存：

```cpp
struct Node {
    char mark;
    char flag;
    int size;
};

struct Node {
    int size;
    char mark;
    char flag;
};
```

上述两种方式得到的 `Node` 大小均为 8，相较于最初版本节省了 4 字节的内存。**所以，结构体中成员变量的顺序将会影响程序使用内存的大小。**

#### 2.3 程序运行时的进程地址空间

在未出现“分页虚拟内存”管理机制之前，操作系统对内存空间采用“分段”的方式进行管理：即将相似的数据放在一起，例如文本段、初始化数据段、未初始化数据段。时至今日，此类内存布局仍在使用，并结合分页虚拟内存共同实现对物理内存的管理。

- 文本段：又称为代码段，文本段中保存了程序的机器语言指令，所以这部分内容是只读的，并且可能会被多个进程所共享。
- 初始化数据段：包含了显示初始化的全局变量和静态变量。
- 未初始化数据段：包含了未显示初始化的全局变量和静态变量。
- 堆栈：实现函数调用的重要内存区域，由栈帧组成，栈帧中保存了函数的参数、局部变量、调用方的栈帧地址以及返回值等。当一个函数调用返回时该函数的栈帧可能会被其它函数所占用，所以栈帧中的变量是易失的。
- 堆：动态内存分配区，`malloc` 函数族所申请的内存就源于堆中，堆内存中的变量不易失，其生命周期可由程序控制。

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/Snorlax/pointer/memory.png)

```cpp
#include <stdlib.h>

int number = 1024;                  /* 全局已初始化数据区 */
int *pointer;                       /* 全局未初始化数据区 */

static int size = 4096;             /* 全局静态已初始化数据区 */

int func(int number, int size) {
    
    int result;                     /* 栈帧存储区 */

    result = number * size;

    return result;
}

int main() {
    /* 堆 */
    char *student = (char *)malloc(sizeof(char) * number);

    /* 函数内部使用 static 修饰的变量将会作为静态变量 */
    static int total_socre;
    
    free(student);
    
    return 0;
}
```

需要说明一点的是，当函数调用返回时，并不会清理该函数所使用的栈帧，当下一个函数调用时直接进行覆盖。同时因为栈帧是易失的以及语言特性，使得 C 语言中的变量无法从栈中逃逸。

#### 2.4 柔性数组

在结构体中声明一个数组成员变量时，可能会直接定义一个指针，动态申请内存时，依次申请结构体和结构体中数组的内存：

```cpp
#include <stdlib.h>

typedef struct heap {
    long size;
    int *elements;
} Heap;

int main() {
    long size = 1024;

    Heap *heap = (Heap *)malloc(sizeof(Heap));
    int *elements = (int *)malloc(sizeof(int) * size);

    heap->size = size;
    heap->elements = elements;

    free(heap);
    free(elements);
}
```

内存的申请和释放均需要两步操作，固然我们可以封装关于 `Heap` 结构体内存申请和释放的相关方法，但是还是会有些麻烦。所以，C99 允许在结构体中定义的一个长度为零的数组，作为一个占位符，运行时再为其分配内存。

```cpp
#include <stdio.h>
#include <stdlib.h>

typedef struct heap {
    long size;
    int elements[0];
} Heap;

int main() {
    Heap heap = {0};

    printf("heap size: %ld \n", sizeof(heap));

    printf("heap address: %p \n", &heap);
    printf("heap.size address: %p \n", &heap.size);
    printf("heap.elements address: %p \n", &heap.elements);
}
```

运行结果为：

```bash
heap size: 8 
heap address: 0x7ffe67b7c390 
heap.size address: 0x7ffe67b7c390 
heap.elements address: 0x7ffe67b7c398 
```

所以，对于结构体中的零长数组而言，并不占用内存空间，但是会为其分配一个内存占位，这就为我们动态地创建数组打下了基础：

```cpp
#include <stdio.h>
#include <stdlib.h>

typedef struct heap {
    long size;
    int elements[0];
} Heap;

int main() {
    long size = 1024;
    
    // 一次性初始化完毕
    Heap *heap = (Heap *)malloc(sizeof(Heap) + sizeof(int) * size);

    heap->size = size;

    for (int i = 0; i < size; i++) {
        heap->elements[i] = i;
    }

    printf("size address: %p \n", &heap->size);
    printf("elements address: %p \n", heap->elements);
    printf("elements[1] address: %p \n", &heap->elements[1]);

    free(heap);
}

// size address: 0x55581becf2a0 
// elements address: 0x55581becf2a8 
// elements[1] address: 0x55581becf2ac
```

虽然柔性数组很方便，但是相比于指针定义仍然有不少的缺陷，例如**零长数组必须定义在结构体最后一个成员中，且结构体中不能有多个零长数组**。

（未完待续......










































