---
title: Python代码报错记录集
description:
slug: python_error_notes
date: 2024-08-14
image: 
categories:
    - Python
tags:
    - Python
    - error_notes
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---

### 1. ModuleNotFoundError: No module named 'xxx'

```python
    from PIL import Image, ImageFilter
    ModuleNotFoundError: No module named 'PIL'

    # 缺少依赖PIL，PIL已被Pillow替代,安装
    pip install Pillow
```

``` python
    import pytesseract
    ModuleNotFoundError: No module named 'pytesseract'

# 缺少依赖pytesseract,安装
    pip install pytesseract
```
### 2. pytesseract.pytesseract.TesseractNotFoundError: tesseract is not installed or it's not in your PATH
**没有安装tesseract-ocr,可以到这里 https://github.com/UB-Mannheim/tesseract/wiki 下载安装，并自行配置环境变量指向安装目录**
```python
Traceback (most recent call last):
  File "D:\Python\Python312\Lib\site-packages\pytesseract\pytesseract.py", line 275, in run_tesseract
    proc = subprocess.Popen(cmd_args, **subprocess_args())
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "D:\Python\Python312\Lib\subprocess.py", line 1026, in __init__
    self._execute_child(args, executable, preexec_fn, close_fds,
  File "D:\Python\Python312\Lib\subprocess.py", line 1538, in _execute_child
    hp, ht, pid, tid = _winapi.CreateProcess(executable, args,
                       ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
FileNotFoundError: [WinError 2] 系统找不到指定的文件。

During handling of the above exception, another exception occurred:
...
 File "D:\Python\Python312\Lib\site-packages\pytesseract\pytesseract.py", line 486, in image_to_string
    return {
           ^
  File "D:\Python\Python312\Lib\site-packages\pytesseract\pytesseract.py", line 489, in <lambda>
    Output.STRING: lambda: run_and_get_output(*args),
                           ^^^^^^^^^^^^^^^^^^^^^^^^^
  File "D:\Python\Python312\Lib\site-packages\pytesseract\pytesseract.py", line 352, in run_and_get_output
    run_tesseract(**kwargs)
  File "D:\Python\Python312\Lib\site-packages\pytesseract\pytesseract.py", line 280, in run_tesseract
    raise TesseractNotFoundError()
pytesseract.pytesseract.TesseractNotFoundError: tesseract is not installed or it's not in your PATH. See README file for more information.
```

### 3. Please make sure the TESSDATA_PREFIX environment variable is set to your "tessdata" directory. Failed loading language \'chi_sim\' Tesseract couldn\'t load any languages! Could not initialize tesseract.
**到这里下载各种语言的训练数据。 https://github.com/tesseract-ocr/tessdata**

```python
 File "D:\Python\Python312\Lib\site-packages\pytesseract\pytesseract.py", line 486, in image_to_string
    return {
           ^
  File "D:\Python\Python312\Lib\site-packages\pytesseract\pytesseract.py", line 489, in <lambda>
    Output.STRING: lambda: run_and_get_output(*args),
                           ^^^^^^^^^^^^^^^^^^^^^^^^^
  File "D:\Python\Python312\Lib\site-packages\pytesseract\pytesseract.py", line 352, in run_and_get_output
    run_tesseract(**kwargs)
  File "D:\Python\Python312\Lib\site-packages\pytesseract\pytesseract.py", line 284, in run_tesseract
    raise TesseractError(proc.returncode, get_errors(error_string))
pytesseract.pytesseract.TesseractError: (1, 'Error opening data file D:\\Program Files\\Tesseract-OCR/tessdata/chi_sim.traineddata Please make sure the TESSDATA_PREFIX environment variable is set to your "tessdata" directory. Failed loading language \'chi_sim\' Tesseract couldn\'t load any languages! Could not initialize tesseract.')
```

### 4. Exception redis.clients.jedis.exceptions.JedisDataException: value sent to redis cannot be null
``` java
// 原始代码这样写的
    jedis.rpush(key, null, "0", "0");
```
原因是尝试向 Redis 中写入了空值（null）。根据 Redis 的设计，它不允许存储 null 值。
在代码中，当没有查询到数据时，会尝试向 Redis 写入 null, "0", "0" 这三个值。其中 null 是不允许的，因此触发了异常。
要解决这个问题，可以考虑以下几种方法：
不写入任何值：如果查询不到数据，则不向 Redis 写入任何内容。
写入特殊值：用一个特殊值代替 null，比如空字符串 "" 或者 -1 表示无效或未找到的数据。
这里提供一种修改方案，即使用空字符串替换 null: