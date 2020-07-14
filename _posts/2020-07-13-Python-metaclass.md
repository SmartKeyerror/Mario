---
layout: post
cover: false
navigation: true
title: 揭开Python元类(metaclass)神秘的面纱
date: 2020-07-13 10:06:25
tags: ['Python']
subclass: 'post tag-fiction'
author: smartkeyerror
categories: smartkeyerror
---

Python语言的`metaclass`特性一直是初学者的"噩梦"，当初博主在学习元类时也是一头雾水，但是一旦真正的理解了什么是"动态语言"之后，元类就不再神秘与难以理解了。Python这门动态语言最大的特性就是不需要一个类的字节码就能够在运行时创建出一个类，这是理解元类最为关键的信息。

<!---more--->

### 1. 基础知识汇总

#### 1.1 stackoverflow

首先，强烈推荐阅读stackoverflow上关于`metaclass`的回答，作者并没有使用什么高级词汇，就算英语稀烂也能看的懂。

> https://stackoverflow.com/a/6581949/12523821

#### 1.2 类属性和实例属性

类属性表示绑定在一个类上的属性，而实例属性则是绑定在不同实例上的属性，类属性只有一份，而实例属性则可以有多份。当实例属性和类属性重名，并通过实例获取该属性时，会返回实例属性，而不是类属性。

```python
class Hugo(object):
    name = None
    def __init__(self, name):
        self.name = name

if __name__ == "__main__":
    Hugo.name = "smart"
    print(Hugo.name)      # "smart"

    hugo = Hugo("raven")  
    print(hugo.name)      # "raven"

    print(Hugo.name)      # "smart"
```


#### 1.3 `__new__`方法和`__init__`方法

在Python中，实际创建对象的过程是由`__new__`方法控制的，该方法接收class对象(cls)。而`__init__`方法则是在`__new__`方法所创建的对象实例上，进行属性的赋值或者其它操作，所以接收实例对象(self)。

当想要控制创建对象的过程时，应该使用`__new__`方法，例如常用的单例模式，而不是使用`__init__`方法:

```python
from threading import Lock

class SingletonClass(object):
    instance = None
    lock = Lock()
    
    def __new__(cls, *args, **kwargs):
        if cls.instance:
            return cls.instance
        with cls.lock:
            # double check
            if not cls.instance:
                cls.instance = object.__new__(cls)
            return cls.instance
```

#### 1.4 MRO

Python是通过MRO列表来实现类的继承的，MRO列表的构造由C3线性化算法实现。实际上，类的继承层级关系最终会表现成包含所有基类的线性顺序表。

```python
class Parent(object):
    def __init__(self):
        print("Parent init")

class Children(Parent):
    def __init__(self):
        super(Children, self).__init__()
        print("Children init")

class Grandchildren(Children):
    def __init__(self):
        super(Grandchildren, self).__init__()
        print("Grandchildren init")

if __name__ == "__main__":
    print(Grandchildren.__mro__)
```

运行结果为:

```bash
(
    <class '__main__.Grandchildren'>, 
    <class '__main__.Children'>, 
    <class '__main__.Parent'>, 
    <class 'object'>
)
```

其顺序与继承顺序刚好相反，也就是说，通过类的`__mro__`属性即可找到该类的所有父类，包括`object`类。

Python同时也提供了内建的反射函数，来返回某个类的MRO列表:

```python
def getmro(cls):
    return cls.__mro__
```


### 2. metaclass 

我们已经知道了`metaclass`是创建一个类的工具，通过`metaclass`能够更加灵活地动态地创建一个类，其中一个非常重要的结果就是能够获取到"子类"的全部信息，例如类属性、类方法等。

```python
class HugoMetaclass(type):
    def __new__(mcs, name, bases, attrs):
    
        for name, value in attrs.items():
            print(f"get class field: {name}===>{value}")
            
        return super().__new__(mcs, name, bases, attrs)

class Hugo(metaclass=HugoMetaclass):
    name = "smart"
    gender = "male"
```

运行上述代码将会打印出`Hugo`类的所有属性信息:

```bash
get class field: __module__===>__main__
get class field: __qualname__===>Hugo
get class field: name===>smart
get class field: gender===>male
```

其中`__module__`和`__qualname__`为内部属性，而`name`和`gender`则是用户自定义的类属性。可以看到，在`HugoMetaclass。__new__`方法中，完全能够获取到`Hugo`类的相关类属性，那么更进一步地来说，不管用户定义了什么样的类属性，都可以使用`metaclass`在创建该类之前获取到该类的所有属性。这就为诸如ORM、表单验证等基础服务提供了构建的基础。

#### 2.1 metaclass的应用

`type`的`__new__`方法接收4个参数，分别为类对象，类名称，父类元组以及类属性。这四个参数中最为关键的就是父类元组和类属性，通常项目中使用`metaclass`时也是和这两个参数频繁打交道。

##### 2.1.1 父类元组

```python
class HugoMetaclass(type):
    def __new__(mcs, name, bases, attrs):
        print(bases)
        return super().__new__(mcs, name, bases, attrs)

class Hugo(metaclass=HugoMetaclass):
    pass

class HugoChild(Hugo):
    pass
```

运行后将得到以下结果:

```bash
()
(<class '__main__.Hugo'>,)
```

