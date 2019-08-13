---
title: logging源码解析之DictConfig
urlname: logging/DictConfig
categories: python
tags:
  - python
  - logging
  - 源码解析
description: 本篇文章中详细介绍了dict配置格式、加载原理，以及深入分析了dictConfig()函数的实现机制.
p: /python/logging源码解析之DictConfig
date: 2018-03-10 14:20:11
updated: 2018-03-10 14:20:11
---
### 一、引言
在logging模块中，可以通过三种方式配置日志记录：
- 使用调用`baseConfig()`函数显式创建`logger`、`handler`和`formatter`。
- 创建日志配置文件并使用`fileConfig()`函数读取它。
- 创建配置信息字典并将其传递给`dictConfig()`函数。
第一种方式我们在`{% post_link python/logging源码解析之RootLogger RootLogger %}`这篇文章中已经详细说明其用法，第二种在`{% post_link python/logging源码解析之FileConfig FileConfig %}`文章中介绍，在本篇文章中我们讨论通过字典配置日志记录的实现，官方建议使用字典配置。

如果对logging日志模块中的几个概念还有不明白的，可以参考之前介绍的几篇文章，这里不再过多涉及。

源码路径: python3.6/logging/config.py
### 二、字典配置
`logging configuration`需要列出要创建的各种对象以及它们之间的连接，例如：
- 首先创建一个名为`console`的`handler`；
- 然后创建一个名为`startup`的`logger`并将其消息发送到`console`的`handler`。 
这些对象不限于`logging`模块中提供的对象，因为您可以编写自己的`formatter`或`handler`类。 这些类的参数可能还需要包含外部对象，例如`sys.stderr`。 
描述这些对象和连接的语法会在下面定义。
#### 1. 基本配置
传递给`dictConfig()`的字典必须包含以下键：
- `version`  - 设置为表示模式版本的整数值。 目前唯一有效的值是1，但是使用此`key`可以使模式发展，同时仍然保持向后兼容性。
其他所有的`key`都是可选的，但如果存在，则将按如下所述进行解释。 在下面提到“字典配置”的所有情况下，将检查特殊的`()`键以查看是否需要自定义实例化，如果需要，用定义对象中描述的机制来创建实例；否则，上下文用于确定要实例化的内容。
##### 1. 1 `formatters`
其对应的`value`是一个`dict`，其中每个`key`都是一个`formatter ID`，其对应的`value`都是一个描述如何配置相应的`Formatter`实例的字典，字典中`format`和`datefmt`(默认值为`None`)这两个`key`，用于构造`Formatter`实例。
##### 1.2 `filters`
其对应的`value`是一个`dict`，其中每个`key`都是一个`filter id`，其对应的`value`都是一个描述如何配置相应`Filter`实例的字典，字典中`name`(默认为空字符串)这个`key`，用于构造`logging.Filter`实例。
##### 1.3 `handlers`
其对应的`value`是一个`dict`，其中每个`key`都是一个`handler id`，其对应的`value`都是一个描述如何配置相应`Handler`实例的字典，字典中有如下`key`:
(1). `class`(强制性)，这是`handler`类的完全限定名称;
(2). `level`(可选)，`handler`的级别;
(3). `formatter`(可选)，此`handler`的`formatter id`;
(4). `filter`(可选)，此`handler`的`filter id`列表。
其它所有`key`都作为关键字参数传递给`handler`的构造函数。 例如，给定代码段：
```yml
handlers:
  console:
    class : logging.StreamHandler
    formatter: brief
    level   : INFO
    filters: [allow_foo]
    stream  : ext://sys.stdout
  file:
    class : logging.handlers.RotatingFileHandler
    formatter: precise
    filename: logconfig.log
    maxBytes: 1024
    backupCount
```
其中ID为`console`的`handler`被实例化为`logging.StreamHandler`，使用`sys.stdout`作为输出流。 
ID为`file`的`handler`被实例化为`logging.handlers.RotatingFileHandler`，其关键字参数filename ='logconfig.log'，maxBytes = 1024，backupCount = 3。
##### 1.4 `loggers`
其对应的`value`是一个`dict`，其中每个`key`都是一个`logger id`，其对应的`value`都是一个描述如何配置相应`Logger`实例的字典，字典中有如下`key`:
(1). level(可选)，`logger`的`level`;
(2). propagate(可选)，`logger`的`propagate`设置;
(3). filters(可选)，`logger`的`filter id`的列表。
(4). handlers(可选)， `logger`的`handler id`列表。
将根据指定的`level`，`propagate`，`filters`和`handlers`配置创建指定的`logger`。
##### 1.5 `root`
这是`root`记录器的配置。 配置的处理将与任何记录器一样，但`propagate`设置不适用。
##### 1.6 `incremental`
是否将`dict`配置处理为现有配置的增量配置。 此值默认为`False`，意味着指定的配置将使用与现有`fileConfig()`API使用的语义相同的语义替换现有配置。
如果指定的值为True，则按照“增量配置”一节中的说明处理配置。
##### 1.7 `disable_existing_loggers`
是否要禁用现有的所有非`root`记录器。 此设置与`fileConfig()`中相同名称的参数意义一样。 如果不存在，则此参数默认为`True`， 如果`incremental`为`True`，则忽略此值。
#### 2. 增量配置
增量配置很难有完全的灵活性。例如，由于`filters`和`formatters`等对象是匿名的，因此一旦设置了配置，在增量配置时就无法引用此类匿名对象。

