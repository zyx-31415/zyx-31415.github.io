---
title: "关于使用ebooklib进行多个txt转epub及epub合并与追加时遇到的小问题"
date: 2023-08-19T11:11:03+08:00
tags: ['ebooklib','python','epub','txt转epub']
categories: ['code']
top: true
---

# 23-08-19_关于使用ebooklib进行多个txt转epub及epub合并与追加时遇到的小问题

# 前言

不记得是半个月还是一个月之前，我下了一些短篇小说(当然，数量挺多)，如果导入我的电纸书，那场面，锣鼓喧天，鞭炮齐鸣，红旗招展，人山人海(其实就是会出现上百页，不好归档，笑)。所以我能忍吗，我刚学的python(其实学了好久，头发一掉一大把)。还搞不定你这小家伙。开搞

# 正文

## txt转epub

先介绍一下我的文件的结构

标题.txt

	内容为正文，不带其他内容

好了，开始编写

首先，导入库

​`from ebooklib import epub`​

然后，新建一个epub对象
​`book = epub.EpubBook()`​

接着，设置一下元信息

```python
	book.set_identifier("这里是id") #注意，格式为'id+数字'
	book.set_title("这里设置标题")
	book.set_language("cn")
	book.add_author("这里设置作者")
	book.add_metadata("DC", "description")
```

接下来，设置一下样式，作为一个懒人，自然是用默认的啦

```python
    style = 'body { font-family: Times, Times New Roman, serif; }'
    nav_css = epub.EpubItem(uid="style_nav",
                            file_name="style/nav.css",
                            media_type="text/css",
                            content=style)
    book.add_item(nav_css)
```

(敲黑板)现在，重点来了，在写代码之前，我们得了解一下epub文件的结构

epub文件可以看作一个zip文件，结构是这样的

example.epub

-- EPUB

	-- 001.xhtml

	-- 002.xhtml

	-- ...... .xhtml

	-- nav.xhtml

	-- style

		-- nav.css

-- META-INF

	--元信息文件

其中，META-INF这个文件夹暂时不用考虑，主要看EPUB文件夹

001.xhtml之类的xhtml文件是每一章的标题及内容

而nav.xhtml便是我们的目录，其中包含了每一个章节的标题及指向的xhtml文件

但是我们拥有的只有txt文件，而epub转换的时候需要HTML格式的，我们该如何做呢？

简单，加一下`<h1>`和`<p>`标签就好了

具体代码如下

首先，读取文件

```python
	def _read_txt(txt_name):  # 读取TXT
    	with open(txt_name, 'r', encoding='utf-8') as txt:
    	    txt_condent = txt.read()
    	    pass
    	return txt_condent
```

当然，还得获得一下绝对路径下的文件路径

```python
	def _t_files(path):  # 查找单一文件夹下所有文件与子文件夹
    	dirs = []
    	files = []
    	for item in os.scandir(path):
        	if item.is_dir():
            	dirs.append(item.path)
         	   pass
     	   elif item.is_file():
    	        files.append(item.path)
     	   pass
    	return dirs, files
```

由于为了准确，写的是绝对路径，所以还得分一下文件名，以便获取标题

```python
	def _get_title(dir):  # 获得绝对路径下的文件名
    	t_title = dir.split('\\')[-1].split('.')[0]
    	return t_title
```

可以转化HTML格式了

```python
	def _txt2html(txt_name):  # TXT转单个章节
    	txt_condent = _read_txt(txt_name)  # 读取txt内容
    	txt_condent_list = txt_condent.split('\n')  # 分段
    	text_name = _get_title(txt_name)  # 获取文件名
    	txt_html = '<h1>' + text_name + '</h1>'  # 设置标题
    	# print(txt_condent_list)
    	for temp in txt_condent_list:
        	txt_html += '<p>' + temp + '</p>'  # 段落转HTML格式
    	return text_name, txt_html
```

