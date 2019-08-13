---
title: logging源码解析之RootLogger
urlname: logging/RootLogger
categories: python
tags:
  - python
description: 在本篇文章中，我们介绍一个name为root的Logger实例，其是一个全局变量，调用模块方法记录日志默认都是用这个logger处理的.
p: /python/logging源码解析之RootLogger
date: 2018-02-24 15:54:42
updated: 2018-02-24 15:54:42
---
### 一、引言
通过前面几篇文章的分析，我们已经熟悉了`logging`模块的几个重要概念，今天我们就重点介绍一个特殊的`logger`，即`root`，它是`RootLogger`类的一个实例，今天我们就围绕这个特殊的`looger`进行讨论。
### 二、`RootLogger`
`RootLogger`与其他`Logger`并没有什么不同，除了它必须要有日志记录级别，并且层次结构中只有一个实例。
```python
class RootLogger(Logger):
    def __init__(self, level):
        Logger.__init__(self, "root", level)
```
`RootLogger`类是以`root`为`name`的`logger`。
还定义了一个全局变量`root`：
```python
root = RootLogger(WARNING)
Logger.root = root
Logger.manager = Manager(Logger.root)
```
这个`root`是一个以`level=WARNING`初始化的`RootLogger`实例，且将这个`logger`赋值给`Logger`类的`root`属性，又将以`Logger.root`初始化的`Manager`类赋值给`Logger`类的`manager`属性。

> 综上可知，任何`logger`的根节点也就是最上层的节点都是`root`，除非自己改动`Logger.root`的值以及`Logger.manager`的值。

### 四、`basicConfig`
```python
def basicConfig(**kwargs):
    # 添加线程安全性以防有人错误地从多个线程调用basicConfig()
    _acquireLock()
    try:
        if len(root.handlers) == 0:
            handlers = kwargs.pop("handlers", None)
            if handlers is None:
                if "stream" in kwargs and "filename" in kwargs:
                    raise ValueError("'stream' and 'filename' should not be specified together")
            else:
                if "stream" in kwargs or "filename" in kwargs:
                    raise ValueError("'stream' or 'filename' should not be specified together with 'handlers'")
            if handlers is None:
                filename = kwargs.pop("filename", None)
                mode = kwargs.pop("filemode", 'a')
                if filename:
                    h = FileHandler(filename, mode)
                else:
                    stream = kwargs.pop("stream", None)
                    h = StreamHandler(stream)
                handlers = [h]
            dfs = kwargs.pop("datefmt", None)
            style = kwargs.pop("style", '%')
            if style not in _STYLES:
                raise ValueError('Style must be one of: %s' % ','.join(
                    _STYLES.keys()))
            fs = kwargs.pop("format", _STYLES[style][1])
            fmt = Formatter(fs, dfs, style)
            for h in handlers:
                if h.formatter is None:
                    h.setFormatter(fmt)
                root.addHandler(h)
            level = kwargs.pop("level", None)
            if level is not None:
                root.setLevel(level)
            if kwargs:
                keys = ', '.join(kwargs.keys())
                raise ValueError('Unrecognised argument(s): %s' % keys)
    finally:
        _releaseLock()
```
`basicConfig`函数可以设置日志记录系统的基本配置。
> 如果`root logger`已配置`handler`，则此函数不执行任何操作。 这是一种便宜的方法，旨在通过简单的脚本来执行`logging`模块的一次性配置，默认创建一个写入`sys.stderr`，且`format=BASIC_FORMAT`的`StreamHandler`，并将此`handler`添加到`name=root`的`logger`中。

这个函数可以指定许多可选的关键字参数，可以改变默认行为，可选参数如下：
- filename:  使用指定的文件名r创建FileHandler，而不是StreamHandle。
- filemode:  指定打开文件的模式，如果指定了filename（如果未指定filemode，则默认为'a'）。
- format:    指定`handler`使用的格式字符串。
- datefmt:   指定使用的日期/时间格式。
- style:     如果指定了格式字符串，则使用此字符串指定格式字符串的类型(可能的值'％'，'{'，'$'，对于％-formatting，:meth：`str.format`和:class：`string .Template`-默认为'％')。
- level:     将`root logger`级别设置为指定的级别。
- stream:    使用指定的流初始化StreamHandler。 请注意，此参数与'filename'不兼容 - 如果两者都存在，则忽略'stream'。
- handlers: 如果指定，这应该是已经创建的可迭代的`handler`，它将被添加到`root logger`中。 `handler`列表中如果有`handler`没有分配`formatter`，那将使用在此函数中设置的的`formatter`。

