---
title: logging源码解析之Manager
urlname: logging/Manager
categories: python
tags:
  - python
description: 在本篇文章中介绍了logging模块的Manager类，其主要维护了Logger实例的层次结构。
p: /python/logging源码解析之Manager
date: 2018-02-10 09:45:22
updated: 2018-02-10 09:45:22
---
### 一、引言
之前我们已经详细讨论了`{% post_link python/logging源码解析之LogRecord LogRecord %}`、`{% post_link python/logging源码解析之Formatter Formatter %}`、`{% post_link python/logging源码解析之Filter Filter %}`和`{% post_link python/logging源码解析之Handler Handler %}`，接下来在我们介绍另一重要概念之前，我们还需要详细介绍一个类——`Manager`，它对`Logger`很重要，主要管理`logger`的层次结构。
### 二、`PlaceHolder`
类`PlaceHolder`实例在`logger`层次结构`Manager`中被用来代替未定义的`logger`节点。 此类仅供内部使用，不作为公共API的一部分。
#### 1. 定义
```python
class PlaceHolder(object):
```
#### 2. `_init__`
```python
def __init__(self, alogger):
        self.loggerMap = {alogger: None}
```
`__init__`方法中初始化了一个`dict`结构的属性`loggerMap`，初始值是给定的`logger`作为`key`值，`None`作为`value`值的字典；
#### 3. `append`
```python
    def append(self, alogger):
        if alogger not in self.loggerMap:
            self.loggerMap[alogger] = None
```
`append`方法是将指定的`logger`添加到属性`loggerMap`字典中，`key`值为给定的`logger`，而`value`值是`None`。

> 综上可知，类`PlaceHolder`中仅有一个属性`loggerMap`，是一个字典，其中`key`值是一个`logger`实例，而对应的`value`值总为`None`。

### 三、`_loggerClass`
`_loggerClass`是一个全局变量，定义的是初始化`logger`实例所用的类，定义了两个关于它的函数。
#### 1. `setLoggerClass`
```python
def setLoggerClass(klass):
       if klass != Logger:
        if not issubclass(klass, Logger):
            raise TypeError("logger not derived from logging.Logger: " + klass.__name__)
    global _loggerClass
    _loggerClass = klass
```
`setLoggerClass`函数是设置被用于实例化`logger`时的类，且将`_loggerClass`变量设置为模块级全局变量。 该类应该定义`__init __()`方法，这样只需要一个`name`参数，而`__init __()`应该调用`Logger .__ init __()`。
#### 2. `getLoggerClass`
```python
def getLoggerClass():
    return _loggerClass
```
`getLoggerClass`函数返回实例化`logger`时使用的类。

举个例子:
```python
def logger_class():
    import logging
    print(logging._loggerClass)

if __name__ == '__main__':
    logger_class()
    
Output: <class 'logging.Logger'>
```
由上面例子可知，默认实例化`logger`的类就是`'logging.Logger`类。
### 四、`Manager`
> [正常情况下]只有一个Manager实例，它管理`logger`的层次结构。
#### 1. 定义
```python
class Manager(object):
```
#### 3. `__init__`
```python
    def __init__(self, rootnode):
        self.root = rootnode
        self.disable = 0
        self.emittedNoHandlerWarning = False
        self.loggerDict = {}
        self.loggerClass = None
        self.logRecordFactory = None
```
`__init__`方法使用`logger`层次结构的根节点，也就是`rootnode`参数来初始化`Manager`的属性`root`。
#### 4. `setLoggerClass`
```python
    def setLoggerClass(self, klass):
        if klass != Logger:
            if not issubclass(klass, Logger):
                raise TypeError("logger not derived from logging.Logger: " + klass.__name__)
        self.loggerClass = klass
```
`setLoggerClass`方法用了设置在使用`Manager`类实例化`logger`时要使用的类。
#### 5. `setLogRecordFactory`
```python
    def setLogRecordFactory(self, factory):
        self.logRecordFactory = factory
```
`setLogRecordFactory`方法用了设置在使用此Manager实例化日志`record`时要使用的工厂。

> 接下来几个方法就是重点了，它们实现了`Manager`类是如何管理`logger`的层次结构的，它们的父节点和子节点是如何形成的。

#### 6. `getLogger`
```python
    def getLogger(self, name):
        rv = None
        if not isinstance(name, str):
            raise TypeError('A logger name must be a string')
        _acquireLock()
        try:
            if name in self.loggerDict:
                rv = self.loggerDict[name]
                if isinstance(rv, PlaceHolder):
                    ph = rv
                    rv = (self.loggerClass or _loggerClass)(name)
                    rv.manager = self
                    self.loggerDict[name] = rv
                    self._fixupChildren(ph, rv)
                    self._fixupParents(rv)
            else:
                rv = (self.loggerClass or _loggerClass)(name)
                rv.manager = self
                self.loggerDict[name] = rv
                self._fixupParents(rv)
        finally:
            _releaseLock()
        return rv
```
`getLogger`方法是`Manager`类的核心方法，用来获取指定`name`的`logger`，如果不存在则创建它。 参数`name`是以`.`点分隔的分层名称，例如“a”，“a.b”，“a.b.c”或类似名称。

