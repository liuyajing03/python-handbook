---
title: logging源码解析之Logger
urlname: logging/Logger
categories: python
tags:
  - python
  - logging
  - 源码解析
description: 本篇文章介绍logging模块中的Logger类，以及如何利用之前介绍的LogRecord、Filter、Formatter和Handler来处理用户定义的日志.
p: /python/logging源码解析之Logger
date: 2018-02-17 10:42:33
updated: 2018-02-17 10:42:33
---
### 一、引言
在[模块对象Manager](logging/Manager.md)这篇文章中，我们介绍了*Manager*类是如何管理*Logger*对象的层次结构，那今天介绍`logging`模块中的重要对象*Logger*就轻松多了。

### 二、Logger
#### 1. 定义
```python
class Logger(Filterer):
```
`Logger`类是继承自`Filterer`类的。
#### 2. `__init__`
```python
    def __init__(self, name, level=NOTSET):
        Filterer.__init__(self)
        self.name = name
        self.level = _checkLevel(level)
        self.parent = None
        self.propagate = True
        self.handlers = []
        self.disabled = False
```
`__init__`方法接受参数`name `和`level`来初始化`logger`实例，其中`level`参数默认是`NOTSET`；我们还注意到其初始化了一个`handlers`属性值为`[]`。
#### 3. `setLevel`
```python
    def setLevel(self, level):
        self.level = _checkLevel(level)
```
这个方法通过调用`_checkLevel`函数来设置日志的级别，参数`level`必须是一个`int`类型或者`str`类型。
#### 4. `getEffectiveLevel`
```python
    def getEffectiveLevel(self):
        logger = self
        while logger:
            if logger.level:
                return logger.level
            logger = logger.parent
        return NOTSET
```
`getEffectiveLevel`方法获取此`logger`的有效级别。向上循环遍历`logger`层次结构，查找其上方节点的日志级别，返回找到的第一个值不为`0`的级别，否则返回`0`。
#### 5. `isEnabledFor`
```python
    def isEnabledFor(self, level):
        if self.manager.disable >= level:
            return False
        return level >= self.getEffectiveLevel()
```
`isEnabledFor`方法用来判断给定的级别`level`在此`logger`中是否有效。
#### 6. `addHandler`
```python
    def addHandler(self, hdlr):
        _acquireLock()
        try:
            if not (hdlr in self.handlers):
                self.handlers.append(hdlr)
        finally:
            _releaseLock()
```
`addHandler`方法是将给定的`handler`对象添加到属性`handlers`列表中。

`addHandler`方法没有限制可以添加的`handler`数量。有时候，应用程序需要将严重类的消息记录在一个文本文件，而将错误类或其他等级的消息输出在控制台中。要进行这样的设定，只需多配置几个日志`handler`即可，在应用程序代码中的日志记录调用可以保持不变。示例如下:
```python
import logging

logger = logging.getLogger('simple_example')
logger.setLevel(logging.DEBUG)
# create file handler which logs even debug messages
fh = logging.FileHandler('spam.log')
fh.setLevel(logging.DEBUG)
# create console handler with a higher log level
ch = logging.StreamHandler()
ch.setLevel(logging.ERROR)
# create formatter and add it to the handlers
formatter = logging.Formatter('%(asctime)s - %(name)s - %(levelname)s - %(message)s')
ch.setFormatter(formatter)
fh.setFormatter(formatter)
# add the handlers to logger
logger.addHandler(ch)
logger.addHandler(fh)

# 'application' code
logger.debug('debug message')
logger.info('info message')
logger.warning('warn message')
logger.error('error message')
logger.critical('critical message')
```
需要注意的是，'application' 代码并不关心是否有多个日志处理器。示例中所做的只是添加和配置了两个日志处理器，并且运行后日志既可以在控制台输出，也会在日志文件输出。
当运行后，你会看到控制台如下所示：
```python
2018-02-17 21:34:05,693 - simple_example - ERROR - error message
2018-02-17 21:34:05,694 - simple_example - CRITICAL - critical message
```
而在文件中会看到像这样:
```python
2018-02-17 21:34:05,692 - simple_example - DEBUG - debug message
2018-02-17 21:34:05,693 - simple_example - INFO - info message
2018-02-17 21:34:05,693 - simple_example - WARNING - warn message
2018-02-17 21:34:05,693 - simple_example - ERROR - error message
2018-02-17 21:34:05,694 - simple_example - CRITICAL - critical message
```
当然也可以给不同的`handler`设置不同的`formatter`，这样两份日志的格式就不一样了。