#### 1. 记录日志到文件
一种非常常见的情况是将日志事件记录到文件，请确认启动新的Python 解释器，不要在上一个环境中继续操作:
```python
import logging
logging.basicConfig(filename='example.log',level=logging.DEBUG)
logging.debug('This message should go to the log file')
logging.info('So should this')
logging.warning('And this, too')
```
现在，如果我们打开日志文件，我们应当能看到日志信息：
```python
DEBUG:root:This message should go to the log file
INFO:root:So should this
WARNING:root:And this, too
```
该示例同样展示了如何设置日志追踪级别的阈值。该示例中，由于我们设置的阈值是 `DEBUG`，所有信息全部打印。

如果你想从命令行设置日志级别，例如：
```python
--log=INFO
```
并且在一些`loglevel`变量中你可以获得 `--log`命令的参数，你可以使用：
```python
getattr(logging, loglevel.upper())
```
通过`level`参数获得你将传递给`basicConfig()`的值。你需要对用户输入数据进行错误排查，可如下例：
```python
# assuming loglevel is bound to the string value obtained from the
# command line argument. Convert to upper case to allow the user to
# specify --log=DEBUG or --log=debug
numeric_level = getattr(logging, loglevel.upper(), None)
if not isinstance(numeric_level, int):
    raise ValueError('Invalid log level: %s' % loglevel)
logging.basicConfig(level=numeric_level, ...)
```
对`basicConfig()`的调用应该在调用`debug()`，`info()`等的前面。因为它被设计为**一次性**的配置，只有第一次调用会进行操作，随后的调用不会产生有效操作。

如果多次运行上述脚本，则连续运行的消息将追加到文件`example.log`。 如果你希望每次运行重新开始，而不是记住先前运行的消息，则可以通过将上例中的调用更改为来指定`filemode`参数:
```python
logging.basicConfig(filename='example.log', filemode='w', level=logging.DEBUG)
```
输出将与之前相同，但不再追加进日志文件，因此早期运行的消息将丢失。
#### 2. 更改显示消息的格式
要更改用于显示消息的格式，你需要指定要使用的`format`:
```python
import logging
logging.basicConfig(format='%(levelname)s:%(message)s', level=logging.DEBUG)
logging.debug('This message should appear on the console')
logging.info('So should this')
logging.warning('And this, too')
```
这将输出：
```python
DEBUG:This message should appear on the console
INFO:So should this
WARNING:And this, too
```
对于可以出现在格式字符串中的全部内容，你可以参考以下文档`{% post_link python/logging源码解析之LogRecord LogRecord %}` ，但为了简单使用，你只需要`levelname` (严重性)，`message`(事件描述，包括可变数据)。
#### 3. 在消息中显示日期/时间
要显示事件的日期和时间，你可以在格式字符串中放置 '%(asctime)s'
```python
import logging
logging.basicConfig(format='%(asctime)s %(message)s')
logging.warning('is when this event was logged.')
```
默认打印这样的格式：
```python
2018-02-24 11:41:42,612 is when this event was logged.
```
日期/时间显示的默认格式（如上所示）类似于`ISO8601`或`RFC 3339`。 如果你需要更多地控制日期/时间的格式，请为`basicConfig`提供`datefmt`参数，如下例所示:
```python
import logging
logging.basicConfig(format='%(asctime)s %(message)s', datefmt='%m/%d/%Y %I:%M:%S %p')
logging.warning('is when this event was logged.')
```
这会显示如下内容：
```python
02/24/2018 11:46:36 AM is when this event was logged.
```
`datefmt`参数的格式与`time.strftime()`支持的格式相同。

### 五、模块级函数
#### 1. `getLogger`
```python
def getLogger(name=None):
    if name:
        return Logger.manager.getLogger(name)
    else:
        return root
```
`getLogger`返回具有指定名称的`logger`，必要时创建它。 如果未指定名称，则返回`root logger`。

