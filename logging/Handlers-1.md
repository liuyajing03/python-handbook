---
title: /python/logging源码解析之Handlers(一)
urlname: logging/Handlers/1
categories: python
tags:
  - python
description: python
p: /python/logging源码解析之Handlers(一)
date: 2018-03-17 14:42:55
updated: 2018-03-17 14:42:55
---
### 一、引言
在`{% post_link python/logging源码解析之Handler Handler %}`这篇文章中，我们主要介绍了两个基本的Handler类`StreamHandler`和`FileHandler`，但它们远不能满足我们的需求，因此在`logging`模块中我们又单独定义了`14`个`Handler`类，我们分三篇文章详细介绍它们。

源码路径: 
在模块中有如下常量:
```python
DEFAULT_TCP_LOGGING_PORT = 9020
DEFAULT_UDP_LOGGING_PORT = 9021
DEFAULT_HTTP_LOGGING_PORT = 9022
DEFAULT_SOAP_LOGGING_PORT = 9023
SYSLOG_UDP_PORT = 514
SYSLOG_TCP_PORT = 514

_MIDNIGHT = 24 * 60 * 60  # 一天的秒数
```
### 二、`BaseRotatingHandler`
> `BaseRotatingHandler`类是用于处理滚动日志文件的`Handler`的基类。 不应该直接用于实例化，而是使用`RotatingFileHandler`或`TimedRotatingFileHandler`这两个类。
#### 1. 定义
```python
class BaseRotatingHandler(logging.FileHandler):
```
`BaseRotatingHandler`类是继承自`logging.FileHandler`类，也是我们之前介绍过的。
#### 2. `__init__`
```python
    def __init__(self, filename, mode, encoding=None, delay=False):
        logging.FileHandler.__init__(self, filename, mode, encoding, delay)
        self.mode = mode
        self.encoding = encoding
        self.namer = None
        self.rotator = None
```
`__init__`方法使用指定的`filename`进行流式日志记录。
#### 3. `emit`
```python
    def emit(self, record):
        try:
            if self.shouldRollover(record):
                self.doRollover()
            logging.FileHandler.emit(self, record)
        except Exception:
            self.handleError(record)
```
`emit`方法首先检测是否应该滚动日志，如果需要，则处理日志文件的滚动，否则继续，然后将`record`输出到文件。方法`shouldRollover`与`doRollover`在此类中均没有定义，故需要在其子类中定义。
#### 4. `rotation_filename`
```python
    def rotation_filename(self, default_name):
        # 如果属性namer不是可调用的，则直接使用default_name的值
        if not callable(self.namer):
            result = default_name
        else:
            # 返回处理后的值
            result = self.namer(default_name)
        return result
```
`rotation_filename`方法返回滚动日志时修改日志文件的文件名，提供此方法以便可以提供自定义文件名。
#### 5. `rotate`
```python
    def rotate(self, source, dest):
        # source：源文件名。 这通常是基本文件名，例如'test.log中'
        # dest：目标文件名。 这通常是滚动后的，例如，'test.log.1'。
        if not callable(self.rotator):
            # Issue 18940: A file may not have been created if delay is True.
            # 如果rotator属性是不可调用，则直接重命名
            if os.path.exists(source):
                os.rename(source, dest)
        else:
            self.rotator(source, dest)
```
`rotate`方法实现滚动日志时更改源日志文件的日志名。
### 三、`RotatingFileHandler`
> `RotatingFileHandler`类用于记录到一组文件的`Handler`，当当前日志文件达到特定大小时，该文件从一个文件切换到下一个文件;
> 默认情况下，日志文件无限增长，如果你使用`RotatingFileHandler`作为`Handler`的类，那可以指定`maxBytes`和`backupCount`的值，以允许文件以预定大小进行滚动;
> 只要当前日志文件的长度接近`maxBytes`，就会发生滚动。 如果`backupCount>=1`，系统将连续创建与基本文件具有相同路径名的新文件，但附加扩展名“.1”，“.2”等;
> 例如，如果`backupCount=5`，基本文件名为“app.log”，则会显示“app.log”，“app.log.1”，“app.log.2”，...“app.log.5”。 日志正在写入的文件始终是“app.log”——当它被填满时，它被关闭并重命名为“app.log.1”，如果文件“app.log.1”，“app.log.2” 等存在，则分别重命名为”app.log.2“，”app.log.3“等;
> 如果`maxBytes`为零，则不会发生翻转。

