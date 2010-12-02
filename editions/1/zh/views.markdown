## 使用视图来查找你的数据 ##

视图在用于完成多种目标时都很有用:

* 在数据库中找出与某一特定进程相关的文档.
* 从你的文档中找出数据, 并用特定的方式展现出来.
* 创建高效的索引, 使用文档内的任何值或者任何结构来找出文档.
* 使用这些索引来表示文档间的关系.
* 最后, 使用视图, 你可以用你文档内的数据作各种各样的计算. 比如, 一个视图, 可以回答像"你的公司在上周, 上个月, 或者上一年使用了多少资金" 类似的问题.

### 什么是视图 ###

让我们来看看各个不同的案例. 首先: 以特定的顺序, 因特定的目的来提取数据. 对于一个首页来说, 我们希望有一个以日期排序的日志标题列表. 在我们展示视图是如何工作时, 会使用到一组示例文档. 它们是"设计文档"这个章节中的文档的删减版, 但其实一样也无所谓.

        {
          "_id":"biking",
          "_rev":"AE19EBC7654",

          "title":"Biking",
					"body":"My biggest hobby is mountainbiking. The other day...",
					"date":"2009/01/30 18:04:11"
 		    }

        {
					"_id":"bought-a-cat",
					"_rev":"4A3BBEE711",
					
					"title":"Bought a Cat",
					"body":"I went to the the pet store earlier and brought home a little kitty...",
					"date":"2009/02/17 21:13:39"
    		}

        {
					"_id":"hello-world",
					"_rev":"43FBA4E7AB",
					
					"title":"Hello World",
					"body":"Well hello and welcome to my new blog...",
					"date":"2009/01/15 15:52:20"
	    	}
          
上面三个文档会作为例子. 注意, 文档是通过"_id"来排序的, 这也它们在数据库被保存的方式. 现在我们来定义一个视图. "Get Started"章节向你展示了如何在Futon--CouchDB的管理客户端--里创建一个视图. 如果你忘记怎么做了, 回去看看那一节. 当我们向你展示一些代码的时候, 如果没有相应的解释, 请你容忍我们一下.

				function(doc) {
					if(doc.date && doc.title) {
						emit(doc.date, doc.title);
					}
				}

这是一个map函数, 它是用JavaScript写的. 如果你对JavaScript不熟悉, 但用过C或者其他类C的语言, 比如Java, PHP或者C#, 那么这个看起来应该比较熟悉. 一个简单的函数定义.

你把视图函数以字符串的形式存储在设计文档的"views"域中. 你不用自己来跑视图函数. 而是来查询你的视图. CouchDB拿到源代码然后在你的视图有定义的每个文档上帮你运行它. 你通过查询你的视图来得到视图结果.

所有的map函数有一个单一的参数:doc. 这是你数据库中的一个单一文档. 我们的map函数检查文档是否有date和title属性--幸运的是我们所有的文档都有--然后以这两个域作为参数调用内置的emit()函数.

emit()函数总是接受两个参数. 第一个是key, 第二个是value. emit(key, value)函数在你的视图结果里创建一个记录. 还有一点, emit()函数可以map函数里被多次调用, 这样可以在单一文档的视图结果里创建多个记录, 但是我们还没有那么做.

CouchDB接受任何你传入emit()函数的东西并把它们放进一个列表里. 列表里的每一行包含了我们的key和value. 更重要的是, 这个列表是根据key来排序的, 在我们的例子是doc.date. 一个视图结果的最重要的一个特性是, 它是用key来排序的. 我们以后会一次又一次回到这一点是来. 别走开~

				Key	                     Value
				"2009/01/15 15:52:20"	   "Hello World"
				"2009/01/30 18:04:11"	   "Biking"
				"2009/02/17 21:13:39"	   "Bought a Cat"
				
如果你有仔细阅读前面几段的话, 有一句话你一定会有深刻的印象: "当你查询你的视图, CouchDB拿到源代码然后在你的视图有定义的每个文档上帮你运行它". 如果你有很多的文档, 那会占用相当久的时间, 你会怀疑这到底有没有效率. 是的, 这会变得很没有效率, 但CouchDB设计避免任何额外的消耗: 它只在你第一次查询你的视图的时候在所有的文档上跑一次. 如果一个文档被改变了, map函数只会为这个单一的文档跑一次, 重新生成key和value.

