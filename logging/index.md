---
title: logging源码解析之Introduction
urlname: logging/Introduction
categories: python
tags:
  - python
  - logging
  - 源码解析
description: 本篇文章中主要对logging模块进行简要的介绍，以及对__init__文件中定义的几个模块级变量以及常量的说明.
p: /python/logging源码解析之Introduction
date: 2018-01-06 14:13:39
updated: 2018-01-06 14:13:39
---
### 一、引言
在我们执行代码时，总希望将一些执行时的数据或者错误记录到指定的日志文件或者标准输出、错误输出等，在Python中我们用的最多的模块就是`logging`了，`logging`模块是在`python 2.3`版本时被引入为内置模块的。那么它究竟是如何将日志按照我们定义的格式输出到指定位置的呢？从今天开始，我们就来详细分析下这个模块的内部机制。

源码路径: `lib/python3.6/logging/`
### 二、准备工作
`logging`模块包含三个`.py`文件，分别是：
- `__init__.py`: 定义处理日志记录的各种类；
- `config.py`: 定义不同的读取日志配置的函数；
- `handlers.py`: 定义扩展的`Handler`类。

我们首先分析`__init__.py`这个文件，在这个文件中定义了我们所熟知的所有`logging`的对象：
- [LogRecord](logging/LogRecord.md)
// - `Formatter`
// - `Filter`
// - `Handler`
// - `Logger`

针对这些对象的实现及意义我们将一一进行分析。

在这个模块的开头定义了几个常量和简单函数，一起来看一下：
#### 1. 杂项数据
```python
# _startTime 用作计算事件的相对时间
_startTime = time.time()

# raiseExceptions用于查看是否应传播处理期间的异常
raiseExceptions = True

# 如果您不想在日志中使用线程信息，请将其设置为零
logThreads = True

# 如果您不想在日志中使用multiprocessing处理信息，请将其设置为零
logMultiprocessing = True

# 如果您不希望日志中包含进程信息，请将此值设置为零
logProcesses = True
```
这个数据在`import logging`时就已经可以访问到了，也可以直接设置它们，示例如下:
```python
In [11]: import logging

In [12]: logging._startTime
Out[12]: 1562583406.426662

In [13]: logging._startTime
Out[13]: 1562583406.426662

In [14]: logging.raiseExceptions
Out[14]: True

In [15]: logging.logThreads
Out[15]: True

In [16]: logging.logMultiprocessing
Out[16]: True

In [17]: logging.logProcesses
Out[17]: True

In [18]: logging.logProcesses = False

In [19]: logging.logProcesses
Out[19]: False

```
#### 2. `level`常量
```python
CRITICAL = 50
FATAL = CRITICAL
ERROR = 40
WARNING = 30
WARN = WARNING
INFO = 20
DEBUG = 10
NOTSET = 0
```
这部分定义了关于日志不同`level`用不同的`int`类型的数字表示，全部大写说明这些是常量数据。日志功能应以所追踪事件级别或严重性而定，各级别适用性如下（以严重性递增）：
```table
| 级别(-)    | 何时使用|
|DEBUG     | 细节信息，仅当诊断问题时适用。|
|INFO         | 确认程序按预期运行                        |
|WARNING| 表明有已经或即将发生的意外(例如：磁盘空间不足)。程序仍按预期进行|
|ERROR      | 由于严重的问题，程序的某些功能已经不能正常执行|
|CRITICAL  | 严重的错误，表明程序已不能继续执行|
```
默认的级别是`WARNING`，意味着只会追踪该级别及以上的事件，除非更改日志配置。

所追踪事件可以以不同形式处理。最简单的方式是输出到控制台，另一种常用的方式是写入磁盘文件。
一个非常简单的例子:
```python
import logging
logging.warning('Watch out!')  # 打印一条消息
logging.info('I told you so')  # 不会打印
```
如果你在命令行中输入这些代码并运行，你将会看到：
```python
WARNING:root:Watch out!
```
`INFO`消息并没有输出到命令行，因为默认级别是`WARNING`。打印的信息包含事件的级别以及在日志调用中的对于事件的描述，例如“Watch out!”。暂时不用担心“root”部分：之后会作出解释。输出格式可按需要进行调整，`format`选项同样会在之后作出解释。

