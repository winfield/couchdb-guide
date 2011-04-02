## Recipes ##

本章会展示一些一般任务在CouchDB里的最佳实践. 我们会给出易懂的一步步的指导.

### 银行 ###

银行的业务是严肃的. 它们需要严肃的数据库来存储严肃的交易和严肃的帐户信息. 它们不能丢失任何钱. 他们同时也不能造出钱来. 一个银行必须在所有时候都处于平衡状态.

传统的观点认为一个数据库需要支持事务才能算可以在严肃的场合使用. CouchDB不支持传统意义上的事务(虽然它是以事务的方式工作的), 所以你会认为CouchDB并不适合于存储银行数据. 另外, 你会认为把钱放在沙发里安全吗? 好吧, 我们会. 本章会解释为什么.

### 会计不使用橡皮擦 ### 

假设你想要给你在纽约的堂弟保罗寄100美元. 在那个遥远的年代里, 你必须要跑去纽约亲自把钱交给保罗, 或者你会通过平信来邮寄这笔钱. 这两种方法都相当的不方便, 所以人们开始找别的法子. 然后有一天, 银行开始提供服务可以管理好你的钱, 确保钱会平安的送到保罗的手中. 当然, 他们会收一点点的费用, 但是人们愿意付这点小钱, 免的要跑一趟纽约. 在幕后, 银行会派某一个人把钱送到保罗的银行--这和以前跑腿的过程没什么两样, 只是换了另一个人来处理这种麻烦事. 银行也可以批量的进行资金的转移; 而不需要单独派人. 他们可以每周收集一次到纽约的钱, 然后把它们一次性送到. 如果遇到了什么问题--比如, 收款人已不再是那个银行的客户(记得吗, 在那个年代, 从东海岸到西海岸要花费数周)--钱就会送回到原来的帐户里.

最终, 现代化的银行系统被采用了, 而真正的资金送出送回从此停止. 银行只在帐面上有钱, 可以不使用真正的货币而进行资金的交互. 虽然老的概念还存在于我们的脑子里. 要从我们的银行帐户中转给某人一笔钱, 银行需要把钱从我们的帐户里取出来, 然后放到接收帐户里. 但是, 在现在的年代里, 我们希望什么事都可以是实时的. 要在Amazon上订点东西, 只需要点几下鼠标, 它们马上就会发邮件给你了. 所以, 为什么银行的交易要更长的时间呢?

现在, 银行都已经实现了电子化(并且已经实现了相当长的一段时间了). 当我们要作一个资金转移时, 希望它可以实时完成, 并且还希望它会像以前那样的方式处理: 从我的帐户里把钱拿出来, 加到保罗的帐户上, 如果出现了什么问题, 就把钱放回我的帐户. 虽然逻辑上是这样没错, 但在幕后却并非如此, 并且在计算机出现之前也不是这么做的.

当你去银行, 要求转一笔钱给保罗的时候, 会计人员会记录下这个事件, 发起一笔交易. 交易会包含日期, 数量和收款人. 还记得吧, 银行需要一直保持在平衡状态. 从你的帐户取出的钱不能凭空消失. 会计人员会把钱移动到一个银行为你维护的转移帐户中. 现在你的帐户余额是现有帐户里的资金和转移帐户中的资金的和. 这时候, 银行会检查保罗的帐户是否和你描述的一样, 并检查资金是否可以被安全送到. 如果检查没问题, 资金就会通过另外一个独立的交易, 从转移帐户中转到保罗的帐户中. 一切都处于平衡状态. 注意, 这里有多个独立的交易, 而不一个大型的, 包含了一系列动作的交易.

现在, 让我们想像一下, 如此有错误发生了: 假设保罗的帐户已经不存在了. 银行进行转移帐户批量操作时, 检查到了这一点. 另一个交易会因此产生, 它会把钱从转移帐户移回你的帐户. 注意, 把钱移回你的帐户不是取消上一个交易. 而是上一个交易的一个逆向交易.