视图结果存储在一个B-tree里, 和用来保存你的文档的结构一样. 视图B-tree被存储在它们自己的文件里, 如果要高性能的CouchDB使用环境, 你可以把视图保存在它们自己的磁盘里. B-tree提供了非常快速的, 根据key来查询的行的能力, 以及查询一个key范围的能力. 在我们的例子里, 一个单一的视图可以回答所有的关于时间的问题: "给我所有的一周的博客文章" 或者 "上个月", "上一年". 非常干净. 关于更多的CouchDB如何使用B-tree的信息, 请看附录的"The Power of B-Trees".

当我们查询我们的视图, 我们会得到一个由日期排序的文档列表, 每一行同时还包含文章标题, 这样我们便可以组合到文章的链接. 上面的列表图示只是一个图形化的视图结果的示意. 实际上的结果是JSON形式的, 而且包含少量的元数据.

				{
					"total_rows": 3,
					"offset": 0,
					"rows": [
						{
							"key": "2009/01/15 15:52:20",
							"id": "hello-world",
							"value": "Hello World"
						},

						{
							"key": "2009/02/17 21:13:39",
							"id": "bought-a-cat",
							"value": "Bought a Cat"
						},

						{
							"key": "2009/01/30 18:04:11",
							"id": "biking",
							"value": "Biking"
						}
					]
		    }

好吧, 我们又说谎了, 实际上的结果可没有这样的可读的格式, 不包含缩进和新行, 但上面这个更有利于你(和我们)的阅读和理解. 还有, 那个"id"域又是从哪里来的, 之前并没有啊. 观察的好, 我们之前忽略掉它是为了避免混乱. CouchDB会在视图结果里自动包含文档的id. 我们在组合到博客文章链接时也会用到它.

### 高效的查找 ###

让我们继续第二个视图的案例: "创建高效的索引, 使用文档的任意值或结构来查找文档". 关于高效的索引, 我们已经解释过了, 但是还是跳过了一点细节. 现在正是结束关于更复杂的map函数的讨论的好时机.

首先, 让我们回到B-tree来. 我们解释过, 存储根据key排序的视图结果的B-tree只会被创建一次, 当你查询过了一个视图, 以后所有的查询都会去读取B-tree, 而不会去执行map函数. 当你改变了一个文档, 或者增加或删除了一个文档, 会发生什么事呢? 简单: CouchDB足够聪明, 可以找到由一个特定文档产生的那一列. 它会把它们标记成不可用的, 让他们不再在视图里显示出来. 如果一个文档被删除了, 很好, B-tree结果正好反映出了数据库的状态. 如果一个文档更新了, 新的文档会执行map函数, 新的结果会被插入到B-tree的正确位置; 新增加文档也是以这样的方法处理. "The Power of  B-Trees"附录演示了B-tree是一种符合我们要求的非常高效的数据结构.

关于效率的讨论, 还有一点是要加以说明的: 通常在两次视图查询之间会有多个文档被更新了. 前面一段所解释的机制会被运用于数据库的所有改变, 因为最后那次视图查询是一个批量的操作, 这使得速度变得更加的快, 也让你的资源被更好的被利用起来.

#### 查找单个文档 ####

来谈谈更为复杂的map函数. 前面我们说过"使用文档内的任何值或者任何结构来找出文档.", 我们已经解释了如何通过以一个域排序的视图来提取值(例子里我们的date域). 同样的机制被用在快速查找里. 得到一个视图结果的查询URI是/database/_design/designdocname/_view/viewname. 这会给你一个视图里所有记录的列表. 我们只有三个文档, 所以列表很短, 但如果有成千上万个文档, 列表就会变得很长. 你可以在URI里加上*视图参数*来约束一个结果集. 比方说, 我们知道一篇博客日志的日期. 为了找到这个单一的文档, 我们可以使用 /blog/_design/docs/_view/by_date?key="2009/01/30 18:04:11" 这个URI来得到"Biking"这篇日志. 记住, 你可以在key参数里放进任何你想要的东西来约束emit()函数. 不管你在那里加什么参数, 我们现在可以使用它来快速而准确的查找了.

