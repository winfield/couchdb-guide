## View Cookbook for SQL Jockeys ##

本章是一个关于一些常见的SQL查询在CouchDB要如何得到相同的结果的集合. 这里关键要记住的是, CouchDB和SQL数据库的工作方式完全不同, 并且SQL世界的最佳实践并不能很好的或者说完全不能原原本本的翻译到CouchDB里. 本章节"cookbook"假设你对CouchDB的基础熟悉, 像创建更新数据库及文档等.

### 使用视图 ###

在SQL你是怎么做的:

				CREATE TABLE

或者:

				ALTER TABLE

使用视图是一个两步走的过程: 首先你要定义一个视图; 然后你再来查询它. 这类似于使用CREATE TABLE或者ALTER TABLE定义一个表结构(带有索引), 然后使用SQL语句来查询它.

### 定义一个视图 ###

定义视图是通过在CouchDB数据库创建特殊的文档来完成的. 唯一特殊之处就是文档的_id, 它是由"_design/-"开头的, 比如, _design/application. 除此之外, 它只是一个普通的CouchDB文档. 为了让CouchDB知道你是在定义一个视图, 你需要在设计文档中使用特殊的内容. 下面就是一个例子:

				{
					"_id": "_design/application",
					"_rev": "1-C1687D17",
					"views": {
						"viewname": {
							"map": "function(doc) { ... }",
							"reduce": "function(keys, values) { ... }"
						}
					}
				}

我们定义了一个叫viewname的视图. 视图的定义包含了两个函数: map函数和reduce函数. 指定一个reduce函数并不是必须的. We'll look at the nature of the functions later. 注意视图名字可以是任何你喜欢: users, by-name, 或者by-date. 这只是几个例子.

一个设计文档也可以包含多个视图定义, 每个视图都有一个唯一的名字:

				{
					"_id": "_design/application",
					"_rev": "1-C1687D17",
					"views": {
						"viewname": {
							"map": "function(doc) { ... }",
							"reduce": "function(keys, values) { ... }"
						},
						"anotherview": {
							"map": "function(doc) { ... }",
							"reduce": "function(keys, values) { ... }"
						}
					}
				}

### 查询视图 ###

The name of the design document and the name of the view are significant for querying the view. To query the view viewname, you perform an HTTP GET request to the following URI:
设计文档的名字和视图的名字对于查询一个视图来说很重要. 要想查询一个视图, 可以通过一个带有如下URI的HTTP GET请求:

				/database/_design/application/_view/viewname

database是你的设计文档所在的数据库. 接下来是设计文档的名字, 然后是视图的名字, 以前缀_view/开始. 要查询另外一个视图, 就只要把URI的视图名字换成另外一个就行了. 如果你要查询在另外一个设计文档中的视图, 则要改变设计文档的名字.

### MapReduce 函数 ### 

MapReduce是一个用两个过程来解决问题的概念, 这两个过程分别被命名为map和reduce. map会在CouchDB里一个一个的查找所有的文档, 然后创建一个map结果. map结果一个经排序的key/value对的集合. key和value都可以在map函数里指定. 一个map函数可以在每个文档上调用0到N次的emit(key, value)函数, 每次执行都会创建map结果集中的一行.

CouchDB很聪明, 在每个文档上只会跑一次map函数, 即便是在一个视图的子查询里. 只有文档变更了或者对于新创建文档map函数才会在它们上面重新运行.

#### Map函数 ###

Map函数在每个文档上独立的运行. 它们不能改变文档, 并且它们不能和外界交流--它们没有副作用. 这是必要的, 因为这样CouchDB才能在当有一个文档变更时, 不需要重新运算所有的结果才能保证正确性.

map的结果如下所示:

				{"total_rows":3,"offset":0,"rows":[
				{"id":"fc2636bf50556346f1ce46b4bc01fe30","key":"Lena","value":5},
				{"id":"1fb2449f9b9d4e466dbfa47ebe675063","key":"Lisa","value":4},
				{"id":"8ede09f6f6aeb35d948485624b28f149","key":"Sarah","value":6}
				]}