#### 1. 定义
```python
class RotatingFileHandler(BaseRotatingHandler):
```
可见，`RotatingFileHandler`类是继承自我们上面介绍的`BaseRotatingHandler`类。
#### 2. `__init__`
```python
    def __init__(self, filename, mode='a', maxBytes=0, backupCount=0, encoding=None, delay=False):
        # 如果需要翻滚日志，则使用其他模式没有意义。 
        # 例如，如果指定了'w'，则先前运行的日志将丢失，因为日志文件将在每次运行时被截断。
        if maxBytes > 0:
            mode = 'a'
        BaseRotatingHandler.__init__(self, filename, mode, encoding, delay)
        self.maxBytes = maxBytes
        self.backupCount = backupCount
```
`__init__`方法主要调用了`BaseRotatingHandler`类的`__init__`方法，但多了两个属性`maxBytes`和`backupCount`。
#### 3. `doRollover`
```python
    def doRollover(self):
        if self.stream:
            # 关闭日志流
            self.stream.close()
            self.stream = None
        if self.backupCount > 0:
            # backupCount>0
            # 比如backupCount=5，遍历顺序为[4,3,2,1] 
            for i in range(self.backupCount - 1, 0, -1):
                sfn = self.rotation_filename("%s.%d" % (self.baseFilename, i))
                dfn = self.rotation_filename("%s.%d" % (self.baseFilename, i + 1))
                # 将dfn文件删除，将 sfn 的文件名命名为 dfn
                if os.path.exists(sfn):
                    if os.path.exists(dfn):
                        os.remove(dfn)
                    os.rename(sfn, dfn)
            # 将现在的日志文件命名为后缀为".1"的文件名
            # 也就是“app.log”命名为"app.log.1"
            dfn = self.rotation_filename(self.baseFilename + ".1")
            if os.path.exists(dfn):
                os.remove(dfn)
            self.rotate(self.baseFilename, dfn)
        if not self.delay:
            self.stream = self._open()
```
`doRollover`方法是指定日志滚动的。
#### 4. `shouldRollover`
```python
    def shouldRollover(self, record):
        if self.stream is None:  # delay设置为True
            self.stream = self._open()
        if self.maxBytes > 0:  # 是否滚动日志
            msg = "%s\n" % self.format(record)
            self.stream.seek(0, 2)  # due to non-posix-compliant Windows feature
            if self.stream.tell() + len(msg) >= self.maxBytes:
                return 1
        return 0
```
`shouldRollover`方法用来判断是否应该滚动日志，通过查看指定的`record`是否会导致文件超出我们的大小限制。
### 四、`TimedRotatingFileHandler`
> 用于记录到文件的`Handler`，按照时间间隔滚动日志。如果`backupCount>0`，则在完成翻转时，只保留`backupCount`文件——删除最旧的文件。

