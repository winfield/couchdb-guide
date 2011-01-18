## Scaling Basics ##

扩展是一个很宽泛的话题. 很难找到一个唯一的定义. Everyone and her grandmother都有自己的对于扩展的定义. Most definitions are valid, but they can be contradicting. 更糟的是, 还有很多错误的关于扩展的观点. 为了能真正的定义什么是扩展, 人们需要用一把尖刀来找出那些最重要的部分. 

首先, 扩展无关于某样特定的技能或技术; 扩展, 或者说扩展性, 是一个特定架构的一种属性. 什么是需要扩展的几乎在所有的项目里都有所不同.

Scaling is specialization.

—Joe Stump, Lead Architect of Digg.com and SimpleGeo.com

Joe的引用是我们找到的关于扩展的最精确的描述. 它同时也是一个很模糊(wishy-washy)的描述, 不过那也是扩展的本质. 举个例子: 像Facebook.com这样的一个网站--有着大量的用户以及与这些用户相关的数据, 另外每天还有更多更多的用户加入进来--会想要在数据层面上扩展其用户数据. 作为对比, Flickr.com的核心和Facebook一样也是用户以及关于用户的数据, 但是在Flickr这个例子里, 增长最快的数据是用户上传的图片. 这些图片并不一定要保存在数据库里, 所以扩展图片存储是Flickr的成长之路.

It is common to think of scaling as scaling out. This is shortsighted. Scaling can also mean scaling in—that is, being able to use fewer computers when demand declines. More on that later.

These are just two services. There are a lot more, and every one has different things they want to scale. CouchDB is a database; we are not going to cover every aspect of scaling any system. We concentrate on the bits that are interesting to you, the CouchDB user. We have identified three general properties that you can scale with CouchDB:
这只是两个服务. 还有很多其他的服务, 而且它们各自都有着自己想要扩展的东西. CouchDB是一个数据库; 我们不打算要照顾到扩展系统的方方面面. 我们专注于那些你, CouchDB用户所感兴趣的部分. 我们定义了三个可以在CouchDB里扩展的一般属性:

* 读请求
* 写请求
* 数据

### 扩展读请求 ###

一个读请求从数据库里获取一片信息. 它会经过下列的步骤. 首先, HTTP服务器模块需要接受请求. 它会打开一个Socket来发送数据. 接下来, HTTP请求处理模块会分析请求并把请求转向至合适的CouchDB子模块. 对于一个单一的文档来说, 请求就会来到数据库模块, 在那里文档数据会从文件系统里被查找出来然后再原路返回回去.

所有的这些都需要处理时间以及足够多的sockets(或者文件描述符)可用. 服务器的存储后端必须能够处理所有的读请求. 还有其他一些事情可以限制系统接受更多的读请求; 这里, 最基本的要点在于, 一个单一的服务器只能处理一定数量的请求. 如果你的应用产生了更多的请求, 你必须要设置另一台服务器让你的应用可以读取.

让人欣慰的是, 读请求是可以被缓存的. 常用的东西可以被存储在内存中并且在瓶颈所在的更高层中就可以被返回. 使用缓存, 请求甚至不会到你的数据库层and thus toll-free. 第18章, 负载均衡解释了这样的场景.

### 扩展写请求 ###

写请求和读请求很像, 只是它更加糟糕一点. 它不仅仅是从硬盘读取一片数据, 还会写回去修改它. 记的吧, 读请求让人欣慰的是它是可以被缓存的. 写请求: 没那么简单. 当一个写请求改变数据时, 缓存必须被通知, 或者客户端必须被告知此时不能使用缓存. 如果你写多个服务器用于写请求的扩展, 那么一个写请求必须在所有的服务器上都起作用. 不管是哪一个场景, 写请求都需要投入更多的工作. 第19章, Clustering讲解了在多个服务器之间扩展写请求的方法.

### 扩展数据 ###

The third way of scaling is scaling data. Today’s hard drives are cheap and have a lot of capacity, and they will only get better in the future, but there is only so much data a single server can make sensible use of. It must maintain one more indexes to the data that uses disk space again. Creating backups will take longer and other maintenance tasks become a pain.
第三种扩展是扩展数据. 现在, 硬盘十分廉价并且容量很大, 而且在未来它们会变得更好, 但是

解决方法是把数据分割成可管理的分块, 并把每一个分块放到一个独立的服务器上. 所以存储有数据分块的服务器构成一个集群, 这个集群包含了所有的数据. 第19章, Clustering会讲到如何创建和使用集群.

尽管我们把读, 写和数据分开讲解, 但实际上它们很少相互独立. 扩展其中的一个会影响到其他的. 在接下来的章节里, 我们既会独立的讲解它们, 也会组合的讲解.

### Basics First ###

Replicaton是所有这些扩展方法的基础. 在我们讲解扩展之前, 第16章, Replication会让你CouchDB极佳的replication特性.