此外，一旦配置完成，就没有一个令人信服的案例可以在运行时任意改变`logger`，`handlers`，`filters`，`formatters`的对象图; 只需设置`level`(在`logger`中设置`propagate=1`)就可以控制`loggers`和`handlers`的详细程度。在多线程环境中以安全的方式任意改变对象图是有问题的; 虽然并非不可能，但实际上增加的复杂性并不值得。

当`dict`配置中`key`为`incremental`存在且为`True`时，系统将完全忽略所有`formatters`和`filters`的设置，并仅处理`handlers`配置中的`level`设置，以及`logger`和`root`中的`level`和`propagate`设置。

使用`dict`配置可以将配置作为`pickled dicts`通过网络发送到套接字侦听器，因此，长时间运行的应用程序的日志记录详细程度(`level`)可以随着时间的推移而改变，而无需停止和重新启动应用程序。
#### 3. 对象连接
`dict`配置描述了一组日志对象——`loggers`，`handlers`，`formatters`，`filters`——它们在对象图中相互联系，因此，`dict`配置需要表示对象之间的连接，这是通过为每个目标对象提供一个明确标识它的id，然后使用源对象配置中的id来指示源和目标对象之间存在具有该id的连接来完成的。
例如，以下YAML代码段：
```yml
formatters:
  brief:
    # configuration for formatter with id 'brief' goes here
  precise:
    # configuration for formatter with id 'precise' goes here
handlers:
  h1: #This is an id
   # configuration of handler with id 'h1' goes here
   formatter: brief
  h2: #This is another id
   # configuration of handler with id 'h2' goes here
   formatter: precise
loggers:
  foo.bar.baz:
    # other configuration for logger 'foo.bar.baz'
    handlers: [h1, h2]
```
(注意：YAML在这里使用，因为它比Python中等价的`dict`形式更具可读性)

`logger`的ID是`logger`的`name`，以`logger`的`name`用来引用这些`logger`，例如，`foo.bar.baz`；`formatter`和`filter`的ID可以是任何字符串值(例如上面的`brief`，`precies`)，它们是瞬态的，因为它们仅对处理字典配置有意义并用于确定对象之间的连接，配置完成后就不会存在了 。

上面的代码片段表明`name`为`foo.bar.baz`的`logger`应该有两个`handler`，它们由`handler id`为`h1`和`h2`描述。 `h1`的`formatter id`是`brief`，而`h2`的`formatter id`是`precies`。
#### 4. 自定义对象
`dict`配置支持用户定义的`handler`，`filter`和`formatter`对象。(`logger`不需要为不同的实例使用不同的类型，因此在此配置模式中不支持用户定义的`logger`类)。

