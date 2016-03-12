# 基于Scrapy的Reddit爬取与分析

供稿人: Daniel Donohue，丹尼尔于2015年9月23日至12月18日期间参加了 NYC 数据科学院组织的为期十二周的数据科学训练营，本篇文章的成果主要基于他的第三个课程项目。

## 简介

NYC 数据科学训练营的第三个课程项目要求我们利用 Python 编写一个爬虫程序。由于我平时在 Reddit 上花费了大量的时间，因此我决定将`Reddit`设为我的目标网站。对于陌生用户而言，[Reddit](https://www.reddit.com/)是一个汇总用户提交的帖子或主题专栏链接("subreddits")的平台，其他用户可以在网站上对他人的帖子进行投票并发表自己的看法。`Reddit`网站上目前有超过 3600 万的登记用户和将近一百万个`subreddits`，该网站上有许多可以爬取的内容。

## 方法

我选取了十个 subreddits，其中五个 subreddits 来自于热门区，另外五个是我喜欢的栏目，我将从网站中爬取博文标题、链接、发表的日期时间、投票人数和相关热门评论的数据。十个 subreddits分别为：gaming, movies, videos, worldnews, science, seahawks, floridaman, circlejerk, totallynotrobots和uwotm8。

Python 中有许多爬虫软件包，最后我选择使用灵活易拓展的`Scrapy`。我先将数据储存到数据库中，然后以`txt`文件形式导出博文的标题和热门评论，最后利用`wordcloud`模块来绘制词云图。

## 细节

当你创建一个新的项目后，`Scrapy`会创建一个内含多个文件的文件夹。第一个文件`items.py`用于定义储存爬取数据的容器：

```python
from scrapy import Item, Field

class RedditItem(Item):
	subreddit = Field()
	link = Field()
	title = Field()
	date = Field()
	bote = Field()
	top_comment = Field()
```

该文件使得我们可以利用 Python 中处理字典的方法来处理爬虫数据，其中`field`的名字代表键值，爬取的数据代表字典中的数值。

第二个文件用于定义蜘蛛程序(Spider)，蜘蛛程序是`Scrapy`中用于定义爬取规则和解析数据的类。首先我们必须导入几个模块：

```python
import re
from bs4 import BeautifulSoup
from scrapy import Spider, Request
from reddit.items import RedditItem
```

在爬虫程序中我们将利用正则表达式和`BeautifulSoup`分别来提取链接和评论内容。接下来我们导入`spider`用于构建蜘蛛程序，最后导入先前定义的`items`。

```python
class RedditSpider(Spider):
	name = 'reddit'
	allowed_domains = ['reddit.com']
	start_urls = [
		'http://www.reddit.com/r/circlejerk',
		'http://www.reddit.com/r/gaming',
		'http://www.reddit.com/r/floridaman',
		'http://www.reddit.com/r/movies',
		'http://www.reddit.com/r/science',
		'http://www.reddit.com/r/seahawks',
		'http://www.reddit.com/r/totallynotrobots',
		'http://www.reddit.com/r/uwotm8',
		'http://www.reddit.com/r/videos',
		'http://www.reddit.com/r/worldnews'
	]
```

上述代码中的`allowed_domains`属性限制了爬虫程序的爬取目标，`start_urls`代表爬虫程序开始工作的链接。接下来我们定义一个解析方法，指导爬虫程序如何处理链接中的内容。

```python
def parse(self, response):
	links = response.xpath('//p[@class="title"]/a[@class="title may-blank"]/@href').extract()
	titles = response.xpath('//p[@class="title"]/a[@class="title may-blank"]/text()').extract()
	dates = response.xpath('//p[@class="tagline"]/time[@class="live-timestamp"]/@title').extract()
	votes = response.xpath('//div[@class="midcol unvoted"]/div[@class="score unvoted"]/text()').extract()
	comments = response.xpath('//div[@id="siteTable"]//a[@class="comments may-blank"]/@href').extract()
```

上述代码表示利用`XPath`从 HTML 文档中提取数据，最终数据以列表形式呈现。这些列表对应的前四个元素将填补`RedditItem`每个实例中对应的字段，但是热门评论的数据需要进一步处理，具体实现方法见下面的代码：

```python
for i, link in enumerate(comments):
	item = RedditItem()
	item['subreddit'] = str(re.findall('/r/[A-Za-z]*8?', link))[3:len(str(re.findall('/r/[A-Za-z]*8?', link)))-2]
	item['link'] = link[i]
	item['title'] = titles[i]
	item['date'] = dates[i]
	if votes[i] = u'\u2022':
		item['vote'] = 'hidden'
	else:
		item['vote'] = int(votes[i])
		
	request = Request(link, callback=self.parse_comment_page)
	request.meta['item'] = item
	yield request
```

对于评论列表中的第i个链接，我们创建一个`RedditItem`实例，用 subreddit 的名字填补 subreddit 字段，用第i个链接填补 link 字段，用第i个标题填补 title 字段。接下来，我们创建一个用于获取评论数据的请求命令，并利用`parse_comment_page`进行解析：

```python
def parse_comment_page(self, response):
	item = response.meta['item']
	top = response.xpath('//div[@class="commentarea"]//div[@class="md"]').extract()
	top_soup = BeautifulSoup(top, 'html.parser')
	item['top_comment'] = top_soup.get_text().replace('\n', '')
	yield item
```

我们再次利用`XPath`从 HTML 文档中提取评论数据，并利用`BeautifulSoup`移除评论数据中的 HTML 标签。最后我们将处理后的数据填补到 comment 字段中。

下一步，我们需要告知`Scrapy`如何利用项目管道工具处理提取出来的数据，管道工具可以用于进一步处理爬虫数据并将其储存到数据库中。我们选择将数据储存到`MongoDB`数据库，这是一个面向文档的数据库，不同于传统的关系型数据库。严格来说，传统的关系型数据库就足够了，但是`MongoDB`拥有更加灵活的数据模型，这有助于未来对该项目的拓展。首先，我们必须在`settings.py`文件中设定数据库的参数：

```python
BOT_NAME = 'reddit'
SPIDER_MODULES = ['reddit.spiders']
NEWSPIDER_MODULES = ['reddit.spiders']
DOWNLOAD_DELAY = 2

ITEM_PIPELINES = {
	'reddit.pipelines.DuplicatesPipeline': 300,
	'reddit.pipelines.MongoDBPipeline': 800,
}

MONGODB_SERVER = 'localhost'
MONGODB_PORT = 27017
MONGODB_DB = 'reddit'
MONGODB_COLLECTION = 'post'
```

设定下载延迟机制是为了避免违反 `Reddit` 的规定，此时我们已经设置好蜘蛛程序、解析器和数据库，现在我们需要利用管道工具(`pipeline.py`)将两者它们连接起来：

```python
import pymongo
from scrapy.conf import settings
from scrapy.exceptions import DropItem
from scrapy import log

class DuplicatesPipeline(object):
	def __init__(self):
		self.ids_seen = set()
		
	def process_item(self, item, spider):
		if item['link'] in self.ids_seen:
			raise DropItem("Duplicate item found: %s" % item)
		else:
			self.ids_seen.add(item['link'])
		return item
		
class MongoDBPipeline(object):
	def __init__(self):
		connection = pymongo.MongoClient(
			settings['MONGODB_SERVER'],
			settings['MONGODB_PORT']
		)
		db = connection[settings['MONGODB_DB']]
		self.collection = db[settings['MONGODB_COLLECTION']]
		
	def process_item(self, item, spider):
		valid = True
		for data in item:
			if not data:
				valid = False
				raise DropItem("Missing{0}!".format(data))
			if valid:
				self.collection.insert(dict(item))
				log.msg("Added to MongoDB database!", 
				level=log.DEBUG, spider = spider)
			return item
```

上述代码中的第一个类用于检测某个链接是否已经处理完毕，对于已经处理过的链接则选择忽略。第二类设定了数据存储参数，其中第一个方法用于连接数据库，第二个方法用于处理数据并将其加到数据库中。最后我们的数据库将如下图所示：

![img1](img1.png)

## `Reddit`词云

至此，爬虫任务已经完毕，接下来我们打算利用可视化技术来刻画评论情况。Python 中的`wordcloud`模块可用于绘制词云图，具体操作方法如下面代码所示：

```python
import sys
import pymongo
reload(sys)

sys.setdefaultcoding('UTF8')

client = pymongo.MongoClient()
db = client.reddit

subreddits = ['/r/circlejerk','/r/gaming','/r/FloridaMan','/r/movies','/r/science',
'/r/Seahawks','/r/totallynotrobots','/r/uwotm8','/r/videos','/r/worldnews']

for sub in subreddits:
	cursor = db.post.find({"subreddit":sub})
	
for doc in cursor:
	with open("text_files/%s.txt" % sub[3:],'a') as f:
		f.write(doc['title'])
		f.write('\n\n')
		f.write(doc['top_comment'])
		f.write('\n\n')
		
client.close()
```

前两行代码将默认的译码格式转换为 UTF-8，这可以避免解码时无法提取评论中的表情符号。最后我们利用这些文本文件来生成词云图：

```python
import numpy as np
from PIL import image
from wordcloud import WordCloud

subs = ['circlejerk','FloridaMan','gaming','movies','science',
'Seahawks','totallynotrobots','uwotm8','videos','worldnews']

for sub in subs:
	text = open('text_files/%s.txt' % sub).read()
	reddit_mask = np.array(Image.open('reddit_mask.jpg'))
	wc = WordCloud(background_color="black",mask=reddit_mask)
	wc.generate(text)
	wc.to_file('wordclouds/%s.jpg' % sub)
```

`WordCloud`对象将以`reddit_mask.jpg`为画布，将词云绘制上去。如下图示所示：

![img2](img2.png)

经过这次项目后，我已经变成`Scrapy`的粉丝，你可以利用它来抓取各种信息。





---

原文链接：http://www.datasciencecentral.com/profiles/blogs/scraping-reddit

原文作者：Daniel Donohue

译者：Fibears

---
