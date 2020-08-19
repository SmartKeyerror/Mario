---
layout: post
cover: false
navigation: true
title: Python虚拟机
date: 2020-08-18 10:50:25
tags: ['Python']
subclass: 'post tag-fiction'
author: smartkeyerror
categories: smartkeyerror
---

我们常说Python一是门解释型语言，只需要敲下`python code.py`就可以运行编写的代码，而无需使用类似于`javac`或者`gcc`进行编译。那么，Python解释器是真的一行一行读取Python源代码而后执行吗? 实际上，Python在执行程序时和Java、C#一样，都是先将源码进行编译生成字节码，然后由虚拟机进行执行，只不过Python解释器把这两步合二为一了而已。

<!---more--->

### 1. Python程序执行过程

事实上，Python程序在执行过程中同样需要编译(Compile)，编译产生的结果称之为字节码，而后由Python虚拟机逐行地执行这些字节码。所以，Python解释器由两部分组成: 编译器和虚拟机。

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/Python/Interpreter/process.png)

上图展示了Python程序的执行过程，以及C程序的编译、汇编与链接过程，从该图中可以非常明显地看出Python与C程序的执行区别。Python如此设计的原因在于将程序的执行与底层硬件进一步地分离，无需担心程序的编译、汇编以及链接过程，使得Python程序相较于C程序而言更加易于移植。

这里再说一下Python和Java的区别。Java在程序执行时必须使用`javac`对源代码进行编译，但是并不直接编译成机器语言，而是和Python一样，编译成字节码，而后由JVM进行执行。从这一点上来看，Python和Java非常类似，只不过Python的编译过程由解释器完成，用户也可以手动的对Python源代码进行编译，生成`.pyc`文件，节省那么一丢丢的时间。

```bash
python -m compileall <dir>
```

通过运行上述命令可对`<dir>`目录下所有的Python文件进行编译，编译结果将会存放于该目录下的`__pycache__`的`.pyc`文件中。


### 2. 编译过程与字节码

在Python的内建函数中，定义了`compile`以及`exec`两个方法，前者将源代码编译成为Code Object对象，Code Object对象中即保存着源代码所对应的字节。而`exec`方法则是运行Python语句或者是由`compile`方法所返回的Code Object。`exec`方法可直接运行Python语句，其参数并一定需要是Code Object。

```python
>>> snippet = "for i in range(3): print(f'Output: {i}')"

>>> result = compile(snippet, "", "exec")

>>> result
<code object <module> at 0x7f8e7e6471e0, file "", line 1>

>>> exec(result)
Output: 0
Output: 1
Output: 2
```

在上述代码中定义了一个非常简单的Python代码片段，其作用就是在标准输出中打印0，1，2这三个数而已。通过`compile`方法对该片段进行编译，得到Code Object对象，并将该对象交由`exec`函数执行。下面来具体看下返回的Code Object中到底包含了什么。

在源码`cpython/Include/code.h`中定义了`PyCodeObject`结构体，即Code Object对象:

```cpp
/* Bytecode object */
typedef struct {
    PyObject_HEAD               /* Python定长对象头 */
    PyObject *co_code;          /* 指令操作码，即字节码 */
    PyObject *co_consts;        /* 常量列表 */
    PyObject *co_names;         /* 名称列表(不一定是变量，也可能是函数名称、类名称等) */
    PyObject *co_filename;      /* 源码文件名称 */
    
    ...                         /* 省略若干字段 */
} PyCodeObject;
```

字段`co_code`即为Python编译后字节码，其它字段在此处可暂时忽略。字节码的格式为人类不可阅读格式，其形式通常是这样的:

```python
>>> result.co_code
b'x\x1ee\x00d\x00\x83\x01D\x00]\x12Z\x01e\x02d\x01e\x01\x9b\x00\x9d\x02\x83\x01\x01\x00q\nW\x00d\x02S\x00'
```

这个时候我们需要一个"反汇编器"来将字节码转换成人类可阅读的格式，"反汇编器"打引号的原因是在Python中并不能称为真正的反汇编器。

```python
>>> import dis
>>> dis.dis(result.co_code)
          0 SETUP_LOOP              30 (to 32)
          2 LOAD_NAME                0 (0)
          4 LOAD_CONST               0 (0)
          6 CALL_FUNCTION            1
          8 GET_ITER
    >>   10 FOR_ITER                18 (to 30)
         12 STORE_NAME               1 (1)
         14 LOAD_NAME                2 (2)
         16 LOAD_CONST               1 (1)
         18 LOAD_NAME                1 (1)
         20 FORMAT_VALUE             0
         22 BUILD_STRING             2
         24 CALL_FUNCTION            1
         26 POP_TOP
         28 JUMP_ABSOLUTE           10
    >>   30 POP_BLOCK
    >>   32 LOAD_CONST               2 (2)
         34 RETURN_VALUE
```