要配置的对象由`dict`描述它们的配置，为了为用户定义的对象实例化提供完全的灵活性，用户需要提供一个可调用的“工厂” ，使用配置`字典`调用它并返回实例化的对象，这里通过特殊键`()`对应的工厂的绝对导入路径来表示。 这是一个具体的例子：
```yml
formatters:
  brief:
    format: '%(message)s'
  default:
    format: '%(asctime)s %(levelname)-8s %(name)-15s %(message)s'
    datefmt: '%Y-%m-%d %H:%M:%S'
  custom:
      (): my.package.customFormatterFactory
      bar: baz
      spam: 99.9
      answer: 42
```
上面的YAML片段定义了三个`formatter`:
-  第一个，带有id `brief`，是具有指定格式字符串的标准`logging.Formatter`实例;
-  第二个，具有id `default`，具有更长的格式并且还明确定义时间格式，并使用这两个格式字符串初始化`logging.Formatter`。 
以Python源代码形式显示，`brief formatter`和`default formatter`具有配置子字典：
```python
{
  'format' : '%(message)s'
}
```
和
```python
{
  'format' : '%(asctime)s %(levelname)-8s %(name)-15s %(message)s',
  'datefmt' : '%Y-%m-%d %H:%M:%S'
}
```
由于这些字典不包含特殊键`()`，因此从上下文推断实例化: 结果创建了标准的`logging.Formatter`实例。 
- 第三个，具有id `custom`的第三个`formatter`的配置子字典是：
```python
{
  '()' : 'my.package.customFormatterFactory',
  'bar' : 'baz',
  'spam' : 99.9,
  'answer' : 42
}
```
这包含特殊键`()`，意味着需要实例化用户自定义的对象。 在这种情况下，将使用指定的可调用工厂。 如果它确实可调用，则直接使用——否则，如果指定的是字符串(如示例中所示)，则将使用常规导入机制定位实际可调用对象。 之后将使用配置子字典中的其余项作为关键字参数调用`callable`。 
在上面的示例中，假定调用返回具有id `custom`的`formatter`：
```python
my.package.customFormatterFactory(bar='baz', spam=99.9, answer=42)
```
键`()`已被用作特殊键，因为它不是有效的关键字参数名称，因此不会与调用中使用的关键字参数的名称冲突， `()`也可以作为助记符，相应的值是可调用的。
#### 5. 访问外部对象
有时配置需要引用配置外部的对象，例如`sys.stderr`，如果使用Python代码构造`dict`配置，这很简单，但是当通过文本文件(例如`JSON`，`YAML`)提供配置时会出现问题。在文本文件中，没有标准方法可以区分`sys.stderr`和字符串`'sys.stderr'`。

为了便于区分，配置系统在字符串值中查找某些特殊前缀并对其进行特殊处理。例如，如果在配置中提供文字字符串`ext://sys.stderr`作为值，则将剥离`ext://`并使用常规`import`机制处理值的其余部分。

这种前缀的处理方式与协议处理类似: 有一种通用的机制来查找与正则表达式匹配的前缀^(?P<prefix>[a-z]+)://(?P<suffix>.*)$，其中，如果识别`prefix`，则以`prefix`相关的方式处理`suffix`，并用处理的结果替换字符串的值。如果未识别`prefix`，则字符串值将保持原样。
#### 6. 访问内部对象
除了外部对象之外，有时还需要引用配置中的对象。这将由配置系统隐式地完成它所知道的事情。
例如，`logger`或`handler`中`level`的字符串值`DEBUG`将自动转换为`logging.DEBUG`的值，`handlers`，`filters`和`formatters`将获取对象ID并解析为适当的目标对象。

但是，对于日志记录模块不知道的用户定义对象，需要更通用的机制。
例如，`logging.handlers.MemoryHandler`，它接受一个`target`参数，这是另一个委托给它的`handler`。由于系统已经知道了这个类，因此在配置中，给定的目标只需要是相关目标处理程序的对象id，系统将从id解析为`handler`序。但是，如果用户定义了一个有`alternate`属性的`handler`: `my.package.MyHandler`，则配置系统将不知道`alternate`引用的`handler`。为了满足这一需求，通用的解决机制允许用户指定：
```yml
handlers:
  file:
    # configuration of file handler goes here

  custom:
    (): my.package.MyHandler
    alternate: cfg://handlers.file
```
文字字符串`cfg://handlers.file`将以类似于具有`ext://`前缀的字符串的方式解析，但只会返回配置而不会导入命名空间。 该机制允许通过`.`或索引进行访问，方式与`str.format`提供的方式类似。 因此，给出以下代码段：
```yml
handlers:
  email:
    class: logging.handlers.SMTPHandler
    mailhost: localhost
    fromaddr: my_app@domain.tld
    toaddrs:
      - support_team@domain.tld
      - dev_team@domain.tld
    subject: Houston, we have a problem.
```
在配置中，字符串`cfg://handlers`将使用`key`为`handlers`的`dict`，字符串`cfg://handlers.email`将使用`handlers`字典中`key`为`email`对应的`dict`，依此类推。字符串`cfg://handlers.email.toaddrs[1]`将解析为`dev_team.domain.tld`，字符串`cfg://handlers.email.toaddrs[0]`将解析为值`support_team@domain.tld`。可以使用`cfg://handlers.email.subject`或等效地`cfg://handlers.email[subject]`来访问`subject`的值。

如果`key`中包含空格或非字母数字字符，则仅需要使用后一种形式。如果索引值仅包含十进制数字，则将尝试使用相应的整数值进行访问，如果需要，将返回字符串值。

