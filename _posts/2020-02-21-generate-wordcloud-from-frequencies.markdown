---
layout: post
title: "根据词频生成词云(Python wordcloud实现)"
author: "BJ大鹏"
header-style: text
tags:
  - Python
---

网上大多数词云的代码都是基于原始文本生成，这里写一个根据词频生成词云的小例子，都是基于现成的函数。

安装词云与画图包
```shell
pip3 install wordcloud
pip3 install matplotlib
```
word_cloud.py(生成词云的程序)
```python
from wordcloud import WordCloud
import matplotlib.pyplot as plt

# 生成词云
def create_word_cloud():
    frequencies = {}
    for line in open("./record.txt"):
        arr = line.split(" ")
        frequencies[arr[0]] = float(arr[1])
    # 支持中文, SimHei.ttf可从以下地址下载：https://github.com/cystanford/word_cloud
    wc = WordCloud(
        font_path="./SimHei.ttf",
        max_words=100,
        width=2000,
        height=1200,
    )
    word_cloud = wc.generate_from_frequencies(frequencies)
    # 写词云图片
    word_cloud.to_file("wordcloud2.jpg")
    # 显示词云文件
    plt.imshow(word_cloud)
    plt.axis("off")
    plt.show()

# 根据词频生成词云
create_word_cloud()
```
record.txt文件示例，第1列是单词，第2列是频率，空格分隔
```
中文 100
英文 2
日语 3
```
运行后得到如下结果：
<img src="/img/wordcloud.jpg" height="400px" width="400px">