#### 1. 定义
```python
class TimedRotatingFileHandler(BaseRotatingHandler):
```
可见，`TimedRotatingFileHandler`类也是继承自我们上面介绍的`BaseRotatingHandler`类。
#### 2. `__init__`
```python
    def __init__(self, filename, when='h', interval=1, backupCount=0, encoding=None, delay=False, utc=False, atTime=None):
        BaseRotatingHandler.__init__(self, filename, 'a', encoding, delay)
        self.when = when.upper()
        self.backupCount = backupCount
        self.utc = utc
        self.atTime = atTime
        # 计算实际滚动时间间隔，单位是秒，并且设置发生滚动时使用的文件名后缀。
        # 参数`when`，支持如下几种类型:
        # S  - 秒
        # M  - 分钟
        # H  - 小时
        # D  - 天
        # MIDNIGHT - 午夜滚动
        # W {0-6}  - 在一周的某一天滚动; 0 - 星期一
        # 不区分大小写
        if self.when == 'S':
            self.interval = 1  # 1秒
            self.suffix = "%Y-%m-%d_%H-%M-%S"
            self.extMatch = r"^\d{4}-\d{2}-\d{2}_\d{2}-\d{2}-\d{2}(\.\w+)?$"
        elif self.when == 'M':
            self.interval = 60  # 一分
            self.suffix = "%Y-%m-%d_%H-%M"
            self.extMatch = r"^\d{4}-\d{2}-\d{2}_\d{2}-\d{2}(\.\w+)?$"
        elif self.when == 'H':
            self.interval = 60 * 60  # 一个小时
            self.suffix = "%Y-%m-%d_%H"
            self.extMatch = r"^\d{4}-\d{2}-\d{2}_\d{2}(\.\w+)?$"
        elif self.when == 'D' or self.when == 'MIDNIGHT':
            self.interval = 60 * 60 * 24  # 一天
            self.suffix = "%Y-%m-%d"
            self.extMatch = r"^\d{4}-\d{2}-\d{2}(\.\w+)?$"
        elif self.when.startswith('W'):
            self.interval = 60 * 60 * 24 * 7  # 一个星期
            if len(self.when) != 2:
                # 必须带上0-6之间的数字，表示周几，否则报错
                raise ValueError("You must specify a day for weekly rollover from 0 to 6 (0 is Monday): %s" % self.when)
            if self.when[1] < '0' or self.when[1] > '6':
                raise ValueError("Invalid day specified for weekly rollover: %s" % self.when)
            self.dayOfWeek = int(self.when[1])
            self.suffix = "%Y-%m-%d"
            self.extMatch = r"^\d{4}-\d{2}-\d{2}(\.\w+)?$"
        else:
            raise ValueError("Invalid rollover interval specified: %s" % self.when)

        self.extMatch = re.compile(self.extMatch, re.ASCII)
        self.interval = self.interval * interval  # 乘以要求的单位
        # The following line added because the filename passed in could be a
        # path object (see Issue #27493), but self.baseFilename will be a string
        filename = self.baseFilename
        if os.path.exists(filename):
            # 获取其最后一次修改的时间
            t = os.stat(filename)[ST_MTIME]
        else:
            # 如果日志不存在，则获取当前时间
            t = int(time.time())
        self.rolloverAt = self.computeRollover(t)
```
`__init__`方法首先调用的`BaseRotatingHandler`类的`__init__`方法，然后初始化日志滚动的时间间隔及滚动后的文件后缀，而且还调用了`computeRollover`方法来获取下一次日志滚动的时间。
#### 3. `computeRollover`
```python
    def computeRollover(self, currentTime):
        result = currentTime + self.interval
        # 如果我们设置的是"midnight"或"w"滚动，则interval已知。
        # 我们需要弄清楚的是下一个滚动日志是什么时候。 
        # 换句话说，如果你在"midnight"滚动，那么你的基本间隔是1天，但你想在午夜开始那一天，而不是现在。 
        # 我们必须获取rolloverAt值，以便在正确的时间触发第一次滚动。 之后，定期间隔将负责其余部分。 请注意，此代码不关心闰秒。
        if self.when == 'MIDNIGHT' or self.when.startswith('W'):
            # This could be done with less code, but I wanted it to be clear
            if self.utc:
                t = time.gmtime(currentTime)
            else:
                t = time.localtime(currentTime)
            currentHour = t[3]
            currentMinute = t[4]
            currentSecond = t[5]
            currentDay = t[6]
            
            # 计算滚动时间atTime的的秒数，比如(atTime=13:45)
            if self.atTime is None:
                # _MIDNIGHT = 24 * 60 * 60
                rotate_ts = _MIDNIGHT
            else:
                rotate_ts = ((self.atTime.hour * 60 + self.atTime.minute) * 60 + self.atTime.second)

            # r 是从现在到下一次滚动之间剩余的秒数
            r = rotate_ts - ((currentHour * 60 + currentMinute) * 60 + currentSecond)
            if r < 0:
                # 滚动时间早于当前时间（例如 self.rotateAt是13:45，现在是14:15），明天滚动，此时 r  就是多于今天日志滚动时间的秒数。
                r += _MIDNIGHT
                # 明天星期几 0-6
                currentDay = (currentDay + 1) % 7
            # 获取下一次滚动日志的时间
            result = currentTime + r
   
            # 如果我们在指定的某一天滚动日志，要加上直到下一次滚动的天数，但是因为我们刚刚计算的时间是从第二天开始的，所以偏移1天。 以下有三种情况(一周0-6为一个间隔)：
            # 1) 滚动日是今天，在这种情况下，什么都不做
            # 2) 滚动的日期在这一间隔的后面，即今天是第2天(星期三)，滚动是在第6天(星期日))。下一次翻滚的日期只是6  -  2  -  1或3。
            # 3) 滚动日期在这一间隔的前面，即今天是第5天(星期六)，滚动的日期在第3天(星期四)，已经过了一天，滚动日是6 - 5 + 3，或4. 在这种情况下， 它是当前一周剩余的天数(1)加上下一周直到滚动日的天数(3)。
            # 上面2)和3)中描述的计算需要增加一天。这是因为上述时间计算将我们带到这一天的午夜，即第二天的开始。
            if self.when.startswith('W'):
                day = currentDay  # 0 is Monday
                if day != self.dayOfWeek:
                    if day < self.dayOfWeek:
                        daysToWait = self.dayOfWeek - day
                    else:
                        daysToWait = 6 - day + self.dayOfWeek + 1
                    newRolloverAt = result + (daysToWait * (60 * 60 * 24))
                    if not self.utc:
                        # 验证 t  和 newRolloverAt 是否为夏令时
                        dstNow = t[-1]
                        dstAtRollover = time.localtime(newRolloverAt)[-1]
                        if dstNow != dstAtRollover:
                            if not dstNow:  # DST kicks in before next rollover, so we need to deduct an hour
                                addend = -3600
                            else:  # DST bows out before next rollover, so we need to add an hour
                                addend = 3600
                            newRolloverAt += addend
                    result = newRolloverAt
        return result
```
`computeRollover`方法根据指定的时间计算日志的滚动时间，可见`atTime`属性仅对`w`和`midnight`有效。这个函数逻辑稍微有些复杂，总结如下：
- 当`when!=midnight`且`when!=w{0-6}`，则直接加上`interval`属性的值返回;
- 当`when=midnight`或`when=w{0-6}`时，首先，根据`atTime`时间计算是当天还是第二天滚动日志：
    (1). 没有设置`atTime`属性，则下次滚动日志的时间为第二天的`00:00:00`;
    (2). 如果设置了`atTime`属性，但此时已经过了`atTime`设置的时间，那滚动时间就是第二天的`atTime`时间，否则就是今天的`atTime`时间;
    同时我们会计算其是一周内的星期几，记为`currentDay`，是`0-6`之间的整数;
