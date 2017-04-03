---
layout: post
title: 协同过滤实战：网络小说推荐系统（一）
category: project
description: 数据的爬取和整理
---
项目源代码参见 [GitHub Repo](https://github.com/ZhiyuanLIPlus/EBooksRecommander)

前一段时间正好看完了《集体智慧编程》中的协同过滤推荐部分，想了想自己现在也越来越难在网上发现能看进去的小说了，就动了用协同过滤算法给自己写一个网络小说推荐工具的念头。

## 数据爬取
在手边没有任何原始数据的情况下，网上抓取数据成为了我获取数据来源的唯一方法。这个时候就不得不祭出爬虫大杀器SCRAPY了。我选择的网站是龙空旗下的[优书网](http://www.yousuu.com/booklist)，这也是我自己平时经常去浏览书评的一个网站。
Scrapy的使用方法很简单，网上的教程一抓一大把，这里就不多赘述了。这里主要说一下我在使用爬虫工具的时候遇到的几个大坑。

### 爬虫被监测甚至IP被封
在我刚开始学写爬虫的时候，并没有一开始就使用Scrapy框架。那时候自己用Python urllib2和正则库手写了一个非常简陋的爬虫加多线程登录去爬取新浪微博的数据。最作死的是，当时我脑袋一抽居然把自己日常使用的微博账号放进了爬虫使用的账号池中，最后当然就是被新浪一下打死。账号被封之后还臊眉耷眼地去跟新浪客服要求恢复账号，结果可想而知。这次为了避免悲剧重演，我在Scrapy的中间件中使用了在向服务器发送请求时随机更换浏览器头这样一个方法。希望可以用这样的方式迷惑服务器后台，避免被发现。

具体代码如下：
```python
class RotateUserAgentMiddleware(UserAgentMiddleware):
    user_agent_list = [
        'Mozilla/5.0 (X11; Linux i686) AppleWebKit/537.31 (KHTML, like Gecko) Chrome/26.0.1410.43 Safari/537.31',
        #这里省略其他浏览器头部分，源码可以在我的Github上找到
        ]

    def __init__(self, user_agent=''):
        self.user_agent = user_agent

    def _user_agent(self, spider):
        if hasattr(spider, 'user_agent'):
            return spider.user_agent
        elif self.user_agent:
            return self.user_agent

        return random.choice(self.user_agent_list)

    def process_request(self, request, spider):
        ua = self._user_agent(spider)
        if ua:
            request.headers.setdefault('User-Agent', ua)
```
然后在setting.py中设置一下
```python
DOWNLOADER_MIDDLEWARES = {
    'eBksSpider.middlewares.RotateUserAgentMiddleware': 543,
}
```
这里的543只是为了让scrapy在启动时明白需要启动的中间件的先后顺序，因为我只用了一个中间件，所以随便写了个数字就好。

这样使用了随机浏览器头的机制之后，这次的爬取进行地很顺利。当然，严谨地说，在对爬虫的敏感性上，微博和优书网也一定不是一个数量级的，所有并不算是一个确定有效的方法。

### 中文编码问题
这个相信是所有使用python2.7系列写爬虫的同学们心中永远的痛了。编码类的科普，网上依然是一抓一大把，这里直接给出我使用的解决方法。
使用Scrapy的Piplines机制，在piplines.py中添加：
```python
tempDictParse = {}
class EbksspiderPipeline(object):
    def __init__(self):
        self.file = codecs.open('test.json', 'w', encoding='utf-8')

    def process_item(self, item, spider):
        tempDictParse['name'] = item['name']
        tempDictParse['publisher'] = item['publisher']
        tempDictParse['publisher_url'] = item['publisher_url']
        tempDictParse['type'] = item['type']
        tempDictParse['numOfLike'] = item['numOfLike']
        tempDictParse['bookList'] = item['bookList']
        tempDictParse['url'] = item['url']

        line = json.dumps(tempDictParse) + "\n"
        self.file.write(line.decode('unicode_escape'))
        return item

    def spider_closed(self, spider):
        self.file.close()
```
同样在设置setting.py中：
```python
ITEM_PIPELINES = {
    'eBksSpider.pipelines.EbksspiderPipeline': 300
}
```
注意，因为这里我们把所有的编码任务都交给了piplines，所以在爬取的时候，我们直接使用extract_first()等，而不是在爬虫代码中进行编码操作。

### Json dump
在数据序列化方面，我最后选择的是用json。一个最常见的问题是，当你在scrapy中自定义了Item作为爬取数据的存储形式时。json包是没有办法把自定义item类dump出去的。解决方法依然有很多，我选择了最简单的一种。用系统的dict类转存了一下自定义item类中的数据，再dump，因为最后的数据量其实只是大概10k左右的object，所以这样的内存消耗完全可以接受。当然如果在抓取时，直接使用dict类的话应该就完全不需要这一步了。

### Debug
初次使用Scrapy的话难免会有各种错误需要调试，这里推荐的方法是在ide中调用命令行达到debug的目的。

在与scrapy.cfg同路径的地方新建一个main.py文件，添加代码
```python
from scrapy import cmdline
cmdline.execute("scrapy crawl ysXpathTest -o test.json".split())
```

在程序中按自己意愿添加breakpoint,然后debug这个main.py文件，就可以调试代码了。

## 数据整理
因为最后爬到的数据并没有我一开始想象的那么多，大概也就10k+左右的书单，所有我也没有使用任何复杂的数据结构，简简单单的一个嵌套的dict类。
```python
{
    'UserA':{
                'BookA': 3.5,
                'BookB': 5,
                'BookC': 4,
                'BookD': 3
            }
    'UserB':{
                'BookB': 1.5,
                'BookC': 2,
                'BookE': 1
            }
    #...etc..
}
```

使用dict嵌套最基本的原因当然是希望可以快速提取数据。
