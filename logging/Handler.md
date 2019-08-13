---
title: logging源码解析之Handler
urlname: logging/Handler
categories: python
tags:
  - python
  - logging
  - 源码解析
description: 在本篇文章中介绍logging的Handler概念，主要是把LogRecord实例格式化后的字符串输出到定义的stream中。
p: /python/logging源码解析之Handler
date: 2018-02-03 14:15:34
updated: 2018-02-03 14:15:34
---
### 一、引言
我们之前介绍了`{% post_link python/logging源码解析之LogRecord LogRecord %}`、`{% post_link python/logging源码解析之Formatter Formatter %}`、`{% post_link python/logging源码解析之Filter Filter %}`，今天我们介绍`logging`模块中的另一重要概念`Handler`。
首先，介绍几个变量和函数：
#### 1. 变量
```python
_handlers = weakref.WeakValueDictionary()  # map of handler names to handlers
_handlerList = []  # added to allow handlers to be removed in reverse of order initialized
```
模块级变量`_handlers`表示的是`handler.name`到`handler`的映射；
`_handlerList`维护的是`handler`的内部清理列表，将初始化的`handler`引用添加到`_handlerList`列表里，以允许`handler`以初始化的顺序反向删除。
#### 2. `_removeHandlerRef`
```python
def _removeHandlerRef(wr):
    acquire, release, handlers = _acquireLock, _releaseLock, _handlerList
    if acquire and release and handlers:
        acquire()
        try:
            if wr in handlers:
                handlers.remove(wr)
        finally:
            release()
```
这个函数实现了从列表`_handlerList`中删除`handler`引用。
> 当globals设置为None时，可以在模块拆卸期间调用此函数。 它也可以从另一个线程调用。 所以我们需要先占用必要的全局变量并检查它们是否为None，以防止在解释器关闭期间出现竞争条件和故障。
#### 3. `_addHandlerRef`
```python
def _addHandlerRef(handler):
    _acquireLock()
    try:
        _handlerList.append(weakref.ref(handler, _removeHandlerRef))
    finally:
        _releaseLock()
```
这个函数实现的是将使用`weakref`将`handler`添加到列表`_handlerList`，示例如下:
```python
def handler_test():
    import logging
    print(logging._handlers)
    print(logging._handlerList)

if __name__ == '__main__':
    handler_test()
    
Output: <WeakValueDictionary at 0x10a8ec7b8>
[<weakref at 0x10a92bef8; to '_StderrHandler' at 0x10a92ac18>]
```
### 二、`Handler`
现在正式分析`Handler`类。
#### 1. 定义
```python
class Handler(Filterer):
```
`Handler`实例将`logging`事件分派到特定目标。这是`Handler`的基类，充当占位符，定义`Handler`接口。`Handler`可以根据需要使用`Formatter`实例来格式化`record`。 默认情况下，不指定`formatter`， 在这种情况下，日志记录由`record.message`确定的“原始”消息生成。
#### 2. `__init__`
```python
    def __init__(self, level=NOTSET):
        Filterer.__init__(self)
        self._name = None
        self.level = _checkLevel(level)
        self.formatter = None
        # Add the handler to the global _handlerList (for cleanup on shutdown)
        _addHandlerRef(self)
        self.createLock()
```
`__init__`方法初始化实例 - 将属性`formatter`设置为None，将属性`filters`设置为空。
#### 3. `createLock`
```python
    def createLock(self):
        if threading:
            self.lock = threading.RLock()
        else:  # pragma: no cover
            self.lock = None
```
`createLock`方法设置属性`lock`，实现获取线程锁从而对底层`I/O`进行序列化访问。
#### 4. `get_name`
```python
    def get_name(self):
        return self._name
```
`get_name`方法获取`Handler`实例的`_name`属性的值。
#### 5. `set_name`
```python
    def set_name(self, name):
        _acquireLock()
        try:
            if self._name in _handlers:
                del _handlers[self._name]
            self._name = name
            if name:
                _handlers[name] = self
        finally:
            _releaseLock()
```
`set_name`方法为`Handler`实例的`_name`属性设置值，并将其重新加入到`_handlers`字典中。
```python
name = property(get_name, set_name)
```
将`get_name`和`set_name`设置为`property`类型的属性。
#### 6. `acquire`
```python
    def acquire(self):
        if self.lock:
            self.lock.acquire()
```
`acquire`方法实现了获取`I/O`线程锁。
#### 7. `release`
```python
    def release(self):
        if self.lock:
            self.lock.release()
```
`release`方法实现了释放`I/O`线程锁。
#### 8. `setLevel`
```python
    def setLevel(self, level):
        self.level = _checkLevel(level)
```
`setLevel`方法实现了设置`handler`的日志级别，`level`参数必须是`int`或者`str`。
#### 9. `format`
```python
    def format(self, record):
        if self.formatter:
            fmt = self.formatter
        else:
            fmt = _defaultFormatter
        return fmt.format(record)
```
`format`方法实现了格式化指定的`record`，如果设置了属性`formatter`，就使用它进行格式化日志记录。 否则就使用模块的默认`formatter`，即`_defaultFormatter`来进行格式化日志记录并将其以字符串形式返回。
#### 10. `emit`
```python
    def emit(self, record):
        raise NotImplementedError('emit must be implemented by Handler subclasses')
```
实际上`emit`方法执行记录指定`record`所需的任何操作。。此版本旨在由子类实现，因此引发`NotImplementedError`。
#### 11. `handle`
```python
    def handle(self, record):
        rv = self.filter(record)
        if rv:
            self.acquire()
            try:
                self.emit(record)
            finally:
                self.release()
        return rv
```
`handle`方法实现的是将通过`filter`方法的`record`执行`emit`方法，通过获取/释放`I/O`线程锁执行`emit`方法，从而完成对`record`的包装。 返回的是`record`是否通过过滤器。
#### 12. `setFormatter`
```python
    def setFormatter(self, fmt):
        self.formatter = fmt
```
`setFormatter`方法用来设置`handler`的`formatter`属性。
#### 13. `flush`
```python
    def flush(self):
        pass
```
`flush`方法确保刷新所有日志记录输出。此版本不执行任何操作，旨在由子类实现。
#### 14. `close`
```python
    def close(self):
        # get the module data lock, as we're updating a shared structure.
        _acquireLock()
        try:  # unlikely to raise an exception, but you never know...
            if self._name and self._name in _handlers:
                del _handlers[self._name]
        finally:
            _releaseLock()
```
`close`方法用于整理`handler`使用的任何资源。这里实现的是从`handler`的内部映射`_handlers`中删除`handler`。子类应确保重写的`close()`方法调用它。
#### 15. `handleError`
```python
    def handleError(self, record):
        if raiseExceptions and sys.stderr:  # see issue 13807
            t, v, tb = sys.exc_info()
            try:
                sys.stderr.write('--- Logging error ---\n')
                traceback.print_exception(t, v, tb, None, sys.stderr)
                sys.stderr.write('Call stack:\n')
                # Walk the stack frame up until we're out of logging,
                # so as to print the calling context.
                frame = tb.tb_frame
                while (frame and os.path.dirname(frame.f_code.co_filename) ==
                    __path__[0]):
                    frame = frame.f_back
                if frame:
                    traceback.print_stack(frame, file=sys.stderr)
                else:
                    # couldn't find the right stack frame, for some reason
                    sys.stderr.write('Logged from file %s, line %s\n' % (
                        record.filename, record.lineno))
                # Issue 18671: output logging message and arguments
                try:
                    sys.stderr.write('Message: %r\n'
                                     'Arguments: %s\n' % (record.msg,
                                                          record.args))
                except Exception:
                    sys.stderr.write('Unable to print the message and arguments'
                                     ' - possible formatting error.\nUse the'
                                     ' traceback above to help find the error.\n'
                                     )
            except OSError:  # pragma: no cover
                pass  # see issue 5971
            finally:
                del t, v, tb
```
`handleError`方法实现的是处理在调用`emit`方法期间发生的错误。当在调用`emit()`方法期间遇到异常时，应该从`handler`调用此方法；如果`raiseExceptions`为`True`，则会以静默方式忽略异常；也可以使用自定义的`handler`替换它，将正在处理的`record`传递给此方法。
#### 16. `__repr__`
```python
    def __repr__(self):
        level = getLevelName(self.level)
        return '<%s (%s)>' % (self.__class__.__name__, level)
```
### 三、`StreamHandler`
类`StreamHandler`是继承自`Handler`类的。
#### 1. 定义
```python
class StreamHandler(Handler):
    terminator = '\n'
```
`StreamHandler`是一个`handler`类，它将已经过格式化的日志记录写入流。 **请注意，此类不会关闭流，因为可能会使用`sys.stdout`或`sys.stderr`。**
#### 2. `__init__`
```python
    def __init__(self, stream=None):
        Handler.__init__(self)
        if stream is None:
            stream = sys.stderr
        self.stream = stream
```
`__init__`初始化`handler`，首先调用父类`Handler`的`__init__`方法，然后设置`stream`属性，如果未指定`stream`参数，则使用`sys.stderr`。
#### 3. `flush`
```python
    def flush(self):
        self.acquire()
        try:
            if self.stream and hasattr(self.stream, "flush"):
                self.stream.flush()
        finally:
            self.release()
```
重写的`flush`方法实现的是刷新`stream`的功能。
#### 4. `emit`
```python
    def emit(self, record):
        try:
            msg = self.format(record)
            stream = self.stream
            stream.write(msg)
            stream.write(self.terminator)
            self.flush()
        except Exception:
            self.handleError(record)
```
重写的`emit`方法实现的是将一个`record`写入`stream`中。
> 如果指定了`formatter`属性，则用其对`record`进行格式化；然后使用尾随换行符`terminator`属性的值将`record`写入流中。 如果存在异常信息，则使用`traceback.print_exception`对其进行格式化并附加到流中。 如果流具有'encoding'属性，则它用于确定如何输出到流。