#### 3. `_levelToName`
```python
_levelToName = {
    CRITICAL: 'CRITICAL',
    ERROR: 'ERROR',
    WARNING: 'WARNING',
    INFO: 'INFO',
    DEBUG: 'DEBUG',
    NOTSET: 'NOTSET',
}
```
全局变量`_levelToName`是一个字典，定义的是`int`类型的`level`表示与`str`类型的`level`表示之间的对应关系，如下：
```python
In [20]: logging._levelToName
Out[20]: 
{0: 'NOTSET',
 10: 'DEBUG',
 20: 'INFO',
 30: 'WARNING',
 40: 'ERROR',
 50: 'CRITICAL'}
```
#### 4. `_nameToLevel`
```python
_nameToLevel = {
    'CRITICAL': CRITICAL,
    'FATAL': FATAL,
    'ERROR': ERROR,
    'WARN': WARNING,
    'WARNING': WARNING,
    'INFO': INFO,
    'DEBUG': DEBUG,
    'NOTSET': NOTSET,
}
```
全局变量`_nameToLevel`是一个`dict`类型，定义的是`str`类型的`level`表示与`int`类型的`level`表示之间的对应关系，如下:
```python
In [21]: logging._nameToLevel
Out[21]: 
{'CRITICAL': 50,
 'DEBUG': 10,
 'ERROR': 40,
 'FATAL': 50,
 'INFO': 20,
 'NOTSET': 0,
 'WARN': 30,
 'WARNING': 30}
```
可以看到还定义了一个级别为`NOTSET`，其对应的`int`值为`0`，后面我们会讨论到。
#### 5. `getLevelName`
```python
def getLevelName(level):
    result = _levelToName.get(level)
    if result is not None:
        return result
    result = _nameToLevel.get(level)
    if result is not None:
        return result
    return "Level %s" % level
```
这个函数返回日志级别`level`的`str`或者`int`表示。说明如下：
- 如果给定的`level`是`int`类型且在`_levelToName`中有对应的`value`值，则返回其`str`类型的表示;
- 如果给定的`level`是`str`类型且在`_nameToLevel`中有对应的`value`值，则返回其`int`类型的表示;
- 否则，返回字符串`"Level ％s" ％level`。
例如：
```python
In [22]: logging.getLevelName(0)
Out[22]: 'NOTSET'

In [23]: logging.getLevelName('WARN')
Out[23]: 30

In [24]: logging.getLevelName('DEBUG')
Out[24]: 10

In [25]: logging.getLevelName(100)
Out[25]: 'Level 100'

In [26]: logging.getLevelName('debug')
Out[26]: 'Level debug'
```
#### 6. `addLevelName`
```python
def addLevelName(level, levelName):
    _acquireLock()
    try:  # unlikely to cause an exception, but you never know...
        _levelToName[level] = levelName
        _nameToLevel[levelName] = level
    finally:
        _releaseLock()
```
这个函数是将`levelName`与`level`相关联，**参数level 与 levelName 没有任何限制**：
- 在`_levelToName`变量中添加`{level: levelName}`；
- 在`_nameToLevel`变量中添加`{levelName: level}`。