它是一个根据key的值排序的列表. id是被自动加入的, 表示创建此行结果的文档. value就是你要找的值, 上面的例子里是女孩的年龄.

产生这个结果的map函数如下所示:

				function(doc) {
					if(doc.name && doc.age) {
						emit(doc.name, doc.age);
					}
				}

它包含了一个if语句作为检查来保证我们操作的是正确的域, 然后它调用以name和age分别作为key和value调用emit函数.

#### Reduce函数 ####

Reduce函数将会在下面的"Aggregate Functions"部分里讲解.

### 通过Key来查找 ###

在SQL里你会这么做:

				SELECT field FROM table WHERE value="searchterm"

用途: 得到一个Key("searchterm")相对应的结果(可能是一条记录或者一组记录).

为了查找快速, 在不考虑存储机制的情况下, 索引是必需的. 索引是一个用于优化搜索以及读取速度的数据结构. CouchDB的map结果被存储于一个类似的索引中, 它是一个B+树.

To look up a value by "searchterm", we need to put all values into the key of a view. All we need is a simple map function:
要想根据"searchterm"来查找值, 我们需要把所有的值放在视图的key里面. 我们只需要一个简单的map函数:

				function(doc) {
					if(doc.value) {
						emit(doc.value, null);
					}
				}

这个map函数会创建一个拥有域value文档列表并且根据value域的值进行排序. 要找出所有符合"searchterm"的记录, 我们查询这个视图并通过加入查询参数的方式指定查询条件:

				/database/_design/application/_view/viewname?key="searchterm"

Consider the documents from the previous section, and say we’re indexing on the age field of the documents to find all the five-year-olds:
再考虑下前面那个部分里讲到的文档, 假设我们要索引文档的年龄域来找出所有5岁大的:

				function(doc) {
					if(doc.age && doc.name) {
						emit(doc.age, doc.name);
					}
				}

查询:

				/ladies/_design/ladies/_view/age?key=5

结果:

				{"total_rows":3,"offset":1,"rows":[
				{"id":"fc2636bf50556346f1ce46b4bc01fe30","key":5,"value":"Lena"}
				]}

简单吧.

注意你必须要先emit一个值. 视图结果每一行都包含了其相对应的文档ID. 我们可以使用它来从文档本身中查找更多的数据. 我们也可以使用?include_docs=true参数让CouchDB来为我们读取文档.

### 通过一个前缀来查找 ###

在SQL中, 我们是怎么做的:

				SELECT field FROM table WHERE value LIKE "searchterm%"

用途: 找出所有的value域以"searchterm"开头的文档. 举了例子, 假如你为每个文档存储了一个MIME类型(像是text/html或者是image/jpg), 然后你想要找出所有的根据MIME类型判断是图片的文档.

这个问题的解决方案和前面那个例子很类似: 你所需要的只是一个map函数, 它要比前面那个更聪明些. 但是首先, 先来看看示例文档:

				{
					"_id": "Hugh Laurie",
					"_rev": "1-9fded7deef52ac373119d05435581edf",
					"mime-type": "image/jpg",
					"description": "some dude"
				}

