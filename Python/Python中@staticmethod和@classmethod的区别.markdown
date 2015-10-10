# Python中@staticmethod和@classmethod的区别

标签（空格分隔）： Python

---
本文翻译自：[Difference between @staticmethod and @classmethod in Python](http://www.pythoncentral.io/difference-between-staticmethod-and-classmethod-in-python/)

[TOC]

## Python中的类方法与静态方法
在本文中，我会尝试解释静态方法和类方法是什么以及他们之间的区别。静态方法和类方法都使用装饰符（decorators)来定义静态方法和类方法，跟普通方法定义一样。假如想对Python的修饰符有个基本了解，可以看一下这篇文章《[Python Decorators Overview](http://www.pythoncentral.io/python-decorators-overview/)》。

## 简单方法、静态方法和类方法
在类中最常用的方法是实体方法，例如，类实体作为方法的第一个参数。

例如，一个基本的类实体方法可能如下所示：
```python
class Kls(object):
    def __init__(self, data):
        self.data = data
        
    def prind(self):
        print(self.data)
        
ik1 = Kls('arun')
ik2 = Kls('seema')

ik1.printd()
ik2.printd()
```
这将得到如下输出：
```python
arun
seema
```
![](http://www.pythoncentral.io/wp-content/uploads/2013/02/instancemethod.png)
看完样例代码和上面的图之后：
- 在箭头1和2中，实参分别传给方法中的self,data两个形参；
- 在箭头3，self参数指向ik1这个实体；
- 在箭头4，我们不需要将实体作为参数提供给方法，就像它能自己处理参数说明；

现在如果我们只想与类交互，而与不是类的实体交互，怎么办呢？我们可以在类外面写一个简单的方法来做这件事，但这样会将与代码与类的关系放在了类之外。这可能会导致未来的代码维护问题，如下：
```python
def get_no_of_instances(cls_obj):
    return cls_obj.no_inst
    
class Kls(object):
    no_inst = 0
    
    def __init__(self):
        Kls.no_inst = Kls.no_inst +１
        
ik1 = Kls()
ik2 = Kls()

print(get_no_of_instances(Kls))
```
得到如下输出：
```
2
```
### @classmethod
现在我们想要做的是在类中创建一个方法，该方法使用类对象而不是实体。如果我们想不使用实体，那我们所需要做的如下所示：
```python
def iget_no_of_instance(ins_obj):
    return ins_obj.__class__.no_inst
    
class Kls(object):
    no_inst = 0
    
    def __init__(self):
        Kls.no_inst = Kls.no_inst +１

ik1 = Kls()
ik2 = Kls()
print iget_no_of_instance(ik1)
```
```
2
```
使用Python2.2之后的功能，我们可以在类中创建一个方法，使用@classmethod修饰符。
```python
class Kls(object):
    no_inst = 0
    
    def __init__(self):
        Kls.no_inst = Kls.no_inst + 1
        
    @classmethod
    def get_no_of_instance(cls_obj):
        return cls_obj.no_inst
        
ik1 = Kls()
ik2 = Kls()

print ik1.get_no_of_instance()
```
得到如下输出：
```
2
2
```
这个优势在于：不管我们是通过类实体还是类调用该方法，它会将类作为第一个参数传给该方法。

### @staticmethod
经常会有些功能与类相关，但不需要类或者实体做任何操作。可能是设置环境变量，改变另一个类的属性等等。在这样的情况下，我们同样可以使用一个方法，尽管这样可能会导致后续代码维护的问题。

以下是一个样例：
```python
IND = 'ON'

def checkind():
    return (IND == 'ON')

class Kls(object):
    def __init__(self,data):
        self.data = data
        
def do_reset(self):
    if checkind():
        print('Reset done for:', self.data)
        
def self_db(self):
    if checkind():
        self.db = 'new db connection'
        print('DB connection made for:', self.data)
        
ik1 = Kls(12)
ik1.do_reset()
ik1.self_db()
```
得到如下输出：
```
Reset done for: 12
DB connection made for: 12
```
如果我们在此使用@staticmethod修饰符，我们可以把所有代码放在相关联的位置。
```python
IND = 'ON'
 
class Kls(object):
    def __init__(self, data):
        self.data = data
 
    @staticmethod
    def checkind():
        return (IND == 'ON')
 
    def do_reset(self):
        if self.checkind():
            print('Reset done for:', self.data)
 
    def set_db(self):
        if self.checkind():
            self.db = 'New db connection'
        print('DB connection made for: ', self.data)
 
ik1 = Kls(12)
ik1.do_reset()
ik1.set_db()
```
得到如下输出：
```
Reset done for: 12
DB connection made for: 12
```

下面是一个更全面的例子，还有图说明。
### @staticmethod和@classmethod如何不同
```python
class Kls(object):
    def __init__(self, data):
        self.data = data
 
    def printd(self):
        print(self.data)
 
    @staticmethod
        def smethod(*arg):
            print('Static:', arg)
 
    @classmethod
        def cmethod(*arg):
            print('Class:', arg)
```
```python
>>> ik = Kls(23)
>>> ik.printd()
23
>>> ik.smethod()
Static: ()
>>> ik.cmethod()
Class: (<class '__main__.Kls'>,)
>>> Kls.printd()
TypeError: unbound method printd() must be called with Kls instance as first argument (got nothing instead)
>>> Kls.smethod()
Static: ()
>>> Kls.cmethod()
Class: (<class '__main__.Kls'>,)
```
接下来用图来解释到底发生了什么：
![](http://www.pythoncentral.io/wp-content/uploads/2013/02/comparison.png)