下面我会举几个例子分析一下这个方法的执行逻辑，顺便也会分析`_fixupChildren`和`_fixupParents`这两个方法。
##### 1. Example 1
条件: `name`参数为`A.B`，但`A`不存在，也就是`loggerDict`是一个空字典。
> 分析如下：
> 1. 初始化`rv=None`，获取全局线程锁;
> 2. 不满足`if`条件，进入`else`代码块，初始化一个`name=A.B`的`logger`实例并将其赋值给`rv`，并将`rv`的`manager`属性设置为`self`，也就是此`Manager`实例，并将`A.B`添加到`loggerDict`中，其对应的`value`值就是`rv`;
> 3. 调用方法`_fixupParents`，参数是`rv`，内部执行逻辑如下：
>(1). 参数`alogger=rv`，则`alogger.name=A.B`，那`i=1`，此方法中的局部变量`rv`初始值为`None`;                     
> (2). 第一次进入`while`循环，此时`substr=A`，`substr`满足`if`条件，然后`A`作为`key`值添加到字典`loggerDict`中，其对应的`value`是以`alogger`初始化的一个`PlaceHolder`实例`plcaehodler`;
> (3). 第`2`步执行完后，`i=-1`，`rv=None`，不满足`while`条件退出循环;
> (4). 由于`rv=None`，所以执行`rv=self.root`，也就是`rv`为根节点，最后把`alogger`的父节点设置为`rv`，也就是根节点;
> 4. 最后，释放全局线程锁，执行结束。

综上可知，如果`name=A.B`在字典中`loggerDict`不存在，就创建它，并将其添加到`loggerDict`字典中，而如果它的某个上层节点也不存在，就用它自身生成一个`PlaceHolder`对象替代它的某个上层节点添加到`loggerDict`字典中，直到它的某个上层节点存在为止，如果它的上层节点均不存在，那就把根节点设置为其父节点。

所以在这个例子中，属性`loggerDict`字典中添加了两个`key`，其中一个是属性`A.B`，其对应的`value`值是一个`Logger('A.B')`实例；另一个是`A`，其对应的`value`是`PlaceHolder(Logger('A.B'))`。
##### 2. Example 2
条件：`name`参数为`A`，以 Example 1 的结果为前提条件。
> 分析如下：
> 1. 初始化`rv=None`，获取全局线程锁;
> 2. 此时满足`if`条件，在`loggerDict`中其对应的`value`值是`PlaceHolder(Logger('A.B'))`，是一个`PlaceHolder`实例，进入第二层`if`条件，此时`ph=PlaceHolder(Logger('A.B'))`，`rv`为一个`name=A`的`Loggger`实例，且其`manager`属性为此`Manager`实例，然后将`loggerDict`中其对应的`value`值设置为`rv`;
> 3. 调用方法`_fixupChildren`，参数是`ph`和`rv`，内部执行逻辑如下：
> (1). 参数`alogger=rv`，则`alogger.name=A`，那么初始化`name=A`，`namelen=1`;  
> (2). 遍历`ph`的属性`loggerMap`中的`key`列表，也就是`logger`列表，此时`key`列表中只有一个`key`，那就是`Logger('A.B')`，根据上面的分析，其父节点为根节点，默认是`root`;
> (3). 此时满足`if`条件，将`Logger('A.B')`的父节点赋值给`Logger('A')`的父节点，也就是说`Logger('A')`的父节点是根节点，然后再将`Logger('A.B')`的父节点设置为`Logger('A')`;
> 4. 调用方法`_fixupParents`，参数是`rv`，内部执行逻辑如下：
>(1). 参数`alogger=rv`，则`alogger.name=A`，那`i=-1`，此方法中的局部变量`rv`初始值为`None`;                     
> (2). 不满足`while`条件，由于`rv=None`，所以执行`rv=self.root`，也就是`rv`为根节点，最后把`alogger`的父节点设置为`rv`，也就是根节点;

从这个例子可以看出这个方法还可以修正`Logger`之间的层次结构，如果我们先创建了子节点，那其不存在的父节点就用`PlaceHolder`对象代替，其后如果还要创建其父节点，则会自动修正其父子节点的关系，确保`Logger`之间总保持层次结构。

#### 7. `_fixupParents`
```python
    def _fixupParents(self, alogger):
        name = alogger.name
        # 返回‘.’在字符串name中最后一次出现的位置(从右向左查询)
        # 如: name="gunicorn.test.error", 则 i=13
        i = name.rfind(".")
        rv = None
        while (i > 0) and not rv:
            substr = name[:i]
            if substr not in self.loggerDict:
                self.loggerDict[substr] = PlaceHolder(alogger)
            else:
                obj = self.loggerDict[substr]
                if isinstance(obj, Logger):
                    rv = obj
                else:
                    assert isinstance(obj, PlaceHolder)
                    obj.append(alogger)
            i = name.rfind(".", 0, i - 1)
        if not rv:
            rv = self.root
        alogger.parent = rv
```
`_fixupParents`方法就是用来为给定的`logger`设置`parent`节点的，从而确保从给定的`logger`到根节点的`logger`之间一直有`logger`或`placeholder`对象。

具体分析见上面，这里不再详细讨论。
#### 8. `_fixupChildren`
```python
    def _fixupChildren(self, ph, alogger):
        name = alogger.name
        namelen = len(name)
        for c in ph.loggerMap.keys():
            # The if means ... if not c.parent.name.startswith(nm)
            if c.parent.name[:namelen] != name:
                alogger.parent = c.parent
                c.parent = alogger
```
`_fixupChildren`方法确保`ph`实例中`loggerMap`的所有`key`值(也就是`Logger`对象)设置正确的父节点。

具体分析见上面，这里不再详细讨论。
### 五、总结
> 本篇文章中讨论的`Manager`类主要是为`Logger`类的讨论做铺垫，其实现的功能主要是为所有的`Logger`维护了一个层级关系，如果指定`logger`的上一节点不存在，就用一个`PlaceHolder`代替，如果存在，那就是这个子节点的父节点。