> 在编写和测试应用程序时，创建能过滤不同等级消息的日志处理器是很有用的。不要使用 print 去调试，而是使用 `logger.debug`: 它不像打印语句需要在调试结束后注释或删除掉，你可以把它们保留在源码中并不输出。当需要再次调试时，只需要改变日志记录器或处理器的过滤等级即可。

#### 7. `removeHandler`
```python
    def removeHandler(self, hdlr):
        _acquireLock()
        try:
            if hdlr in self.handlers:
                self.handlers.remove(hdlr)
        finally:
            _releaseLock()
```
`removeHandler`方法是将给定的`handler`对象从属性`handlers`列表中删除。
#### 8. `hasHandlers`
```python
    def hasHandlers(self):
        c = self
        rv = False
        while c:
            if c.handlers:
                rv = True
                break
            if not c.propagate:
                break
            else:
                c = c.parent
        return rv
```
`hasHandlers`方法用来查看此`logger`是否配置了`handler`。
> 在`logger`层次结构中，循环遍历此`logger`及其父节点的所有`handlers`属性。 如果找到属性`handlers`的值不是空列表，则返回`True`，否则返回`False`。 每当找到“propagate”属性设置为零(不向上传播)的`logger`时，就停止搜索层次结构——这是检查是否存在`handler`的最后一个`logger`。
#### 9. `callHandlers`
```python
    def callHandlers(self, record):
        c = self
        found = 0
        while c:
            for hdlr in c.handlers:
                found = found + 1
                if record.levelno >= hdlr.level:
                    hdlr.handle(record)
            if not c.propagate:
                c = None  # break out
            else:
                c = c.parent
        if (found == 0):
            if lastResort:
                if record.levelno >= lastResort.level:
                    lastResort.handle(record)
            elif raiseExceptions and not self.manager.emittedNoHandlerWarning:
                sys.stderr.write("No handlers could be found for logger \"%s\"\n" % self.name)
                self.manager.emittedNoHandlerWarning = True
```
`callHandlers`方法将`LogRecord`日志记录实例传递给所有相关的`handler`进行处理，在[Handler](logging/Handler.md)这篇文章中我们可以知道，`handler`处理日志记录就是将格式化后的字符串输出到流中。
> 在`logger`层次结构中循环遍历此`logger`及其父项的所有`handler`， 如果未找到`handler`，则向`sys.stderr`输出一次性错误消息。 每当找到“propagate”属性设置为零的`logger`时，就停止搜索层次结构——这将是调用`handler`的最后一个`logger`。