接下来便是添加章节对象

```python
    chaplist = [] #初始化章节列表
    for i in range(len(txt_file_line)):
        title, html = _txt2html(txt_file_line[i]) #获得标题以及转化为HTML格式的正文
        chap = epub.EpubHtml(title=title,
                             file_name='00{}.xhtml'.format(i), #这里是设置xhtml文件的名称，记住，xhtml的文件名是唯一的
                             lang='cn',
                             content=html)
        chaplist.append(chap)
```

现在，我们有了一个装满章节的列表，可以开始设置目录了

```python
    # 把所有chapters导入book里，并添加目录和书脊
    for chap in chaplist:
        book.add_item(chap)
    book.toc = chaplist
    book.spine = chaplist
    # 添加默认的 NCX and Nav file
    book.add_item(epub.EpubNcx())
    book.add_item(epub.EpubNav())
```

最后，别忘了保存

```python
    # 保存
    epub.write_epub('./'+'epub_abc.epub', book)
```

好了，我们可以将一群txt文件合并为epub文件了

接下来我们要说说如何合并及追加epub文件了

## 合并epub

源代码如下

```python
		# 导入模块
		import ebooklib
		from ebooklib import epub
		from bs4 import BeautifulSoup


		def _new_book(time):
    		# 创建EpubBook
    		book = epub.EpubBook()

  	    	# 设置书籍基本信息
 	   		book.set_identifier('id' + time)
  	   		book.set_title('epub_' + time)
 	   		book.set_language('cn')
    		book.add_author('writer')

    		# 添加样式表
    		style = 'body {font-family: Times, Times New Roman, serif;}'
    		nav_css = epub.EpubItem(
        	uid='nav_css', file_name='style/nav.css', media_type='text/css', content=style)
    		book.add_item(nav_css)
    		return book


		def _read_book(book_dir):
    	# 读取旧EPUB书籍
    	old_book = epub.read_epub(book_dir)

    	# 过滤掉旧书中的nav.html
    	docs = [item for item in old_book.get_items_of_type(
        ebooklib.ITEM_DOCUMENT) if item.file_name != 'nav.xhtml']

    	# 将文档转换为EpubHtml添加到列表
    	chaplist = []
    	for doc in docs:
        	if doc.media_type == 'application/xhtml+xml':
            	# print(doc.title)
            	contect = doc.content
            	soup = BeautifulSoup(contect, "html.parser")
            	title = soup.find('title').text
            	chap = epub.EpubHtml(
                title=title, file_name=doc.file_name, lang=doc.lang, content=contect)
            	chaplist.append(chap)
    	return chaplist


	def _write_book(chaplist, book, time):
    	# 添加所有章节文档
    	for chap in chaplist:
        	book.add_item(chap)

    	# 设置书籍目录和导航
    	book.toc = chaplist
    	book.spine = chaplist

    	# 添加NCX和导航文件
    	book.add_item(epub.EpubNcx())
    	book.add_item(epub.EpubNav())

    	# 生成EPUB文件
    	epub.write_epub('h_epub_' + time + '.epub', book)


	def _merge_epub(f_book_dir, s_book_dir, time):
    	n_book = _new_book(time)
    	f_chaplist = _read_book(f_book_dir)
    	len_f_c = len(f_chaplist)
    	s_chaplist = _read_book(s_book_dir)
    	# 重新命名第二个书籍章节文件名,避免重复
    	for chap in s_chaplist:
        	chap.file_name = '00'+f'{len_f_c}.xhtml'
        	len_f_c += 1
    	n_chaplist = f_chaplist + s_chaplist
    	_write_book(n_chaplist, n_book, time)



```

接下来，让我们逐步分析

首先看`_new_book`​函数

