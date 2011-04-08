## 高性能 ##

本章会教会你如何用最快的方式插入数据到CouchDB以及如何以最快的方法来查询数据. 同时也将解释为什么使用不同的技术会产生不同的性能.

The take-home message: bulk operations result in lower overhead, higher throughput, and more space efficiency. If you can’t work in bulk in your application, we’ll also describe other options to get throughput and space benefits. 最后, 我们会描述如何使用Erlang来直接和CouchDB交互, 如果你想要把CouchDB整合到一个非HTTP协议的服务器, 像是SMTP(email)或者XMPP(chat), 这将会是个非常有用的技术.

### Good Benchmarks Are Non-Trivial ###

每个应用程序都是不同的. 对于性能的要求不总是那么明显. 不同的情况需要使用不同的参数. 一种经典的trade-off就是latency versus throughput. 并发性是另外一个因素. 许多数据库平台在处理100个并发客户端时和处理1000个或更多的并发客户端时, 做法有很大的不同. 有些数据需要进行串行化处理, 这就是增加响应客户端的总时间(延时), 以及服务器的负载. 我们认为简单的数据结构和读取模式可以在应用程序的可缓存性和可扩展性方面作出很大的不同, 这点我们晚点在来介绍.

The upshot: 真正的性能测试需要真实的环境. 模拟一个环境是很困难的. Erlang在强负载下会表现的更好(特别是在多核的环境里), 所以我们经常会看到一些测试并不能给予CouchDB足够大的负载使其崩溃.

让我们来看一个典型的web应用. Craigslist并非真是这样运作的(因为我们也并不知道Craigslist是如何动作的), 但是对于展示性能测试的问题, 这已经足够接近了.

你有一个web服务器, 某个中间件, 以及一个数据库. 当一个用户请求进来时, web数据库处理网络和解析HTTP请求. 请求接着被转交给中间件, 它会找出哪个服务需要运行, 然后启动这个服务来处理请求. 中间件可能会和你的数据库以及其他的一些外部资源像是文件或者远程web服务打交道. 最后请求返回至web服务器, 它会发出结果的HTML. 这个HTML会包含存在于web服务器的其他资源的链接(像是CSS, JS, 或者图片文件). 每次都会执行这一过程. 虽然每次都会稍有不同, 但是一般来霁, 所有的请求都是类似的. 并且在此过程中, 会有一些缓存来保存中间结果以避免重新计算的昂贵代价.

这里有很多动态的部分. 从上往下, 对所有的组件进行评估来找出瓶颈在哪是非常复杂的(但是能找出来当然是好的). We start making up numbers now. 绝对的值并不重要; 只有它们相互之间的关系才是重要的. 比如说, 一个请求需要1.5秒(1,500毫秒)才能完全的返回给浏览器.

在像Craigslist这样的简单例子里, 会有初始的HTML, 一个CSS文件, 一个JS文件, 以及一个favicon. 除了HTML, 这些都是静态资源, 它们需要从硬盘(或者内存)中读取出来, 然后返回给浏览器并生成它们. 对于性能来说, 最值得做的事情就是保持数据尽可能的小(GZIP压缩, 高压缩率的JPG), 并且避免同时请求它们(浏览器HTTP层面的缓存).  Making the web server any faster doesn’t buy us much (yeah, hand wavey, but we don’t want to focus on static resources here). 让我们假设所有的静态资源都需要500毫秒来处理和生成.

Read all about improving client experience with proper use of HTTP from Steve Souders, web performance guru. His YSlow tool is indispensable for tuning a website.

生成初始HTML需要占用我们1000毫秒. 网络延时(请查看第一章, 为什么使用CouchDB?)也会花掉我们200毫秒. 再让我们假设HTTP解析, 中间件处理路由和运行, 以及访问数据库都会占用相同的时间, 每样200毫秒.