`dis`方法将返回字节码的助记符(mnemonics)，和汇编语言非常类似，从这些助记符的名称上我们就可以大概猜出解释器将要执行的动作，例如`LOAD_NAME`加载名称，`LOAD_CONST`加载常量。所以，我们完全可以将这些助记符看作是汇编指令，而指令的操作数则在助记符后面描述。例如`LOAD_NAME`操作，其操作数的下标为0，而在源代码中使用过的名称保存在`co_names`字段中，所以`LOAD_NAME  0`即表示加载`result.co_names[0]`:

```python
>>> result.co_names[0]
'range'
```

又比如`LOAD_CONST`操作，其操作数的下标也为0，只不过这次操作数不再保存在`co_names`，而是`co_consts`中，所以`LOAD_CONST  0`则表示加载`result.co_consts[0]`:

```python
>>> result.co_consts[0]
3
```

由于Code Object对象保存了常量、变量、名称等一系列的上下文内容，所以可以直接对该对象进行反汇编操作:

```python
>>> dis.dis(result)
  1           0 SETUP_LOOP              30 (to 32)
              2 LOAD_NAME                0 (range)
              4 LOAD_CONST               0 (3)
              ...
```

现在，我们可以对Python字节码做一下小结了。Python在编译某段源码时，并不会直接返回字节码，而是返回一个Code Object对象，字节码则保存在该对象的`co_code`字段中。由于字节码是一个二进制字节序列，无法直接进行阅读，所以需要通过"反汇编器"(`dis`模块)将字节码转换成人类可读的助记符。助记符的形式和汇编语言非常类似，均由操作指令+操作数所组成。

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/Python/Interpreter/Compile.png)


### 3. 命名空间与作用域

Python的命名空间与作用域经常被开发者所忽略，在未深入了解Python虚拟机之前，我个人也认为这些东西并不重要。但是，命名空间和变量作用域将会是Python虚拟机在执行过程中一个非常重要的一环。

命名空间实际上是名称到对象的一种映射，本质上就是一个键-值对，所以大部分的命名空间由`dict`实现。命名空间可以分为3类: 内置命名空间，全局命名空间与局部命名空间，在作用域存在嵌套的特殊情况下，可能还会有闭包命名空间。

#### 3.1 内置命名空间(Build-in)
Python语言内置的名称，例如内置函数名(`len`, `dis`)，内置异常(`Exception`)等。

```python
>>> import builtins
>>> builtins.__dict__
```

#### 3.2 全局命名空间(Global)

全局命名空间以模块进行划分，每一个模块中都包含了`dict`对象，其中保存了模块中的变量名、类名、函数名等等。

#### 3.3 局部命名空间(Local)

局部命名空间可以简单的认为就是函数的命名空间，例如函数参数，在函数中定义的局部变量。

在局部命名空间中，有一个非常特殊的情况，就是类定义:

```python
class Reader(object):

    BUFFER_SIZE = 4096
    
    class Inner(object):
        def __init__(self):
            print(BUFFER_SIZE)
```

在执行`Reader.Inner()`语句时，将会抛出`NameError`的异常，表示`BUFFER_SIZE`未定义。其原因在于类的嵌套并不等同于函数嵌套

#### 3.4 闭包命名空间(Enclosing)

当出现嵌套函数定义时，或者作用域嵌套时，Python将会把内层作用域所依赖的所有外层命名存储在一个特殊的命名空间中，也就是闭包命名空间。

```python
def foo(func):
    def wrapper(*args, **kwargs):
        logger.info(f"Execute func: {func.__name__}")
        func(*args, **kwargs)
    return wrapper
```

闭包指函数，而不是类，所以在类的嵌套中，将不会存在闭包命名空间:

```python
class Reader(object):

    BUFFER_SIZE = 4096
    
    class Inner(object):
        def __init__(self):
            print(BUFFER_SIZE)
```

在执行`Reader.Inner()`语句时，将会抛出`NameError`的异常，表示`BUFFER_SIZE`未定义。

当语句需要查找变量`X`时，将会按照 Local -> Enclosing -> Global -> Builtin 的顺序进行查找，俗称LEGB规则。

