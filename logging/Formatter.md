---
title: logging源码解析之Formatter
urlname: logging/Formatter
categories: python
tags:
  - python
  - logging
  - 源码解析
description: 在本篇文章中将详细介绍logging模块中的Formatter类，也就是如何将logRecord实例按照一定的格式转化成text文本格式，也就是我们在日志输出文件中看到的内容.
p: /python/logging源码解析之Formatter
date: 2018-01-20 14:15:09
updated: 2018-01-20 14:15:09
---
### 一、引言
在[LogRecord](logging/LogRecord)这篇文章中，我们已经介绍了`logging`模块中`LogRecord`类，那在本篇文章中将详细介绍`logging`模块中的`Formatter`类，`Formatter`类的作用就是将一个`Logrecord`实例按照我们既定的日志输出格式转化成`text`格式。
不过在正式分析`Formatter`类之前，我们先分析下日志输出格式有哪些风格可以使用。
### 二、类`PercentStyle`
首先，我们看一下字符串格式化运算符`%`的定义及使用，以方便我们后面的分析。
#### 1. `%`
字符串格式化运算符`%`是根据指定的格式格式化字符串。
语法如下：
```python
%[key][flags][width][.precision][length type]conversion type % values
```
接下来介绍语法中各个部分的含义:
##### 1.1 `%`
> 语法中符号`%`是必须的，指定开头的标记。

##### 1.2 `key`
> 语法中`[key]`是一个映射`key`，是可选的，也就是可省略的。
这部分的格式是由带括号的字符序列组成(`e.g. (somename)`)，那和`%`结合起来用就是`%(somename)`。

##### 1.3 `flags`
> 语法中`[flags]`也是可选的。这是一个标记位，会影响语法中`conversion type`这一部分的结果，它有以下几种类型：
- '#':  值转换将使用`alternate form`(在下面定义) ;
- '0':  对于数值类型的`values`，转换将用`0`填充;
- '-':  保留转换后的值(如果`0`与`-`同时存在，则覆盖`0`转换);
- ' ':  (空格)在标记转换产生的正数(或空字符串)之前应留空。
- '+':  符号字符('+'或' - ')将在转换之前(覆盖“space”标志)。
可以存在长度修饰符(h，l或L)，但是因为它不是Python所必需的，所以会被忽略——例如， `%ld`与`%d`相同。

##### 1.4 `width`
> 语法中`[width]`也是可选的。，表示的是最小字段宽度。
> 如果指定为 `*` (星号)，则从`values`的元组的下一个元素读取实际宽度，并且要转换的对象在最小字段宽度和可选`precision`之后。

##### 1.5 `precision`
> 语法中`[.precision]`是可选的，表示精度，以`.`(点)表示，后跟精度位。 
> 如果指定为 `*` (星号)，则从`values`的元组的下一个元素读取实际宽度，并且转换的值在精度之后。

##### 1.6 `length type`
> 语法中`[length type]`是可选的，表示长度修饰符;

##### 1.7 `conversion type`
> 语法中`[conversion type]`是可选的，表示转换类型。它有以下几个可选类型:
- ‘d’:  有符号整数小数;
- ‘i’:  有符号整数小数;
- ‘o’:  有符号的八进制。 如果结果的开头字符不是`0`，则`alternate form`会在左侧填充格式(`width`设置)和数字之间插入开头字符`0`;
- ‘u’:  过时类型 - 它与'd'相同。 见PEP 237;
- ‘x’ or ‘X’:  有符号的十六进制。如果结果的开头字符不是`0x`或`0X`，则`alternate form`会在左侧填充格式(`width`设置)和数字之间插入开头字符`0x`或`0X`(取决于使用“x”还是“X”格式); 
- ‘e’ or ‘E’:  浮点指数格式。 `alternate form`会让结果始终包含小数点，即使后面没有数字。`precision`确定小数点后的位数，默认为6;
- ‘f’ or ‘F’:  浮点小数格式。 `alternate form`会让结果始终包含小数点，即使后面没有数字也是如此。`precision`确定小数点后的位数，默认为6;
- ‘g’ or ‘G’:  浮点格式。 如果指数小于-4或不小于精度，则使用指数格式，否则使用小数格式。  `alternate form`会让结果始终包含小数点，并且不会删除尾随零，否则它们将被删除。`precision`确定小数点前后的有效位数，默认为6;
- ‘c’:  单个字符(接受整数或单个字符串);
- ‘r’:  `String`(使用`repr()`)转换任何Python对象)，`precision`确定使用的最大字符数;
- ‘s’:  `String`(使用`str()`转换任何Python对象); 如果提供的对象或格式是unicode字符串，则生成的字符串也将是unicode。`precision`确定使用的最大字符数;
- ‘%’:  没有转换参数，致使结果中出现`%`字符。

