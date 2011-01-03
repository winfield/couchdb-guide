## 存储文档 ##

文档是CouchDB的中心数据结构. 想要最好的理解和使用CouchDB, 你需要以文档的方式来思考. 这一章节将会讲解从设计文档到保存文档的整一个生命周期. 我们会通过阅读文档, 以及用视图来操作和查询文档来一步步的学习. 在下一个部分里, 你还会看到CouchDB是如何把文档转换为其他格式的.

文档是自包含的数据单元. 你也许听到过"记录"这个词来描述类似的东西. 你的数据一般是由一些小的原数据--像整型和数据串--组合而成的. 文档是这些原数据的第一层抽象. 它们对原数据作了一些结构化和逻辑划分. 一个人的身高可能用一个整型来表示(176), 但是这个整型通过是一个更大型数据结构的一部分, 这个更大的数据结构包含了标签("height": 176)以及一些相关的数据({"name":"Chris", "height": 176})

要在文档里放入多少数据, 这取决于你的应用, 以及你将要如何使用视图(以后)有一点关系. 但一般来说, 一个文档对应于编程语言中一个对象实例. 如果你运行的是一个在线商店, 那么就会有销售商品以及商品的售出记录以及评论. 它们都可以作为对象, 然后作为文档. 

文档和那些有作者以及CRUD(create, read, update, delete)操作的一般对象不同. 基于文档的软件(像文字处理和表格处理之类的)以存储文档为中心来构建其存储模型, 这样作者可以找到他们创建的文档. 类似的, 在一个CouchDB应用里, 你会发现自己在展现层(presentation layer)有很大的自由. 比如, 不在Controller里加入时间戳到数据里, 而是让用户可以自己控制时间戳, 这样你会可以轻松的得到一个草稿状态以及在未来发布文章的能力(通过把当前时间作为endkey来查询文档的发布时间).

因为有验证函数的存在, 你不必担心垃圾数据会让系统产生错误. 通常在基于文档的软件里, 客户端应用编辑和操作数据, 然后把文档保存回去. As long as you give the user the document she asked you to save, she’ll be happy.

假设你的用户可以在商品("lovely book")上作评论; 你可以选择把评论用一个数组的形式保存在商品这个文档上. 这样做让找出商品的评论变得非常容易, 但是, 就像人们说的, "这样没有可扩展性." 一件受欢迎的商品可能会有上百条, 上千条甚至更多的评论.

在这个例子里, 不把评论存储在商品文档里, 把它们建模到另外一些文档集合里会更好. 获取集合有几种特定的模式, 这些CouchDB都很容易做到. 你可能只想要每次只展示10条或20条, 并且提供前一页和后一页链接. 把评论作为独立的实体来处理, 你就可以通过视图来对它们进行分组. 一个组可以是整个集合或者是10条或者20条的分片, 根据它们所属的商品来排序, 这样就能很容易的找到你想要的数据集.

一条原则(a rule of thumb): 把所有在应用里将会单独处理的东西作为文档来处理. 商品是独立的, 评论是独立的, 但你不需要把它们分成更小的分块来处理. 把文档有意义的进行分组, 视图是一个方便的办法. 

让我们一步步来构建我们的示例应用, 展示下在实际中如何使用文档.

### JSON文档格式 ###

设计任何应用的第一步(在你知道程序是用来做什么的以及确定了用户界面后)是决定应用将来用来代表和存储的数据的格式. 我们的示例博客是由JavaScript写的. 就在前面, 我们说过文档基本上代表了数据对象. 在这个例子里, 它们之间有一个确切的对应关系. CouchDB借用了JavaScript的JSON数据格式; 这允许我们在编程时可以把文档直接作为原生对象来使用. 这实在是太方便了, 并且这么做可以减少遇到的麻烦(如果你曾使用过ORM系统, 你应该会知道我们所指的意思).

让我们来设计博客日志的JSON格式数据的草稿. 我们知道每篇日志我们需要一个作者, 一个标题, 和一个正文. 我们知道我们将会使用文档ID来找出文档, 这样URL就能对搜索引擎友好, 而且我们想要以日志创建日期来排列它们.

JSON是如何工作的应该非常直观. 大括号({})括起了对象, 对象是key/value的列表. Key是用双引号("")引起来的字符串. 最后, 一个value可以是一个字符串, 一个整型, 一个对象, 或者一个数组([]). key和value以分号(:)分隔, 多对key和value以逗号(,)分隔. 这就是JSON. 对JSON格式的完整描述, 请参考附录E, JSON初步.

