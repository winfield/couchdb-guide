## 复制 ##

本章将介绍CouchDB的全世界最棒的复制系统. 复制系统可以同步得到两份相同的数据库拷贝, 这样不管用户在哪里, 都能以相当低的延时来访问数据. 这些数据库可以运行在同一个服务器, 也可以是在不同的服务器上--CouchDB对此不作区分. 如果你改变了其中一份数据库的拷贝, 复制系统会将这些改变发送到另一份拷贝.

复制是一个单方面的操作: 发送给CouchDB一个包含了源数据库与目标数据库的HTTP请求, 然后CouchDB就会把源数据库的改变发送到目标数据库. 这就是复制的全部, 把一样东西说成是全世界最棒的, 但却只需要一句话来解释它似乎是有些奇怪. 但是为什么CouchDB的复制系统如此强大, 其中一部分原因也在于其简单性.

让我们来看看复制是如何工作的:

				POST /_replicate HTTP/1.1
				{"source":"database","target":"http://example.org/database"}

这个调用会发送所有的在本地数据库"database"里的文档到远程数据库"http://example.org/database. 如果一个数据库和你发送POST /_replicate HTTP请求过去的是在同一个CouchDB实例上, 那它就被认为是一个本地数据库. 所有其他的都被认为是远程数据库.

If you want to send changes from the target to the source database, you just make the same HTTP requests, only with source and target database swapped. That is all.
如果你想要从目标数据库发送改变到源数据库, 你可以发送相同的HTTP请求, 只需要把源数据库和目标数据库的位置交换一下. 这样就行了.

				POST /_replicate HTTP/1.1
				{"source":"http://example.org/database","target":"database"}

A remote database is identified by the same URL you use to talk to it. CouchDB replication works over HTTP using the same mechanisms that are available to you. This example shows that replication is a unidirectional process. Documents are copied from one database to another and not automatically vice versa. If you want bidirectional replication, you need to trigger two replications with source and target swapped.

### The Magic ###

当你叫CouchDB复制一个数据库到另一个时, 它会去比较两个数据库, 找出那些源数据库和目标数据库不同的文档, 然后一组组的发送改变的文档到目标数据库直到所有的改变都被发送完毕. 改变包括新增文档, 改变的文档, 和删除的文档. 目标数据库上已经存在的有着相同版本的文档不会被发送; 只有更新版本的才会. 

CouchDB中的数据库有一个序列号, 每次数据库被改变时, 它都会增加1. CouchDB会记住哪一个变更对应于哪一个序列号. 用这种方式, CouchDB就可以回答像这样的问题, "从序列号212到现在, 数据库做了哪些变更?", 只需要返回一个新增文档和改变文档的列表就行. 用这种方式找出数据库间的变更是一种有效的操作. 它也增加的复制的健壮性.

CouchDB views use the same mechanism when determining when a view needs updating and which documents to replication. You can use this to build your own solutions as well.

你可以在一个CouchDB实例上使用复制来创建一个数据库的镜像, 以便用来测试代码而不必担心数据丢失或者用于查找老版本的数据库数据. 但如果你在不同地理位置的两台或者多台计算机上使用复制系统, 那会更加有趣. 

在不同的服务器上, 也许它们之间的距离有成百上千公里远, 问题就会产生. 服务器会崩溃, 网络连接会中断, 还有其他的糟糕事情会发生. 当一个复制的过程被中断时, 它会使得两个正在进行复制的CouchDB处于一个inconsistant状态. 然后, 当问题解决后, 你再触发复制, 它就会从中断的地方继续进行复制.

### 通过管理界面简单的复制 ###

你可以在浏览器中通过Futon--CouchDB的内建管理界面--来运行复制. 启动CouchDB然后在浏览器中打开http://127.0.0.1:5984/_utils/. 在右侧, 你会看到一个列表, 点击"Replication". 

Futon会展示一个用来运行复制的界面. 你既可以从列表里选择本地数据库也可以填写远程数据库的URL来指定一个源数据与目标数据库地址, 

点击Replicate按钮, 等待一会, 然后再看看屏幕的下方, CouchDB会在那里显示一些关于复制的统计数据, 也或者, 如果有错误发生了, 会显示一些解释信息.

好啦! 你已经运行了第一次复制.

### 关于复制的细节 ###

