## 变更通知 ##

假设你正使用CouchDB来创建一个消息服务. 每个用户都有一个收件箱, 其他用户会把信息发送至收件箱. 当用户想要阅读所有收到的消息时, 他们只要打开收件箱就可以看到所有的消息.

目前来看, 一切很简单, 但是这需要用户一直不停的点击刷新按钮, 来检查是不是收到了新的消息. 这通常被称为轮询(polling). 大量的用户会产生了大量的请求, 而在大多数时候, 他们不做任何事情只是在刷新所有已读的消息.

如果能在新消息收到时, 让CouchDB给你一个提示是不是很棒呢? 数据库的_changes API正是用来实现这个的.

刚才描述的场景可以看成是一个缓存过期问题; 它是指, 什么时候我才知道, 现在所显示的东西已经和底层数据不相对应了呢? 任何形式的缓存过期, 不管是后端/前端相关的, 都可以使用_changes API.

_changes同样被设计并适用于一个数据库的确切活动, 不管是用于简单的展示, 还是用来同样重要的--对一个新增文档(或者一个文档变更)作出响应.

使用_changes API系统的美妙之处在于它们是解耦的. 一个程序只对最新的变更感兴趣, 而不需要知道是哪一个程序创建了新文档, 反之亦然.

下面就是一个变更对象的样子:

				{"seq":12,"id":"foo","changes":[{"rev":"1-23202479633c2b380f79507a776743d5"}]}

有三个域:

seq
:   文档创建或者变更时产生的数据库的更新序列号.

id
:   文档的ID.

changes
:   一个域的数组, 默认包含文档的版本号, 但也可以包含文档冲突以及其他的信息.

_changes API在每个数据库里都可以使用. 每个请求都可以取得一个单一数据库中的变更. 但是如果需要多个数据库的变更, 你也只要简单的发送多个请求到多个数据库的_changes API就行了.

让我们创建一个在本章节作为例子使用的数据库:

				> HOST="http://127.0.0.1:5984"
				> curl -X PUT $HOST/db
				{"ok":true}

有三种方式来获取通知: 轮询(polling)(默认的), 长轮询(long polling)和连续变更(continuous). 每一种适用于不同的场景, 我们会对每一种都进行仔细的讨论.

### 轮询变更 ###

在前面的例子里, 我们试图去避免轮询(polling)的方式, 但是它实在是一个非常简单的办法, 并且在有些时候是唯一的可以解决问题的方法. 因为它是最简单的方案, 所以它成为了changes API的默认方式.

让我们来看看我们的测试数据库是如何做的. 首先, 作出请求(我们又一次使用到了curl):

				curl -X GET $HOST/db/_changes

结果很简单:

				{"results":[

				],
				"last_seq":0}

什么都没有, 因为我们还没有在数据库放入任何东西--这没什么让人惊奇的. 但是你应该可以猜到当有什么放入时我们就会看到有结果了. 让我们来创建一个文档:

				curl -X PUT $HOST/db/test -d '{"name":"Anna"}'

CouchDB响应:

				{"ok":true,"id":"test","rev":"1-aaa8e2a031bca334f50b48b6682fb486"}

现在我们再跑一次变更请求:

				{"results":[
				{"seq":1,"id":"test","changes":[{"rev":"1-aaa8e2a031bca334f50b48b6682fb486"}]}
				],
				"last_seq":1}

我们得到了新文档的通知. 非常棒! 但是等一下--当我们创建了文档并且已经得到了像版本号之类的信息后, 为什么我们还要请求changes API来再一次得到它呢? 还记得吗, changes API的目的就是让你可以构建解耦的系统. 创建文档的程序很有可能和向数据库请求变更的程序并不是同一个, 因为前者早就知道放入数据库的是什么了(虽然概念有些模糊, 但是这个程序也可能对其他程序作出的变更感兴趣).

在后台, 我们创建另外一个文档. 让我们来看看这次数据库的变更是什么样的:

				{"results":[
				{"seq":1,"id":"test","changes":[{"rev":"1-aaa8e2a031bca334f50b48b6682fb486"}]},
				{"seq":2,"id":"test2","changes":[{"rev":"1-e18422e6a82d0f2157d74b5dcf457997"}]}
				],
				"last_seq":2}