还有一个可能的错误情况: 假设你没有足够的钱转出100美元给保罗. 这在银行进行任何扣款交易时, 由会计人员(或者软件)检查出来. 出于责任, 银行不能假装一个操作没有发生过; 它必须把每一个操作运作都在日志里记录下来. 撤消是通过一个显式的逆向操作来完成的, 而不是通过恢复或者删除一个已存的交易. "会计不使用橡皮擦", 这是引用了Pat Helland的话, 它是交易处理系统方面的高级架构师, 曾工作于微软和Amazon.

再重复一次, 一个交易可以成功或者失败, 没有第三种可能. CouchDB可以保证只会成功或者失败的唯一操作是单一文档的写入. 任何需要用到交易的操作都需要结合单一文档写入. 如果业务逻辑检测到一个错误发生(比如, 没有足够的资金), 就必须创建一个逆向的交易.

让我们来看一个CouchDB的例子. 我们在前面提到过, 你的帐户余额是一个合计. 如果坚持这一观点去看问题的话, 事情就非常简单了. 我们不会去更新两个帐户的余额(你的和保罗的, 或者你的帐户和你的转移帐户), 只是简单的创建一个交易文档来描述正在做的事情, 然后使用一个视图来合计余额.

想像一下有很多交易: 

				...
				{"from":"Jan","to":"Paul","amount":100}
				{"from":"Paul","to":"Steve","amount":20}
				{"from":"Work","to":"Jan","amount":200}
				...

在CouchDB中, 单一文档的写入的原子化的. 查询一个视图会强制更新所有改变文档的视图索引. 视图结果总是和文档中的数据保持一致. 这保证了银行会一直处于平衡状态. 当然还会有其他很多的交易, 但这些已经足够用以演示的了.


如何读取目前的帐户余额呢? 简单--创建一个MapReduce视图:

				function(transaction) {
					emit(transaction.from, transaction.amount * -1);
					emit(transaction.to, transaction.amount);
				}
				function(keys, values) {
					return sum(values);
				}

看起来并不难吧? 我们会把它保存在设计文档_design/account中的视图balance里. 让我们来找出一月份的余额吧:

				curl 'http://127.0.0.1:5984/bank/_design/account/_view/balance?key="Jan"'

CouchDB响应:

				{"rows":[
				{"key":null,"value":100}
				]}

看起来不错! 现在让我们来看看银行是不是真的处在平衡状态. 所有交易的和应该是0

				curl http://127.0.0.1:5984/bank/_design/account/_view/balance

CouchDB响应:

				{"rows":[
				{"key":null,"value":0}
				]}

#### 收尾 ####

这个例子应该已经解释清楚了, 那些需要强一致性的应用, 如果可以把大的事务分成几个小的事务, 是可以使用CouchDB来实现的. 银行可以算是严肃业务的一个近似, 所以你可以放心的把重要的业务逻辑建模到CouchDB的小事务上来.

### 排序列表 ###

视图让你可以使用数据的任何值来进行排序--甚至是复杂的JSON, 就像我们在前面几章有看到的那样. 根据时间排序可以让用户快速的找到想要的东西; 在一个以按字母顺序排序的名字列表里, 找出要找的名字也要简单的多. 人们总是会自然的诉诸于分治(divide-and-conquer)算法(听起来很熟悉?), 而不会去考虑另外一大块的输入集, 因为他们知道要找的名字不会出现在那里. 类似的, 根据号码和日期排序, 为用户管理大断增长的数据帮了大忙.

还有一种排序方法更加模糊一些. 搜索引擎会根据相关性来排序结果. 相关结果是搜索引擎认为的, 根据你的搜索条件(以及潜在的, 搜索历史和浏览历史)最相关的那些结果. 还有其他的一些系统, 它们试图去影响先前的相关性结果, 但是它们几乎都不可能猜出用户感兴趣的是什么. 计算机不擅长于猜测那是众所周知的.