So far, we’ve skipped over the result from a replication request. Now is a good time to look at it in detail. Here’s a nicely formatted example:
到目前为止, 我们忽略了复制请求的结果. 现在让我们来仔细的看看. 下面是一个格式化好的例子:

				{
					"ok": true,
					"source_last_seq": 10,
					"session_id": "c7a2bbbf9e4af774de3049eb86eaa447",
					"history": [
						{
							"session_id": "c7a2bbbf9e4af774de3049eb86eaa447",
							"start_time": "Mon, 24 Aug 2009 09:36:46 GMT",
							"end_time": "Mon, 24 Aug 2009 09:36:47 GMT",
							"start_last_seq": 0,
							"end_last_seq": 1,
							"recorded_seq": 1,
							"missing_checked": 0,
							"missing_found": 1,
							"docs_read": 1,
							"docs_written": 1,
							"doc_write_failures": 0,
						}
					]
				}

"ok": true, 这部分和其他的响应相似, 告诉我们一切正常. source_last_seq包含了此次复制的源数据库的更新序列. 每一次复制请求都会被赋予一个session_id, 它只是一个UUID;  you can also talk about a replication session identified by this ID.

The next bit is the replication history. CouchDB maintains a list of history sessions for future reference. The history array is currently capped at 50 entries. Each unique replication trigger object (the JSON string that includes the source and target databases as well as potential options) gets its own history. Let’s see what a history entry is all about.
下一个部分是复制历史. CouchDB会维护一个历史会话的列表用于将来的参照. 

再次的, 这里的session_id只是为了方便. 复制的开始和结束时间也被记录. _last_seq域记录了这个会话开始和结束时数据库的更新序列. 再次的, recorded_seq是目标数据库的更新序列. 如果一个复制进程中途中断了然后又重新开始, 那它就会和end_last_seq不同. missing_checked是目标数据库已经有了的不需要被复制的文档的数量. missing_found是指没有的文档的数量.

最后三个--docs_read, docs_written, 和doc_write_failures--显示了我们从源数据库读取了多个文档, 写进了目标数据库多少文档, 以及多少写入失败了. 如果一切正常, 那么_read和_written应该是相同的且doc_write_failures应该是0. 如果不是, 就可以知道在复制过程中, 有什么出错了. 可能的失败原因会是任意一方的服务器崩溃, 网络连接中断, 或者validate_doc_update函数拒绝了一个文档的写入.

一个普遍的场景是在管理员帐号启用的节点之间进行复制. 设计文档只能由管理员创建, 所以如果进行没有管理员认证的复制, 那么在复制过程中写入设计文档就会失败, 并被记录为doc_write_failures. 如果你拥有管理员帐号, 请确保在复制请求中加入认证:

				> curl -X POST http://127.0.0.1:5984/_replicate  -d '{"source":"http://example.org/database", "target":"http://admin:password@e127.0.0.1:5984/database"}'

### 连续复制 ###

现在你已经了解了复制在底层是如何工作的了, 我们再讲一个小技巧. 如果你在复制请求里加入"continuous":true, CouchDB复制在发送完所有的变更文档后不会停止. 它会监听CouchDB的_changes API(请察看第20章, 变更提醒)并自动的将源数据库中的新改变文档发送至目标数据库. 实际上, 它们并不是立刻被复制; 有一个复杂的算法用于在服务器空闲时进行复制, 以达到最大的性能. 这个算法很复杂并且还在不断改进中, 所以在这里讲解它没有太大的意义.

				> curl -X POST http://127.0.0.1:5984/_replicate -d '{"source":"db", "target":"db-replica", "continuous":true}'

在本书写作之时, CouchDB在重启后还不能记住连续复制. 所以现在, 在重启CouchDB后, 你需要再一次的触发它们. 将来, CouchDB会允许你定义一个永久的连续复制, 这样即便服务器重启了你也不必再做任何操作.

### 就这些? ###

复制是接下来几个章节的基础. 请务必让自己已经理解了本章节的内容. 如果还有些不明白, 就再看一遍, 然后在Futon里耍耍复制界面.

我们还没有告诉你关于复制的全部内容. 接下来的几章会讲解如何处理复制冲突(请查看第17章, 冲突处理), 如何使用一组同步的CouchDB实例进行负载均衡(请查看第18章, 负载均衡), 以及如何如何构建一个CouchDB集群来处理更多的数据或者更多的写请求(请查看第19章, 集群).