然后，如果`when=midnight`，则直接返回计算的值即可；但是对于`when!=w{0-6}`，我们还要讨论三种情况:
    设置的滚动星期几是`dayOfWeek`属性的值，也是`0-6`之间的整数;
    (1). 如果`currentDay = dayOfWeek`是一致的，那就直接返回;
    (2). 如果`currentDay > dayOfWeek`，则需要等待的天数是`6 - currentDay + dayOfWeek + 1`;
    (3). 如果`currentDay < dayOfWeek`，则需要等待的天数是`dayOfWeek - currentDay`;
    最后还要考虑夏令营时间的设置;
    所以，要在`when=midnight`求得的时间的基础上还要加上要等待的天数，就是`when=midnight`时下一个滚动日志的时间。

#### 4. `shouldRollover`
```python
    def shouldRollover(self, record):
        t = int(time.time())
        if t >= self.rolloverAt:
            return 1
        return 0
```
`shouldRollover`方法用来判断是否应发生翻转，`record`参数未使用，因为我们只是比较时间，但它是必需的，因为要确保方法签名是相同的。
#### 5. `getFilesToDelete`
```python
    def getFilesToDelete(self):
        dirName, baseName = os.path.split(self.baseFilename)
        fileNames = os.listdir(dirName)
        result = []
        prefix = baseName + "."
        plen = len(prefix)
        for fileName in fileNames:
            if fileName[:plen] == prefix:
                suffix = fileName[plen:]
                if self.extMatch.match(suffix):
                    result.append(os.path.join(dirName, fileName))
        if len(result) < self.backupCount:
            result = []
        else:
            result.sort()
            result = result[:len(result) - self.backupCount]
        return result
```
`getFilesToDelete`方法用来获取滚动时确定要删除的文件，比之前使用`glob.glob()`方法更具体。
#### 6. `doRollover`
```python
    def doRollover(self):
        if self.stream:
            self.stream.close()
            self.stream = None
        
        currentTime = int(time.time())
        dstNow = time.localtime(currentTime)[-1]
        # 获得此序列开始的时间并转换为TimeTuple
        t = self.rolloverAt - self.interval
        if self.utc:
            timeTuple = time.gmtime(t)
        else:
            timeTuple = time.localtime(t)
            dstThen = timeTuple[-1]
            if dstNow != dstThen:
                if dstNow:
                    addend = 3600
                else:
                    addend = -3600
                timeTuple = time.localtime(t + addend)
        dfn = self.rotation_filename(self.baseFilename + "." + time.strftime(self.suffix, timeTuple))
        if os.path.exists(dfn):
            os.remove(dfn)
        self.rotate(self.baseFilename, dfn)
        if self.backupCount > 0:
            for s in self.getFilesToDelete():
                os.remove(s)
        if not self.delay:
            self.stream = self._open()
        # 重新计算下一次滚动日志的时间
        newRolloverAt = self.computeRollover(currentTime)
        while newRolloverAt <= currentTime:
            newRolloverAt = newRolloverAt + self.interval
        # If DST changes and midnight or weekly rollover, adjust for this.
        if (self.when == 'MIDNIGHT' or self.when.startswith('W')) and not self.utc:
            dstAtRollover = time.localtime(newRolloverAt)[-1]
            if dstNow != dstAtRollover:
                if not dstNow:  # DST kicks in before next rollover, so we need to deduct an hour
                    addend = -3600
                else:  # DST bows out before next rollover, so we need to add an hour
                    addend = 3600
                newRolloverAt += addend
        self.rolloverAt = newRolloverAt
```
`doRollover`方法实现日志的滚动:
- 日志滚动时会在文件名后附加日期/时间戳，以文件的开始而不是当前时间命名文件;
- 如果设置了`backupCount`，那么我们必须获取匹配文件名列表，对它们进行排序并删除日期/时间戳最老的文件名;
- 最后重新计算下一次日志滚动的时间。

### 