要计算机让找出什么是最相关的, 最简单的办法就是让用户自己来划定优先级. 拿待办事项应用来说吧: 它允许用户重新排序待办事项, 这样他们就可以知道哪项是接下来要做的工作. 而其底层的问题--保持一个用户定义的排序列表--这在很多其他地方也会被用到.

#### 一个整数的列表 ####

让我们继续来看待办事项应用这个例子. 有一个实现方法很简单: 每个待办事项都保存一个整数来指定其在列表中的位置. 我们使用一个视图来得到所有待办事项的正确排序. 首先, 我们需要一些示例文档:

				{
					"title":"Remember the Milk",
					"date":"2009-07-22T09:53:37",
					"sort_order":2
				}

				{
					"title":"Call Fred",
					"date":"2009-07-21T19:41:34",
					"sort_order":3
				}

				{
					"title":"Gift for Amy",
					"date":"2009-07-19T17:33:29",
					"sort_order":4
				}

				{
					"title":"Laundry",
					"date":"2009-07-22T14:23:11",
					"sort_order":1
				}

接下来, 我们要创建一个带有一个简单的map函数的视图, 这个map函数会产生记录并根据文档的sort_order域进行排序. 视图结果和我们想的一样:

				function(todo) {
					if(todo.sort_order && todo.title) {
						emit(todo.sort_order, todo.title);
					}
				}

				{
					"total_rows": 4,
					"offset": 0,
					"rows": [
						{
							"key":1,
							"value":"Laundry",
							"id":"..."
						},
						{
							"key":2,
							"value":"Remember the Milk",
							"id":"..."
						},
						{
							"key":3,
							"value":"Call Fred",
							"id":"..."
						},
						{
							"key":4,
							"value":"Gift for Amy",
							"id":"..."
						}
					]
				}

看起来问题相当简单嘛, 但是你能发现其中存在的问题吗? 给你一个提示: 如果要把a gift for Amy的优先级调高于remember the milk, 我们应该怎么做? 理论上来说, 这项工作的要求很简单:

1, 把"Remember the Milk"的sort_order赋值给"Gift for Amy."

2, 把"Remember the Milk"以及所有低于其优先级的待办事项的sort_order都加1.

而在底层, 需要做的事情有很多. 在CouchDB里, 你需要载入所有的文档, 增加它们的sort_order, 然后再保存回去. 如果你拥有大量的待办事项(我是有一堆的), 那么这将会是一个巨大的工程. 也许会有一个更好的实现方法.

#### 一个浮点数的列表 ####

改进办法很简单: 我们使用浮点数, 取代整数来指定一个排序位置:

				{
					"title":"Remember the Milk",
					"date":"2009-07-22T09:53:37",
					"sort_order":0.2
				}

				{
					"title":"Call Fred",
					"date":"2009-07-21T19:41:34",
					"sort_order":0.3
				}

				{
					"title":"Gift for Amy",
					"date":"2009-07-19T17:33:29",
					"sort_order":0.4
				}

				{
					"title":"Laundry",
					"date":"2009-07-22T14:23:11",
					"sort_order":0.1
				}

视图仍然保持不变. 读取列表和先前的实现方法一样简单. 而重新排序则要比先前的容易多了. 应用程序前端可以保存一份sort_order的值, 当移动一个待办事项时, 保存这一次移动, 这样我们不仅有了新位置的值, 还有其上和其下两个待办事项的sort_order.

让我们来把"Gift for Amy"到"Remember the Milk"上面. 此时, 它的上下两个待办事项的sort_order分别是0.1和0.2. 要保存"Gift for Amy"的正确sort_order, 我们只需要简单的使用其上其下两个待办事项sort_order的中间值就可以了: (0.1 + 0.2) / 2 = 0.3 / 2 = 0.15.

