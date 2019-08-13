---
title: logging源码解析之FileConfig
urlname: logging/FileConfig
categories: python
tags:
  - python
  - logging
  - 源码解析
description: 本篇文章主要剖析了fileConfig这个函数加载日志配置文件的内部机制，以及详细说明了配置文件的定义格式.
p: /python/logging源码解析之FileConfig
date: 2018-03-03 14:19:56
updated: 2018-03-03 14:19:56
---
### 一、引言
在logging模块中，可以通过三种方式配置日志记录：
- 使用调用`baseConfig()`函数显式创建`logger`、`handler`和`formatter`。
- 创建日志配置文件并使用`fileConfig()`函数读取它。
- 创建配置信息字典并将其传递给`dictConfig()`函数。
第一种方式我们在`{% post_link python/logging源码解析之RootLogger RootLogger %}`这篇文章中已经详细说明其用法，在本篇文章中我们主要讨论通过文件配置日志记录的实现。

如果对logging日志模块中的几个概念还有不明白的，可以参考之前介绍的几篇文章，这里不再过多涉及。

源码路径: python3.6/logging/config.py
### 二、 文件配置
`fileConfig()`理解的配置文件格式基于`configparser`([官方文档](https://docs.python.org/3/library/configparser.html))。 该文件必须包含名为`[loggers]`，`[handlers]`和`[formatters]`的部分，这些部分按名称标识文件中定义的每种类型的实体，对于每个这样的实体，都有一个单独的部分，用于标识该实体的配置方式:
- 对于`[loggers]`部分中名为`log01`的记录器，相关配置详细信息保存在`[logger_log01]`部分中;
- `[handlers]`部分中名为`hand01`的处理程序将其配置保存在名为`[handler_hand01]`的部分中;
- `[formatters`]部分中名为`form01`的格式化程序将在名为`[formatter_form01]`的部分中指定其配置。 必须在名为`[logger_root]`的部分中指定`root`记录器配置。

> `fileConfig()`API比`dictConfig()`API旧，并且不提供涵盖日志记录的某些方面的功能。 例如，您无法使用`fileConfig()`配置`Filter`对象。 如果需要在日志记录配置中包含`Filter`实例，则需要使用`dictConfig()`。 请注意，以后配置功能增强功能将添加到`dictConfig()`中，因此在方便的时候可以考虑使用`dictConfig()`API来设置日志。

下面给出了文件中这些部分的示例:
```ini
[loggers]
keys=root,log02,log03,log04,log05,log06,log07

[handlers]
keys=hand01,hand02,hand03,hand04,hand05,hand06,hand07,hand08,hand09

[formatters]
keys=form01,form02,form03,form04,form05,form06,form07,form08,form09
```
#### 1. `[loggers]`
`root`记录器必须指定`level`和`handlers`列表，下面给出了根记录器部分的示例：
```ini
[logger_root]
level=NOTSET
handlers=hand01
```
- `level`可以是`DEBUG`，`INFO`，`WARNING`，`ERROR`，`CRITICAL`或`NOTSET`之一。 仅对于`root`记录器，`NOTSET`表示将记录所有消息。 `level`值在`logging`    模块里的命名空间的上下文中是`eval()`。
- `handlers`是以逗号分隔的`handler`名称列表，必须出现在`[handlers]`部分中，并在配置文件中包含相应的部分。

> 对于除`root`记录器之外的记录器，还需要一些其他信息， 以下示例说明了这一点:

```ini
[logger_parser]
level=DEBUG
handlers=hand01
propagate=1
qualname=compiler.parser
```
- `level`和`handlers`信息与`root`记录器中的一样，但如果将非`root`记录程序的`level`指定为`NOTSET`，则系统会在层次结构的较高位置查询`logger`以确定`logger`的有效级别。 
- `propagate`: 设置为`1`表示消息必须从此`logger`传播到`logger`层次结构上方的`handler`，或者`0`表示消息不会传播到层次结构中的`handler`。 
- `qualname`: 是记录器的分层通道名称，也就是说应用程序用于获取`logger`的`name`。

#### 2. `[handler]`
指定`handler`配置的部分由以下示例:
```ini
[handler_hand01]
class=StreamHandler
level=NOTSET
formatter=form01
args=(sys.stdout,)
```
- `class`: 表示`handler`的类(由`logging`模块的命名空间中的`eval()`确定)
- `level`: `logger`的日志级别，`NOTSET`被认为是“记录所有内容”。
- `formatter`: 表示此`handler`的格式化程序的键名称。 如果为空，则使用默认格式化程序`logging._defaultFormatter`。 如果指定了名称，则它必须出现在`[formatters]`部分中，并在配置文件中具有相应的部分。
- `args`: 当在日志包的命名空间的上下文中使用`eval()`时，`args`是`handler`类的构造函数的参数列表， 如果未提供，则默认为`()`。
- `kwargs`: 当在日志包的命名空间的上下文中使用`eval()`时，`kwargs`是`handler`类的构造函数的关键字参数`dict`，如果未提供，则默认为`{}`。

```ini
[handler_hand02]
class=FileHandler
level=DEBUG
formatter=form02
args=('python.log', 'w')

[handler_hand03]
class=handlers.SocketHandler
level=INFO
formatter=form03
args=('localhost', handlers.DEFAULT_TCP_LOGGING_PORT)

[handler_hand04]
class=handlers.DatagramHandler
level=WARN
formatter=form04
args=('localhost', handlers.DEFAULT_UDP_LOGGING_PORT)

[handler_hand05]
class=handlers.SysLogHandler
level=ERROR
formatter=form05
args=(('localhost', handlers.SYSLOG_UDP_PORT), handlers.SysLogHandler.LOG_USER)

[handler_hand06]
class=handlers.NTEventLogHandler
level=CRITICAL
formatter=form06
args=('Python Application', '', 'Application')

[handler_hand07]
class=handlers.SMTPHandler
level=WARN
formatter=form07
args=('localhost', 'from@abc', ['user1@abc', 'user2@xyz'], 'Logger Subject')
kwargs={'timeout': 10.0}

[handler_hand08]
class=handlers.MemoryHandler
level=NOTSET
formatter=form08
target=
args=(10, ERROR)

[handler_hand09]
class=handlers.HTTPHandler
level=NOTSET
formatter=form09
args=('localhost:9022', '/log', 'GET')
kwargs={'secure': True}
```
#### 3. `[formatter]`
指定`formatter`配置的部分以下列为代表:
```ini
[formatter_form01]
format=F1 %(asctime)s %(levelname)s %(message)s
datefmt=
class=logging.Formatter
```
- `format`是整体格式字符串。
- `datefmt`是`strftime()`兼容的日期/时间格式字符串。 如果为空，则包替换的内容几乎等于指定日期格式字符串`％Y-％m-％d％H:％M:％S`。 此格式还指定使用逗号分隔符附加到使用上述格式字符串的结果的毫秒数。 这种格式的示例时间是`2003-01-23 00:29:50,411`。
- `class`: 是可选的。 它指示`formatter`类的名称(点模块和类名)此选项对于实例化`Formatter`子类很有用， `Formatter`的子类可以以扩展或压缩格式呈现异常回溯。

### 三、`fileConfig`
上面我们详细说明了配置文件各个部分的意义及配置格式，而读取配置文件通常调用的就是函数`logging.config.fileConfig()`，今天我们将详细讨论它是如何工作的，源码如下:
```python
def fileConfig(fname, defaults=None, disable_existing_loggers=True):
    import configparser

    if isinstance(fname, configparser.RawConfigParser):
        cp = fname
    else:
        cp = configparser.ConfigParser(defaults)
        if hasattr(fname, 'readline'):
            cp.read_file(fname)
        else:
            cp.read(fname)

    formatters = _create_formatters(cp)

    # 关键部分
    logging._acquireLock()
    try:
        # 清空logging._handlers和logging._handlerList中的数据
        logging._handlers.clear()
        del logging._handlerList[:]
        # handlers将自己添加到logging._handlers
        handlers = _install_handlers(cp, formatters)
        _install_loggers(cp, handlers, disable_existing_loggers)
    finally:
        logging._releaseLock()
```
`fileConfig`函数从`ConfigParser`格式的文件中读取日志记录配置的，我们注意到这个函数中首先调用的`configparser`的方法读取配置文件，然后又调用了`_create_formatters`, `_install_handlers`, `_install_loggers`等几个函数，我们下面将一一分析。

注意一下: `fileConfig`函数中的局部变量`cp`是一个`key/value`结构。

> 应用程序可以多次调用，允许用户从各种预先配置的配置中进行选择(如果开发人员提供了一种机制来提供选择并加载所选择的配置)。

我们先看几个公共函数吧：
```python
def _resolve(name):
    name = name.split('.')
    used = name.pop(0)
    found = __import__(used)
    for n in name:
        used = used + '.' + n
        try:
            found = getattr(found, n)
        except AttributeError:
            __import__(used)
            found = getattr(found, n)
    return found
    
def _strip_spaces(alist):
    return map(lambda x: x.strip(), alist)
```
这两个函数在下面的几个函数中多次被调用，其中`_resolve`函数是将点名称解析为全局对象，示例如下：
```python
In [1]: from logging import config

In [2]: config._resolve("handlers.RotatingFileHandler")
---------------------------------------------------------------------------
ModuleNotFoundError                       Traceback (most recent call last)
<ipython-input-2-764e9cb53c24> in <module>()
----> 1 config._resolve("handlers.RotatingFileHandler")

~/anaconda3/lib/python3.6/logging/config.py in _resolve(name)
     92     name = name.split('.')
     93     used = name.pop(0)
---> 94     found = __import__(used)
     95     for n in name:
     96         used = used + '.' + n

ModuleNotFoundError: No module named 'handlers'

In [3]: config._resolve("logging.handlers.RotatingFileHandler")
Out[3]: logging.handlers.RotatingFileHandler
```
所以从这个例子可以看出`name`参数是个字符串类型，且字符串的值需是合法的python的模块路径(以`.`连接的)。

`_strip_spaces`函数是将`alist`中每个元素的的首尾空白字符去掉，返回的是一个`map`类型，是可迭代对象，示例如下:
```python
In [9]: result = config._strip_spaces([' test ', ' test1', 'test2 '])

In [10]: result
Out[10]: <map at 0x1090c7550>

In [11]: for r in result:
    ...:     print(r)
    ...:     
test
test1
test2
```

#### 1. `_create_formatters`
```python
def _create_formatters(cp):
    flist = cp["formatters"]["keys"]
    if not len(flist):
        return {}
    flist = flist.split(",")
    flist = _strip_spaces(flist)
    formatters = {}
    for form in flist:
        sectname = "formatter_%s" % form
        fs = cp.get(sectname, "format", raw=True, fallback=None)
        dfs = cp.get(sectname, "datefmt", raw=True, fallback=None)
        stl = cp.get(sectname, "style", raw=True, fallback='%')
        c = logging.Formatter
        class_name = cp[sectname].get("class")
        if class_name:
            c = _resolve(class_name)
        f = c(fs, dfs, stl)
        formatters[form] = f
    return formatters
```
> `_create_formatters` 函数就是读取`[formatters]`部分的`keys`字段的`formatter`列表，然后再读取每个`formatter`对应的部分配置并根据其配置创建对应的`formatter`实例，添加到`dict`结构的`formatters`，最后将`formatters`返回。

#### 2. `_install_handlers`
```python
def _install_handlers(cp, formatters):
    hlist = cp["handlers"]["keys"]
    # 如果为空，则直接返回
    if not len(hlist):
        return {}
    # 获取定义的多个handler的列表
    hlist = hlist.split(",")
    hlist = _strip_spaces(hlist)
    handlers = {}
    fixups = []  # 用于处理程序间引用
    # 循环遍历handler的列表
    for hand in hlist:
        # 每个handler的section都是handler_xxxx这种形式
        section = cp["handler_%s" % hand]
        # 从handler_xxxx中读取相应的key值
        klass = section["class"]
        fmt = section.get("formatter", "")
        try:
            klass = eval(klass, vars(logging))
        except (AttributeError, NameError):
            klass = _resolve(klass)
        args = section["args"]
        args = eval(args, vars(logging))
        # 用指定的类实例化handler
        h = klass(*args)
        # 为handler设置日志级别
        if "level" in section:
            level = section["level"]
            h.setLevel(level)
        # 为handler设置formatter
        if len(fmt):
            h.setFormatter(formatters[fmt])
        if issubclass(klass, logging.handlers.MemoryHandler):
            target = section.get("target", "")
            if len(target):  # the target handler may not be loaded yet, so keep for later...
                fixups.append((h, target))
        # 将生成的handler添加到handlers中
        handlers[hand] = h
    # now all handlers are loaded, fixup inter-handler references...
    for h, t in fixups:
        h.setTarget(handlers[t])
    return handlers
```
> `_install_handlers` 函数就是读取`[handlers]`部分的`keys`字段的`handler`列表，然后再读取每个`handler`对应的部分配置并根据其配置创建对应的`handler`实例，添加到`dict`结构的`handlers`，最后将`handlers`返回。

#### 3. `_install_loggers`
```python
def _install_loggers(cp, handlers, disable_existing):
    # 首先配置root
    llist = cp["loggers"]["keys"]
    llist = llist.split(",")
    llist = list(map(lambda x: x.strip(), llist))
    llist.remove("root")
    section = cp["logger_root"]
    root = logging.root
    log = root
    if "level" in section:
        level = section["level"]
        log.setLevel(level)
    for h in root.handlers[:]:
        root.removeHandler(h)
    hlist = section["handlers"]
    if len(hlist):
        hlist = hlist.split(",")
        hlist = _strip_spaces(hlist)
        for hand in hlist:
            log.addHandler(handlers[hand])

    # 现在处理其它的logger，我们不想丢失现有的logger，因为其他线程可能有指向它们的指针。 
    # existing中包含所有现有的logger，当我们加载完成新配置时，就将删除这个列表中的所有记录器
    existing = list(root.manager.loggerDict.keys())
    # 使用排序列表，可以更轻松地找到子记录器。
    existing.sort()
    # 我们将保留现有记录器的所有子节点的记录器列表
    child_loggers = []
    # 设置新的logger
    for log in llist:
        # 除了root，在文件中定义logger的section都是以logger_开头的
        section = cp["logger_%s" % log]
        # 从指定logger对应的section中读取qualname的value值，也就是logger的name
        qn = section["qualname"]
        # 从指定logger对应的section中读取propagate的value值，是一个int类型的，默认值是1
        propagate = section.getint("propagate", fallback=1)
        # 获取name=qn的logger对象，如果不存在，则创建
        logger = logging.getLogger(qn)
        # 如果qn在existing列表中存在
        if qn in existing:
            i = existing.index(qn) + 1  # 从qn之后的条目开始
            prefixed = qn + "."
            pflen = len(prefixed)
            num_existing = len(existing)
            while i < num_existing:
                # 找到其子节点的logger, 并将其加入到child_loggers中
                if existing[i][:pflen] == prefixed:
                    child_loggers.append(existing[i])
                i += 1
            # 将qn从existing列表中删除
            existing.remove(qn)
        # 为logger设置日志级别
        if "level" in section:
            level = section["level"]
            logger.setLevel(level)
        # 将这个logger中的handler清空
        for h in logger.handlers[:]:
            logger.removeHandler(h)
        # 为logger设置propagate和disabled属性
        logger.propagate = propagate
        logger.disabled = 0
        # 从指定logger对应的section中读取handlers的value值，可以指定多个，以','分隔
        # 并将这多个handler添加到logger中
        hlist = section["handlers"]
        if len(hlist):
            hlist = hlist.split(",")
            hlist = _strip_spaces(hlist)
            for hand in hlist:
                logger.addHandler(handlers[hand])

    # 禁用任何旧记录器。 没有必要删除它们，因为其他线程可能会继续保留引用，通过禁用它们，可以阻止它们执行任何日志记录。 
    # 但是，不能禁用列表child_loggers中的记录器，因为它们是我们新生成的这些logger的子节点 
    _handle_existing_loggers(existing, child_loggers, disable_existing)
```
> `_install_loggers` 函数就是读取`[loggers]`部分的`keys`字段的`logger`列表，首先读取`[logger_root]`部分创建`root`记录器，然后再读取其余的每个`logger`对应的部分配置并根据其配置分别创建对应的`logger`实例。

函数`_handle_existing_loggers`定义如下:
```python
def _handle_existing_loggers(existing, child_loggers, disable_existing):
    root = logging.root
    for log in existing:
        logger = root.manager.loggerDict[log]
        # 如果logger在child_loggers存在，就将其propagate设置为True，意味着让其父节点处理其日志记录
        if log in child_loggers:
            logger.level = logging.NOTSET
            logger.handlers = []
            logger.propagate = True
        else:
            # 否则就禁用这个logger    
            logger.disabled = disable_existing
```
### 四、总结
在这篇文章中我们详细说明了配置文件的定义格式，也剖析了`fileConfig`这个函数加载日志配置文件的内部机制，可以看出，其格式还是围绕`Formatter`, `Handler`, `Logger`这三个概念进行读取并加载的，但是并不支持`Filter`对象的设置。