#### 10. `handle`
```python
    def handle(self, record):
        if (not self.disabled) and self.filter(record):
            self.callHandlers(record)
```
`handle`方法为给定的`record`调用`callHandlers`方法，前提是`logger`的`disabled`属性为`False`且`record`可以通过`logger`的所有过滤器。
此方法中的`record`可是是从套接字接收的unpickled记录，也可以是本地创建的记录。
#### 11. `makeRecord`
```python
    def makeRecord(self, name, level, fn, lno, msg, args, exc_info, func=None, extra=None, sinfo=None):
        rv = _logRecordFactory(name, level, fn, lno, msg, args, exc_info, func, sinfo)
        if extra is not None:
            for key in extra:
                if (key in ["message", "asctime"]) or (key in rv.__dict__):
                    raise KeyError("Attempt to overwrite %r in LogRecord" % key)
                rv.__dict__[key] = extra[key]
        return rv
```
`makeRecord`是一个工厂方法，可以在子类中重写以创建专用的`LogRecord`实例。
#### 12. `findCaller`
```python
    def findCaller(self, stack_info=False):
        f = currentframe()
        # On some versions of IronPython, currentframe() returns None if
        # IronPython isn't run with -X:Frames.
        if f is not None:
            f = f.f_back
        rv = "(unknown file)", 0, "(unknown function)", None
        while hasattr(f, "f_code"):
            co = f.f_code
            filename = os.path.normcase(co.co_filename)
            if filename == _srcfile:
                f = f.f_back
                continue
            sinfo = None
            if stack_info:
                sio = io.StringIO()
                sio.write('Stack (most recent call last):\n')
                traceback.print_stack(f, file=sio)
                sinfo = sio.getvalue()
                if sinfo[-1] == '\n':
                    sinfo = sinfo[:-1]
                sio.close()
            rv = (co.co_filename, f.f_lineno, co.co_name, sinfo)
            break
        return rv
```
`findCaller`方法实现的是找到调用者的堆栈帧，以便我们可以记下源文件名，行号和函数名。

这个方法中调用的`currentframe`函数我们在`{% post_link python/logging源码解析之Introduction Introduction %}`这篇文章中有介绍。

> 现在我们围绕这个方法分析下它的内部机制:
> 1. 在方法的开头调用了`currentframe`函数，并将其执行结果赋值给`f`，那么`f`就是在堆栈调用中`depth`为`3`的一个`frame`对象;
> 2. 如果`f`不为`None`，则获取它的上一个堆栈调用`frame`对象，也就是在堆栈调用中`depth`为`4`;
> 3. 进入`while`循环，如果`f`的源文件等于全局变量`_srcfile`的值，则继续获取它的上一个堆栈调用`frame`对象，也就是在堆栈调用中`depth`为`5`，并进入下一轮循环，直到其`f`源文件不等于全局变量`_srcfile`的值;
> 4. 此时如果`stack_info==True`，则获取其堆栈调用信息;
> 5. 然后获取`f`的源文件、代码行、函数名和堆栈信息一起返回。

#### 13. `getChild`
```python
    def getChild(self, suffix):
        if self.root is not self:
            suffix = '.'.join((self.name, suffix))
        return self.manager.getLogger(suffix)
```
`getChild`方法用来获取一个`logger`，它是此`logger`也就是`self`的子节点。

> 这是一种方便的方法，例如`logging.getLogger('abc').getChild('def.ghi')`与 `logging.getLogger('abc.def.ghi')`是一样的。

