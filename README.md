# 功能描述：
本程序一开始是为了解决“人类手动去看贴吧的小说更新很浪费时间”的问题而设计的。它自动去爬取指定贴吧的置顶精品帖子（就是小说的更新），然后判断是否是更新的内容，若是，则发送邮件给你（也是可以在程序里自定义指定的）。节约时间可以好好学习！！！

> 但是，天有不测风云，我写完这个程序不到半个月，百度贴吧的小说类贴吧就全部被关闭清理了，据说是版权问题。现在又重新开启了，不过已经没有更新小说了。所以这个程序也就仅剩下参考功能了。

上面用markdown的“引用”着重点出的话是我之前的记录。
不过最近将整个程序模块化，抽象出可重用、独立的模块，使得可以很轻松迁移到别的项目里，比如这里我就实现了爬取[另外一个小说网站](http://dazhuzai.net.cn/)的更新，以及爬取学院网站的通知的更新，还有一个爬取[王垠的博客](http://yinwang.org)（王垠是何许人也，哈哈，挺大争议性的人物），不过这不是重点，重点是我想表达，这个程序现在已经可以轻松迁移到各种相似的用途上了！！！

（更新于2016-09-02）


**在使用的时候请注意要**：

1. 修改发送列表（不要发给我）
2. 修改send\_email模块使用自己申请的邮箱或搭建的邮件服务器，请不要用我申请的账号，因为一个免费账号每天发送邮件的频率和次数好像是有限制的，为了你好我也好^__^


# 开发环境
python3（注意只支持python3） + ubuntu 16.10 + Crontab

其中Crontab是ubuntu上的一个定时任务机制，用来设置定时指定这个脚本的。用法很简单，请自行搜索。 

1. 在shell中输入命令crontab -e（首次运行时需要选择编辑器，建议选自己熟悉的，比如vim）； 
2. 然后在里面输入`* * * * * python3 /home/jacket/NovelUpdate/main.py >> /home/jacket/NovelUpdate/data/log.txt`，保存即可。 注意上面的路径需要根据自己的实际来改，还有我把输出重定向导出为日志，如果不需要也可以去掉。
3. ps，上面的命令是每分钟都做一次，这样子日志很快就很大了！所以我还定制了每天的日志清除任务

值得注意的是，随着检测更新的任务的增多，我发现有些是需要实时的（比如喜欢看的小说、必须看的通知），而有些的实时性要求很低（比如一些技术博客），所以我在crontab中区分了这两种任务，最后的话，我的crontab脚本这样写：

```
# 每分钟都检测一遍“实时的检测任务”
* * * * * python3 /home/jacket/NovelUpdate/RealTimeChecker.py >> /home/jacket/NovelUpdate/data/RealTimeLog.txt

# 每隔十分钟才检测一遍“非实时的检测任务”
0,10,20,30,40,50 * * * * python3 /home/jacket/NovelUpdate/NotRealTimeChecker.py >> /home/jacket/NovelUpdate/data/NotRealTimeLog.txt

# 每天的中午12点清除log文件
* 12 * * * rm /home/jacket/NovelUpdate/data/RealTimeLog.txt /home/jacket/NovelUpdate/data/NotRealTimeLog.txt
```


# 程序逻辑
一开始我写爬取百度小说贴吧的程序，没有模块化设计，全部逻辑写在一个类里，过了几个月回头看，太丑了，扩展性超级差。所以就按照MVC架构来思考模块化：

## Model
数据层分三部分：

1. 存储已经爬取到的数据；
2. 获取指定页面的目标数据（用**正则表达式**匹配出目标内容）；
3. 支持查找，以便逻辑层判断数据是否为更新。

对于第一点，其实正规一点应该使用数据库的，不过考虑到一本小说最多也就2000章，杀鸡不需要牛刀，所以使用文件系统来存储也很简单维护。所以我是将历史的更新存储到一个数据库db_update里，然后将新获取到的数据与历史数据对比，如果没有匹配的，说明就是更新了。

对于第二点，设计了一个SinglePageSpider来封装urllib库的请求操作，直接传入网址就可以获取到目标数据，所谓的目标数据是通过编写正则表达式来匹配页面得到的！

第三点，简单匹配，也可以考虑做hash等。


## View
这里其实是：

1. 根据更新内容（来自Model层），生成邮件内容；
2. 可靠发送邮件。

生成邮件内容这个简单，如何做到“**可靠发送邮件**”呢？（如果不可靠，这个爬虫的乐趣就少很多了）
其实关键是“**可靠**”，为什么会不可靠呢？发邮件不是一发就出去了吗？
这个涉及到我使用的免费的新浪邮箱的个人邮箱服务，很多限制，有时网络不好，有时邮箱限制发送的次数、时间间隔等等，都会导致失败。

我把发送失败的邮件保存到数据库中，然后在下一次激活的时候会发送，直到它们被发送成功为止！由于我目前设定crontab是每分钟都执行一次爬虫程序，所以邮件发送的延迟还是很小的。


## Control
逻辑层把前面两层组织起来了：

1. 给Model层指定页面去爬取数据，并判断是否为更新；
2. 若是更新，则扔给“可靠邮件发送器”发送邮件通知。

## 关于发送失败情况的进一步讨论
> 很明显发送失败了，是不能将其内容写入数据库db_update里的，因为这样下次就会把这些内容判断为不是更新的内容，从而忽略掉它们。
那应该怎么办呢？

> **逻辑上**这应该是一个**原子操作**：发送成功+写入数据库，或者发送失败+不写入数据库。
> 但是不写入数据库的话，要丢弃掉这些数据，然后等下一次爬取的时候再尝试发送？

> 这样确实可以，但是有个问题——就常识来说，邮件服务器的故障一般是持续的（作最坏的打算），不会说马上一两分钟就解决。那万一它这个故障持续了挺久的，并且那些更新了却没有发送成功的数据被后来更新的数据给挤掉了（因为主页一般只有2-3个置顶贴），那这部分数据**就真的丢失了**！！！
> **所以不能丢弃掉这些数据**，但是逻辑上又不能写入数据库db_update里，那应该怎么办呢？

> 我的做法是——还是实际上写入数据库db_update里，但是会另外建立一个数据库fail来存储这些发送失败的数据，在每次检测的时候也检测是否有发送失败的数据并尝试重新发送，直到发送成功才将fail里的数据清空。这样子相当于逻辑上是没有写入到db_update里的，但同时又保证了不会有因为邮件服务器故障时间的长短而导致数据丢失的问题！！！

好吧，说人话，还是从MVC架构的角度来考虑，逻辑层从数据层拿到更新的数据之后，传递给View层，View就负责可靠地将这些内容发送邮件成功，这样职责分明，逻辑层不必管View层是怎么实现的，而View专心发送好邮件就好了。

# 代码结构
## main.py
这是主程序，在里面可自定义发送的邮箱列表、想要爬取的贴吧地址、执行多种不同的监测更新模块。

## SimpleDB.py
这是我自己封装的一个简易数据库抽象类，抽象了这个爬虫对于数据处理的接口，目前支持的操作有:建表、插入、查找、获取整张表数据、删除整张表的数据的操作。


# SimpleDBUsingSqlite3.py
继承SimpleDB并实现其接口，使用sqlite3实现。


# SimpleDBUsingFS.py
继承SimpleDB并实现其接口，底层其实是用了文件系统来做（效率低）。


## SinglePageSpider.py
这是对python3的urllib的请求操作的上层封装，很容易获取单个页面的内容。

## send_email.py
这是一个发送邮件的功能函数，使用smtp协议发送邮件，需要自行申请或搭建一个邮件服务器来实现，当然，也可以换成另外的通讯模式，因为发送邮件是一个黑盒子，里面的功能逻辑可以随便改，对其它模块没有任何影响！！！（前提是返回值、参数等的约定保持一致）

## SendMailReliablly.py
这是基于send_email.py提供的api封装的一个“可靠邮件发送器”！一旦发送失败，它会将未发送成功的邮件内容保存进数据库中，等待下次激活时发送。而这个重新发送的过程对于用户来说是透明的，完全由程序自己控制的，所以我称之为“可靠放心的”！

## 监测模块UpdateMonitorBaseClass.py
至此，所有的模块对于整个爬虫监测系统来说，都是可重用的！想要迁移到新的业务需求上时，只需要简单使用我封装好的监测模块即可！如果有特殊的业务需求（下面有例子），也可以通过简单继承基类和重写函数就可以达到，可重用性超级高（个人觉得）！！！！

比如，想监测我们学院官网通知的更新，只需要知道官网网址，以及匹配通知链接的正则表达式，然后简单调用：
```python
# 参数的含义分别是：发送列表，标题，要爬取的页面url，匹配内容的正则表达式
sdcs = UpdateMonitorBaseClass(email_send_list,
		'学院官网',
		'http://sdcs.sysu.edu.cn/',
		'<a href="(http://sdcs.sysu.edu.cn/\?p=\d*)" title="([^"]*)"[^<]*</a>',
		'条通知'
		)
sdcs.checkUpdate()
```


## DaZhuZai.py（已去除）
由于百度小说贴吧被查封，现在不再更新了，所以就找了另外一个专门的网站来做。当然，最好的办法还是：**支持正版**，到正版官网去看和订阅，自然会有很好的更新通知服务。

由于我目前测试用的网站里面有一个推荐以往精彩章节的功能，所以经常会爬取到以前的章节，所以这个时候就需要继承基类UpdateMonitorBaseClass来重写getTargetContent函数就足够了！！！

其实这里只是为了说明，只需要继承UpdateMonitorBaseClass并实现其getTargetContent接口就可以很好地自定义需要。

但其实我后期已经扩展了UpdateMonitorBaseClass的功能，所以基本不需要自定义了，用好UpdateMonitorBaseClass的参数即可！！！


```python
# -*- coding:utf-8 -*-
import os
import re

from SinglePageSpider import SinglePageSpider
from UpdateMonitorBaseClass import UpdateMonitorBaseClass


class DaZhuZai(UpdateMonitorBaseClass):
	def getTargetContent(self):
		"""爬取大主宰中文网的帖子，不过由于它的页面太过丰富，所以需要重写基类的这个获取目标内容的函数

		Returns: a list of update chapters' relative path and its title
		such as [('http://dazhuzai.net.cn/dazhuzai-1309.html', 'title 1'), ]
		"""
		page = SinglePageSpider().getPage(self.url)
		result = re.findall(self.pattern, page)
		if result:
			newestResult = []
			for update in result:
				if int(update[0]) > 1300:
					url = 'http://dazhuzai.net.cn/dazhuzai-{0}.html'.format(update[0])
					newestResult.append([url, update[1]])
			return newestResult
		else:
			return None

```


# 优化与深入
## 1. 每次监测更新是线性遍历比较
线性遍历不一定慢，因为只要匹配到有，就可以退出了！！！假设我们逆序遍历文件里的行，那么一切就搞定了，很快！
举个例子，我们已经爬取到的数据是：
```
tie1
tie2
tie3
tie4
...
tie1000
```
最新的是tie1000，那么假设没有更新，我们还是只爬取到tie1000，如果从头到尾顺序遍历过来的话，那么比较次数是1000次，而如果逆序遍历，只需要比较一次就可以了！！！

可以控制文件指针来实现，不过我还在想有没有更好的办法（因为控制文件指针移动来读取每行数据太麻烦啦）


## 2. 数据库更换
其实python里内置了sqlite（效率应该可以比现在高挺多，可以用索引之类的来快速查找），哈哈，不过自己造轮子的感觉很爽！！！
我将这个爬虫里对Model层的接口需求抽象出来了，然后分别用文件系统、sqlite3来实现这些接口，然后就随心所欲调用了，只需要将想使用的class传给UpdateMonitorBaseClass的构造函数即可！比如：
```
googleBlog = UpdateMonitorBaseClass(
		email_send_list,
		'Google blog',
		'http://developers.googleblog.cn/',
		"<a href='(http://developers.googleblog.cn/[^']*)' itemprop='url' title='([^']*)'>[^<]*</a>",
		dbClass = SimpleDBUsingSqlite3,
		tips = '篇博客'
		)
```