看到了吗, 结果中有了一行新的, 代表了新创建的文档? 此外, 第一个创建的文档在列表中再一次出现了. changes API的默认结果是数据库看到的所有变更的历史.

我们已经了解了"seq":1的变更, 对它已经不再感兴趣了. 我们可以告诉changes API这一点, 通过在查询参数里使用since=1:

				curl -X GET $HOST/db/_changes?since=1

这会返回所有这个序列号之后的变更:

				{"results":[
				{"seq":2,"id":"test2","changes":[{"rev":"1-e18422e6a82d0f2157d74b5dcf457997"}]}
				],
				"last_seq":2}

说到选项, 使用style=all_docs可以在每一行结果的changes数组中得到更多的版本和冲突信息. 如果你想要明确的指定默认行为, 选项值是main_only.

### 长轮询(Long Polling) ###

长轮询(long polling)是浏览器发明的, 用于消除普通轮询实现存在的一个问题的一项技术: 如果没有变更发生它就不会运行任何请求. 长轮询(long polling)是这样做的: 当作出一个到长轮询(long polling) API的请求时, 会打开到CouchDB的HTTP连接直至变更结果中出现新的一行, 并且你和CouchDB都要保持这个HTTP连接存在. 一旦新结果出现, 连接就被中断. 

这对于低频率的更新很有用. 如果一个客户端产生了大量的变量, 你就会发现自己打开了许多的新连接, 这时这种实现方式相比于普通的轮询就没有什么优势了. 这项技术的另一个结果是对于每一个请求长轮询(long polling)变更通知的客户端, CouchDB都必须要保持一个HTTP连接打开. CouchDB很能胜任这样的任务, 它就是被设计成可以处理大量并发请求的. 但是你必须要保证, 你的操作系统允许CouchDB可以使用至少和所需要处理的长轮询(long polling)客户端数量相同的套接字(当然, 还有其他用于正常请求连接的).

要作一个长轮询(long polling)请求, 在查询参数里加入feed=longpoll. 下面的列表里, 我们加入的时间戳用来清楚展现发生的事.

				00:00: > curl -X GET "$HOST/db/_changes?feed=longpoll&since=2"
				00:00: {"results":[
				00:10: {"seq":3,"id":"test3","changes":[{"rev":"1-02c6b758b08360abefc383d74ed5973d"}]}
				00:10: ],
				00:10: "last_seq":3}

在00:10, 我们在后台创建了另一个文档, CouchDB正确的发送给了我们变更通知. 注意我们使用的参数since=2来避免得到任何先前变更的通知. 还要注意的是, 必须在curl命令中使用双引号, 因为我们使用了一个ampersand, 它在我们的shell里是一个特殊字符.

长轮询(long polling)请求的选项和普通的轮询请求的一样.

网络很不可靠, 所以有时你不知道到底是没有变更呢, 还是网络连接哪里出问题了. 如果你在查询参数里加入另外一个选项, hearheat=N, 这里N是一个数字, 那么CouchDB就会每N毫秒发送我你一个newline字符. 有了这个newline字符, 你就知道还没有变更通知, 而CouchDB仍然准备好了在变更发生时通知你.

### 连续变更 ###

长轮询很棒, 但是你仍然需要为每一个变更通知作一次HTTP请求. 对于浏览器来说, 这是唯一的可以避免普通轮询所带有的问题的方法. 但是浏览器并不是唯一的和CouchDB打交道的客户端. 如果你使用Python, Ruby, Java, 或者任何其他语言, 那么还有另外一个选择. 

连续变更API允许你使用一个HTTP连接就能够在有变更发生时收到通知. 你发送一个连续变更API请求, 然后你和CouchDB都将"永远"的保持这个连接打开. CouchDB在事件发生时会发送给你newline字符--正好和长轮询相反--此后它还会继续保持连接打开, 等待发送下一个通知.

