---
title: "略谈python中如何将图片转为pdf"
date: 2023-11-13T21:19:08+08:00
tags: ['pdf','python','img2pdf','图片转pdf']
categories: ['code']
top: true
---

# 略谈python中如何将图片转为pdf #

##  1.前言

有的时候，我们有多个目录且存在大量的图片，如图

![](/img/1113/00.png)

如果我们想要将每个目录合并为一个文件，一般人们会将其压缩为压缩包，但是这样子操作十分不便，压缩之后压缩包也会分散在各个目录，十分的~~蓝廋~~难受，作为一名~~闲的蛋疼~~喜欢折腾又~~不会搜索~~独立自主的小白，自然是自己写代码啦(主要是之前找到的软件太难用又有bug，只能自己在网上找资料了)

## 2.代码

### 2.1总览

让我们看看项目的代码

这是`main.py`的代码

```python
import  api as a
def main():
    _root_dir = input('输入根目录')
    re_dir_list = a._find_all(_root_dir)
    # print(re_dir_list)
    for dir in re_dir_list:
        title = a._get_title(dir)
        a._i2p(_root_dir,title,dir)

if __name__ == '__main__':
    main()
```

单独看这段代码毫无作用，重点是在下面的`api.py`的代码

```python
import os
import img2pdf
def _t_files(path):# 查找单一目录下所有文件与子目录
    dirs = []
    files = []
    for item in os.scandir(path):
        if item.is_dir():
          dirs.append(item.path)
          pass
        elif item.is_file():
          files.append(item.path)
        pass
    return dirs,files

def _find_all(dir): # 递归查询最底层目录
    temp_dirs,temp_files = _t_files(dir)
    re_list = []
    if temp_dirs == []:
        re_list.append(dir)
        pass
    else:
        for dirs in temp_dirs:
            re_list+=_find_all(dirs)
            pass
        pass
    return re_list

def _get_title(dir): # 获得最底层目录名
    t_title = dir.split('\\')[-1]
    return t_title

def _i2p(root_dir,title,dir): # 图片转pdf
    if os.path.exists(root_dir+'\\pdf') == False:
        os.mkdir(root_dir+'\\pdf')
    t_dir,file = _t_files(dir)
    file_name = root_dir+'\\pdf\\'+title+'.pdf'
    print(title)
    with open(file_name,'wb') as f:
        f.write(img2pdf.convert(file))

```

这是整个项目的核心(废话，整个项目就两个文件)，接下来让我们详细将这些代码细细刨析一下。

---

### 2.2详解

整个项目分为分割可复用功能的`api.py`与整合功能的`main.py`，项目存在的优点与不足如下

---

#### 优点~~(就别在自己脸上贴金了吧)~~

可复用

思路清晰

可读程度高

#### 缺点/不足

未进行报错处理~~(主要是自己用，有什么报错自己修，也不会遇上炒饭，笑)~~

未进行文件类别分筛~~(实在是懒得写了，一个everything就可以提前处理了)~~

---

好了，让我们开始详细分析吧

---

```python
import  api as a
def main():
    _root_dir = input('输入根目录')
    re_dir_list = a._find_all(_root_dir)
    # print(re_dir_list)
    for dir in re_dir_list:
        title = a._get_title(dir)
        a._i2p(_root_dir,title,dir)

if __name__ == '__main__':
    main()
```

`main.py`这个文件主要是调用`api.py`中的各种函数，提供输入并串联各函数以进行处理。对于我们来说，逐行与`api.py`中的函数对照分析是个很好的分析思路

---

```python
import  api as a
def main():
    _root_dir = input('输入根目录')
```

这几行不是重点，就不说了

---

```python
re_dir_list = a._find_all(_root_dir)
```

这段代码对应的函数如下

```python
def _find_all(dir): # 递归查询最底层目录
    temp_dirs,temp_files = _t_files(dir)
    re_list = []
    if temp_dirs == []:
        re_list.append(dir)
        pass
    else:
        for dirs in temp_dirs:
            re_list+=_find_all(dirs)
            pass
        pass
    return re_list
```

功能在函数里面已经说明了，就是查找到`dir`目录下最底层的所有目录并返回列表`re_list`，在讲这个函数之前，我们得先看这个，`temp_dirs,temp_files = _t_files(dir)`，这行代码所指向的是这个函数

