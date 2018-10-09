---
layout:     post
title:      "使用pynlpir增强jieba分词的准确度"
subtitle:   "Use pynlpir to enhance the accuracy of jieba participle."
date:       2018-10-7 8:00:00
author:     "XuXinkun"
header-img: "img/post-bg-pynlpir-jieba.jpg"
tags:

---


在使用jieba分词时，发现分词准确度不高。特别是一些专业词汇，比如`堡垒机`，只能分出`堡垒`，并不能分出`堡垒机`。这样导致的问题是很多时候检索并不准确。
经过对比测试，发现[nlpir](http://ictclas.nlpir.org/)进行分词效果更好。但是nlpir的效率和各种支持又没有jieba那么好，因此采用了一种折中的方案。
就是先用nlpir生成字典，然后使用jieba利用字典进行分词。

首先安装pynlpir。pynlpir的相关说明可以参考https://pynlpir.readthedocs.io/en/latest/index.html。

```
// 安装
$ pip install pynlpir
// 证书更新
$ pynlpir update
```

而后为jieba生成字典。jieba支持的字典格式为`单词 词频`，中间用空格隔开，每行一个单词。
使用pynlpir生成词典的方式如下：

```Python
import pynlpir
pynlpir.open()
f = open("doc.txt", "r")
s= f.readlines()
s = '\n'.join(s)
f.close()
key_words = pynlpir.get_key_words(s, max_words=1000, weighted=True)
for key_word in key_words:
    print '%s %s' % (key_word[0], int(key_word[1]*10))
```

这里之所以为每个`词频*10`，主要是为了加强其权重。而后再使用jieba利用该字典进行分词。至于jieba分词如何使用词典，可以参考https://github.com/fxsjy/jieba/blob/master/test/test_userdict.py。这里就不再重复了。

对于sphinx-doc，其最新版本也是使用的jieba分词。同样可以使用本方法来提升其分词的准确率。
中文分词引入可以参考https://www.chenyudong.com/archives/sphinx-doc-support-chinese-search.html。
在conf.py中，配置`html_search_options = {'dict': '/usr/lib/jieba.txt'}`，加入字典的路径。这里一定要绝对路径。相对路径不能生效。