一共需要创建两个类: `Hugo`和`HugoChild`，`Hugo`类直接使用`HugoMetaclass`创建，所以其父类元组为空。而`HugoChild`直接继承自`Hugo`，所以其父类为`Hugo`。所以，可以通过`bases`参数来判断当前创建的类是否需要进行处理。

```python
class HugoMetaclass(type):
    def __new__(mcs, name, bases, attrs):
    
        parents = [b for b in bases if isinstance(b, HugoMetaclass)]
        if not parents:
            return super().__new__(mcs, name, bases, attrs)

        # 这里所创建的类都是Hugo的子类, 而不是Hugo类
        return super().__new__(mcs, name, bases, attrs)
```

##### 2.1.2 类属性

类属性是"子类"中最为重要的数据，可以说元类的最终目的就是为了根据类属性创建出一个模板，将该模板数据保存在类中。

```python
class HugoMetaclass(type):
    def __new__(mcs, name, bases, attrs):

        parents = [b for b in bases if isinstance(b, HugoMetaclass)]

        # 对Hugo类不做任何处理
        if not parents:
            return super().__new__(mcs, name, bases, attrs)

        klass = super().__new__(mcs, name, bases, attrs)

        # 保存attrs中所有的int类型数据
        klass.declared_fields = {}

        for name, value in attrs.items():
            if isinstance(value, int):
                klass.declared_fields[name] = value

        return klass
        
class Hugo(metaclass=HugoMetaclass):
    pass

class HugoChild(Hugo):
    name = "smart"
    age = 24

if __name__ == "__main__":
    print(HugoChild.declared_fields)
```

上面创建了一个`int`类型的"模板"，并保存在了`declared_fields`这个字典中。注意不要将`declared_fields`挂到`mcs`上，`mcs`就是`HugoMetaclass`，变量绑定到`mcs`上会丢失一些信息，导致程序出现BUG。

那么如果`HugoChild`又有子类呢? 上述方式是否能够将`HugoChild`和其子类的属性一起获取到呢?

```python
class HugoChild(Hugo):
    name = "smart"
    age = 24

class HugoGrandChild(HugoChild):
    height = 180

if __name__ == "__main__":
    print(Hugo.declared_fields)
    print(HugoChild.declared_fields)
    print(HugoGrandChild.declared_fields)
```

这时候会发现，这三个类的`declared_fields`结果都是`{'height': 180}`，`age`字段丢失了。原因也很简单，在创建`HugoGrandChild`类时，`declared_fields`被重新声明成了空字典，所以`HugoChild`中的类属性就会丢失。那么有没有什么办法能够得到完整版呢? 这就需要用到上面所提到的MRO列表了。

我们可以通过MRO列表，来获取到`HugoGrandChild`的所有父类，而后逐一的遍历找出类型为`int`的类属性，保存在`declared_fields`这个字典中。

```python
class HugoMetaclass(type):
    def __new__(mcs, name, bases, attrs):

        parents = [b for b in bases if isinstance(b, HugoMetaclass)]

        # 对Hugo类不做任何处理
        if not parents:
            return super().__new__(mcs, name, bases, attrs)

        # 保存attrs中所有的int类型数据
        klass = super().__new__(mcs, name, bases, attrs)

        klass.declared_fields = {}

        for name, value in attrs.items():
            if isinstance(value, int):
                klass.declared_fields[name] = value

        # 遍历__mro__列表并找出类型为`int`的类属性, 保存在字典中
        for parent in klass.__mro__:
            for name, value in getattr(parent, 'declared_fields', parent.__dict__).items():
                if isinstance(value, int):
                    klass.declared_fields[name] = value

        return klass

if __name__ == "__main__":
    print(HugoChild.declared_fields)
    print(HugoGrandChild.declared_fields)
```

其运行结果为:

```bash
{'age': 24}
{'height': 180, 'age': 24}
```

如此一来，`HugoGrandChild`在继承了`HugoChild`之后，也能够获取到其中的相关字段，并且父类不会受到子类的影响。

上述代码中存在一些重复的代码片段，将其抽离出来，使代码结构更加清晰:

```python
def is_instance_or_subclass(val, class_):
    try:
        return issubclass(val, class_)
    except TypeError:
        return isinstance(val, class_)

def _get_fields(attrs, field_class):
    fields = [
        (field_name, attrs.get(field_name))
        for field_name, field_value in list(attrs.items())
        if is_instance_or_subclass(field_value, field_class)
    ]
    return fields

def _get_fields_by_mro(klass, field_class):
    mro = klass.__mro__
    return sum(
        (
            _get_fields(
                getattr(base, 'declared_fields', base.__dict__),
                field_class,
            )
            for base in mro[:0:-1]
        ),
        [],
    )

class HugoMetaclass(type):
    def __new__(mcs, name, bases, attrs):

        parents = [b for b in bases if isinstance(b, HugoMetaclass)]

        # 对Hugo类不做任何处理
        if not parents:
            return super().__new__(mcs, name, bases, attrs)

        # 保存attrs中所有的int类型数据
        klass = super().__new__(mcs, name, bases, attrs)

        class_fields = _get_fields(attrs, int)
        inherited_fields = _get_fields_by_mro(klass, int)
        klass.declared_fields = dict(class_fields + inherited_fields)

        return klass
```

### 3. 小结

`metaclass`并不神秘，得益于Python是动态语言，可以在运行时动态地创建一个类的特性，我们能够在事前去创建一些有用的"模板"，在运行时将模板和数据有机的结合起来，最终呈现出宛如魔术般的效果。