多次调用`logging.getLogger('someLogger')`时会返回对同一个`logger`对象的引用。 这不仅是在同一个模块中，在其他模块调用也是如此，只要是在同一个Python解释器进程中。 是应该引用同一个对象，此外，应用程序可以在一个模块中定义和配置父logger，而在另外单独的模块中创建（但不配置）子logger，对子logger的所有调用都将传给父logger，实例如下:
主模块:
```python
import logging
import auxiliary_module

# 用'spam_application'创建logger
logger = logging.getLogger('spam_application')
logger.setLevel(logging.DEBUG)
# 创建filehandler，level为DEBUG
fh = logging.FileHandler('spam.log')
fh.setLevel(logging.DEBUG)
# 创建streamhandler，level为ERROR
ch = logging.StreamHandler()
ch.setLevel(logging.ERROR)
# 创建formatter并将formatter添加到handler中
formatter = logging.Formatter('%(asctime)s - %(name)s - %(levelname)s - %(message)s')
fh.setFormatter(formatter)
ch.setFormatter(formatter)
# 向logger中添加handler
logger.addHandler(fh)
logger.addHandler(ch)

logger.info('creating an instance of auxiliary_module.Auxiliary')
a = auxiliary_module.Auxiliary()
logger.info('created an instance of auxiliary_module.Auxiliary')
logger.info('calling auxiliary_module.Auxiliary.do_something')
a.do_something()
logger.info('finished auxiliary_module.Auxiliary.do_something')
logger.info('calling auxiliary_module.some_function()')
auxiliary_module.some_function()
logger.info('done with auxiliary_module.some_function()')
```
辅助模块`auxiliary_module`:
```python
import logging

# create logger
module_logger = logging.getLogger('spam_application.auxiliary')

class Auxiliary:
    def __init__(self):
        self.logger = logging.getLogger('spam_application.auxiliary.Auxiliary')
        self.logger.info('creating an instance of Auxiliary')

    def do_something(self):
        self.logger.info('doing something')
        a = 1 + 1
        self.logger.info('done doing something')

def some_function():
    module_logger.info('received a call to "some_function"')
```
输出结果会像这样:
```python
2018-02-24 23:47:11,663 - spam_application - INFO - creating an instance of auxiliary_module.Auxiliary
2018-02-24 23:47:11,665 - spam_application.auxiliary.Auxiliary - INFO - creating an instance of Auxiliary
2018-02-24 23:47:11,665 - spam_application - INFO - created an instance of auxiliary_module.Auxiliary
2018-02-24 23:47:11,668 - spam_application - INFO - calling auxiliary_module.Auxiliary.do_something
2018-02-24 23:47:11,668 - spam_application.auxiliary.Auxiliary - INFO - doing something
2018-02-24 23:47:11,669 - spam_application.auxiliary.Auxiliary - INFO - done doing something
2018-02-24 23:47:11,670 - spam_application - INFO - finished auxiliary_module.Auxiliary.do_something
2018-02-24 23:47:11,671 - spam_application - INFO - calling auxiliary_module.some_function()
2018-02-24 23:47:11,672 - spam_application.auxiliary - INFO - received a call to 'some_function'
2018-02-24 23:47:11,673 - spam_application - INFO - done with auxiliary_module.some_function()
```
#### 2.     `log`
```python
def critical(msg, *args, **kwargs):
    if len(root.handlers) == 0:
        basicConfig()
    root.critical(msg, *args, **kwargs)


fatal = critical


def error(msg, *args, **kwargs):
    if len(root.handlers) == 0:
        basicConfig()
    root.error(msg, *args, **kwargs)


def exception(msg, *args, exc_info=True, **kwargs):
    error(msg, *args, exc_info=exc_info, **kwargs)


def warning(msg, *args, **kwargs):
    if len(root.handlers) == 0:
        basicConfig()
    root.warning(msg, *args, **kwargs)


def warn(msg, *args, **kwargs):
    warnings.warn("The 'warn' function is deprecated, use 'warning' instead", DeprecationWarning, 2)
    warning(msg, *args, **kwargs)


def info(msg, *args, **kwargs):
    if len(root.handlers) == 0:
        basicConfig()
    root.info(msg, *args, **kwargs)


def debug(msg, *args, **kwargs):
    if len(root.handlers) == 0:
        basicConfig()
    root.debug(msg, *args, **kwargs)


def log(level, msg, *args, **kwargs):
    if len(root.handlers) == 0:
        basicConfig()
    root.log(level, msg, *args, **kwargs)
```
这些方法的逻辑大致相同，我就放在一起讨论了，都是先判断`root`的`handler`列表是否为空，如果为空，则调用`basicConfig`函数，以设置日志系统的默认配置，然后调用`logger`对应的输出日志的方法。这些函数里的`logger`是`root`，这些函数都是模块级，也就是我们直接调用`logging.info()`就可以写日志了，默认输出到`sys.stderr`，格式为`%(levelname)s:%(name)s:%(message)s`。