If you now set out to improve one part of the big puzzle that is your web app and gain 10 ms in the database access time, this is probably time not well spent (unless you have the numbers to prove it).

然而, 把一个单一的请求这样分块并且来看每个组件要花费多少时间同样是一种误导. 即便在正常的负载下, 只有一小部分的时间是花费在数据库上的, 这也并不能告诉你在注意峰值时会发生什么. 如果所有的请求都涌向同一个数据库, 那么任何的锁行为都会堵塞大量的web请求. 在正常负载时, 你的数据库可能只占用了总时间的一小部分, 但是在spike时, 它可能就会变成一个瓶颈, 进而影响整个应用服务器在峰值时的运行. CouchDB可以最小化这种影响, 通过为每一个连接都配给一个Erlang进程, 来保证所有的客户端都可以得到处理, 而只会增加一点点的延时.

### 高性能的CouchDB ###

现在你知道了数据库性能只是整个web性能的一个小部分, 我们会给你一些小提示来处理CouchDB的大多数性能问题.

CouchDB从头到尾就是被设计用于高并发的环境的, 这是大多数web应用面临的情况. 然而, 有时间, 我们需要导入一大块的数据到CouchDB或者要把另一个数据库的数据完全的转移过来. 又或者我们要构建一个自定义的Erlang应用, 它需要从更底层连接CouchDB, 而不是通过HTTP.

#### 硬件 ####

通常人们会想要知道他们应该使用的什么类型的硬件, 内存多大, 什么样的CPU型号等等. 真正的答案的是CouchDB很足够灵活, 它可以跑在任何硬件上, 从智能手机到集群. 所以答案多种多样. 

更多的内存会更好, 因为CouchDB大量的使用的文件系统的缓存. CPU的核心数对于创建视图比处理文档更加重要. 固态硬盘(SSD)非常棒因为they can append to a file while loading old blocks, with a minimum of overhead. 当它们变得越来越快, 越来越便宜时, 对于CouchDB将会变得方便.

#### An Implementation Note ####

我们不会再在这里长篇大论的讨论B-tree, 但是理解CouchDB的数据格式是了解哪种策略可以得到最佳的性能的关键. 每次有更新时, CouchDB会从磁盘里读取指向更新文档的B-tree节点或者是一个key的范围, 里面可以找到新文档的_id.

这种读取通常可以从文件系统缓存里得到, 除非被更新的文档处于一个很长时间都没有被读取的区域里. 在这些情况下, 就必须要磁盘里查找了, 这会阻塞写入并且还有其他的一些连锁反应. 防止此类的磁盘查找就是CouchDB性能优化要做的事.

在本章中, 我们会使用JavaSciprt测试套件里拿来的数字. 虽然它不是最准确的, 但是它使用的策略(记录在10内可以被保存的文档的数量)不坏. 用于性能测试的硬件并不是很高端: 只是一个老型号的白色的Intel Core 2 Duo的Macbook(还记得这个型号吗, 你?).

在CouchDB运行在5984端口上时, 你可以在CouchDB源代码目录的bench目录下, 通过运行./runner.sh自己来运行性能测试.

### Bulk Inserts and Mostly Monotonic DocIDs ###

Bulk inserts are the best way to have seekless writes. Random IDs force seeking after the file is bigger than can be cached. Random IDs also make for a bigger file because in a large database you’ll rarely have multiple documents in one B-tree leaf.

Optimized Examples: Views and Replication

If you’re curious what a good performance profile is for CouchDB, look at how views and replication are done. Triggered replication applies updates to the database in large batches to minimize disk chatter. Currently the 0.11.0 development trunk boasts an additional 3–5x speed increase over 0.10’s view generation.

Views load a batch of updates from disk, pass them through the view engine, and then write the view rows out. Each batch is a few hundred documents, so the writer can take advantage of the bulk efficiencies we see in the next section.

### Bulk Document Inserts ###