#### 5. `__repr__`
```python
    def __repr__(self):
        level = getLevelName(self.level)
        name = getattr(self.stream, 'name', '')
        if name:
            name += ' '
        return '<%s %s(%s)>' % (self.__class__.__name__, name, level)
```
综上可知，`StreamHandler`类实现的就是将`record`输出到`stream`的`handler`。
### 四、`FileHandler`
类`FileHandler`是一个继承自`StreamHandler`类的`handler`，这个类实现的是将格式化的日志记录写入磁盘文件。
#### 1. 定义
```python
class FileHandler(StreamHandler):
```
#### 2. `__init__`
```python
    def __init__(self, filename, mode='a', encoding=None, delay=False):
        # Issue #27493: add support for Path objects to be passed in
        filename = os.fspath(filename)
        # 使用绝对路径，否则当前目录更改时，使用它的派生类可能会成为一个cropper
        self.baseFilename = os.path.abspath(filename)
        self.mode = mode
        self.encoding = encoding
        self.delay = delay
        if delay:
            # 这里不打开流，但是我们仍然需要调用Handler构造函数来设置level，formatter，lock等。
            Handler.__init__(self)
            self.stream = None
        else:
            StreamHandler.__init__(self, self._open())
```
`__init__`方法中将打开指定的文件并将其用作日志记录流。
#### 3. `close`
```python
    def close(self):
        self.acquire()
        try:
            try:
                if self.stream:
                    try:
                        self.flush()
                    finally:
                        stream = self.stream
                        self.stream = None
                        if hasattr(stream, "close"):
                            stream.close()
            finally:
                # Issue #19523: call unconditionally to
                # prevent a handler leak when delay is set
                StreamHandler.close(self)
        finally:
            self.release()
```
`close`方法实现了关闭`stream`。
#### 4. `_open`
```python
    def _open(self):
        return open(self.baseFilename, self.mode, encoding=self.encoding)
```
`_open`方法使用(原始)模式和编码打开`baseFilename`并返回结果流。
#### 5. `emit`
```python
    def emit(self, record):
        if self.stream is None:
            self.stream = self._open()
        StreamHandler.emit(self, record)
```
`emit`方法实现了将日志`record`发送到流中。而在`emit`方法中，如果由于在构造函数中指定了`delay`而没有打开流，则在调用父类`StreamHandler`的emit之前打开它。
#### 5. `__repr__`
```python
    def __repr__(self):
        level = getLevelName(self.level)
        return '<%s %s (%s)>' % (self.__class__.__name__, self.baseFilename, level)
```
### 五、`_StderrHandler`
类`_StderrHandler`是一个继承自`StreamHandler`类的`handler`，这个类类似于使用`sys.stderr`的`StreamHandler`，但是`stream`属性总是`sys.stderr`，而不是通过构造函数进行设置的`sys.stderr`。
#### 1. 定义
```python
class _StderrHandler(StreamHandler):
```
#### 2. `__init__`
```python
    def __init__(self, level=NOTSET):
        Handler.__init__(self, level)
```
`__init__`方法仅是调用了`Handler`类的`__init__`方法。
#### 3. `stream`
```python
    @property
    def stream(self):
        return sys.stderr
```
`stream`方法添加了`property`特性，将覆盖`__init__`方法中设置的`stream`属性。
在`logging`模块中还定义了与之相关的两个变量，如下:
```python
_defaultLastResort = _StderrHandler(WARNING)
lastResort = _defaultLastResort
```
即将值为`WARNING`的`level`参数传入类`_StderrHandler`生成一个`Handler`实例，并将其赋值给模块级变量`_defaultLastResort`；接着又将变量`_defaultLastResort`赋值给模块级变量`lastResort`。
### 六、`NullHandler`
```python
# Null handler
class NullHandler(Handler):
    def handle(self, record):
        """Stub."""

    def emit(self, record):
        """Stub."""

    def createLock(self):
        self.lock = None
```
这是一个空的`handler`类。
> 它旨在避免“无法为`logger`XXX找到`handler`“的一次警告。 这对于库代码很重要，库代码可能包含记录事件的代码。 如果库的用户未配置日志记录，则可能会生成一次性警告; 为了避免这种情况，库开发人员只需要实例化`NullHandler`并将其添加到库模块或包的顶级`logger`中。
### 七、总结
到这里，关于`logging`模块的`Handler`概念就解析完了，这里主要定义了三个`handler`类，主要是将`record`输出到`stream`中，`stream`可以是标准输出流`sys.stdout`、错误输出流`sys.stderr`或者文件。