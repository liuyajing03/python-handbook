---
title: logging源码解析之LogRecord
urlname: logging/LogRecord
categories: python
tags:
  - python
  - logging
  - 源码解析
description: 在本篇文章中将详细介绍logging模块中的LogRecord类.
p: /python/logging源码解析之LogRecord
date: 2018-01-13 14:11:45
updated: 2018-01-13 14:11:45
---
### 一、引言
在[Introduction](index.md)这篇文章中，我们已经简单介绍了`logging`模块，接下来我们就开始详细分析`logging`模块中的`LogRecord`类。
那这个类是做什么呢？答案就是通过`logging`模块内的`LogRecord`类生成日志输出的记录的。
### 二、`LogRecord`
定义如下：
```python
class LogRecord(object):
```
这是一个以`object`为基类的类。
#### 1. `__init__`
接下来我们来分析`LogRecord`的`__init__`函数:
```python
    def __init__(self, name, level, pathname, lineno, msg, args, exc_info, func=None, sinfo=None, **kwargs):
        ct = time.time() # 获取当前时间
        self.name = name # 日志记录的 name
        self.msg = msg  # 日志记录的 msg 

        if (args and len(args) == 1 and isinstance(args[0], collections.Mapping) and args[0]):
            args = args[0]
        self.args = args # 日志记录 args 参数
        self.levelname = getLevelName(level) # 获取设置日志日录level对应的文本
        self.levelno = level # 设置日志级别
        self.pathname = pathname # 设置日志路径
        try:
            self.filename = os.path.basename(pathname)
            self.module = os.path.splitext(self.filename)[0]
        except (TypeError, ValueError, AttributeError):
            self.filename = pathname
            self.module = "Unknown module"
        self.exc_info = exc_info
        self.exc_text = None  # used to cache the traceback text
        self.stack_info = sinfo
        self.lineno = lineno
        self.funcName = func 
        self.created = ct # 获取日志记录的产生时间
        self.msecs = (ct - int(ct)) * 1000
        self.relativeCreated = (self.created - _startTime) * 1000 # 获取日志记录的相对时间
        if logThreads and threading:
            self.thread = threading.get_ident()
            self.threadName = threading.current_thread().name
        else:  # pragma: no cover
            self.thread = None
            self.threadName = None
        if not logMultiprocessing:  # pragma: no cover
            self.processName = None
        else:
            self.processName = 'MainProcess'
            mp = sys.modules.get('multiprocessing')
            if mp is not None:
                # Errors may occur if multiprocessing has not finished loading
                # yet - e.g. if a custom import hook causes third-party code
                # to run when multiprocessing calls import. See issue 8200
                # for an example
                try:
                    self.processName = mp.current_process().name
                except Exception:  # pragma: no cover
                    pass
        if logProcesses and hasattr(os, 'getpid'):
            self.process = os.getpid()
        else:
            self.process = None
```
可见`__init__`方法就是设置日志记录的各个字段，我们来详细分析以下。

首先，分析`if (args and len(args) == 1 and isinstance(args[0], collections.Mapping) and args[0])`，这个`if`语句中有`4`个判断，前两个判断很好理解，那第三个判断`isinstance(args[0], collections.Mapping)`是什么意思呢？如下：
    ```python
    In [6]: amap = dict(a=1)
    
    In [7]: import collections
    
    In [8]: isinstance(amap, collections.Mapping)
    Out[8]: True
    ```
由此可见，`dict`对象也是一个`collections.Mapping`对象。
那第4个判断语句是什么意思呢？这是为了确认`collections.Mapping`对象不为空或者不能为`0`值。
由此可见`args`参数肯定是一个列表或者元组。