注意, 如果多行记录有相同的key(也许我们会设计一个视图, 它的key是日志的作者的名字), key参数的查询会返回多于一行的记录.

#### 查找多个文档 ####

我们想要"得到上个月的所有日志". 如果现在是二月份, 通过/blog/_design/docs/_view/by_date?startkey="2010/01/01 00:00:00"&endkey="2010/02/00 00:00:00", 很容易就能得到结果. startkey和endkey参数指定了一个查找的区间.

To make things a little nicer and to prepare for a future example, we are going to change the format of our date field. Instead of a string, we are going to use an array, where individual members are part of a timestamp in decreasing significance. This sounds fancy, but it is rather easy. Instead of:

为了看起来更加美观, 也为了给将来的例子做准备, 我们要改变下日期域的格式. 我们将会用数组来代替字符串, 数组里的值按时间戳重要性降序排列. 这听起来很美妙, 但其实做起来很容易. 原来的格式:

				{
					"date": "2009/01/31 00:00:00"
				}

现在的格式:

				"date": [2009, 1, 31, 0, 0, 0]

我们的map函数不需要因此而改变, 但我们的视图结果看起来会有些变化. 见表2, “New view results”.

				Key	                          Value
				[2009, 1, 15, 15, 52, 20]	    "Hello World"
				[2009, 2, 17, 21, 13, 39]	    "Biking"
				[2009, 1, 30, 18, 4, 11]	    "Bought a Cat"

表2. New view results

而我们的查询则变为
/blog/_design/docs/_view/by_date?key=[2009, 1, 1, 0, 0, 0] 和 
/blog/_design/docs/_view/by_date?key=[2009, 01, 31, 0, 0, 0]. 你所需要关心的是, 这只是语法上的一个变化, 而不是语义上的. 但这向你展示了视图的强大. 你不仅可以使用像字符串, 整数之类的标题来创建索引, 也可以使用JSON作为视图的key. 比如, 我们把我们文档打上了一系列的标签, 现在我们想把所有的标签都列出来, 但我们又不想处理那些没有被打过标签的文档.

				{
					...
					tags: ["cool", "freak", "plankton"],
					...
				}

				{
					...
					tags: [],
					...

				}
				 function(doc) {
					if(doc.tags.length > 0) {
						for(var idx in doc.tags) {
							emit(doc.tags[idx], null);
						}
					}
				}

这里展示了一些新的东西. 你可以在这个(if(doc.tags.length > 0))结构里使用条件判断而不只是值. 这也展示了一个map函数是如何在同一个文档上调用多次emit()函数的. 最后, 你可以传null, 而不是一个具体的值给value参数. 这对于key参数也同样适用.	我们会在后面看到它们的用途.

#### 结果的倒序 ####

要取得视图结果的倒序, 可以使用descending=true这个查询参数. 如果你使用了startkey这个参数, 你会发现CouchDB返回了和你想要的不一样的记录甚至没有记录. 这是怎么回事?

当你明白了底层的查询选项是如何工作时, 一切就变得容易理解了. 为了快速查找, 视图被存储在一个树结构里. 当你查询一个视图时, CouchDB是这样做的:

1, 从最顶上开始读取, 如果startkey存在, 就从startkey指定的位置开始读取.

2, 一次返回一行, 直至结尾, 如果endkey存在, 就直到endkey指定的位置.

如果你指定了descending=true, 读取的方向倒序了, 而不是视图中记录的排序被倒序了. 另外, 上面的两步走过程没有变化.

比如, 你有一个这样的视图结果:

				Key	   Value
				0	     "foo"
				1	     "bar"
				2	     "baz" 

如果查询条件是:?startkey=1&descending=true. CouchDB会怎么做呢? 看上面的第一步: 它会跳到startkey所在的位置, 在这个例子里是1, 然后反方向读取记录直至视图的结尾. 所以这个查询的结果会是:

				Key    Value
				1	     "bar"
				0	     "foo"