图1, "JSON日志格式"展示了一个符合我们要求的文档. 很酷的是, 我们马上就构建好了这个文档. 我们不需要去定义一个模版, 我们也不用去定义如何来展现数据. 我们只要创建我们想要的文档就行了. 对象的要求在应用的开发过程中会一直改变. 只需要创建一个满足要求的新文档就行了, 改进变得非常容易.

Figure 1. The JSON post format

Do I really look like a guy with a plan? You know what I am? I’m a dog chasing cars. I wouldn’t know what to do with one if I caught it. You know, I just do things. The mob has plans, the cops have plans, Gordon’s got plans. You know, they’re schemers. Schemers trying to control their little worlds. I’m not a schemer. I try to show the schemers how pathetic their attempts to control things really are.

—The Joker, The Dark Knight

让我们再更仔细一点的来看看文档. 文档最前面的两个域(_id和_rev)是CouchDB本身的"家务事", 它们作为一个特定文档实例的认证. _id很简单: 如果我存储了什么到CouchDB, 它会创建_id并把它返回给我. 我可以使用_id来构建得到数据的URL.

Your document’s _id defines the URL the document can be found under. Say you have a database movies. All documents can be found somewhere under the URL /movies, but where exactly?

If you store a document with the _id Jabberwocky ({"_id":"Jabberwocky"}) into your movies database, it will be available under the URL /movies/Jabberwocky. So if you send a GET request to /movies/Jabberwocky, you will get back the JSON that makes up your document ({"_id":"Jabberwocky"}).

_rev(或者说版本ID)描述了一个文档的版本. 每一个改变都会创建一个新的文档版本(它同样是自包含的)并且更新_rev. 这很有用因为每当你保存一个文档时, 你必须提供最新的_rev, 这样CouchDB才能知道你处理的是最新版本的文档.

我们在第2章, 最终一致性里提到过这个. 版本ID扮演的角色相当于把文档写入CouchDB的MVCC系统时的守门人. 文档是共享的资源; 许多客户端可以同时读取和改写它们. 为了确保两个在改写文档的客户端不会互相踩到对方的脚, 每个客户端在提交更新时都必须提供它认为是最新的版本ID. 如果服务器上的版本ID符合客户端提供的, CouchDB会接受这次更改. 反之, 则更新会被拒绝. 客户端应该读取最新的版本, 合并改变, 然后再试尝试保存.

在一机制保证了两件事: 一个客户端只能改写一个它所知道的版本, 并且它不能跳过其他客户端所做的更改. 这一机制不需要CouchDB把任何文档加锁就可以工作. 这保证了没有客户端需要等待其他客户端完成工作才可以操作文档. 更新是串行化的, 所以CouchDB永远不会试图用比你的硬盘转速还快的速度写入文档, 并且这也意味着两个冲突的更改不会被同时写入.

### 超越_id和_rev: 你的文档数据 ###

如果你已经完全理解了文档的_id和_rev域, 那就让我们来看看文档的其他部分.

				{
					"_id":"Hello-Sofa",
					"_rev":"2-2143609722",
					"type":"post",

首先是文档的类型. 注意这是一个应用层面的参数, 而不是CouchDB特有的. 对于CouchDB来说, 这只是一个以type为名字的key/value而已. 对于我们来说, 因为我们要往Sofa里面加入博客日志, type就有了更深层次的意思. Sofa使用type域来决定使用哪种验证. 然后, 它就放心的在视图和用户界面里依赖文档的类型了. 这样做就避免了每次使用之前都要去检查要用到的域以及嵌套的JSON值. 这纯粹只是一个惯例, 你可以自己决定自己想要的结构, 然后通过文档的结构来认出其类型("有一个三个元素的数组存在的文档"-a.k.a. duck typing). 我们只是认为这是一个简单的方法, 希望你也会同意.

					"author":"jchris",
					"title":"Hello Sofa",

author和title域是在日志创建是设置的. title域是可改变的, 但是author域因为安全方面的原因是由验证函数锁定的. 只有作者(author)才可以编辑日志.

					"tags":["example","blog post","json"],

Sofa’s tag system just stores them as an array on the document. This kind of denormalization is a particularly good fit for CouchDB.
Sofa的标签系统只是把标签以一个数组的形式存储在文档里而已. 

  "format":"markdown",
  "body":"some markdown text",
  "html":"<p>the html text</p>",
Blog posts are composed in the Markdown HTML format to make them easy to author. The Markdown format as typed by the user is stored in the body field. Before the blog post is saved, Sofa converts it to HTML in the client’s browser. There is an interface for previewing the Markdown conversion, so you can be sure it will display as you like.

  "created_at":"2009/05/25 06:10:40 +0000"
}
The created_at field is used to order blog posts in the Atom feed and on the HTML index page.