```python
	def _new_book(time):
    	# 创建EpubBook
    	book = epub.EpubBook()

    	# 设置书籍基本信息
    	book.set_identifier('id' + time)
    	book.set_title('epub_' + time)
    	book.set_language('cn')
    	book.add_author('writer')

    	# 添加样式表
    	style = 'body {font-family: Times, Times New Roman, serif;}'
    	nav_css = epub.EpubItem(
        	uid='nav_css', file_name='style/nav.css', media_type='text/css', content=style)
    	book.add_item(nav_css)
    	return book

```

这个很简单，看了上面文章的都知道，主要是生成一个设置好元数据的epub对象

接下来看`_read_book`​函数

```python
	def _read_book(book_dir):
    	# 读取旧EPUB书籍
    	old_book = epub.read_epub(book_dir)

    	# 过滤掉书中的nav.html
    	docs = [item for item in old_book.get_items_of_type(
        	ebooklib.ITEM_DOCUMENT) if item.file_name != 'nav.xhtml']

    	# 将文档转换为EpubHtml添加到列表
    	chaplist = []
    	for doc in docs:
        	if doc.media_type == 'application/xhtml+xml':
            	# print(doc.title)
            	contect = doc.content
            	soup = BeautifulSoup(contect, "html.parser")
            	title = soup.find('title').text
            	chap = epub.EpubHtml(
                	title=title, file_name=doc.file_name, lang=doc.lang, content=contect)
            	chaplist.append(chap)
    	return chaplist
```

首先，读取已有的epub文件

​`old_book = epub.read_epub(book_dir)`

然后，获得epub文件中的xhtml文件，当然，要把 nav.xhtml 去掉

​`​    docs = [item for item in old_book.get_items_of_type(ebooklib.ITEM_DOCUMENT) if item.file_name != 'nav.xhtml']`​

这时要注意，获得的docs列表里面的都是EpubItem类型的数据，所以我们要转换一下

```python
    	# 将文档转换为EpubHtml添加到列表
    	chaplist = []
    	for doc in docs:
        	if doc.media_type == 'application/xhtml+xml':
            	# print(doc.title)
            	contect = doc.content
            	soup = BeautifulSoup(contect, "html.parser")
            	title = soup.find('title').text
            	chap = epub.EpubHtml(
                	title=title, file_name=doc.file_name, lang=doc.lang, content=contect)
            	chaplist.append(chap)
```

可能有人会问为什么要用bs4库，我等会会详细讲一下

好了，可以返回一个写好章节的列表了

​`return chaplist`​

至于`_write_book`​函数，也很简单

```python
	def _write_book(chaplist, book, time):
    	# 添加所有章节文档
    	for chap in chaplist:
        	book.add_item(chap)

    	# 设置书籍目录和导航
    	book.toc = chaplist
    	book.spine = chaplist

    	# 添加NCX和导航文件
    	book.add_item(epub.EpubNcx())
    	book.add_item(epub.EpubNav())

    	# 生成EPUB文件
    	epub.write_epub('h_epub_' + time + '.epub', book)
```

这个没啥好讲的，就是将设置好的章节列表转换为目录再写进去

最后，就是合并的主函数`_merge_epub`​函数了

```python
	def _merge_epub(f_book_dir, s_book_dir, time):
    	n_book = _new_book(time)  # 新建epub文件
    	f_chaplist = _read_book(f_book_dir)
    	len_f_c = len(f_chaplist)
    	s_chaplist = _read_book(s_book_dir)
    	# 重新命名第二个书籍章节文件名,避免重复
    	for chap in s_chaplist:
        	chap.file_name = '00'+f'{len_f_c}.xhtml'
        	len_f_c += 1
    	n_chaplist = f_chaplist + s_chaplist
    	_write_book(n_chaplist, n_book, time)

```

首先，利用上面的`_new_book`​函数，新建一个epub数据

​`n_book = _new_book(time)`​

然后，读取第一个epub文件的章节列表

​`f_chaplist = _read_book(f_book_dir)`​

为了后面的去重工作，得先输出列表长度