这对于不频繁的和频繁的通知来说都非常好, 并且它和长轮询有着相同的结果: 会拥有大量的长时间的HTTP连接存在. 但是再一次的, CouchDB可以轻松的支持它们.

在参数中加入feed=continuous来作出一个连续变更的API请求. 下面就是结果, 这次我们还是加上了时间戳. 在00:10和00:15, 我们分别创建了一个新文档:

				00:00: > curl -X GET "$HOST/db/_changes?feed=continuous&since=3"
				00:10: {"seq":4,"id":"test4","changes":[{"rev":"1-02c6b758b08360abefc383d74ed5973d"}]}
				00:15: {"seq":5,"id":"test5","changes":[{"rev":"1-02c6b758b08360abefc383d74ed5973d"}]}

Note that the continuous changes API result doesn’t include a wrapping JSON object with a results member with the individual notification results as array items; it includes only a raw line per notification. Also note that the lines are no longer separated by a comma. Whereas the regular and long polling APIs result is a full valid JSON object when the HTTP request returns, the continuous changes API sends individual rows as valid JSON objects. The difference makes it easier for clients to parse the respective results. The style and heartbeat parameters work as expected with the continuous changes API.

### 过滤器 ###

变更通知API以及它的三种模式已经给了你很多选项来请求处理变更. 变更过滤器则给了你另一个层次的灵活性. 让我们假设第一个场景中的消息拥有优先级, 并且一个用户只对于高优先级的消息感兴趣.

我们来讲讲过滤器. 类似于视图函数, 一个过滤器是一个JavaScript函数, 它被存储于一个设计文档, 并在之后由CouchDB来执行. 它们被放入一个特殊的域filters中用你喜欢的名字命名. 下面就是一个例子:

				{
					"_id": "_design/app",
					"_rev": "1-b20db05077a51944afd11dcb3a6f18f1",
					"filters": {
						"important": "function(doc, req) { if(doc.priority == 'high') { return true; }
						else { return false; }}"
					}
				}

要想查询这个查询API的过滤器, 在参数中加入filter=desingnocname/filtername:

				curl "$HOST/db/_changes?filter=app/important"

这时结果里面就只包含也过滤器函数返回true的文档变更了--在我们的例子里就是那些优级为高的文档变更. 很棒吧, 但CouchDB会把它带到一个更高的层次.

让们再拿最开始那个用户相互发送消息的应用来做例子. 这次不是每个用户拥有一个收件箱数据库, 我们为所有用户只使用一个单一的数据库作为收件箱. 这时用户如何收到属于她自己的新消息呢?

我们可以使用一个带有查询参数的过滤函数:

				function(doc, req)
				{
					if(doc.name == req.query.name) {
						return true;
					}

					return false;
				}

现在如果你在查询里加入参数?name=Steve, 过滤函数就只会返回那些文档的name域等于"Steve"的文档. 如果你想要查询另外一个用户的变更, 只要改变查询参数就可以了(比如name=Joe).

在查询中加入一个参数是件很简单的事情. 不过如何防止Steve在查询中加入参数name=Joe来查看Joe的收件箱呢? CouchDB可以防止此类事件的发生吗? 如果不能我们就不会提出这个问题了, 是吧, 哈?

过滤函数的req参数中包含一个域userCtx, 它是指用户上下文. 它包含了先前通过HTTP认证的用户的信息. 具体的来说, req.userCtx.name包含了作出这个过滤变更查询的用户的用户名. 我们可以确定这个用户就是他本人, 因为他已经在CouchDB的认证系统里经过认证了. 有了这个, 我们甚至不需要动态的过滤参数了(尽管在其他的情况下, 它还是有用的).

如果你已经把CouchDB设置为需要认证, 则用户必须要作出经过认证的请求, 我们的过滤函数则变成这样:

				function(doc, req)
				{
					if(doc.name) {
						if(doc.name == req.userCtx.name) {
							return true;
						}
					}

					return false;
				}

### 收尾 ###

变更API让你可以在不同的场景中构建复杂的通知系统, 并且与其他独立的异步的组件共存. 与复制系统一起, 这个API是构建分布式, 高可用性, 高性能的CouchDB集群的基石.