如果再一次查询视图, 我们就可以得到想要的结果了:

				{
					"total_rows": 4,
					"offset": 0,
					"rows": [
						{
							"key":0.1,
							"value":"Laundry",
							"id":"..."
						},
						{
							"key":0.15,
							"value":"Gift for Amy",
							"id":"..."
						},
						{
							"key":0.2,
							"value":"Remember the Milk",
							"id":"..."
						},
						{
							"key":0.3,
							"value":"Call Fred",
							"id":"..."
						}
					]
				}

这一实现方法的负面因素是, 随着重新排序待办事项数量的增加, 浮点精度会成为一个问题. 一个解决办法是不去管它, 认为一个单一的用户不会越过限制. 或者, 在用户不活跃的时候, 使用一个管理任务来把整个列表重置为1位小数. 

这一实现方法的优势则在于, 只需要处理一个文档. 这就使得保存新的列表顺序, 以及更新用来维护排序列表的视图变得非常高效, 因为只需要把变更的文档合并到索引中就行了.

### 分页 ###

这一部分解释了如何来对视图结果进行分页. 分页是一种用户界面(UI)模式, 它可以用来显示大量的记录(结果集), 而不必一次就全部载入所有记录. 一个固定大小的子集--称之为"页"--会被展示出来, 其中带有上一页, 下一页的链接或者按钮, 可以跳转到邻近的页面. 

我们假设你已经熟悉了如何创建和查询文档, 视图, 以及视图的多个查询选项.

#### 示例数据 ####

为了有数据可以使用, 我们创建一个乐队的列表, 每一个乐队作为一个文档:

				{ "name":"Biffy Clyro" }

				{ "name":"Foo Fighters" }

				{ "name":"Tool" }

				{ "name":"Nirvana" }

				{ "name":"Helmet" }

				{ "name":"Tenacious D" }

				{ "name":"Future of the Left" }

				{ "name":"A Perfect Circle" }

				{ "name":"Silverchair" }

				{ "name":"Queens of the Stone Age" }

				{ "name":"Kerub" }

#### 视图 ####

我们需要一个简单的map函数, 用来生成一个以字母顺序排序的乐队名字列表. 这应该很简单, 但是我们加入了一些额外的过滤器, 去掉了乐队名字前缀中的"The"和"A":

				function(doc) {
					if(doc.name) {
						var name = doc.name.replace(/^(A|The) /, "");
						emit(name, null);
					}
				}

视图结果就是一个以字母顺序排序的乐队名字的列表. 现在假设我们想要每5个一页来显示乐队名字, 并且想要下一页和上一页的链接(如果不在第一页的话).

我们在前面几章已经学过了如何使用startkey, limit以及skip参数了. 在这里, 将再次使用到它们. 首先, 我们来看一下完整的结果集:

{"total_rows":11,"offset":0,"rows":[
  {"id":"a0746072bba60a62b01209f467ca4fe2","key":"Biffy Clyro","value":null},
  {"id":"b47d82284969f10cd1b6ea460ad62d00","key":"Foo Fighters","value":null},
  {"id":"45ccde324611f86ad4932555dea7fce0","key":"Tenacious D","value":null},
  {"id":"d7ab24bb3489a9010c7d1a2087a4a9e4","key":"Future of the Left","value":null},
  {"id":"ad2f85ef87f5a9a65db5b3a75a03cd82","key":"Helmet","value":null},
  {"id":"a2f31cfa68118a6ae9d35444fcb1a3cf","key":"Nirvana","value":null},
  {"id":"67373171d0f626b811bdc34e92e77901","key":"Kerub","value":null},
  {"id":"3e1b84630c384f6aef1a5c50a81e4a34","key":"Perfect Circle","value":null},
  {"id":"84a371a7b8414237fad1b6aaf68cd16a","key":"Queens of the Stone Age","value":null},
  {"id":"dcdaf08242a4be7da1a36e25f4f0b022","key":"Silverchair","value":null},
  {"id":"fd590d4ad53771db47b0406054f02243","key":"Tool","value":null}
]}