通过HTTP, 最快的将数据导入CouchDB的模式是_bulk_docs. 批量文档API可以在一个POST请求中接受一个文档的集合, 并在一个索引操作里把它们全部存储进CouchDB里.

在使用一种脚本语言导入一组数据时应该批量文档API. 这样可以比单独的更新快10到100倍, 并且在大多数语言里都很容易实现.

影响批量操作性能的主要因素是更新的大小, 它同时包括总数据的大小以及更新的文档的数量的大小.

wrong translated...下面是四个不同数量级的批量文档插入, 从100个文档, 然后是1,000个, 5,000个, 10,000个:

				bulk_doc_100
				4400 docs
				437.37574552683895 docs/sec

				bulk_doc_1000
				17000 docs
				1635.4016354016355 docs/sec

				bulk_doc_5000
				30000 docs
				2508.1514923501377 docs/sec

				bulk_doc_10000
				30000 docs
				2699.541078016737 docs/sec

You can see that larger batches yield better performance, with an upper limit in this test of 2,700 documents/second. With larger documents, we might see that smaller batches are more useful. For references, all the documents look like this: {"foo":"bar"}

虽然每秒2,700个文件已经很不错了, 我们想要更加强大的! 接下来, 我们要同时运行几个批量文档插入.

With a different script (using bash and cURL with benchbulk.sh in the same directory), we’re inserting large batches of documents in parallel to CouchDB. With batches of 1,000 docs, 10 at any given time, averaged over 10 rounds, I see about 3,650 documents per second on a MacBook Pro. Benchbulk also uses sequential IDs.
使用一个不同的脚本(在同一个目录下, 利用bash和cURL配合benchbulk.sh), 我们

We see that with proper use of bulk documents and sequential IDs, we can insert more than 3,000 docs per second just using scripting languages.

### Batch Mode ###

为了避免单一文件写入时的索引和硬盘同步开销, 有一个选项可以使CouchDB在内存中保存批量的文档. 当达到一定界限时, 或者在用户触发时, 再冲刷(flushing)到硬盘上. 批量选项没有普通更新所拥有的数据完整性保证, 所以它只能在某些较近更新的潜在丢失可以被接受时使用.

因为在冲刷发生以前, 批量模式只把更新存储在内存中, 那些在保存到CouchDB时出现错误的更新就会丢失. 默认的, CouchDB会每秒一次把内存中的更新冲刷到硬盘, 数据丢失因此仍然是很小的. 为了反映batch=ok模式时的数据完整性减弱, HTTP返回码会变成202 Accepted, 而不是201 Created.

批量模式的理想使用场景是日志类型应用, 那里会有许多分布式的写入, 每一个都会存储不连续的事件到CouchDB中. 在一个普通的日志场景中, 接受在极少的情况丢失少量的更新, 来换取更大的处理能力是值得的.

There is a pattern for reliable storage using batch mode. It’s the same pattern as is used when data needs to be stored reliably to multiple nodes before acknowledging success to the saving client. In a nutshell, the application server (or remote client) saves to Couch A using batch=ok, and then watches update notifications from Couch B, only considering the save successful when Couch B’s _changes stream includes the relevant update. We covered this pattern in detail in Chapter 16, Replication.
使用批量模式时的可靠数据存储有一个模式. 它和在不知道客户端保存是否成功时, 在多个节点上可靠的存储数据时使用的模式的相同的. 简言之, 应用服务器(或者远程客户端)使用batch=ok在节点A进行存储, 然后观察来自节点B的变更通知, 

				batch_ok_doc_insert
				4851 docs
				485.00299940011996 docs/sec

This JavaScript benchmark only gets around 500 documents per second, six times slower than the bulk document API. However, it has the advantage that clients don’t need to build up bulk batches.
这一JavaScript基准测试只得到了每秒500个文档的结果, 比块(bulk)文档API要慢上六倍. 然而, 它的优势在于, 客户端不需要创建bulk batches.

### 单文档插入 ###