​`len_f_c = len(f_chaplist)`

接下来，读取第二个epub文件的章节列表

​`s_chaplist = _read_book(s_book_dir)`​

由于两个文件的xhtml文件名会有重复，得重命名一下

```python
    	# 重新命名第二个书籍章节文件名,避免重复
    	for chap in s_chaplist:
        	chap.file_name = '00'+f'{len_f_c}.xhtml'
        	len_f_c += 1
```

好了，合并一下重命名后的列表

​`n_chaplist = f_chaplist + s_chaplist`​

最后，写入新文件

​`_write_book(n_chaplist, n_book, time)`​

这样，合并文件就写好了

最后，让我们看看如何追加epub文件

## 追加epub文件

由于文件数量过多，我总不可能将所有txt文件长期保存(虽然还是这么做了)，所以还是得写出追加文件啊。

这是源代码

​`epub_api.py`

```python
	from ebooklib import epub
	import file as f


	def _set_meta(book, time):  # 设置元数据
	    book.set_identifier("id"+time)
	    book.set_title("h_epub_"+time)
	    book.set_language("cn")
	    book.add_author("zyx")
	    book.add_metadata("DC", "description", "h")
	    style = 'body { font-family: Times, Times New Roman, serif; }'
	    nav_css = epub.EpubItem(uid="style_nav",
	                            file_name="style/nav.css",
	                            media_type="text/css",
	                            content=style)
	    book.add_item(nav_css)
	    # 添加默认的 NCX and Nav file
	    book.add_item(epub.EpubNcx())
	    book.add_item(epub.EpubNav())
	    return book


	def _book_chaplist(book, time, chaplist):  # 写入目录文件
	    for chap in chaplist:
	        book.add_item(chap)
	    book.toc = chaplist
	    book.spine = chaplist
	    epub.write_epub('./'+'h_epub_'+time+'.epub', book)


	def _change_chap_list(old_list, dir):  # 合并新旧列表
	    dir, txt_file_line = f._t_files(dir)  # 读取TXT文件夹
	    old_len = len(old_list)  # 标记旧列表长度
	    new_chaplist = old_list
	    for i in range(len(txt_file_line)):  # 开始合并
	        title, html = f._txt2html(txt_file_line[i])
	        chap = epub.EpubHtml(title=title,
	                             file_name='00{}.xhtml'.format(i+old_len),
	                             lang='cn',
	                             content=html)
	        new_chaplist.append(chap)
	        pass
	    return new_chaplist

```

​`epub_add.py`

```python
	import ebooklib
	from ebooklib import epub
	import epub_api as api
	from bs4 import BeautifulSoup


	def _old_chaplist_read(book_dir):  # 读取旧epub文件
	    # 读取旧EPUB书籍
	    old_book = epub.read_epub(book_dir)
	
	    # 过滤掉旧书中的nav.html
	    docs = [item for item in old_book.get_items_of_type(
	        ebooklib.ITEM_DOCUMENT) if item.file_name != 'nav.xhtml']

	    # 将文档转换为EpubHtml添加到列表
	    chaplist = []
	    for doc in docs:
	        if doc.media_type == 'application/xhtml+xml':
	            # print(doc.title)
	            contect = doc.content
	            soup = BeautifulSoup(contect, "html.parser")
	            title = soup.find('title').text
	            chap = epub.EpubHtml(
	                title=title, file_name=doc.file_name, lang=doc.lang, content=contect)
	            chaplist.append(chap)
	    return chaplist


	def _new_book_write(old_book_dir, dir, time):  # 生成新epub文件
	    old_chaplist = _old_chaplist_read(old_book_dir)
	    new_book = epub.EpubBook()
	    new_chaplist = api._change_chap_list(old_chaplist, dir)
    	api._set_meta(new_book, time)
	    api._book_chaplist(new_book, time, new_chaplist)

```