#### 设置 ####

分页的机制非常简单:

* 先显示第一页
* 如果还有更多的记录需要显示, 就加上下一页的链接
* 显示接下来的页面
* 如果不是第一页, 就加上上一页的链接
* 如果还有更多的记录需要显示, 就加上下一页的链接

或者用JavaScript伪码片段来表示就是:

				var result = new Result();
				var page = result.getPage();

				page.display();

				if(result.hasPrev()) {
					page.display_link('prev');
				}

				if(result.hasNext()) {
					page.display_link('next');
				}

#### 慢速分页(不要使用这种方式) ####

不要使用这一方法! 我们在这里展示它, 是因为它看上去似乎很自然, 而你需要知道为什么这样做不好. 为了要从结果集中得到最初的5条记录, 可以使用?limit=5查询参数:

				curl -X GET http://127.0.0.1:5984/artists/_design/artists/_view/by-name?limit=5

结果:

				{"total_rows":11,"offset":0,"rows":[
					{"id":"a0746072bba60a62b01209f467ca4fe2","key":"Biffy Clyro","value":null},
					{"id":"b47d82284969f10cd1b6ea460ad62d00","key":"Foo Fighters","value":null},
					{"id":"45ccde324611f86ad4932555dea7fce0","key":"Tenacious D","value":null},
					{"id":"d7ab24bb3489a9010c7d1a2087a4a9e4","key":"Future of the Left","value":null},
					{"id":"ad2f85ef87f5a9a65db5b3a75a03cd82","key":"Helmet","value":null}
				]}

通过比较total_rows的值和limit的值, 我们可以判断是否有更多的页面需要展示. 我们还可以通过offset的值知道, 现在处于第一页上. 我们可以计算出skip=的值来得到下一页的结果集:

				var rows_per_page = 5;
				var page = (offset / rows_per_page) + 1; // == 1
				var skip = page * rows_per_page; // == 5 for the first page, 10 for the second ...

所以我们用下面这样的方式来进行查询:

				curl -X GET 'http://127.0.0.1:5984/artists/_design/artists/_view/by-name?limit=5&skip=5'

注意, 我们必须要使用'(单引号)来转义&这个字符, 它在shell里是一个特殊字符.

结果如下:

				{"total_rows":11,"offset":5,"rows":[
					{"id":"a2f31cfa68118a6ae9d35444fcb1a3cf","key":"Nirvana","value":null},
					{"id":"67373171d0f626b811bdc34e92e77901","key":"Kerub","value":null},
					{"id":"3e1b84630c384f6aef1a5c50a81e4a34","key":"Perfect Circle","value":null},
					{"id":"84a371a7b8414237fad1b6aaf68cd16a","key":"Queens of the Stone Age","value":null},
					{"id":"dcdaf08242a4be7da1a36e25f4f0b022","key":"Silverchair","value":null}
				]}

实现hasPrev()和hasNext()方法也是相当直观的:

				function hasPrev()
				{
					return page > 1;
				}

				function hasNext()
				{
					var last_page = Math.floor(total_rows / rows_per_page) +
						(total_rows % rows_per_page);
					return page != last_page;
				}

#### The dealbreaker #####

所有这些看起来都很简单和直观, 但是这种实现方法有一个致命的错误. 还记得视图结果是如何从底层的B-tree索引中产生的吗: CouchDB会跳到第一条记录(或者如果提供startkey这个参数的话, 那么就是满足这一参数的第一条记录), 然后一条一条的接着读取, 直到没有记录为止(或者是符合了limit或endkey的参数, 如果有提供的话).

skip参数是这样工作的: 除了会跳到第一条记录来开始读取, skip会跳过指定数量的记录, 但是CouchDBw还是从第一行开始读取了; 它只是没有把那些跳过的记录返回而已. 如果指定了skip=100, 那么CouchDB仍然会读取这100条记录, 只是不会为它们创建输出而已. 这听起来也还不算是太坏, 但其实这样做是非常不好的, 你可能会有1000, 甚至是10000条需要跳到的记录. 这时, CouchDB会读取大量的不必要的记录. 