#### 14. `_log`
```python
    def _log(self, level, msg, args, exc_info=None, extra=None, stack_info=False):
        sinfo = None
        if _srcfile:
            # IronPython不跟踪Python框架，因此findCaller在某些版本的IronPython上引发了异常。 我们在这里将它捕获，以便IronPython可以使用日志记录。
            try:
                # 从堆栈调用信息中获取源文件名、行号、函数名、堆栈信息等
                fn, lno, func, sinfo = self.findCaller(stack_info)
            except ValueError:  # pragma: no cover
                fn, lno, func = "(unknown file)", 0, "(unknown function)"
        else:  # pragma: no cover
            fn, lno, func = "(unknown file)", 0, "(unknown function)"
        if exc_info:
            if isinstance(exc_info, BaseException):
                exc_info = (type(exc_info), exc_info, exc_info.__traceback__)
            elif not isinstance(exc_info, tuple):
                exc_info = sys.exc_info()
       # 创建log record
        record = self.makeRecord(self.name, level, fn, lno, msg, args, exc_info, func, extra, sinfo)
        # 调用所有的handler处理record
        self.handle(record)
```
`_log`方法定义了一个低级`logging`例程，它创建一个`LogRecord`，然后调用此`logger`的所有`handler`来处理该`record`。
#### 15. `log`
```python
    def log(self, level, msg, *args, **kwargs):
        if not isinstance(level, int):
            if raiseExceptions:
                raise TypeError("level must be an integer")
            else:
                return
        if self.isEnabledFor(level):
            self._log(level, msg, args, **kwargs)
```
`log`方法使用`int`类型的`level`级别输出日志`msg %ards`，如果要传递异常信息，请设置`exc_info==True`：
例如: ` logger.log(level, "We have a %s", "mysterious problem", exc_info=1)`
另外，也可以调用`level`对应的方法名，即`logger.debug(msg, *args, **kwargs)`，如下:
```python
    def debug(self, msg, *args, **kwargs):
        if self.isEnabledFor(DEBUG):
            self._log(DEBUG, msg, args, **kwargs)

    def info(self, msg, *args, **kwargs):
        if self.isEnabledFor(INFO):
            self._log(INFO, msg, args, **kwargs)

    def warning(self, msg, *args, **kwargs):
        if self.isEnabledFor(WARNING):
            self._log(WARNING, msg, args, **kwargs)

    def warn(self, msg, *args, **kwargs):
        warnings.warn("The 'warn' method is deprecated, "
                      "use 'warning' instead", DeprecationWarning, 2)
        self.warning(msg, *args, **kwargs)

    def error(self, msg, *args, **kwargs):
        if self.isEnabledFor(ERROR):
            self._log(ERROR, msg, args, **kwargs)

    def exception(self, msg, *args, exc_info=True, **kwargs):
        self.error(msg, *args, exc_info=exc_info, **kwargs)

    def critical(self, msg, *args, **kwargs):
        if self.isEnabledFor(CRITICAL):
            self._log(CRITICAL, msg, args, **kwargs)

    fatal = critical
```
这里就不再多说了。
#### 16. `__repr__`
```python
    def __repr__(self):
        level = getLevelName(self.getEffectiveLevel())
        return '<%s %s (%s)>' % (self.__class__.__name__, self.name, level)
```
最后还定义了一个全局变量:
```python
_loggerClass = Logger
```
关于这个全局变量的的设置和获取在`{% post_link python/logging源码解析之Manager Manager %}`中有介绍，可以参考。
### 三、`LoggerAdapter`
`LoggerAdapter`类这个类设计的像`Logger`，所以可以直接调用`debug()`、`info()`、`warning()`、`error()`、`exception()`、 `critical()` 和 `log()`。 这些方法在对应的`Logger`中使用相同的签名，所以可以交替使用两种类型的实例。，代码如下:
```python
class LoggerAdapter(object):
    def __init__(self, logger, extra):
        self.logger = logger
        self.extra = extra

    def process(self, msg, kwargs):
        kwargs["extra"] = self.extra
        return msg, kwargs

    # Boilerplate convenience methods
    def debug(self, msg, *args, **kwargs):
        self.log(DEBUG, msg, *args, **kwargs)

    def info(self, msg, *args, **kwargs):
        self.log(INFO, msg, *args, **kwargs)

    def warning(self, msg, *args, **kwargs):
        self.log(WARNING, msg, *args, **kwargs)

    def warn(self, msg, *args, **kwargs):
        warnings.warn("The 'warn' method is deprecated, use 'warning' instead", DeprecationWarning, 2)
        self.warning(msg, *args, **kwargs)

    def error(self, msg, *args, **kwargs):
        self.log(ERROR, msg, *args, **kwargs)

    def exception(self, msg, *args, exc_info=True, **kwargs):
        self.log(ERROR, msg, *args, exc_info=exc_info, **kwargs)

    def critical(self, msg, *args, **kwargs):
        self.log(CRITICAL, msg, *args, **kwargs)

    def log(self, level, msg, *args, **kwargs):
        if self.isEnabledFor(level):
            msg, kwargs = self.process(msg, kwargs)
            self.logger.log(level, msg, *args, **kwargs)

    def isEnabledFor(self, level):
        if self.logger.manager.disable >= level:
            return False
        return level >= self.getEffectiveLevel()

    def setLevel(self, level):
        self.logger.setLevel(level)

    def getEffectiveLevel(self):
        return self.logger.getEffectiveLevel()

    def hasHandlers(self):
        return self.logger.hasHandlers()

    def _log(self, level, msg, args, exc_info=None, extra=None, stack_info=False):
        return self.logger._log(
            level,
            msg,
            args,
            exc_info=exc_info,
            extra=extra,
            stack_info=stack_info,
        )

    @property
    def manager(self):
        return self.logger.manager

    @manager.setter
    def manager(self, value):
        self.logger.manager = value

    @property
    def name(self):
        return self.logger.name

    def __repr__(self):
        logger = self.logger
        level = getLevelName(logger.getEffectiveLevel())
        return '<%s %s (%s)>' % (self.__class__.__name__, logger.name, level)
```
> 此类是一个`Logger`的适配器，可以更容易地在`record`输出中指定上下文信息。当你创建一个`LoggerAdapter`的实例时，你会传入一个`Logger`的实例和一个包含了上下文信息的字典对象`extra`。当你调用一个`LoggerAdapter`实例的方法时，它会把调用委托给内部的`Logger`的实例，并为其整理相关的上下文信息。
> 此类中的`process()`方法是将上下文信息添加到日志的输出中，它传入日志消息`msg`和日志调用的关键字参数`kwargs`，并传回(隐式的)这些修改后的内容(把`extra`上下文信息放入`kwargs`中)去调用底层的`logger.log`方法，这样就可以将`extra`里的键值对传入到`LogRecord`实例的`__dict__ `中，从而通过`Formatter`的实例直接使用定制的字符串，实例能找到这个字典类对象的键。

