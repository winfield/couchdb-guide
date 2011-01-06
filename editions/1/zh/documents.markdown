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

Sofa的标签系统只是把标签以一个数组的形式存储在文档里而已. 这样的denormalization处理特别适合于CouchDB.

					"format":"markdown",
					"body":"some markdown text",
					"html":"<p>the html text</p>",

日志使用Markdown HTML格式来写, 这对于作者来说也很容易. 用户输入的Markdown格式内容存储在body域中. 在保存日志之前, Sofa会在用户浏览器中把它转化为HTML. 会有一个界面用来预览Markdown的转化, 所以你可以确定什么是你想显示的.

					"created_at":"2009/05/25 06:10:40 +0000"
				}

create_at域用于在Atom feed和HTML首页里排序日志.

### 编辑页面 ###

为了能够得到日志, 我们首先要创建的第一个页面就是创建和编辑日志的界面.

编辑比只是显示日志让访问者阅读要复杂的多, 但这也意味着在看完本章后, 你将会见识到大多数的技术, 这些技术我们也可能用于其他的章节. 

首先要看的是曾经用于生成HTML页面的show函数. 如果你还不了解它, 请阅读第8章, Show函数来学习它的API. 我们会把它放在Sofa代码的上下文里来看, 这样你就能看到它们是如何组合在一起的了.

				function(doc, req) {
					// !json templates.edit
					// !json blog
					// !code vendor/couchapp/path.js
					// !code vendor/couchapp/template.js

Sofa的编辑页面show函数很直观. 在开始部分, 我们导入要用到的一些重要的模版和库. 重要的部分是那个!json宏, 它会从templates目录中导入edit.html模版. 这些宏会在Sofa被部署到CouchDB时由CouchApp运行. 要了解更多的关于宏的信息, 请看第13章, Show Documents in Custom Formats.

				 // we only show html
					return template(templates.edit, {
						doc : doc,
						docid : toJSON((doc && doc._id) || null),
						blog : blog,
						assets : assetPath(),
						index : listPath('index','recent-posts',{descending:true,limit:8})
					});
				}

函数剩下的部分就很简单了. 我们只是从文档中取回数据填入HTML模版中. 为了防止有文档还不存在, 我们会确保把docid设置为空. 这使得我们可以使用相同的模版来创建新日志和编辑已存在的日志.

#### HTML模版 ####

The only missing piece of this puzzle is the HTML that it takes to save a document like this.

在你的浏览器中, 访问http://127.0.0.1:5984/blog/_desing/sofa/_show/edit, 并且用文本编辑器打开源文件templates/edit.html(或者在浏览器选择查看源代码). 一切都已经准备就绪; 所有我们需要做的事就是用JavaScript把它和CouchDB连接起来. 请看图2, "HTML listing for edit.html"

和任何其他的web应用一样, HTML中最重要的部分就是用于接受用户编辑的表单. 编辑表单会捕获一些基本的数据: 日志标题, 日志内容(Markdown格式), 以及任何用户输入的标签.

				<!-- form to create a Post -->
				<form id="new-post" action="new.html" method="post">
					<h1>Create a new post</h1>
					<p><label>Title</label>
						<input type="text" size="50" name="title"></p>
					<p><label for="body">Body</label>
						<textarea name="body" rows="28" cols="80">
						</textarea></p>
					<p><input id="preview" type="button" value="Preview"/>
						<input type="submit" value="Save &rarr;"/></p>
				</form>

我们使用纯的HTML文档, 它包含有一个普通的HTML表单. 我们会使用JavaScript把用户输入转化为一个JSON文档然后把它保存到CouchDB中. 为了集中讨论CouchDB, 我们不会过多的讨论这里使用的JavaScript. 它是Sofa特有一些应用代码, CouchApp的JavaScript辅助方法以及用于操作界面元素的jQuery代码的组合. 基本的概念就是它会等待用户点击"保存", 然后在发送到CouchDB之前, 执行一些回调函数.

Figure 2. HTML listing for edit.html

### 保存一个文档 ###