然后，我们先看个例子：
```python
def test():
    logging.error('test log %(a)s', {'a': 'error'})

if __name__ == '__main__':
    import logging

    logging.basicConfig(level=logging.WARNING, format='%(asctime)s - %(filename)s[line:%(lineno)d] - %(levelname)s: %(message)s')

    test()
    
 2019-07-09 22:30:16,451 - glogging.py[line:461] - ERROR: test log error
```
上面的例子就是正常的执行结果，那么这个日志记录的`logrecord`示例中的字段是什么样子呢?
```python
{'args': {'a': 'error'},
 'created': 1562724688.263686,
 'exc_info': None,
 'exc_text': None,
 'filename': 'glogging.py',
 'funcName': 'test',
 'levelname': 'ERROR',
 'levelno': 40,
 'lineno': 455,
 'module': 'glogging',
 'msecs': 263.685941696167,
 'msg': 'test log %(a)s',
 'name': 'root',
 'pathname': '/Users/yajingliu/Projects/PycharmProjects/gunicorn/gunicorn/glogging.py',
 'process': 18111,
 'processName': 'MainProcess',
 'relativeCreated': 0.5970001220703125,
 'stack_info': None,
 'thread': 140736003502976,
 'threadName': 'MainThread'}
```
而传入`__init__`的参数值是下面这个样子的:
```python
name= root
level= 40
pathname= /Users/yajingliu/Projects/PycharmProjects/gunicorn/gunicorn/glogging.py
lineno= 455
msg= test log %(a)s
args= ({'a': 'error'},)
exc_info= None
func= test
extra= None
sinfo= None
```
由此可知，`__init__`方法中各个参数的含义:
- `name`: 参数`name`是一个`Logger`对象的`name`属性值；
- `level`: 参数`level`是日志的级别，这里传入的是`int`形式的；
- `pathname`: 参数`pathname`就是生成日志的文件路径；
- `lineno`: 参数`lineno`就是生成日志记录的语句所在文件中的行；
- `msg`: 参数`msg`就是提交日志的 message 信息；
- `args`: 参数`args`是一个元组，是`msg`中的格式化所需的值；
- `exc_info`:
- `func`: 参数`func`就是生成日志记录的语句所在的函数名；
- `extra`:
- `sinfo`: 参数`sinfo`就是调用堆栈信息。

综上，我们可以知道`LgRecord`类实例中的属性值都是根据传入的参数生成的。
#### 2. `__str__`
```python
    def __str__(self):
        return '<LogRecord: %s, %s, %s, %s, "%s">' % (self.name, self.levelno, self.pathname, self.lineno, self.msg)
        
    __repr__ = __str__                                      
```
#### 3. `getMessage`
```python
    def getMessage(self):
        msg = str(self.msg)
        if self.args:
            msg = msg % self.args
        return msg
```
这个方法是将任何用户提供的`args`与`msg`合并后，并返回此`LogRecord`的消息。
综上所述，可知：
- `LogRecord`实例表示正在记录的事件。
- 每次记录某些内容时都会创建LogRecord实例，它们包含与记录事件相关的所有信息。 
- 传入的主要信息是`msg`和`args`，它们使用`str(msg)％args`组合以创建记录的消息字段。 
- 该记录还包括诸如创建记录的时间，进行日志记录调用的源代码行以及要记录的任何异常信息之类的信息。

以上就是关于类`LogRecord`的定义及方法的说明了。
### 三、 `_logRecordFactory`
默认情况下我们使用类`LogRecord`来生成日志记录，但我们也可以通过设置使用我们自定义的方法来生成日志记录，如下：
```python
#   确定实例化日志记录时要使用的类。
_logRecordFactory = LogRecord

# 设置在实例化日志记录时使用的factory。参数 factory 一个可调用的函数，用于实例化日志记录, 默认是LogRecord
def setLogRecordFactory(factory):
    global _logRecordFactory
    _logRecordFactory = factory

# 返回在实例化日志记录时要使用的factory。
def getLogRecordFactory():
    return _logRecordFactory

# 创建一个LogRecord对象，其属性由指定的字典定义。
# 此函数可以用来将通过套接字连接中(作为 dict 发送)接收的日志记录事件转换为 LogRecord 实例。
def makeLogRecord(dict):
    rv = _logRecordFactory(None, None, "", 0, "", (), None, None)
    rv.__dict__.update(dict)
    return rv
```
可以看出这几个函数为我们提供了设置、获取日志记录处理的 factory，默认是`LogRecord`类对象，我们也可以自定义。

### 四、总结
这个类很简单，实现的就是根据所传参数生成日记记录相关的属性，主要在`Logger`类中调用，我们后续会有相关说明。