如果你需要一个其他的方法，比如说，想要在消息字符串前后增加上下文信息，你只需要创建一个`LoggerAdapter`的子类，并覆盖它的`process()`方法来做你想做的事情，以下是一个简单的示例:
```python
class CustomAdapter(logging.LoggerAdapter):
    # 此示例适配器期望传入的类似于dict的对象具有“connid”键，其括号中的值将添加到日志消息之前。
    def process(self, msg, kwargs):
        return '[%s] %s' % (self.extra['connid'], msg), kwargs
```
可以这样使用:
```python
logger = logging.getLogger(__name__)
adapter = CustomAdapter(logger, {'connid': some_conn_id})
```
然后，你记录在适配器中的任何事件消息前将添加`some_conn_id`的值。

如果使用除字典之外的其它对象传递上下文信息，你不需要将一个实际的字典传递给`LoggerAdapter`——你可以传入一个实现了`__getitem__` 和`__iter__`的类的实例，这样它就像是一个字典，这对于你想动态生成值（而字典中的值往往是常量）将很有帮助。

### 四、总结
到这里，`logging`模块中的四大概念`Formatter`, `Filter`, `Handler`, `Logger`就分析完了：
- `Formatter`是将`logrecord`按照定义的日志输出格式转换化为字符串形式;
- `Filter`是过滤指定条件的`logrecord`，一个`logger`可以有多个`filter`;
- `Handler`是将格式化后的字符串输出到流中，一个`logger`可以有多个`handler`;
- `Logger`是将这些概念结合起来，从而将日志输出。

通过调用`Logger`类的实例来执行日志记录。 每个实例都有一个名称，它们在概念上以`.`作为分隔符排列在命名空间的层次结构中。 例如，名为 'scan' 的记录器是记录器 'scan.text' ，'scan.html' 和 'scan.pdf' 的父级。 记录器名称可以是你想要的任何名称，并指示记录消息源自的应用程序区域。

在命名记录器时使用的一个好习惯是在每个使用日志记录的模块中使用模块级记录器，命名如下:
```
logger = logging.getLogger(__name__)
```
这意味着记录器名称跟踪包或模块的层次结构，并且直观地从记录器名称显示记录事件的位置。