一般来说, skip应该只使用一个个位数的值. 但在有些案例中, 指定一个较大的值也是合理的, 这也给你的处理方式是否有潜在问题敲响了警钟. 最后, 为了计算, 你需要一个reduce函数, 然后在每个页面调用来得到正确的数量. 这也会存在潜在的错误.

#### 快速分页(应该使用这种方式) ####

正确的解决方法也难不了多少. 我们不再把结果集以多少记录一页进行分割, 而是每次读取10条, 再使用startkey来跳到接下来的10条记录. 我们还会使用skip参数, 但只给它赋值为1.

下面是其工作方式:

* 从视图中请求row_per_page + 1条记录
* 显示row_per_page条记录, 把row_per_page+1那条记录保存为next_startkey和next_startkey_docid
* 作为页面信息, 把startkey和next_startkey保存下来.
* 使用next_*的值来创建下一页的链接, 使用另外保存的值来创建上一页的链接

用于找出下一页的技巧相当简单. 每次读取11条记录, 而不是10条, 但是只显示10条记录, 然后使用第11条记录中的值来作为下一页的startkey. 要生成上一页的链接则只需要简单的把当前的页面的startkey带到下一页就行了. 如果不存在上一个startkey, 那么我们就是处在第一页. 如果得到的结果集中记录的数量等于或小于rows_per_page, 那么就不再生成下一页的链接. 这种方式被叫做链式列表分页(linked list pagination). 因为我们是一页一页的读取记录, 而不是直接跳到一个已经预先生成好的页面. 虽然这里还有一个需要注意的地方. 你能发现它吗?

CouchDB视图的key不要求是唯一的; you can have multiple index entries read. What if you have more index entries for a key than rows that should be on a page? startkey会跳到第一条记录, 如果CouchDB没有一个另外的参数可以来区分的话, 那就搞砸了. 所有有相同值的key在内部都是以docid排序的, docid就是生成视图记录的文档的ID. 你可以使用startkey_docid和endkey_docidc参数来得到这些记录的子集. 对于分页来说, 我们用不到endkey_docid, 但是startkey_docid要用到. 如果用于找出下一页的那条多余记录的key和当前的startkey相同, 在这种情况下, 也只是在这种情况下, 除了startkey和limit参数以外, 你还需要使用startkey_docid来进行分页. 

要注意, *_docid参数只能作为*key参数的附加条件使用, 并且只适用于把视图结果进一步缩小至一个单一key的时候. 它不能单独使用(唯一的一个例外是内置的_all_docs视图, 它已经是由文档ID来排序的了).

这种实现方法的优势在于, 所有的key操作都是建立在视图背后快速的B-tree索引上的. 显示一个页面不需要扫描大量的不必要的记录.

#### Jump to Page ####

链式列表(linked list)风格分页方式的一个弊端在于, 你不能预先计算好某一页码的记录. 跳转到一个特定的页面是做不到的. 如果有人提出此类问题, 我们本能的反应是, "连Google也没有那样做!", 然后我们倾向于不管这个问题. Google总是在第一页假装它已经找出了另外10页的结果. 只有在你点击第二次的时候(实际上很少有人会这么做), Google才会显示出更多的页面链接. 如果继续翻页, 就会得到先前页面的链接和接下来10页的链接, 但没有更多的了. 预先计算好20页需要的startkey和startkey_docid是很容易的操作, 而要实际计算好每一页的记录将会是一个极其巨大的工程.

如果你真的需要在所有文档的范围上, 跳转到一个特定页码(我们已经看到过有这样需求的应用了), 你仍然可以维护一个整数索引来作为视图索引, 然后使用一个混合的实现来解决分页问题.