not so well translated...做法是提取出这种类型的前缀并把它放入视图的索引中. 我们使用一个正则表达式来匹配这个前缀:

				function(doc) {
					if(doc["mime-type"]) {
						// from the start (^) match everything that is not a slash ([^\/]+) until
						// we find a slash (\/). Slashes needs to be escaped with a backslash (\/)
						var prefix = doc["mime-type"].match(/^[^\/]+\//);
						if(prefix) {
							emit(prefix, null);
						}
					}
				}

We can now query this view with our desired MIME type prefix and not only find all images, but also text, video, and all other formats:
现在我们可以通过这个视图来查找想要的MIME类型了, 并且不只限于图片, 还可以是文本, 视频, 以及其他格式:

				/files/_design/finder/_view/by-mime-type?key="image/"

### Aggregate Functions ###

How you would do this in SQL:
在SQL, 我们是这么做的:

				SELECT COUNT(field) FROM table

Use case: calculate a derived value from your data.

我们还没有讲解过reduce函数. reduce函数和SQL中aggregate函数类似. 它们从多个文档中计算出一个值. 

为了解释清楚reduce函数的机制, 我们创建来创建一个并不是很合理的例子. 但是这个例子很容易理解. 在后面我们会讲些更加有用的reduce函数.

Reduce functions operate on the output of the map function (also called the map re⁠sult or intermediate result). The reduce function’s job, unsurprisingly, is to reduce the list that the map function produces.
reduce函数在map函数的输出上(也被叫做map结果或者中间结果)进行操作. reduce函数的工作, 没什么令人惊奇的, 就是减少map函数产生的列表.

这就是我们的求和reduce函数的样子:

				function(keys, values) {
					var sum = 0;
					for(var idx in values) {
						sum = sum + values[idx];
					}
					return sum;
				}

下面是另外一种方法, 更加符合JavaScript的风格:

				function(keys, values) {
					var sum = 0;
					values.forEach(function(element) {
						sum = sum + element;
					});
					return sum;
				}

reduce函数接受两个参数: 一个key的列表和一个value的列表. 对于我们的求和来说, 我们可以忽略掉key这个列表而只考虑value这个列表. 我们遍历整个列表并把每个值都加起来, 在函数结束时返回这个值.

我们会看到在map和reduce函数之间有一个不同. map函数使用emit()来创建它的结果, 而reduce函数则返回一个值.

For example, from a list of integer values that specify the age, calculate the sum of all years of life for the news headline, “786 life years present at event.” A little contrived, but very simple and thus good for demonstration purposes. Consider the documents and the map view we used earlier in this chapter.
比如, 从一个表示年龄的整数值的列表里, 计算所有年龄的和. 虽然没什么用, 但是很简单也演示了我们的目的. 考虑下我们本章节前面的文档和map视图.

用于计算所有女孩总年龄的reduce函数是:

				function(keys, values) {
					return sum(values);
				}

注意, 不同前两个reduce函数, 这里我们使用了CouchDB预定义的sum()函数. 它所做的和之前两个所做的事情相同, 但求和太常见了, 所有CouchDB就默认包含了.

reduce视图的结果如下所示:

				{"rows":[
				{"key":null,"value":15}
				]}

所以年龄的和在我们的文档里是15. 这就是我们想要的. 结果的key这个域是null, 因为我们找不出哪些文档参与的创建reduce结果的过程. 我们会在后面讲到更加的更加高级的reduce的用法.

通常来说, reduce函数应该是一个单一的scalar值. 也就是说是, 一个整数; 一个字符串; 或者一个较小的, 固定大小的包含aggregated值的列表或对象. 它应该永远不会返回多个值或类似的. 如果你"错误"的使用了reduce, CouchDB会给你一个警告:

				{"error":"reduce_overflow_error","message":"Reduce output must shrink more rapidly: Current output: ..."}

### 得到唯一的值 ###

在SQL, 这是如何做到的:

				SELECT DISTINCT field FROM table

要得到唯一的值不是只要加个关键词那么简单. 但是一个reduce视图和一个特殊的查询参数可以给我们相同的结果. 让我们假设你想要一个所有用户给自己打的标签的列表, 并且没有重复.

首先, 我们来看看原始的文档. 这里我们省略掉了_id和_rev属性:

				{
					"name":"Chris",
					"tags":["mustache", "music", "couchdb"]
				}

				{
					"name":"Noah",
					"tags":["hypertext", "philosophy", "couchdb"]
				}

				{
					"name":"Jan",
					"tags":["drums", "bike", "couchdb"]
				}

接下来, 我们需要一个所有标签的列表. 只要一个map函数就行了:

				function(dude) {
					if(dude.name && dude.tags) {
						dude.tags.forEach(function(tag) {
							emit(tag, null);
						});
					}
				}

结果如下:

				{"total_rows":9,"offset":0,"rows":[
				{"id":"3525ab874bc4965fa3cda7c549e92d30","key":"bike","value":null},
				{"id":"3525ab874bc4965fa3cda7c549e92d30","key":"couchdb","value":null},
				{"id":"53f82b1f0ff49a08ac79a9dff41d7860","key":"couchdb","value":null},
				{"id":"da5ea89448a4506925823f4d985aabbd","key":"couchdb","value":null},
				{"id":"3525ab874bc4965fa3cda7c549e92d30","key":"drums","value":null},
				{"id":"53f82b1f0ff49a08ac79a9dff41d7860","key":"hypertext","value":null},
				{"id":"da5ea89448a4506925823f4d985aabbd","key":"music","value":null},
				{"id":"da5ea89448a4506925823f4d985aabbd","key":"mustache","value":null},
				{"id":"53f82b1f0ff49a08ac79a9dff41d7860","key":"philosophy","value":null}
				]}

As promised, these are all the tags, including duplicates. Since each document gets run through the map function in isolation, it cannot know if the same key has been emitted already. At this stage, we need to live with that. To achieve uniqueness, we need a reduce:
就预料的一样, 这些就是所有的标签了, 包括重复的. 因为每个文档都在map函数里独立的运行, 所以不可能知道之前已经有同样的key产生了. 目前为止, 我们只能做到这样了. 为了达到唯一性, 我们还需要一个reduce函数:

				function(keys, values) {
					return true;
				}

This reduce doesn’t do anything, but it allows us to specify a special query parameter when querying the view:
这个reduce没有做任何事情, 但是它可以在我们查询视图时指定一个特殊的查询参数:

				/dudes/_design/dude-data/_view/tags?group=true

CouchDB响应:

				{"rows":[
				{"key":"bike","value":true},
				{"key":"couchdb","value":true},
				{"key":"drums","value":true},
				{"key":"hypertext","value":true},
				{"key":"music","value":true},
				{"key":"mustache","value":true},
				{"key":"philosophy","value":true}
				]}

在这个例子里, 我们可以忽略掉value这部分, 因为它永远都是true, 但是结果里包含了我们所有的标签并且没有重复!

稍做改变, 我们可以把reduce也使用起来. 让我们来找出那些重复的标签重复了几次. 为了计算每个标签的频繁度, 我们使用前面已经学过的求和. 在map函数里, 我们生成值为1, 而不是之前的null:

				function(dude) {
					if(dude.name && dude.tags) {
						dude.tags.forEach(function(tag) {
							emit(tag, 1);
						});
					}
				}

In the reduce function, we return the sum of all values:
在reduce函数里, 我们返回了所有值的和:

				function(keys, values) {
					return sum(values);
				}

现在, 如果我们用参数?group=true来查询视图, 我们会得到每个标签的出现次数:

				{"rows":[
				{"key":"bike","value":1},
				{"key":"couchdb","value":3},
				{"key":"drums","value":1},
				{"key":"hypertext","value":1},
				{"key":"music","value":1},
				{"key":"mustache","value":1},
				{"key":"philosophy","value":1}
				]}

### Enforcing Uniqueness ###

How you would do this in SQL:
在SQL中是如何做的:

				UNIQUE KEY(column)

Use case: your applications require that a certain value exists only once in a database.
用途: 应用程序需要找出数据库一个只存在一次的特定值.

这在CouchDB里很简单, 每次文档都必须有一个唯一的_id域. 如果你需要数据库的一个唯一的值, 只要把它们作为文档的_id域, CouchDB就可以为你保证其唯一性的.

虽然还有一种特殊情况: 在分布式的系统里, 你运行了不止一个的CouchDB节点, 它们接受写请求时, CouchDB只能保证在一个点里保证, 或者在CouchDB之外来保证. CouchDB会允许两个相同的ID被写入两个不同的节点里. 在复制时, CouchDB会检测到一个冲突并给相应的文档打上标记.` 