例如：
```python
In [35]: logging.addLevelName(5, 'debug')

In [36]: logging._levelToName
Out[36]: 
{0: 'NOTSET',
 5: 'debug',
 10: 'DEBUG',
 20: 'INFO',
 30: 'WARNING',
 40: 'ERROR',
 50: 'CRITICAL'}

In [37]: logging._nameToLevel
Out[37]: 
{'CRITICAL': 50,
 'DEBUG': 10,
 'ERROR': 40,
 'FATAL': 50,
 'INFO': 20,
 'NOTSET': 0,
 'WARN': 30,
 'WARNING': 30,
 'bbb': 'aaa',
 'debug': 5}

```
#### 7. `_checkLevel`
```python
def _checkLevel(level):
    if isinstance(level, int):
        rv = level
    elif str(level) == level:
        if level not in _nameToLevel:
            raise ValueError("Unknown level: %r" % level)
        rv = _nameToLevel[level]
    else:
        raise TypeError("Level not an integer or a valid string: %r" % level)
    return rv
```
这个函数是为了检测给定的`level`是否合法，`level`参数可以是`int`类型或者`str`类型(`str`类型的`level`必须是`_nameToLevel`中的`key`)，返回的是`level`的`int`数值表示，示例如下：
```python
In [38]: logging._checkLevel(2)
Out[38]: 2

In [39]: logging._checkLevel('oo')
---------------------------------------------------------------------------
ValueError                                Traceback (most recent call last)
<ipython-input-39-aa2a66382bec> in <module>()
----> 1 logging._checkLevel('oo')

~/anaconda3/lib/python3.6/logging/__init__.py in _checkLevel(level)
    193 
    194 def _checkLevel(level):
--> 195     if isinstance(level, int):
    196         rv = level
    197     elif str(level) == level:

ValueError: Unknown level: 'oo'

In [40]: logging._checkLevel('debug')
Out[40]: 5

In [41]: logging._checkLevel('WARN')
Out[41]: 30

In [42]: logging._checkLevel(20)
Out[42]: 20
```
#### 8. `_lock`
```python
if threading:
    _lock = threading.RLock()
else:  # pragma: no cover
    _lock = None
```
在`logging`模块中`_lock`被用于对共享数据结构的序列化访问。
`threading.RLock()`是多重锁，在同一线程中可用被多次`acquire`。如果使用`RLock`，那么`acquire`和`release`必须成对出现，即调用了`n`次`acquire`锁请求，则必须调用`n`次的`release`才能在线程中释放锁对象。
查看`_lock`模块锁：
```python
In [43]: logging.threading
Out[43]: <module 'threading' from '/Users/yajingliu/anaconda3/lib/python3.6/threading.py'>

In [44]: logging._lock
Out[44]: <unlocked _thread.RLock object owner=0 count=0 at 0x10fab4e70>
```
#### 9. `_acquireLock`
```python
def _acquireLock():
    if _lock:
        _lock.acquire()
```
这个函数用来获取模块级锁`_lock`从而实现对共享数据的序列化访问，必须调用`_releaseLock()`释放它。
示例如下: 
```python
In [47]: logging._acquireLock()

In [48]: logging._lock
Out[48]: <locked _thread.RLock object owner=140736003502976 count=1 at 0x10fab4e70>
```
#### 10. `_releaseLock`
```python
def _releaseLock():
    if _lock:
        _lock.release()
```
这个函数是用来释放通过调用`_acquireLock()`获取的模块级锁`_lock`的。
示例如下：
```python
In [49]: logging._releaseLock()

In [50]: logging._lock
Out[50]: <unlocked _thread.RLock object owner=0 count=0 at 0x10fab4e70>
```
#### 11. `currentframe`
```python
if hasattr(sys, '_getframe'):
    currentframe = lambda: sys._getframe(3)
else:  # pragma: no cover
    def currentframe():
        """Return the frame object for the caller's stack frame."""
        try:
            raise Exception
        except Exception:
            return sys.exc_info()[2].tb_frame.f_back
```
`sys._getframe([depth])`函数是从调用堆栈中返回一个`frame`对象：
- 如果给出了可选的整数`depth`， `depth`的默认值为零，返回调用堆栈顶部的`frame`对象(也就是调用堆栈中第`0`个`frame`对象)；
- 如果不为`0`，则返回调用堆栈中从顶部数的第`depth`个`frame`对象；
- 如果`depth`比调用堆栈的深度大，则引发`ValueError`。
举例说明：
```python
In [1]: import logging

In [2]: logging.basicConfig(level=logging.WARNING, format='%(asctime)s - %(filename)s[line:%(lineno)d] - %(levelname)s: %(message)s')

In [3]: logging.error('test log')
"堆栈调用中的第三个frame对象": 1372 /Users/yajingliu/anaconda3/lib/python3.6/logging/__init__.py error
"堆栈调用中的第四个frame对象": 1916 /Users/yajingliu/anaconda3/lib/python3.6/logging/__init__.py error
"堆栈调用中的第五个frame对象":  1 <ipython-input-3-33d1d8e41f63> <module>
2019-07-09 20:13:56,745 - <ipython-input-3-33d1d8e41f63>[line:1] - ERROR: test log
```
由于`sys._getframe(3)`中的`depth`参数为`3`，所以我们获取的就是调用堆栈中从顶部数的第三个`frame`对象，但我们可以根据`frame`对象的`f_back`属性获取它的上一个`frame`对象，也就是第`depth+1`个，直到我们获取到最底部的`frame`对象为止，也就是最后一个`frame`对象的`f_back`属性是`None`。

`currentframe`这个可调用对象主要在`Logger`类中用到，届时再详细说明。
#### 12. `_srcfile`
```python
_srcfile = os.path.normcase(addLevelName.__code__.co_filename)
```
执行结果如下:
```python
In [6]: logging.addLevelName.__code__.co_filename
Out[6]: '/Users/yajingliu/anaconda3/lib/python3.6/logging/__init__.py'

In [7]: import os

In [8]: os.path.normcase(logging.addLevelName.__code__.co_filename)
Out[8]: '/Users/yajingliu/anaconda3/lib/python3.6/logging/__init__.py'
```
由上面的执行结果可知`_srcfile`就是`logging`的`__init__.py`的文件路径。
### 三、总结
综上，我们前期的准备工作就结束了，接下来正式开始对`logging`模块的各个类进行详细的分析，文章更新的比较慢，计划每周写一篇，但是会写的比较详细，以便我们从源码解析的过程中对我们的知识查漏补缺。