```python
def _t_files(path):# 查找单一目录下所有文件与子目录
    dirs = []
    files = []
    for item in os.scandir(path):
        if item.is_dir():
          dirs.append(item.path)
          pass
        elif item.is_file():
          files.append(item.path)
        pass
    return dirs,files
```

函数的主要意思是用`os.scandir`读取一个目录下所有的对象(包括文件与目录)，并用`is_dir()`与`is_file()`将它们分类，最后分别返回包含文件与目录路径的列表。

好了，我们现在有了`temp_dirs`,`temp_files`两个列表，现在开始判断，先用`re_list = []`初始化一个临时列表，然后开始进入if语句

```python
if temp_dirs == []:
        re_list.append(dir)
        pass
```

这个还算简单，就是如果`temp_dirs`这个列表是空的，那返回的列表就是我们的输入`dir`

但是接下来的`else`就不一样了

```python
        for dirs in temp_dirs:
            re_list+=_find_all(dirs)
            pass
        pass
```

首先，用`for dirs in temp_dirs`遍历`temp_dirs`列表，然后进入递归，这也是这段函数甚至整个项目的最难点，`re_list+=_find_all(dirs)`，让我们开始推理

假设A目录下有A.1，A.2等目录，在对`A`使用`_find_all`函数后，我们会获得`temp_dirs = [A.1,A.2,...]`，自然，A不是最底层目录(即只有文件的目录)，那A.1，A.2呢，让我们继续向下思考，如果我们`_fin_all(A.1)`，则有两种可能

1. 返回`temp_dirs = [A.1.1,A.1.2,...]`
2. 返回`temp_dirs = []`

如果是状况1，则我们可以继续执行`_find_all(A.1.1)`……

如果是状况2，则`A.1`就是最底层目录

那么，在函数执行完后，父目录的`re_list`等于各子目录的`re_list`之和等于子目录的子目录的`re_list`……直至最后`re_list`等于自己的最底层目录

OK，`re_dir_list = a._find_all(_root_dir)`已经分析完了，让我们继续

---

我们已经获取了所有的最底层目录的列表`re_dir_list`，接下来遍历列表中的每一项，即每一个最底层目录

```python
        title = a._get_title(dir)
        a._i2p(_root_dir,title,dir)
```

`_get_title`函数指向的是这段代码

```python
def _get_title(dir): # 获得最底层目录名
    t_title = dir.split('\\')[-1]
    return t_title
```

主要功能是获取该目录的目录名，在`re_dir_list`中，列表中的目录格式如下`D:\code\github\……`，如果我们要将每一个最底层目录中的图片合并成一个pdf文件，名称取最底层目录的目录名，则我们可以通过`split()`语句分割出各目录的目录名，并且取出最底层目录的名称`t_title`。

最后则是`_i2P(_root_dir,title,dir)`，指向的是这段

```python
def _i2p(root_dir,title,dir): # 图片转pdf
    if os.path.exists(root_dir+'\\pdf') == False:
        os.mkdir(root_dir+'\\pdf')
    t_dir,file = _t_files(dir)
    file_name = root_dir+'\\pdf\\'+title+'.pdf'
    print(title)
    with open(file_name,'wb') as f:
        f.write(img2pdf.convert(file))
```

首先是

```python
    if os.path.exists(root_dir+'\\pdf') == False:
        os.mkdir(root_dir+'\\pdf')
```

这段的功能是检测`root_dir`下是否存在`pdf`目录存放生成的pdf文件，如没有便进行创建

然后`t_dir,file = _t_files(dir)`，主要是获取`dir`目录下所有的文件列表`file`，以便接下来的处理

接下来是`file_name = root_dir+'\\pdf\\'+title+'.pdf'`，这段是拼接生成的pdf文件的整个文件路径

最后是这一段

```python
    with open(file_name,'wb') as f:
        f.write(img2pdf.convert(file))
```

创建`file_name`指向的pdf文件，然后以二进制格式打开，最后写入用`img2pdf.convert(file)`生成的pdf的二进制数据，

# 3.总结

这个项目主要实现的是在一个目录下获取所有最底层目录并将其中的图片合并为pdf文件，主要难点是如何通过循环获取最底层目录。至于我为什么要写这个程序吗……欸嘿。