---
title: logging源码解析之Filter
urlname: logging/Filter
categories: python
tags:
  - python
  - logging
  - 源码解析
description: 本篇文章主要介绍Filter类，也就是对LogRecord实例(日志记录)提供过滤功能。
p: /python/logging源码解析之Filter
date: 2018-01-27 14:14:39
updated: 2018-01-27 14:14:39
---
### 一、引言
> 在分析`Handler`类和`Logger`类之前，我们首先分析下`Filter`类以及管理`Filter`类的`Filterer`类，因为它们都是继承自`Filterer`的。
`Filter`类是为`LogRecord`实例(日志记录)提供过滤功能的。

### 二、`Filter`
定义如下：
```python
class Filter(object):
```
说明如下：
- 过滤器实例用于执行`LogRecord`的任意过滤。
- `Loggers`和`Handlers`可以选择使用`Filter`实例根据需要过滤日志记录。 
- `Filter`类仅允许在`logger`层次结构中低于某个点的事件。 例如，使用`A.B`初始化的过滤器将允许记录器`A.B`，`A.B.C`，`A.B.C.D`，`A.B.D`等记录的事件，但不能记录`A.BB`，`B.A.B`等。如果使用 空字符串，所有事件都可以通过。

#### 1. `__init__`
```python
def __init__(self, name=''):
        self.name = name
        self.nlen = len(name)
```
`__init__`方法会初始化一个`Filter`对象，使用`logger`的`name`进行初始化，该`logger`及其子项将通过过滤器允许其事件。 如果未指定名称，则允许每个事件。
#### 2. `filter`
```python
def filter(self, record):
        if self.nlen == 0:
            return True
        elif self.name == record.name:
            return True
        elif record.name.find(self.name, 0, self.nlen) != 0:
            return False
        return (record.name[self.nlen] == ".")
```
`filter`方法就是确定是否要记录给定的`record`，示例如下：
```python
def filter_test():
    import logging

    log_record = logging.LogRecord(name='root', level=40,
                                   pathname='/Users/yajingliu/Projects/PycharmProjects/gunicorn/gunicorn/glogging.py',
                                   lineno=455,
                                   msg='test log %(a)s',
                                   args=({'a': 'error'},),
                                   exc_info=None,
                                   func='test',
                                   extra=None,
                                   sinfo=None)
    filter_ = logging.Filter(name='test')
    print(filter_.filter(log_record))
    
if __name__ == '__main__':
    filter_test()
    
Output: False
```
在这个例子中我定义了一个`name`为`test`的`filter`，那我生成的`name`为`root`的`record`就不能通过这个过滤器，但是如果我把`record`中的`name`参数改为`test.root`，就可以通过这个`filter`了。
### 三、`Filterer`
```python
class Filterer(object):
```
这是`Logger`和`Handler`的基类，允许它们共享公共代码。
#### 1. `__init__`
```python
    def __init__(self):
        self.filters = []
```
这个方法将过滤器列表初始化为空列表。
#### 2. `addFilter`
```python
    def addFilter(self, filter):
        if not (filter in self.filters):
            self.filters.append(filter)
```
这个方法将指定的 `filter`添加到过滤器列表中。
#### 3. `removeFilter`
```python
    def removeFilter(self, filter):
        if filter in self.filters:
            self.filters.remove(filter)
```
这个方法将指定的 `filter`从过滤器列表删除。
#### 4. `filter`
```python
    def filter(self, record):
        rv = True
        for f in self.filters:
            if hasattr(f, 'filter'):
                result = f.filter(record)
            else:
                result = f(record)  # assume callable - will raise if not
            if not result:
                rv = False
                break
        return rv
```
这个方法通过查询所有过滤器确定记录是否可记录。
- 默认设置是允许记录记录; 
- 任何过滤器不通过，就不允许该日志记录被记录，并会删除记录；
- 如果要删除记录，则返回零值，否则返回非零值；
允许只是`callables`的过滤器，就是说过滤器只要是个可调用对象就可以。