由于在handler执行`handle`方法中会获取线程锁，所以`logging`是线程安全的，从而在多个线程中记录日志并不需要特殊处理，以下示例展示了如何在主线程(起始线程)和其他线程中记录:
```python
import logging
import threading
import time

def worker(arg):
    while not arg['stop']:
        logging.debug('Hi from myfunc')
        time.sleep(0.5)

def main():
    logging.basicConfig(level=logging.DEBUG, format='%(relativeCreated)6d %(threadName)s %(message)s')
    info = {'stop': False}
    thread = threading.Thread(target=worker, args=(info,))
    thread.start()
    while True:
        try:
            logging.debug('Hello from main')
            time.sleep(0.75)
        except KeyboardInterrupt:
            info['stop'] = True
            break
    thread.join()

if __name__ == '__main__':
    main()
```
运行结果会像如下这样:
```python
   0 Thread-1 Hi from myfunc
   3 MainThread Hello from main
 505 Thread-1 Hi from myfunc
 755 MainThread Hello from main
1007 Thread-1 Hi from myfunc
1507 MainThread Hello from main
1508 Thread-1 Hi from myfunc
2010 Thread-1 Hi from myfunc
2258 MainThread Hello from main
2512 Thread-1 Hi from myfunc
3009 MainThread Hello from main
3013 Thread-1 Hi from myfunc
3515 Thread-1 Hi from myfunc
3761 MainThread Hello from main
4017 Thread-1 Hi from myfunc
4513 MainThread Hello from main
4518 Thread-1 Hi from myfunc
```
这表明不同线程的日志像期望的那样穿插输出，当然更多的线程也会像这样输出。

#### 3. `disable`
```python
def disable(level):
    root.manager.disable = level
```
这个函数用来设置`root.manager.disable`为指定的`level`值，从而禁止低于此`level`的日志输出。
#### 4. `shutdown`
```python
def shutdown(handlerList=_handlerList):
    for wr in reversed(handlerList[:]):
        # 可能会发生错误，例如，如果文件被锁定，我们只是在没有设置raiseExceptions时忽略它们
        try:
            h = wr()
            if h:
                try:
                    h.acquire()
                    h.flush()
                    h.close()
                except (OSError, ValueError):
                    # 忽略可能因handler已关闭而导致的错误，但在应用程序退出时仍然会引用它们。
                    pass
                finally:
                    h.release()
        except:  # ignore everything, as we're shutting down
            if raiseExceptions:
                raise
                # else, swallow
```
`shutdown`在日志记录系统中执行清理操作(例如，刷新缓冲区)，应该在程序退出时调用。
在`logging`模块中还有以下程序：
```python
import atexit
atexit.register(shutdown)
```
上面语句实现了在程序退出前自动执行`shutdown`函数。
### 六、`_warnings_showwarning`
在`logging`模块中定义了一个全局变量`_warnings_showwarning`，如下：
```python
_warnings_showwarning = None
```
关于这个全局变量还定义了两个函数：
#### 1. `_showwarning`
```python
def _showwarning(message, category, filename, lineno, file=None, line=None):
    if file is not None:
        if _warnings_showwarning is not None:
            _warnings_showwarning(message, category, filename, lineno, file, line)
    else:
        s = warnings.formatwarning(message, category, filename, lineno, line)
        logger = getLogger("py.warnings")
        if not logger.handlers:
            logger.addHandler(NullHandler())
        logger.warning("%s", s)
```
`_showwarning`函数首先检查文件参数是否为None，然后调用`_warnings_showwarning`函数。 如果指定了文件，它将调用warnings.formatwarning并将结果字符串记录到名为“py.warnings”的`logger`中，且级别是`logging.WARNING`。

> 故`_warnings_showwarning`应该是一个可迭代对象，且能接收`message, category, filename, lineno, file, line`等参数，以确保能正确执行。

#### 2. `captureWarnings`
```python
def captureWarnings(capture):
    global _warnings_showwarning
    if capture:
        if _warnings_showwarning is None:
            _warnings_showwarning = warnings.showwarning
            warnings.showwarning = _showwarning
    else:
        if _warnings_showwarning is not None:
            warnings.showwarning = _warnings_showwarning
            _warnings_showwarning = None
```
`captureWarnings`中，如果`capture`为`True`，则将所有警告重定向到`logging package`。 如果`capture`为`False`，请确保警告不会重定向到日志记录，而是重定向到其原始目标。
### 七、总结
本篇文章中主要介绍了`name=root`的`logger`，且其是一个全局变量，并将它赋值给`Logger.root`，且将`Manager(Logger.root)`赋值给`Loggr.manager`，最后又定义了一系列记录日志的函数，可以让我们调用模块级的函数就可以记录日志。