普通web应用的CouchDB使用场景是单文档的插入. 因为每个插入来自不同的客户端, 且需要一整个HTTP请求和响应的开销, 它的写入能力通常来说是最低的.

最慢的插入CouchDB的用法恐怕就是一个客户端的大量串行写入了. 想像下这样一个场景: 每一个写入都信赖于先前一个写入的结果, 只有这个写入客户端才能正常工作. 单听这描述就会觉得这个场景不太好. 如果你发现你处在了这么个位置, 那么也许还有其他的问题要处理.

一个客户端的串行进行写入, 每秒可以写入258个文档(大概是最差的写入场景了).

				single_doc_insert
				2584 docs
				257.9357157117189 docs/sec

延时提交(Delayed commit)(和连续的UUID结合使用)可能是CouchDB最重要的性能配置选项了. 当它被设置为true时(这是默认的选项), CouchDB允许每个操作完成后不使用显式的fsync. Fsync操作会占用时间(硬盘可能需要进行查找, 在有些平台上硬盘缓存满了, 等等). 所以要求每个操作后都使用fsync极大的限制了CouchDB非块(non-bulk)写入的性能.

延时提交应该就按默认的被设置为true, 除非你的环境里要求当收到更新时必须需要知道(比如CouchDB作为一个更大的事务的一部分时). 也可以使用_ensure_full_commit API来触发fsync(比如, 在做完几个操作以后).

When delayed commit is disabled, CouchDB writes data to the actual disk before it responds to the client (except in batch=ok mode). It’s a simpler code path, so it has less overhead when running at high throughput levels. However, for individual clients, it can seem slow. Here’s the same benchmark in full commit mode:
当延时提交被禁用时, CouchDB会在响应客户端前, 的的确确的把数据写到硬盘上(除非是在batch=ok模式下). 

				single_doc_insert
				46 docs
				4.583042741855135 docs/sec

看看吧, 完整提升启用后的单文档插入--每秒只有4到5个文档! 因为Mac OS X有真正的fsync实现, 所以这是100%真实的结果, 感激吧! 不过别担心, 在块操作时, 完整提交会表现的更好些.

从另一方面来说, 对于大块的数据, 关闭延时提交可以有更好的表现. 这也让我们了解到, 优化一个应用时不按照某本参考书上的做, 总是会得到更好的结果.

### Hovercraft ###

Hovercraft是一个使用Erlang来访问CouchDB的库. Hovercraft基准测试应该会展示出CouchDB硬盘和索引子系统的最快性能表现, 因为它避免了所有的HTTP连接以及JSON转换开销.

Hovercraft主要在是HTTP接口不能满足足够的控制, 或者HTTP接口是多余的时候. 举个例子, 在CouchDB上实现Jabber实时消息系统可能会使用ejabberd和Hovercraft. 最简单的创建一个容错消息队列可能会是RabbitMQ和Hovercraft的结合.

Hovercraft是从一个客户端项目里提取出来的, 这个项目使用CouchDB来存储大量的电子邮件, 把它们作为文档的附件. HTTP接口没有一个简单的方法, 可以使得块数据更新可以结合二进制附件, 所以我们使用Hovercraft, 把一个Erlang SMTP服务器直接连接到了CouchDB. 这样在保持块数据更新高效性的同时, 可以直接把附件存储到硬盘上.

Hovercraft包括了一个基础的测试套件, 我们可以通过它得知每秒可以处理多少文档.

				> hovercraft:lightning().
				Inserted 100000 docs in 9.37 seconds with batch size of 1000.
				(10672 docs/sec)

### Trade-Offs ###

使用工具X可能可以得到5ms的响应时间, 要比市面上其他产品都快一个等级. 编程其实就是一个取舍的过程, 每个人都被约束在这条定律上.

