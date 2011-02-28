## 高性能 ##

This chapter will teach you the fastest ways to insert and query data with CouchDB. It will also explain why there is a wide range of performance across various techniques.
本章会教会你如何用最快的方式插入数据到CouchDB以及如何以最快的方法来查询数据. 同时也解释为什么使用不同的技术会产生不同的性能.

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