> 另外，`Filter`除了过滤`record`之外，也可以使用一个用户定义的类`Filter`在日志输出中添加上下文信息。`Filter `的实例是被允许修改传入的 `LogRecords`，包括添加其他的属性，然后可以使用合适的格式化字符串输出，或者可以使用一个自定义的类`Formatter`。

例如，在一个web应用程序中，正在处理的请求(或者至少是请求的一部分)，可以存储在一个线程本地(`threading.local`) 变量中，然后从`Filter`中去访问。请求中的信息，如IP地址和用户名将被存储在`LogRecord`中。示例如下:
```python
import logging
from random import choice

class ContextFilter(logging.Filter):
    """
    This is a filter which injects contextual information into the log.

    Rather than use actual contextual information, we just use random
    data in this demo.
    """

    USERS = ['jim', 'fred', 'sheila']
    IPS = ['123.231.231.123', '127.0.0.1', '192.168.0.1']

    def filter(self, record):

        record.ip = choice(ContextFilter.IPS)
        record.user = choice(ContextFilter.USERS)
        return True

if __name__ == '__main__':
    levels = (logging.DEBUG, logging.INFO, logging.WARNING, logging.ERROR, logging.CRITICAL)
    logging.basicConfig(level=logging.DEBUG,
                        format='%(asctime)-15s %(name)-5s %(levelname)-8s IP: %(ip)-15s User: %(user)-8s %(message)s')
    a1 = logging.getLogger('a.b.c')
    a2 = logging.getLogger('d.e.f')

    f = ContextFilter()
    a1.addFilter(f)
    a2.addFilter(f)
    a1.debug('A debug message')
    a1.info('An info message with %s', 'some parameters')
    for x in range(10):
        lvl = choice(levels)
        lvlname = logging.getLevelName(lvl)
        a2.log(lvl, 'A message at %s level with %d %s', lvlname, 2, 'parameters')
```
在运行时，产生如下内容:
```python
2018-01-27 22:16:29,900 a.b.c DEBUG    IP: 127.0.0.1       User: fred     A debug message
2018-01-27 22:16:29,900 a.b.c INFO     IP: 127.0.0.1       User: sheila   An info message with some parameters
2018-01-27 22:16:29,900 d.e.f CRITICAL IP: 192.168.0.1     User: fred     A message at CRITICAL level with 2 parameters
2018-01-27 22:16:29,900 d.e.f INFO     IP: 123.231.231.123 User: jim      A message at INFO level with 2 parameters
2018-01-27 22:16:29,901 d.e.f DEBUG    IP: 123.231.231.123 User: jim      A message at DEBUG level with 2 parameters
2018-01-27 22:16:29,901 d.e.f INFO     IP: 123.231.231.123 User: fred     A message at INFO level with 2 parameters
2018-01-27 22:16:29,901 d.e.f INFO     IP: 192.168.0.1     User: fred     A message at INFO level with 2 parameters
2018-01-27 22:16:29,901 d.e.f DEBUG    IP: 123.231.231.123 User: jim      A message at DEBUG level with 2 parameters
2018-01-27 22:16:29,901 d.e.f CRITICAL IP: 192.168.0.1     User: fred     A message at CRITICAL level with 2 parameters
2018-01-27 22:16:29,901 d.e.f ERROR    IP: 123.231.231.123 User: fred     A message at ERROR level with 2 parameters
2018-01-27 22:16:29,901 d.e.f ERROR    IP: 192.168.0.1     User: jim      A message at ERROR level with 2 parameters
2018-01-27 22:16:29,901 d.e.f CRITICAL IP: 123.231.231.123 User: jim      A message at CRITICAL level with 2 parameters
```
### 四、总结
> 虽说这部分内容很简单，但是为了更好的介绍`Handler`和`Logger`，还是有必要了解一下的。