用于日志创建和编辑的JavaScript是围绕图2, "HTML listing for edit.html"中的HTML表单来的. CouchApp jQuery插件提供了一些抽象, 所以我们不需要自己去关心当用户点击提交时, 表单是如何被转化为JSON文档的. $.CouchApp同时还保证用户是已经登录的以及使得其信息在应用里可用. 请看图3, "edit.html的JavaScript回调".

				$.CouchApp(function(app) {
					app.loggedInNow(function(login) {

我们要求CouchApp库做的第一件事情是保证用户已经登录了. 假设答案是肯定的, 我们会继续处理, 生成一个带有编辑器的页面. 这意味着, 我们绑定了一个JavaScript事件处理到表单, 并且指定了要在这个文档上执行的回调函数, 它们在页面载入和文档保存时都会被执行.

Figure 3. JavaScript callbacks for edit.html

						// w00t, we're logged in (according to the cookie)
						$("#header").prepend('<span id="login">'+login+'</span>');
						// setup CouchApp document/form system, adding app-specific callbacks
						var B = new Blog(app);

知道用户已经登录以后, 我们可以在页面的顶端显示他的名字. 变量B只是一个Sofa自己的博客生成代码的shortcut. 它包含了用于把日志内容的Markdown格式转化为HTML的方法, 以及一些其他的东西. 我们把这些函数放到了blog.js, 这样就能把它们从主代码里剥离出来了.

						var postForm = app.docForm("form#new-post", {
							id : <%= docid %>,
							fields : ["title", "body", "tags"],
							template : {
								type : "post",
								format : "markdown",
								author : login
							},

CouchApp的app.docForm()辅助方法用于建立和维护一个CouchDB文档和HTML表单之间的对应关系. 让我们来看看前三个Sofa传入它的参数. 参数id告诉docForm()要把文档保存在哪里. 在创建新文档时, 这个域可以是空的. 我们在参数fields传入一个表单所包含元素的数组, 它们直接对应于CouchDB文档里的JSON域. 最后, 参数template传入的是一个JavaScript对象, 如果是创建一个新的文档, 那么它就是作为一个起始点. 在这个例子里, 我们要保证文档的type是"post", 并且默认的格式是Markdown. 我们还把作者设置为了当前登录用户的登录名.

							onLoad : function(doc) {
								if (doc._id) {
									B.editing(doc._id);
									$('h1').html('Editing <a href="../post/'+doc._id+'">'+doc._id+'</a>');
									$('#preview').before('<input type="button" id="delete"
											value="Delete Post"/> ');
									$("#delete").click(function() {
										postForm.deleteDoc({
											success: function(resp) {
												$("h1").text("Deleted "+resp.id);
												$('form#new-post input').attr('disabled', true);
											}
										});
										return false;
									});
								}
								$('label[for=body]').append(' <em>with '+(doc.format||'html')+'</em>');

onLoad回调函数会在文档来从CouchDB读取前执行. 它会在文档传到表单之前处理一番, 或者设置一些用户界面元素. 在这个例子里, 我们检查文档是否已经有了一个ID. 如果有, 那就意味着它已经被保存了, 所以我们创建一个按钮可以用来删除它, 并且在其之上设置了一个删除的回调函数. 看起来有很多代码, 但对于Ajax应用来说这算是很标准的了. 如果真要在这部分代码挑中毛病的话, 那就是用于创建删除按钮的逻辑可以移到blog.js文件里, 这样我们就可以把更多的用户界面细节移出主业务流程里了.

							},
							beforeSave : function(doc) {
								doc.html = B.formatBody(doc.body, doc.format);
								if (!doc.created_at) {
									doc.created_at = new Date();
								}
								if (!doc.slug) {
									doc.slug = app.slugifyString(doc.title);
									doc._id = doc.slug;
								}
								if(doc.tags) {
									doc.tags = doc.tags.split(",");
									for(var idx in doc.tags) {
										doc.tags[idx] = $.trim(doc.tags[idx]);
									}
								}
							},

docForm的beforeSave()回调函数会在用户点击提交按钮后执行. 在Sofa里, 它会用于设置日志的时间戳, 把日志标题转换为一个易于接受的文档ID(为了更加美观的URL), 以及把字符串形式的文档标签转换为一个数组. 它还会在浏览器里运行Markdown转HTML的代码, 这样当文档被保存后, 应用就可以直接使用HTML了.

							success : function(resp) {
								$("#saved").text("Saved _rev: "+resp.rev).fadeIn(500).fadeOut(3000);
								B.editing(resp.id);
							}
						});

我们在Sofa里最后使用的一个回调函数是success回调函数. 如果文档成功被保存, 它就会被执行. 在我们的例子里, 我们使用它来显示一条消息, 告诉用户她的日志保存成功了, 并且加入了一个到这个日志的链接. 这样在你第一次创建一个日志完成的时候, 你就能点击它来访问日志的永久链接了.

这些就是docForm的回调函数.

						$("#preview").click(function() {
							var doc = postForm.localDoc();
							var html = B.formatBody(doc.body, doc.format);
							$('#show-preview').html(html);
							// scroll down
							$('body').scrollTo('#show-preview', {duration: 500});
						});

Sofa有一个函数用于在保存日志之前预览它们. 因为这并不影响文档是如何保存的, 所以用于监听这个事件的代码没有放到docForm()的回调函数里去.

					}, function() {
						app.go('<%= assets %>/account.html#'+document.location);
					});
				});

最后的一点代码会在用户还没有登录时运行. 它所做的就是把用户跳转到登录页面, 这样他就可以登录, 然后编辑了.

#### 验证 ####

如果一切顺利, 那么当用户点击保存时, 前面的代码就会发送一个JSON文档到CouchDB. 这对于创建用户界面来说很好, 但它没有任何措施来保护数据库免于一些不期望的更新.这就是要用到验证函数的地方了. 如果有一个合适的验证函数, 一个determined的骇客也不能放入任何你不期望的文档到数据库里. 让我们来看看Sofa是如何做的. 要了解看更多关于验证函数的信息, 请浏览第7章, 验证函数.

				function (newDoc, oldDoc, userCtx) {
					// !code lib/validate.js

这一行会从Sofa里导入一个库, 它可以让接下来的代码更加可读. 它只是把请求标记为禁止访问或者未授权的简单包装. 在这一部分里, 我们会集中讨论验证函数的业务逻辑. 注意, 除非你可以Sofa的validate.js, 不然你需要处理更多的Sofa这个库已经抽象出来的原始逻辑

					unchanged("type");
					unchanged("author");
					unchanged("created_at");

这几行的作用就和它们的字面意思一样. 如何文档的类型, 作者, 或者创建时间被改变, 那么它们会抛出一个错误, 告诉用户这个更新被禁止了. 请注意, 这几行并不管这些域有什么样的内容. 它们仅仅是保证任何更新都不能改变这些域的内容.

					if (newDoc.created_at) dateFormat("created_at");

dateFormat辅助方法保证了date(如果有这个域的话)是Sofa的视图期望的格式.

					// docs with authors can only be saved by their author
					// admin can author anything...
					if (!isAdmin(userCtx) && newDoc.author && newDoc.author != userCtx.name) {
							unauthorized("Only "+newDoc.author+" may edit this document.");
					}

如果保存文档的是管理员, 那文档编辑就会被处理. 不然, 就会检查日志作者和现在正在保存日志是不是同一个人. 这样做就保证了作者只能保证它们自己的日志.

					// authors and admins can always delete
					if (newDoc._deleted) return true;

接下来的代码块会检查文档各位域的有效性. 然而如果是删除操作, 根据下面的定义就不会被允许了, 因为它的内容只有_deleted:true. 所以如果是删除操作, 就直接返回验证函数了.

					if (newDoc.type == 'post') {
						require("created_at", "author", "body", "html", "format", "title", "slug");
						assert(newDoc.slug == newDoc._id, "Post slugs must be used as the _id.")
					}
				}

最后, 我们验证日志文档本身的一些域. 在这里我们验证的是那些特定于日志文档的域. 因为我们已经验证了它们的存在, 所以我们可以在视图以及用户界面里放心的使用它们了.

#### 保存你的第一个日志 ####

让我们来看看这些代码是如何一起工作的! 在表单里填入一些数据, 然后点击"保存", 会看到一个成功的返回.

图4, "JSON over HTTP to save the blog post"展示了JavaScript是如何使用HTTP, 把文档PUT到一个以数据库名加文档ID组成的URL上的. 它同时也展示了文档是如何以JSON字符串的形式在PUT请求的内容里发送出去的. 如何你是GET一个文档的URL, 你会看到相同的JSON数据, 但还会带有一个保存到CouchDB时产生的_rev域.

Figure 4. JSON over HTTP to save the blog post

要想看到你保存的文档的JSON版本, 可以到Futon里去浏览它. 访问http://127.0.0.1:5984/_utils/database.html?blog/_all_docs, 你应该会看到一个对应于你刚刚保存的文档的ID. 点击它看看Sofa发送给了CouchDB什么内容.

### 收尾 ###

我们已经讲完了如何设计应用的JSON格式, 如何使用验证函数来加强这些设计, 以及文档是如何被保存的一些基本. 在下一个章节中, 我们会展示如何从CouchDB读取文档, 并把它们在浏览器里展现出来.