### 4. Python虚拟机的执行

#### 4.1 执行上下文——栈帧

在 x86-64 CPU 中包含了16个64位的通用目的寄存器，这些寄存器用于存储数据或者是指针。在这16个通用目的寄存器中，有两个较为特殊的寄存器: %rsp 与 %rbp。%rsp 为栈指针寄存器，表示运行时栈的结束位置，可以简单地理解为栈顶。%rbp 为栈帧指针，用于标识当前栈帧的起始位置。

在x86体系结构中，函数调用是通过栈和栈帧实现的。当一个函数被调用时，首先做的事情就是将调用者栈帧指针入栈，以保留调用关系。其次将为调用的函数创建栈帧，栈帧中包含了函数的参数、创建的局部变量等信息。

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/Python/Interpreter/x86-Stack.png)

回到Python虚拟机中，虚拟机在进行函数调用时，运行方式和x86没什么区别，都是由栈和栈帧所实现的。而栈帧则是由`PyFrameObject`表示，于源码`cpython/Include/frameobject.h`中定义。

```cpp
typedef struct _frame {
    PyObject_VAR_HEAD           /* Python固定长度对象头 */
    struct _frame *f_back;      /* 指向上一个栈帧的指针 */
    PyCodeObject *f_code;       /* Code Object代码对象，其中包含了字节码 */
    
    PyObject *f_builtins;       /* 内建命名空间字典(PyDictObject) */
    PyObject *f_globals;        /* 全局命名空间字典(PyDictObject) */
    PyObject *f_locals;         /* 局部命名空间表(通常是数组) */

    int f_lasti;                /* 上一条指令编号 */
    
    ...
} PyFrameObject;
```

可以看到，在一个栈帧中包含了Code Object代码对象，三个命名空间表，上一个栈帧指针等信息。可以说，`PyFrameObject`对象包含了Python虚拟机执行所需的全部上下文。结合前面提到的字节码和命名空间，我们可以用一张简图来描述。

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/Python/Interpreter/FrameObject.png)

#### 4.2 指令的执行

指令执行的源码均位于`cpython/Python/ceval.c`中，入口函数有两个，一个是`PyEval_EvalCode`，另一个则是`PyEval_EvalCodeEx`，最终的实际调用函数为`_PyEval_EvalCodeWithName`，所以我们只需要关注该函数即可。

`_PyEval_EvalCodeWithName`函数的主要作用为进行函数调用的例常检查，例如校验函数参数的个数、类型，校验关键字参数等。除此之外，该函数将会初始化栈帧对象并将其交给`PyEval_EvalFrame`函数进行处理，最终由`_PyEval_EvalFrameDefault`函数真正的运行指令。

`_PyEval_EvalFrameDefault`函数定义超过了3K行，绝大部分的逻辑其实都是`switch-case`: 根据指令类型执行相应的逻辑。

```cpp
for (;;) {
    switch (opcode) {
        case TARGET(LOAD_CONST): {      /* 加载常量 */
            ...
        }		
        case TARGET(ROT_TWO): {         /* 交换两个变量 */
            ...
        }
        case TARGET(FORMAT_VALUE):{     /* 格式化字符串 */
            ...
        }
```

可以看到`TARGET()`调用中的参数其实就是`dis`方法返回的助记符，当我们在分析助记符的具体实现逻辑时，可以在该文件中找到对应的C实现方法。

#### 4.3 GIL与字节码的执行

对于Python中的容器，例如dict，并没有实现像Java中的`ConcurrentHashMap`，或者是Golang中的`sync.Map`，这是因为Python中的容器(list, dict)本身就是并发安全的，但是在这些容器的源码中并没有发现定义`mutex`，也就是说，Python容器的并发安全并不是通过互斥锁实现的。

实际上，Python容器的并发安全是通过GIL实现的，也就是被广大Pythoner口诛笔伐的全局解释器锁。某一个线程想要运行必须要首先获取全局锁，如此一来，在同一时刻只能有一个线程运行，无法充分利用多核的硬件资源。

Python的线程调度非常类似于CPU的时间片实现，只不过并不是以时间为判断标准，而是以执行字节码的数量作为判断标准。当某一个线程执行了足够多的字节码条数时，当前线程将释放全局锁，唤醒其它线程进行执行。

**所以，得益于GIL的存在，Python容器在进行诸如扩容、缩容操作时，完全不必担心并发问题，因为一条字节码的执行一定是原子性的。**