On the outside, it might appear that everybody who is not using Tool X is a fool. But speed and latency are only part of the picture. We already established that going from 5 ms to 50 ms might not even be noticeable by anyone using your product. Speed may come at the expense of other things, such as:
在外界看来, 所有那些不使用工具X的人都像是傻瓜. 但是速度和延时只是一部分. 一个产品的延时从5ms升高到50ms可能没人会察觉出来. 速度可以换取其他的一些东西, 比如:

内存

为了不用一遍一遍的重复计算, 工具X可能会有一个不错的缓存层, 可以把计算结果保存在内存中. 如果你受限于CPU, 那么这样做不错; 但如果受限于内存, 这样做就不大好了. 这是一个取舍的过程.

并发
The clever data structures in Tool X are extremely fast when only one request at a time is processed, and because it is so fast most of the time, it appears as if it would process multiple requests in parallel. Eventually, though, a high number of concurrent requests fill up the request queue and response time suffers. A variation on this is that Tool X might work exceptionally well on a single CPU or core, but not on many, leaving your beefy servers idling.
工具X的智能数据结构在处理一个单一请求时非常的快, 因为它在大多数时候都是这样快, 所以看起来它在平行处理多个请求时也会同样的快. 虽然, 最终, 大量的并发请求填满了请求队列, 响应时间变得很长. lllll

可靠性

确保数据真正被存储了是一个昂贵的操作. 确保数据保存在一个一致的状态没有损坏也是. 这里有两个需要取舍的地方. 第一个, 在保存到硬盘之前, 使用缓存在内存中存储数据可以保证更高的数据处理能力. 在电源中断或者软硬件崩溃的时候, 这样做就会导致数据丢失. 应用可能接受, 也不可能不能接受这种数据丢失. 第二个, 错误发生后需要进行一致性的检查, 如果你有大量的数据, 这几天要耗费几天的时间. 如果你能承受应用上线, 那这样做没问题, 但很可能你根本承受不了.

要确保了解什么是你需要的, 并且选择那些满足你需要的工具, 而不要只是选择那些有些漂亮数据的工具. 当应用程序为了修复需要下线一天, 而你的客户根本没有耐心等待时, 或者更糟糕的, 数据丢失了时, 谁才是那个傻瓜呢? 

#### 但是...我的老板就是喜欢看那些个破数据! ####

你想要知道哪些数据库, 缓存, 编程语言, 语言框架, 以及工具更快, 更好, 更强壮. 数据会很酷--你可以画出漂亮的图表, 从而通过对比作出选择.

但是, 一个好的执行官首先要知道的事情是, 她所看到的是不足的数据, 因为根据数字画出的图表和真实情况是有出入的. 而那些没有经过仔细调查得出的图表根本没有任何用处.

如果你需要产生一些数字, 要确保明白结果中体现了多少信息, 哪些信息又不能体现出来. 在上交这些数字以前, 还要确保看的人也懂得这些道理. 当然, 最好的方法还是尽可能模拟真实世界的环境进行测试. 虽然这不是件容易的事情.

#### A Call to Arms #####

We’re in the market for databases and key/value stores. Every solution has a sweet spot in terms of data, hardware, setup, and operation, and there are enough permutations that you can pick the one that is closest to your problem. But how to find out? Ideally, you download and install all possible candidates, create a profiling test suite with proper testing data, make extensive tests, and compare the results. This can easily take weeks, and you might not have that much time.

We would like to ask developers of storage systems to compile a set of profiling suites that simulate different usage patterns of their systems (read-heavy and write-heavy loads, fault tolerance, distributed operation, and many more). A fault-tolerance suite should include the steps necessary to get data live again, such as any rebuild or checkup time. We would like users of these systems to help their developers find out how to reliably measure different scenarios.

We are working on CouchDB, and we’d like very much to have such a suite! Even better, developers could agree (a far-fetched idea, to be sure) on a set of benchmarks that objectively measure performance for easy comparison. We know this is a lot of work and the results may still be questionable, but it’ll help our users a great deal when figuring out what to use.