​`epub.api.py`​没啥好讲的，就是上面的代码打包了一下。看得懂上面两章的就可以看懂这个了

重点是`epub_add.py`​

首先，看一下`_old_chaplist_read`​函数

```python
	def _old_chaplist_read(book_dir):  # 读取旧epub文件
    	# 读取旧EPUB书籍
    	old_book = epub.read_epub(book_dir)

    	# 过滤掉旧书中的nav.html
    	docs = [item for item in old_book.get_items_of_type(
    	    ebooklib.ITEM_DOCUMENT) if item.file_name != 'nav.xhtml']

    	# 将文档转换为EpubHtml添加到列表
    	chaplist = []
    	for doc in docs:
    	    if doc.media_type == 'application/xhtml+xml':
    	        # print(doc.title)
    	        contect = doc.content
    	        soup = BeautifulSoup(contect, "html.parser")
    	        title = soup.find('title').text
    	        chap = epub.EpubHtml(
    	            title=title, file_name=doc.file_name, lang=doc.lang, content=contect)
    	        chaplist.append(chap)
    	return chaplist
```

```python
    	# 读取旧EPUB书籍
    	old_book = epub.read_epub(book_dir)
	
    	# 过滤掉旧书中的nav.html
    	docs = [item for item in old_book.get_items_of_type(
    	    ebooklib.ITEM_DOCUMENT) if item.file_name != 'nav.xhtml']
```

这两段看上去很熟悉，看了合并讲解的都知道了，我就不再说一遍了

然后让我们看这一段

```python
    	# 将文档转换为EpubHtml添加到列表
    	chaplist = []
    	for doc in docs:
    	    if doc.media_type == 'application/xhtml+xml':
    	        # print(doc.title)
    	        contect = doc.content
    	        soup = BeautifulSoup(contect, "html.parser")
    	        title = soup.find('title').text
    	        chap = epub.EpubHtml(
    	            title=title, file_name=doc.file_name, lang=doc.lang, content=contect)
    	        chaplist.append(chap)
    	return chaplist
```

这一段看上去也没啥问题，但如果你这么想就错了

让我们看看这几句

```python
	            contect = doc.content
	            soup = BeautifulSoup(contect, "html.parser")
    	        title = soup.find('title').text
    	        chap = epub.EpubHtml(
    	            title=title, file_name=doc.file_name, lang=doc.lang, content=contect)
```

为何要用bs4库呢？

其实在`ebooklib`​库中有直接转换的函数，但我们不能直接用

因为`EpubItem`​是没有标题参数的！

所以我们必须在内容中提取出来标题，才能够进行处理

即

```python
    	        contect = doc.content
    	        soup = BeautifulSoup(contect, "html.parser")
	            title = soup.find('title').text
```

然后才能

```python
    	        chap = epub.EpubHtml(
    	            title=title, file_name=doc.file_name, lang=doc.lang, content=contect)
```

最后让我们把`_new_book_write`​函数看一下

```python
	def _new_book_write(old_book_dir, dir, time):  # 生成新epub文件
    	old_chaplist = _old_chaplist_read(old_book_dir)  # 读取旧epub文件
    	new_book = epub.EpubBook()  # 新建epub对象
    	new_chaplist = api._change_chap_list(old_chaplist, dir)  # 合并新旧章节
    	api._set_meta(new_book, time)  # 设置元数据
    	api._book_chaplist(new_book, time, new_chaplist) #写入对象并保存文件
```

我已经写好了注释，内容看上面说的就好了

# 总结

本文主要讲了如何使用`ebooklib`​库进行epub文件的生成，合并与追加

主要重点有以下几个

1. 设置目录
2. 避免xhtml文件名重复
3. 将`EpubItem`​转换为`EpubHtml`​时需重新设置标题

好了，希望你看完本篇文章后有所收获。中文互联网上讲`ebooklib`​库的文章很少，希望能帮助到你

祝你代码不报错，运行一遍过