##### 1.8 `values`
> 语法中`values`是必须的，是用于替换`conversion type`的`number`，`string`或`sontainer`类型的值。

这里我就不一一举例说明了，有疑问的可以查看[官方文档](https://python-reference.readthedocs.io/en/latest/docs/str/formatting.html)。
#### 2. 定义
```python
class PercentStyle(object):
    default_format = '%(message)s'
    asctime_format = '%(asctime)s'
    asctime_search = '%(asctime)'
```
首先这个类定义了`3`个类属性`default_format`, `asctime_format`, `asctime_search`，且这三个属性的值符合`%`字符串格式化的语法的，所以这个类就是为了对满足`%`运算符语法的字符串进行格式化的。
#### 3. `__init__`
```python
    def __init__(self, fmt):
        self._fmt = fmt or self.default_format
```
`__init__`方法比较简单，就是接收一个`fmt`参数来设置`_fmt`属性，默认值是`default_format`类属性，可知`_fmt`属性的值是一个字符串。
#### 4. `usesTime`
```python
    def usesTime(self):
        return self._fmt.find(self.asctime_search) >= 0
```
`usesTime`的方法实现的是: 只要在`_fmt`属性值中包含字符串`%(asctime)`，就返回`True`，否则返回`False`。
#### 5. `format`
```python
    def format(self, record):
        return self._fmt % record.__dict__
```
`format`函数就是根据传入的`record`实例的`__dict__`属性格式化`_fmt`字符串并返回。

**`record`实例实际上就是一个`LogRecord`对象**，示例如下：
```python
def percent_style_test():
    import logging

    percent_style = logging.PercentStyle(fmt='%(asctime)s - %(filename)s[line:%(lineno)d] - %(levelname)s: %(message)s')

    log_record = logging.LogRecord(name='root', level=40,
                                   pathname='/Users/yajingliu/Projects/PycharmProjects/gunicorn/gunicorn/glogging.py',
                                   lineno=455,
                                   msg='test log %(a)s',
                                   args=({'a': 'error'},),
                                   exc_info=None,
                                   func='test',
                                   extra=None,
                                   sinfo=None)
    log_record.message = log_record.getMessage()
    if percent_style.usesTime():
        log_record.asctime = "2019-07-09 22:30:16"
    print(percent_style.format(log_record))


if __name__ == '__main__':
    percent_style_test()
```
在这个例子中，我们首先定义了一个`percent_style`对象，为其定义了日志输出的格式；然后又初始化了一个`log_record`对象，因为在`log_record.__dict__`中我们没有`asctime`和`message`字段，所以需要为其单独赋值，`message`值就是`LogRecord`中`getMessage`方法返回的值，而`asctime`是一个格式化时间。最后我们调用`percent_style`就按照我们的日志格式定义返回了一个字符串。
综上，类`PercentStyle`主要是对满足`%`运算符语法的字符串进行格式化的。
### 三、类`StrFormatStyle`
#### 1. `str.format()`
这是我们格式化字符串时经常用到的函数，这里不过多说明，可以参考[官方文档](https://docs.python.org/3.6/library/string.html#format-examples)
#### 2. 定义
```python
class StrFormatStyle(PercentStyle):
    default_format = '{message}'
    asctime_format = '{asctime}'
    asctime_search = '{asctime'
```
类`StrFormatStyle`是继承自类`PercentStyle`的，但其三个类属性的值均是`{string}`种形式的。
#### 3. format
```python
    def format(self, record):
        return self._fmt.format(**record.__dict__)
```
类`StrFormatStyle`中只重新定义了`format`方法，调用`str.format()`函数对属性`_fmt`进行格式化并返回格式化后的字符串。
### 四、类`StringTemplateStyle`
#### 1. `string.Template`
`Template`类是`string`中的一个类，可以将一个字符串的格式生成一个模版重复利用，可以参考[官方文档](https://docs.python.org/3.6/library/string.html#template-strings)
#### 2. 定义
```python
class StringTemplateStyle(PercentStyle):
    default_format = '${message}'
    asctime_format = '${asctime}'
    asctime_search = '${asctime}'
```
类`StringTemplateStyle`是继承自类`PercentStyle`的，但其三个类属性的值均是`${string}`种形式的。
我们来看一下这个类的几个方法:
#### 3. `__init__`
```python
    def __init__(self, fmt):
        self._fmt = fmt or self.default_format
        self._tpl = Template(self._fmt)
```
在`__init__`方法中多了一`_tpl`属性，也就是`Template`将属性`_fmt`的值生成了一个模版并赋值给了`_tpl`属性。
#### 4. `usesTime`
```python
    def usesTime(self):
        fmt = self._fmt
        return fmt.find('$asctime') >= 0 or fmt.find(self.asctime_format) >= 0
```
`usesTime`方法是为了确认日志格式中是否有`asctime`这个字段的输出，根据判断语句可知`$asctime`和`${asctime}`两种形式均可。
#### 5. `format`
```python
    def format(self, record):
        return self._tpl.substitute(**record.__dict__)
```
`format`方法还是将格式化后的日志记录返回，此方中调用的是`Template`中的`substitute`方法，`record.__dict__`是一个字典`{key: value}`，那参数`**record.__dict__`就是将字典转换成`key=value`的形式。

综上可知，类`PercentStyle`、类`StrFormatStyle`和`StringTemplateStyle`是定义了三种日志记录的输出格式及其`format`方法实现各自的格式化。
### 五、类Formatter
先看两个模块级常量：
```python
BASIC_FORMAT = "%(levelname)s:%(name)s:%(message)s"

_STYLES = {
    '%': (PercentStyle, BASIC_FORMAT),
    '{': (StrFormatStyle, '{levelname}:{name}:{message}'),
    '$': (StringTemplateStyle, '${levelname}:${name}:${message}'),
}
```
终于到本篇文章的重点了，旗面说过，`Formatter`类的作用就是将一个`Logrecord`实例按照我们既定的日志输出格式转化成`text`格式，那`LogRecord`中的属性及意义请参考 [LogRecord](logging/LogRecord)。
#### 1. 定义
```python
class Formatter(object):
    converter = time.localtime # 时间转换函数
    default_time_format = '%Y-%m-%d %H:%M:%S'  # 默认时间格式字符串
    default_msec_format = '%s,%03d' # 默认msctime格式化字符串
```
类`Formatter`是继承于`object`类的，定义了三个类属性。
#### 2. `__init__`
```python
    def __init__(self, fmt=None, datefmt=None, style='%'):
        if style not in _STYLES:
            raise ValueError('Style must be one of: %s' % ','.join(
                _STYLES.keys()))
        self._style = _STYLES[style][0](fmt)
        self._fmt = self._style._fmt
        self.datefmt = datefmt
```
`__init__`方法接收三个参数: 
- `fmt`: 日志输出的格式化字符串;
- `style`: 字符串类型，必须是`_STYLES`中的`key`，也就是`%、{ 、$`其中之一，默认是`%`;
- `datefmt`: 日志输出的时间格式.

> 由`_STYLES`的定义可知，属性`_style`是一个根据不同的`style`参数而生成的对应的`Style`类的实例，且初始化参数是由`fmt`参数的值；而属性`_fmt`的值就是`_style`对应的`Style`类实例的`_fmt`属性的值，根据我们前面的分析，也就是`fmt`参数的值；最后属性`_datefmt`的值等于参数`datefmt`参数的值。

#### 3. `formatTime`
```python
    def formatTime(self, record, datefmt=None):
        ct = self.converter(record.created)
        if datefmt:
            s = time.strftime(datefmt, ct)
        else:
            t = time.strftime(self.default_time_format, ct)
            s = self.default_msec_format % (t, record.msecs)
        return s
```
`formatTime`方法是对`Logrecord`中的`created`时间进行格式化并返回，默认格式是`default_msec_format`，可以通过`__init__`方法的参数`datefmt`进行自定义。
#### 4. `formatException`
```python
    def formatException(self, ei):
        sio = io.StringIO()
        tb = ei[2]
        # See issues #9427, #1553375. Commented out for now.
        # if getattr(self, 'fullstack', False):
        #    traceback.print_stack(tb.tb_frame.f_back, file=sio)
        traceback.print_exception(ei[0], ei[1], tb, None, sio)
        s = sio.getvalue()
        sio.close()
        if s[-1:] == "\n":
            s = s[:-1]
        return s
```
`formatException`方法实现的是格式化并以字符串形式返回指定的异常信息。这个默认实现只使用`traceback.print_exception()`。
#### 5. `usesTime`
```python
    def usesTime(self):
        return self._style.usesTime()
```
`usesTime`方法主要调用了属性`_style`的`usesTime`方法，用来检查日志输出格式中是否使用`LogRecord`的创建时间。
#### 6. `formatMessage`
```python
    def formatMessage(self, record):
        return self._style.format(record)
```
`formatMessage`方法中调用了`_style`的`format`方法将`LogRecord`实例格式化成日志输出的格式并将格式化后的字符串返回。
#### 7. `formatStack`
```python
    def formatStack(self, stack_info):
        return stack_info
```
`formatStack`方法是格式化`stack_info`，这里是直接返回之。
#### 8. `format`
```python
    def format(self, record):
        # LogRecord类中没有message和asctime属性，调用相应的方法为其添加这两个属性
        record.message = record.getMessage()
        if self.usesTime():
            record.asctime = self.formatTime(record, self.datefmt)
        # 将LogRecord实例格式化为字符串
        s = self.formatMessage(record)
        if record.exc_info:
            # Cache the traceback text to avoid converting it multiple times
            # (it's constant anyway)
            if not record.exc_text:
                record.exc_text = self.formatException(record.exc_info)
        if record.exc_text:
            if s[-1:] != "\n":
                s = s + "\n"
            s = s + record.exc_text
        if record.stack_info:
            if s[-1:] != "\n":
                s = s + "\n"
            s = s + self.formatStack(record.stack_info)
        return s
```
`format`方法中实现是如何将`LogRecord`实例格式化成指定的日志输出格式，最后将格式化后的字符串返回。
> 这个方法首先为`LogRecord`实例添加`message`和`asctime`属性，然后调用`formatMessage`方法将`LogRecord`实例格式化为字符串，然后再在这个字符串后面追加异常信息和堆栈信息后将其返回。

综上可知，类`Formatter`中最核心就是`format`方法了，正式它实现了将`LogRecord`实例格式化成指定的日志输出格式，也就是`text`形式。
最后，定义了一个模块级全局变量来指定默认的`formatter`:
```python
_defaultFormatter = Formatter()
```
### 六、类`BufferingFormatter`
#### 1. 定义
```python
class BufferingFormatter(object):
```
这是一个适合格式化多个`records`的`formatter`。
#### 2. `__init__`
```python
    def __init__(self, linefmt=None):
        if linefmt:
            self.linefmt = linefmt
        else:
            self.linefmt = _defaultFormatter
```
`__init__`方法可以指定`formatter`，用于格式化每个单独的`record`。
#### 3. `formatHeader`
```python
    def formatHeader(self, records):
        return ""
```
这个方法用于返回指定`records`的`header`字符串。
#### 4. `formatFooter`
```python
    def formatFooter(self, records):
        return ""
```
这个方法用于返回指定`records`的`footer`字符串。
#### 5. `format`
```python
    def format(self, records):
        rv = ""
        if len(records) > 0:
            rv = rv + self.formatHeader(records)
            for record in records:
                rv = rv + self.linefmt.format(record)
            rv = rv + self.formatFooter(records)
        return rv
```
`format`方法用于格式化指定的记录并将结果作为字符串返回。
### 七、总结
到这里，关于`logging`模块的日志格式的内容就分析完了，我们知道日志格式有`3`中风格可供选择使用，最后将你的风格以参数形式传入`Formatter`类的`__init__`方法，从而可以调用其对应的`format`方法已完成对它的格式化。