比如给定一个字符串`cfg://handlers.myhandler.mykey.123`，这将解析为`config_dict['handlers']['myhandler']['mykey']['123']`；
如果字符串被指定为`cfg://handlers.myhandler.mykey[123]`，系统将尝试从`config_dict['handlers']['myhandler']['mykey'][123]`中检索其对应的`value`值，如果失败，请回到`config_dict['handlers']['myhandler']['mykey']['123']`。
#### 7. `import`方法和自定义`import`程序
默认情况下，都是使用内置的`__import __()`函数进行导入。 您可能希望用自己的导入机制替换它: 如果是这样，您可以替换`DictConfigurator`或其父类`BaseConfigurator`类的`importer`属性。 但是，您需要注意，因为是通过描述符对类中的函数进行访问的，所以如果您使用Python可调用对象来执行导入，并且希望在类级别而不是实例级别定义它，则需要使用`staticmethod()`进行包装。 例如：
```python
from importlib import import_module
from logging.config import BaseConfigurator

BaseConfigurator.importer = staticmethod(import_module)
```
如果要在配置程序实例上设置`import`可调用对象，则不需要使用`staticmethod()`进行包装。

### 三、`ConvertingXXX`
> 上一节中我们详细介绍了`dict`配置中的语法设置及解析方式，现在我们开始分析`dictConfig`函数是如何解析`dict`配置的。
> 在正式开始之前，我们先看三个类`ConvertingDict`, `ConvertingList`和`ConvertingTuple`，由于这三个类都是以`Converting`开头的，故统称为`ConvertingXXX`。

