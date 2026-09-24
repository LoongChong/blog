---
title: Logger 日志基本用法
categories:
  - Agent开发
description: 
comments: true
toc: true
mathjax: false
copyright: true
date: 2026-09-24 9：45
tags:
  - 日志系统
cover: /img/lachang.png
---

# Logger基本用法

## Logger的基本组成和流程
**Logger系统主要由四个部分组成：**

| 组件           | 作用                                           |
| -------------- | ---------------------------------------------- |
| Logger         | 日志记录器，产生日志                           |
| Handler        | 处理器，决定日志往哪里输出（屏幕？文件？）     |
| Formatter      | 格式器，规定日志长什么样（时间？级别？内容？） |
| Filter（可选） | 过滤器，筛选哪些日志能过                       |

简单理解为：

> Logger产生日志 → Handler接收日志 → Formatter美化日志 → 输出到你想要的地方。

## Logger

> Logger对象就是"日志生产者"，专门负责产生日志消息。

### 创建Logger的方法：

```python
import logging
# 创建Logger对象
logger = logging.getLogger('my_logger')
```

### 设置Logger的日志级别：

> **一句话：它设的是这个 logger 的“最低放行门槛”，也就是阈值（threshold）。**
>
> 准确地说，是给这个 logger 设了一个**整数级别号**，这个号充当“过滤器”：**只有级别号 ≥ 这个值的日志才会被放行**，低于它的直接丢弃。
>
> ```
>    0      10      20      30      40       50
> NOTSET   DEBUG   INFO   WARNING  ERROR   CRITICAL
> ```

```python
logger.setLevel(logging.DEBUG)
```

### Logger的常用方法：

```python
logger.debug('这是调试信息')
logger.info('这是普通信息')
logger.warning('这是警告信息')
logger.error('这是错误信息')
logger.critical('这是严重错误信息')
logger.exception('这是捕获异常信息，包含堆栈')  # 通常在except块中使用

```

## Handler

> Handler对象是“日志搬运工”，负责把日志消息送到指定的地方

### 常用的Handler类型：

|       Handler类型        |            作用            |
| :----------------------: | :------------------------: |
|      StreamHandler       |  输出到控制台（标准输出）  |
|       FileHandler        |         输出到文件         |
|   RotatingFileHandler    | 输出到文件，支持按大小切割 |
| TimedRotatingFileHandler | 输出到文件，支持按时间分割 |
|       SMTPHandler        |         输出到邮件         |
|      SysLogHandler       |       输出到系统日志       |
|       HTTPHandler        |      输出到HTT服务器       |

### 创建Handler

```python
# 输出到控制台
console_handler = logging.StreamHandler()
console_handler.setLevel(logging.DEBUG)

# 输出到文件
file_handler = logging.FileHandler('app.log')
file_handler.setLevel(logging.ERROR)

# 输出到按大小切割的文件（最多10MB，备份5个文件）
rotating_handler = logging.handlers.RotatingFileHandler(
    'app.log', maxBytes=10*1024*1024, backupCount=5
)

# 按时间切割（每天午夜切割，保留7天）
timed_handler = logging.handlers.TimedRotatingFileHandler(
    'app.log', when='midnight', interval=1, backupCount=7
)

```

## Formatter详解

>  Formatter就是"日志化妆师"，负责给日志加上漂亮的格式。

### 格式化占位符:

|    占位符     |                 含义                  |
| :-----------: | :-----------------------------------: |
|  %(asctime)s  | 日期时间，如：2023-01-01 12:00:00,123 |
|   %(name)s    |              记录器名称               |
| %(levelname)s |             日志级别名称              |
|  %(message)s  |             日志消息内容              |

### 创建Formatter的例子：

```python
# 一般格式
formatter = logging.Formatter('%(asctime)s - %(name)s - %(levelname)s - %(message)s')

# 详细格式（带文件名、行号、函数名）
detailed_formatter = logging.Formatter(
    '%(asctime)s [%(levelname)s] %(filename)s:%(lineno)d %(funcName)s() - %(message)s'
)

# 精确时间格式（带毫秒）
precise_formatter = logging.Formatter(
    '%(asctime)s.%(msecs)03d - %(levelname)s - %(message)s',
    datefmt='%Y-%m-%d %H:%M:%S'
)

```

### 设置Formatter到Handler：

```python
console_handler.setFormatter(formatter)
file_handler.setFormatter(detailed_formatter)

```

## 完整实例

```python
import logging

# 1. 创建Logger对象
logger = logging.getLogger('my_logger')
logger.setLevel(logging.DEBUG)  # 设置日志级别

# 2. 创建Handler
console_handler = logging.StreamHandler()
file_handler = logging.FileHandler('app.log')

# 3. 设置Handler的日志级别
console_handler.setLevel(logging.DEBUG)
file_handler.setLevel(logging.ERROR)

# 4. 创建Formatter并绑定到Handler
formatter = logging.Formatter('%(asctime)s - %(name)s - %(levelname)s - %(message)s')
console_handler.setFormatter(formatter)
file_handler.setFormatter(formatter)

# 5. 将Handler添加到Logger
logger.addHandler(console_handler)
logger.addHandler(file_handler)

# 6. 写日志
logger.debug('调试信息')
logger.info('普通信息')
logger.warning('警告信息')
logger.error('错误信息')
logger.critical('严重错误信息')

```

## Logger日志级别详细解释

| 等级     | 数值 | 说明                             | 使用场景                   |
| -------- | ---- | -------------------------------- | -------------------------- |
| DEBUG    | 10   | 调试细节                         | 开发阶段，详细追踪问题     |
| INFO     | 20   | 普通流程信息                     | 常规操作，程序运行状态     |
| WARNING  | 30   | 警告（程序还能跑，但可能出问题） | 需要注意但不是错误的情况   |
| ERROR    | 40   | 错误（程序某些功能无法继续）     | 运行出错，但不会导致崩溃   |
| CRITICAL | 50   | 严重错误（整个程序可能崩掉）     | 紧急情况，程序无法继续运行 |

Logger只会处理大于等于设定等级的日志信息。

> **重要提示**：默认Logger等级是WARNING，所以如果不设置，DEBUG和INFO是不会显示的！