这个结果很可能不是你想要的. 要得到key是1和2的倒序记录, 你需要调换startkey和endkey的值:
endkey=1&descending=true:

				Key    Value
				2	     "baz"
				1	     "bar"

现在看起来好多了. CouchDB开始从视图的结尾开始反方向读取直到endkey指定的位置.

### 得到日志评论的视图 ###

这里我们用了数组作为key来支持reduce的group_level查询参数. CouchDB的视图存储于B-tree文件结构里(这点我们在后面会更加详细的描述). 因为B-tree是结构化的, 我们可以把reduce的中间结果放在树的非叶子节点里, so reduce queries can be computed along arbitrary key ranges in logarithmic time. 见图1, “Comments map function”.

在这个博客应用里, 我们使用带group_level参数的reduce查询来计算每篇日志的评论数量和总的评论数量. 我们通过使用不同的方法来查询同一个视图索引来达到这个目的. 假设有下面这些数组作为key, 并且每个key的value都是1:

				["a","b","c"]
				["a","b","e"]
				["a","c","m"]
				["b","a","c"]
				["b","a","g"]

reduce视图如下:

				function(keys, values, rereduce) {
					return sum(values)
				}

这个reduce会返回在起始和结束key之间的总行数. 所以, 如果参数是startkey=["a","b"]&endkey=["b"](包含了最上面的三行记录), 那么结果就等于3. 效果是用来数出记录数. 如果你想要不依赖记录的value来数出记录数, 你可以把rereduce这个参数打开:

				function(keys, values, rereduce) {
					if (rereduce) {
						return sum(values);
					} else {
						return values.length;
					}
				}

![Comments map function](views/01.png)

Figure 1. Comments map function

这是用来得到我们的示例应用评论数量的reduce视图, 这样map函数就可以来输出评论内容了. 和总是输出value为1相比, 这显然有用的多了. 花些时间来玩玩熟悉下map和reduce函数是值得的. Futon可以用来做这事, 但它给不了你所有的查询参数. 要探索CouchDB增量MapReduce系统的细节和能力, 用你喜欢的语言来写些自己的测试代码是一个非常好的方法.

不管怎么说, 使用带group_level的查询, 其实你是在执行一系列的reduce范围查询: 每做一个level的查询, 得出一个group的结果. 让我们来重新打印下之前的key列表, level 1得到的group:

				["a"]   3
				["b"]   2

group_level=2的:

				["a","b"]   2
				["a","c"]   1
				["b","a"]   2

使用参数group=true会让查询像参数group_level=exact一样工作. 在我们现在的这个例子里, 这会得到每个key, value都是1的结果, 因为没有完全重复的key.

### Reduce/Rereduce ###

We briefly talked about the rereduce parameter to your reduce function. We’ll explain what’s up with it in this section. By now, you should have learned that your view result is stored in B-tree index structure for efficiency. The existence and use of the rereduce parameter is tightly coupled to how the B-tree index works.

我们前面简短的谈了下reduce函数里的rereduce参数. 我们会在这一节里解释它到底是什么有什么用. 讲到这里, 你应该已经知道了为了得到高效率, 视图结果是被存储在B-tree索引结构里的. rereduce参数的存在和使用和B-tree索引是如何工作的, 这两者紧密的联系在一起.

来看示例1, "示例视图结果 (mmm, food)"展示的视图结果.

				"afrikan", 1
				"afrikan", 1
				"chinese", 1
				"chinese", 1
				"chinese", 1
				"chinese", 1
				"french", 1
				"italian", 1
				"italian", 1
				"spanish", 1
				"vietnamese", 1
				"vietnamese", 1

Example 1. Example view result (mmm, food)

如果我们想要找出每个地区有多少dishes, 我们可以重用之前使用过的那个简单的reduce函数:

				function(keys, values, rereduce) {
					return sum(values);
				}

图2, "B-tree索引"展示了一个B-tree索引结构的简化版. 我们缩写了key的字符串.

![B-tree索引](views/02.png)

The view result is what computer science grads call a “pre-order” walk through the tree. We look at each element in each node starting from the left. Whenever we see that there is a subnode to descend into, we descend and start reading the elements in that subnode. When we have walked through the entire tree, we’re done.