每个wrapper都应该有一个`configurator`属性，保存转换时用的实际配置器，具体的配置器我们在后面会详细说明。
#### 1. `ConvertingMixin`
```python
class ConvertingMixin(object):
    def convert_with_key(self, key, value, replace=True):
        result = self.configurator.convert(value)
        # 如果转换后的值不同，且replace为True，则替换它
        if value is not result:
            if replace:
                self[key] = result
            if type(result) in (ConvertingDict, ConvertingList, ConvertingTuple):
                result.parent = self
                result.key = key
        return result

    def convert(self, value):
        result = self.configurator.convert(value)
        if value is not result:
            if type(result) in (ConvertingDict, ConvertingList, ConvertingTuple):
                result.parent = self
        return result
```
类`ConvertingMixin`是这些`ConvertingXXX`类的基类，在这个类中定义了两个方法`convert_with_key`和`convert`。
#### 2. `ConvertingDict`
```python
class ConvertingDict(dict, ConvertingMixin):
    def __getitem__(self, key):
        value = dict.__getitem__(self, key)
        return self.convert_with_key(key, value)

    def get(self, key, default=None):
        value = dict.get(self, key, default)
        return self.convert_with_key(key, value)

    def pop(self, key, default=None):
        value = dict.pop(self, key, default)
        return self.convert_with_key(key, value, replace=False)
```
`ConvertingDict`类继承自`dict`和`ConvertingMixin`，是一个转换字典的warpper。
#### 3. `ConvertingList`
```python
class ConvertingList(list, ConvertingMixin):
    def __getitem__(self, key):
        value = list.__getitem__(self, key)
        return self.convert_with_key(key, value)

    def pop(self, idx=-1):
        value = list.pop(self, idx)
        return self.convert(value)
```
`ConvertingList`类继承自`list`和`ConvertingMixin`，是一个转换列表的warpper。
#### 4. `ConvertingTuple`
```python
class ConvertingTuple(tuple, ConvertingMixin):
    def __getitem__(self, key):
        value = tuple.__getitem__(self, key)
        # 无法替换元组条目
        return self.convert_with_key(key, value, replace=False)
```
`ConvertingTuple`类继承自`tuple`和`ConvertingMixin`，是一个转换元组的warpper。
### 四、`BaseConfigurator`
这个类很重要，是默认解析`dict`配置的类`DictConfigurator`的基类，在前面分析完配置格式之后，源码就好理解多了。
先看在源码文件中定义的一个公共函数，有如下代码:
```python
IDENTIFIER = re.compile('^[a-z_][a-z0-9_]*$', re.I)

def valid_ident(s):
    m = IDENTIFIER.match(s)
    if not m:
        raise ValueError('Not a valid Python identifier: %r' % s)
    return True
```
这里定义了一个全局变量`IDENTIFIER`，是一个正则匹配，而函数`valid_ident`是验证字符串`s`是否能够匹配此正则表达式。
#### 1. 定义
```python
class BaseConfigurator(object):
    CONVERT_PATTERN = re.compile(r'^(?P<prefix>[a-z]+)://(?P<suffix>.*)$')

    WORD_PATTERN = re.compile(r'^\s*(\w+)\s*')
    DOT_PATTERN = re.compile(r'^\.\s*(\w+)\s*')
    INDEX_PATTERN = re.compile(r'^\[\s*(\w+)\s*\]\s*')
    DIGIT_PATTERN = re.compile(r'^\d+$')

    value_converters = {
        'ext': 'ext_convert',
        'cfg': 'cfg_convert',
    }

    # We might want to use a different one, e.g. importlib
    importer = staticmethod(__import__)
```
`BaseConfigurator`是配置器的基类，它定义了一些有用的默认值。
#### 2. `__init__`
```python
    def __init__(self, config):
        self.config = ConvertingDict(config)
        self.config.configurator = self
```
`__init__`方法中定义了一个属性`config`，是一个`ConvertingDict`实例，且将其`configurator`属性设置为`self`，也就是`BaseConfigurator`实例本身。
#### 3. `resolve`
```python
    def resolve(self, s):
        # s 是一个以'.'分隔的字符串，name是一个列表
        name = s.split('.')
        # 取出并删除name中的第一个元素
        used = name.pop(0)
        try:
            # import used 并将这个对象赋值给 found
            found = self.importer(used)
            # 遍历name中剩余的元素
            for frag in name:
                # used依次用'.'与元素连接
                used += '.' + frag
                try:
                    # 检测 found 对象中是否有 frag 属性
                    found = getattr(found, frag)
                except AttributeError:
                    self.importer(used)
                    found = getattr(found, frag)
            return found
        except ImportError:
            e, tb = sys.exc_info()[1:]
            v = ValueError('Cannot resolve %r: %s' % (s, e))
            v.__cause__, v.__traceback__ = e, tb
            raise v
```
`resolve`方法是一个解析`python`模块路径是否可以正常`import`的方法，其中`python`模块路径是一个以`.`连接的字符串。
#### 4. `ext_convert`
```python
    def ext_convert(self, value):
        return self.resolve(value)
```
`ext_convert`方法就是解析`ext://`协议的默认转换器，方法里面调用了`resolve`方法进行解析。
#### 5. `cfg_convert`
```python
    def cfg_convert(self, value):
        # 解析"cfg://"开头的配置，比如"cfg://handlers.myhandler.mykey[123]"，此时value="handlers.myhandler.mykey[123]"
        rest = value
        # WORD_PATTERN = re.compile(r'^\s*(\w+)\s*')，匹配到第一个'.'的位置
        m = self.WORD_PATTERN.match(rest)
        if m is None:
            raise ValueError("Unable to convert %r" % value)
        else:
            # 此例中，m.end()=8，即rest=".myhandler.mykey[123]"，而m.groups()[0]="handlers"
            rest = rest[m.end():]
            # 获取key为"handlers"的dict
            d = self.config[m.groups()[0]]
            # print d, rest
            while rest:
                # DOT_PATTERN = re.compile(r'^\.\s*(\w+)\s*')
                m = self.DOT_PATTERN.match(rest)
                if m:
                    # 在此例中，第一个循环: rest=".myhandler.mykey[123]", m.groups()[0]="myhandler"; 
                    # 第二个循环: rest=".mykey[123]", m.groups()[0]="mykey";
                    d = d[m.groups()[0]]
                else:
                    # 在此例中，第三个循环: rest="[123]"，匹配INDEX_PATTERN正则表达式
                    # INDEX_PATTERN = re.compile(r'^\[\s*(\w+)\s*\]\s*')
                    m = self.INDEX_PATTERN.match(rest)
                    if m:
                        idx = m.groups()[0]
                        # DIGIT_PATTERN = re.compile(r'^\d+$')
                        if not self.DIGIT_PATTERN.match(idx):
                            d = d[idx]
                        else:
                            # 如果索引中是数字类型的，先尝试用数字查找，再尝试用字符串查找
                            try:
                                n = int(idx)  # try as number first (most likely)
                                d = d[n]
                            except TypeError:
                                d = d[idx]
                if m:
                    rest = rest[m.end():]
                else:
                    raise ValueError('Unable to convert %r at %r' % (value, rest))
        # 此时rest为空字符串
        return d
```
`cfg_convert`方法是解析`cfg://`协议的默认转换器，只返回最后对应的值。
#### 6. `convert`
```python
    def convert(self, value):
        if not isinstance(value, ConvertingDict) and isinstance(value, dict):
            # 将一个dict类型的value转换为ConvertingDict，并将其configurator设置为self
            value = ConvertingDict(value)
            value.configurator = self
        elif not isinstance(value, ConvertingList) and isinstance(value, list):
            # 将一个list类型的value转换为ConvertingList，并将其configurator设置为self
            value = ConvertingList(value)
            value.configurator = self
        elif not isinstance(value, ConvertingTuple) and isinstance(value, tuple):
            # 将一个tuple类型的value转换为ConvertingTuple，并将其configurator设置为self
            value = ConvertingTuple(value)
            value.configurator = self
        elif isinstance(value, str):  # str for py3k
            # CONVERT_PATTERN = re.compile(r'^(?P<prefix>[a-z]+)://(?P<suffix>.*)$')
            m = self.CONVERT_PATTERN.match(value)
            # 如果value是一个字符串且满足正则表达式，对其进行转换
            if m:
                d = m.groupdict()
                prefix = d['prefix']
                converter = self.value_converters.get(prefix, None)
                # 获取prefix协议对应的转换方法，目前支持"ext://"和"cfg://"两种
                if converter:
                    suffix = d['suffix']
                    converter = getattr(self, converter)
                    # 调用对应协议的转换方法
                    value = converter(suffix)
        return value
```
`convert`方法是将`value`值转换为适当的类型并返回转换后的`value`。
#### 7. `configure_custom`
```python
    def configure_custom(self, config):
        # 取出特殊key"()"对应的用户自定义的工厂
        c = config.pop('()')
        # 如果不是可调用对象，则import倒入
        if not callable(c):
            c = self.resolve(c)
        # 通过特殊key "." 来定义用户自定义对象的属性值
        props = config.pop('.', None)
        # 检查有效的标识符
        kwargs = dict([(k, config[k]) for k in config if valid_ident(k)])
        # 实例化工厂对象或者调用它
        result = c(**kwargs)
        if props:
            # 为对象添加属性值
            for name, value in props.items():
                setattr(result, name, value)
        return result
```
`configure_custom`方法实现的是使用用户提供的工厂来配置一个对象。
#### 8. `as_tuple`
```python
    def as_tuple(self, value):
        if isinstance(value, list):
            value = tuple(value)
        return value
```
`as_tuple`方法是将列表转换为元组并返回。
### 四、`DictConfigurator`
#### 1. 定义
```python
class DictConfigurator(BaseConfigurator):
```
`DictConfigurator`类继承自`BaseConfigurator`类，实现的是使用类似字典的对象配置日志记录。
#### 2. `configure_formatter`
```python
    def configure_formatter(self, config):
        # 获取自定义formatter的工厂
        if '()' in config:
            factory = config['()']
            try:
                # 根据用户自定义的工厂生成formatter对象
                result = self.configure_custom(config)
            except TypeError as te:
                if "'format'" not in str(te):
                    raise
                # 参数名称从fmt更改为format。 
                # 这样代码可以用于较旧的Python版本（例如，Django）
                config['fmt'] = config.pop('format')
                config['()'] = factory
                result = self.configure_custom(config)
        else:
            fmt = config.get('format', None)
            dfmt = config.get('datefmt', None)
            style = config.get('style', '%')
            cname = config.get('class', None)
            if not cname:
                c = logging.Formatter
            else:
                c = _resolve(cname)
            result = c(fmt, dfmt, style)
        return result
```
`configure_formatter`方法是从一个字典配置中配置一个`formatter`。
#### 3. `configure_filter`
```python
    def configure_filter(self, config):
        if '()' in config:
            # 根据用户自定义的工厂生成filter对象
            result = self.configure_custom(config)
        else:
            name = config.get('name', '')
            result = logging.Filter(name)
        return result
```
`configure_filter`方法是从一个字典配置中配置一个`filter`。
#### 4. `add_filters`
```python
    def add_filters(self, filterer, filters):
        for f in filters:
            try:
                filterer.addFilter(self.config['filters'][f])
            except Exception as e:
                raise ValueError('Unable to add filter %r: %s' % (f, e))
```
`add_filters`方法是将`filters`列表添加到`filterer`对象的`filter`列表中。
#### 5. `configure_handler`
```python
    def configure_handler(self, config):
        config_copy = dict(config)  # 用于在出错的情况下进行恢复
        formatter = config.pop('formatter', None)
        if formatter:
            try:
                formatter = self.config['formatters'][formatter]
            except Exception as e:
                raise ValueError('Unable to set formatter %r: %s' % (formatter, e))
        level = config.pop('level', None)
        filters = config.pop('filters', None)
        if '()' in config:
            # 获取用户自定义的handler类
            c = config.pop('()')
            if not callable(c):
                c = self.resolve(c)
            factory = c
        else:
            cname = config.pop('class')
            klass = self.resolve(cname)
            # handler的特殊情况，指向另一个handler
            if issubclass(klass, logging.handlers.MemoryHandler) and 'target' in config:
                try:
                    th = self.config['handlers'][config['target']]
                    if not isinstance(th, logging.Handler):
                        config.update(config_copy)  # 恢复延迟的cfg
                        raise TypeError('target not configured yet')
                    config['target'] = th
                except Exception as e:
                    raise ValueError('Unable to set target handler %r: %s' % (config['target'], e))
            elif issubclass(klass, logging.handlers.SMTPHandler) and 'mailhost' in config:
                config['mailhost'] = self.as_tuple(config['mailhost'])
            elif issubclass(klass, logging.handlers.SysLogHandler) and 'address' in config:
                config['address'] = self.as_tuple(config['address'])
            factory = klass
        # 获取额外的属性
        props = config.pop('.', None)
        # 获取全部关键字参数
        kwargs = dict([(k, config[k]) for k in config if valid_ident(k)])
        try:
            # 初始化handler类
            result = factory(**kwargs)
        except TypeError as te:
            if "'stream'" not in str(te):
                raise
            # 参数名称从strm更改为stream
            # 这样代码可以用于较旧的Python版本(例如，Django)
            kwargs['strm'] = kwargs.pop('stream')
            result = factory(**kwargs)
        # 为handler添加formatter、设置level、添加filters
        if formatter:
            result.setFormatter(formatter)
        if level is not None:
            result.setLevel(logging._checkLevel(level))
        if filters:
            self.add_filters(result, filters)
        if props:
            # 为handler添加额外属性
            for name, value in props.items():
                setattr(result, name, value)
        return result
```
`configure_handler`方法是从一个字典配置中配置一个`handler`。
#### 6. `add_handlers`
```python
    def add_handlers(self, logger, handlers):
       for h in handlers:
            try:
                logger.addHandler(self.config['handlers'][h])
            except Exception as e:
                raise ValueError('Unable to add handler %r: %s' % (h, e))
```
`add_handlers`方法将`handlers`列表中的`handler`都添加到`logger`中。
#### 7. `common_logger_config`
```python
    def common_logger_config(self, logger, config, incremental=False):
        level = config.get('level', None)
        # 设置logger处理的日志级别
        if level is not None:
            logger.setLevel(logging._checkLevel(level))
        if not incremental:
            # 删除logger中所有的handler
            for h in logger.handlers[:]:
                logger.removeHandler(h)
            handlers = config.get('handlers', None)
            # 将配置中的handler列表添加到logger中
            if handlers:
                self.add_handlers(logger, handlers)
            filters = config.get('filters', None)
            # 将配置中的filter列表添加至logger中
            if filters:
                self.add_filters(logger, filters)
```
`common_logger_config`方法执行`root`和非`root`的`logger`的通用的配置。
#### 8. `configure_logger`
```python
    def configure_logger(self, name, config, incremental=False):
        logger = logging.getLogger(name)
        # 为logger设置日志级别、添加handlers和filters
        self.common_logger_config(logger, config, incremental)
        propagate = config.get('propagate', None)
        if propagate is not None:
            logger.propagate = propagate
```
`configure_logger`方法是从一个字典配置中配置一个非`root`的`logger`。
#### 9. `configure_root`
```python
    def configure_root(self, config, incremental=False):
        root = logging.getLogger()
        self.common_logger_config(root, config, incremental)
```
`configure_root`方法是从一个字典配置中配置一个`root`的`logger`。
#### 10. `configure`
前面介绍了那么多，终于到今天的重点了，这个函数非常长，实现的功能就是从一个`dict`形式的配置中读取并生成相应的日志处理对象。
```python
    def configure(self):
        config = self.config
        # 字典中必须有 version 且值必须为1
        if 'version' not in config:
            raise ValueError("dictionary doesn't specify a version")
        if config['version'] != 1:
            raise ValueError("Unsupported version: %s" % config['version'])
        # 获取incremental的值，是否增量配置
        incremental = config.pop('incremental', False)
        EMPTY_DICT = {}
        # 获取线程锁
        logging._acquireLock()
        try:
            if incremental:
                # 如果设置为增量配置
                # 获取key为handlers对应的dict配置
                handlers = config.get('handlers', EMPTY_DICT)
                for name in handlers:
                    if name not in logging._handlers:
                        raise ValueError('No handler found with name %r' % name)
                    else:
                        try:
                            handler = logging._handlers[name]
                            handler_config = handlers[name]
                            level = handler_config.get('level', None)
                            # 为handler重新设置日志级别
                            if level:
                                handler.setLevel(logging._checkLevel(level))
                        except Exception as e:
                            raise ValueError('Unable to configure handler %r: %s' % (name, e))
                # 获取key为loggers对应的dict配置
                loggers = config.get('loggers', EMPTY_DICT)
                for name in loggers:
                    try:
                        # 设置logger
                        self.configure_logger(name, loggers[name], True)
                    except Exception as e:
                        raise ValueError('Unable to configure logger %r: %s' % (name, e))
                root = config.get('root', None)
                if root:
                    try:
                        # 设置root logger
                        self.configure_root(root, True)
                    except Exception as e:
                        raise ValueError('Unable to configure root logger: %s' % e)
            else:
                # 如果设置为非增量设置，即 incremental=False
                disable_existing = config.pop('disable_existing_loggers', True)

               # 清除logging._handlers字典和logging._handlerList列表
                logging._handlers.clear()
                del logging._handlerList[:]

                # Do formatters first - they don't refer to anything else
                # 首先获取key为formatters的字典配置，因为它没有引用任何对象
                formatters = config.get('formatters', EMPTY_DICT)
                for name in formatters:
                    try:
                        # 根据formatter的配置生成formatter
                        formatters[name] = self.configure_formatter(formatters[name])
                    except Exception as e:
                        raise ValueError('Unable to configure formatter %r: %s' % (name, e))
                # Next, do filters - they don't refer to anything else, either
                # 然后获取key为filters的字典配置，它也没有引用任何对象
                filters = config.get('filters', EMPTY_DICT)
                for name in filters:
                    try:
                        # 根据filters的配置生成filter
                        filters[name] = self.configure_filter(filters[name])
                    except Exception as e:
                        raise ValueError('Unable to configure filter %r: %s' % (name, e))

                # Next, do handlers - they refer to formatters and filters
                # As handlers can refer to other handlers, sort the keys
                # to allow a deterministic order of configuration
                # 然后获取key为handlers的dict配置，它引用了formatters和filters，由于handler可以引用其他handler，因此对key进行排序以保证正确的配置顺序
                handlers = config.get('handlers', EMPTY_DICT)
                deferred = []
                for name in sorted(handlers):
                    try:
                        handler = self.configure_handler(handlers[name])
                        handler.name = name
                        handlers[name] = handler
                    except Exception as e:
                        if 'target not configured yet' in str(e):
                            deferred.append(name)
                        else:
                            raise ValueError('Unable to configure handler %r: %s' % (name, e))

                # Now do any that were deferred
                for name in deferred:
                    try:
                        handler = self.configure_handler(handlers[name])
                        handler.name = name
                        handlers[name] = handler
                    except Exception as e:
                        raise ValueError('Unable to configure handler %r: %s' % (name, e))

                # Next, do loggers - they refer to handlers and filters

                # existing和child_loggers的意义同fileConfig()中的一样
                root = logging.root
                existing = list(root.manager.loggerDict.keys())
                existing.sort()
                child_loggers = []
                # 然后获取key为loggers的dict配置
                loggers = config.get('loggers', EMPTY_DICT)
                for name in loggers:
                    if name in existing:
                        i = existing.index(name) + 1  # 寻找 name 在existing 中的索引
                        prefixed = name + "."
                        pflen = len(prefixed)
                        num_existing = len(existing)
                        while i < num_existing:
                            if existing[i][:pflen] == prefixed:
                                child_loggers.append(existing[i])
                            i += 1
                        existing.remove(name)
                    try:
                        self.configure_logger(name, loggers[name])
                    except Exception as e:
                        raise ValueError('Unable to configure logger %r: %s' % (name, e))

                # 这里的意义同fileConfig()中的一样
                _handle_existing_loggers(existing, child_loggers, disable_existing)

                # And finally, do the root logger
                # 最后，配置 name=root 的logger
                root = config.get('root', None)
                if root:
                    try:
                        # 配置root  logger
                        self.configure_root(root)
                    except Exception as e:
                        raise ValueError('Unable to configure root logger: %s' % e)
        finally:
            # 是否线程锁
            logging._releaseLock()
```
从对源码的注释中可知，`configure`方法读取`dict`配置，依次读取`key`为`formatters`, `filters`, `handlers`和`loggers`的`dict`配置，依次调用其创建函数生成对应的日志对象，从而将这四个对象连接在一起。
#### 11. `dictConfig`
```python
dictConfigClass = DictConfigurator

def dictConfig(config):
    """Configure logging using a dictionary."""
    dictConfigClass(config).configure()
```
此时，加载`dict`配置的主函数`dictConfig()`终于露面了，这是一个模块级函数，接收的`config`参数是个字典，调用的是`dictConfigClass`类的`configure`方法，默认的`dictConfigClass`是`DictConfigurator`，那默认调用的`configure`方法，也就是我们上面分析的那个方法。
### 五、总结
本篇文章中我们详细介绍了`dict`配置的配置方式、解析方法，并深入解析了读取配置的源码，已经非常清楚其内部的真正原理，我们还可以使用自定义的`filter`, `handler`, `formatter`类进行日志配置。
