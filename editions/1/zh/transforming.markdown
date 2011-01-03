## Transforming Views with List Functions ##

就像show函数可以把文档转换为任意一种输出格式, CouchDB的list函数也允许你通过视图生成任何格式的输出. 强大的迭代API可以用来灵活的对记录进行过滤和分组, 也使得生成各种原数据格式--Atom fees, HTML列表, CSV文件, 配置文件, 甚至只是修改过的JSON--变得十分简单.

List函数保存在设计文档的lists域中. 下面是一个包含有两个list函数的示例设计文档:

				{
					"_id" : "_design/foo",
					"_rev" : "1-67at7bg",
					"lists" : {
						"bar" : "function(head, req) { var row; while (row = getRow()) { ... } }",
						"zoom" : "function() { return 'zoom!' }",
					}
				}

### list函数的参数 ### 

list函数接受两个参数, 它们有时候可以被省略, 因为记录数据本身是在函数执行时被加载的. 第一个参数, head, 包含了关于视图的信息. 下面列出的可能会是你可能看到的一个head的JSON表示:

				{total_rows:10, offset:0}

参数request是一个更加丰富的数据结构. 这里的request对象和传入show, update和filter函数中的对象是相同的. 我们在这里详细的说明一下, 就当是作为一个reference. 下面就是一个示例的req对象:


				{
					"info": {
						"db_name": "test_suite_db","doc_count": 11,"doc_del_count": 0,
						"update_seq": 11,"purge_seq": 0,"compact_running": false,"disk_size": 4930,
						"instance_start_time": "1250046852578425","disk_format_version": 4},

数据库信息--在请求一个数据库的URL时得到--被包含在request参数中. This allows you to stamp rendered rows with an update sequence and know the database you are working with.

					"method": "GET",
					"path": ["test_suite_db","_design","lists","_list","basicJSON","basicView"],

HTTP方法和来自客户端的请求地址很有用, 特别是在生成到其他资源的链接的时候.

					"query": {"foo":"bar"},

If there are parameters in the query string (in this case corresponding to ?foo=bar), they will be parsed and available as a JSON object at req.query.
如果在查询字符串中带有参数(在这个例子里对应于?foo=bar), 它们会被解析为JSON对象放在req.query里.

					"headers":
						{"Accept": "text/html,application/xhtml+xml ,application/xml;q=0.9,*/*;q=0.8",
						"Accept-Charset": "ISO-8859-1,utf-8;q=0.7,*;q=0.7","Accept-Encoding":
						"gzip,deflate","Accept-Language": "en-us,en;q=0.5","Connection": "keep-alive",
						"Cookie": "_x=95252s.sd25; AuthSession=","Host": "127.0.0.1:5984",
						"Keep-Alive": "300",
						"Referer": "http://127.0.0.1:5984/_utils/couch_tests.html?script/couch_tests.js",
						"User-Agent": "Mozilla/5.0 Gecko/20090729 Firefox/3.5.2"},
					"cookie": {"_x": "95252s.sd25","AuthSession": ""},

headers给了list和show函数可以返回客户端想要的Content-Type的能力, 以及其他的像cookies之类的好东西. 注意, cookies也同样被解析成了JSON的形式. Thanks, MochiWeb!

					"body": "undefined",
					"form": {},

在HTTP方法是POST的情况下, requset body也同样存在(以格式化的JSON形式, if applicable)

					"userCtx": {"db": "test_suite_db","name": null,"roles": ["_admin"]}
				}

最后, userCtx和发送给validation函数中的一样. 它提供对用户想要被验证的数据库名, 用户名, 以及用户被授予的角色. 在上面的例子中, 你看到的是在CouchDB的"admin party"模式下的一个匿名用户. 除非已经有指定管理员帐户, 否则所有人都是管理员.

关于list函数的参数基本上讲完了. 现在是时候来看看list函数本身的机制了.

### 一个示例list函数 ###

让我们把学到的知识拿来实际使用吧. 在本章介绍里, 我们提到了用list函数来生成配置文件. 有趣的是, 你可以把配置的信息存放在CouchDB中, 然后通过list函数来生成配置文件, 你不需要担心配置文件能被重新生成, 因为你知道它是由一个函数从数据库里生成的, 而不是从其他的信息源. 这个层次的独立性可以保证只要CouchDB运行正常, 配置文件就能生成正常. 因为你不能从其他系统服务, 文件, 或者其他网络资源中获取数据, 你就不可能因为其他外部因素写出不正确的配置文件.

J. Chris got excited about the idea of using list functions to generate config files for the sort of services people usually configure using CouchDB, specifically via Chef, an Apache-licensed infrastructure automation tool. The key feature of infrastructure automation is that deployment scripts are idempotent—that is, running your scripts multiple times will have the same intended effect as running them once, something that becomes critical when a script fails halfway through. This encourages crash-only design, where your scripts can bomb out multiple times but your data remains consistent, because it takes the guesswork out of provisioning and updating servers in the case of previous failures.

Like map, reduce, and show functions, lists are pure functions, from a view query and an HTTP request to an output format. They can’t make queries against remote services or otherwise access outside data, so you know they are repeatable. Using a list function to generate an HTTP server configuration file ensures that the configuration is generated repeatably, based on only the state of the database.

想像一下, 假设你有一个运行中的共享主机平台, 每个用户有一个用名字区分的虚拟主机. 你需要一个配置文件来启动某些节点配置(哪些模式需要使用, 等等), 配置文件中每个用户有一个配置块, 用来配置像用户的HTTP目录, 子域名, 转发端口等等的信息.

				function(head, req) {
					// helper function definitions would be here...
					var row, userConf, configHeader, configFoot;
					configHeader = renderTopOfApacheConf(head, req.query.hostname);
					send(configHeader);

这个函数的第一部分, 我们使用renderTopOfApacheConf(head, req.query.hostname)生成了配置文件的头. 它可能包含了传入到list函数的信息, 像是正在被配置的服务器的内部名字抑或是用户的HTML根目录. 我们不会展示函数的内容, 但你可以想像到它会返回一个长长的字符串, 包含了服务器全局配置和根据视图生成的每个用户的配置.

send(configHeader)函数的调用是list函数可以生成文本的关键. 简单的说, 它只是发送了一个包含了内容字符串的HTTP块到了客户端. 这在幕后, 有一系列的步骤, CouchDB会用一种同步协议与JavaScript解析器对话, 但以程序员的角度来看, send()函数就是用来生成HTTP块的.

Now that we’ve rendered and sent the file’s head, it’s time to start rendering the list itself. Each list item will be the result of converting a view row to a virtual host’s configuration element. The first thing we do is call getRow() to get a row of the view.
我们已经生成和发送的了文件的头, 是时候来生成配置列表本身了. 列表的每一项都是把视图记录转换为虚拟主机配置的结果. 我们要做的第一件事是调用getRow()来得到视图结果中的一行.

					while (row = getRow()) {
						var userConf = renderUserConf(row);
						send(userConf)
					}

while循环会一直运行直到getRow()返回null, CouchDB会这个方式告诉list函数所有的有效记录(根据视图的查询参数)都已经被用完了. 在继续往前讲之前, 让我们来看看在我们得到一条记录后, 做了些什么.

在这个例子里, 我们只是简单的根据记录生成了一个字符串然后返回给了客户端. 当所有的记录都被生成后, 循环结束. 注意, list函数可以提早返回. 可能是当找到一个特定的用户文档后就停止迭代, 也可能是根据配置文件里配置的资源量返回.(or is based on a tally it’s been keeping of some resource allocated in the configuration.)在这些情况下, 循环可以使用break语句或者其他方法提早结束. list函数并不非一定要生成所有传入它的记录.

					configFoot = renderConfTail();
					return configFoot;
				}

最后, 我们结束配置文件, 返回最后的字符串, 它会以HTTP块的形式被发送. list函数的最后一个动作总是返回一个字符串, 它会以最后的HTTP块的形式发送给客户端.

要实际使用我们的配置文件生成函数, 我们可以会运行像下面这样的命令行:

				curl http://localhost:5984/config_db/_design/files/_list/apache/users?hostname=foobar > apache.conf

这条命令会根据保存user中的视图生成我们的Apache配置并保存进一个文件. 多么简单就完成了一个可靠的配置生成器啊!

### list函数的理论 ###

我们已经看过了一个完整的list函数, 让我们再来看看list函数一些有价值的东西.

最明显的就是其迭代风格的API. 因为每一行都是通过getRow()独立的加载, 就很难造成内存泄漏. 在正确使用的情况下, list函数可以生成任意多的记录而不会发生错误.

On the other hand, this API gives you the flexibility to bundle a few rows in a single chunk of output, so if you had a view of, say, user accounts, followed by subdomains owned by that account, you could use a slightly more complex loop to build up some state in the list function for rendering more complex chunks. Let’s look at an alternate loop section:
从另一个方面来讲, 这样的API也给了你可以捆绑少量的记录到一个单一输出块的灵活性. 所以如果你有一个用户帐号的视图, 每个帐号拥有子域名, 你可以使用

				var subdomainOwnerRow, subdomainRows = [];
				while (row = getRow()) {

循环直到到达视图的endkey才会结束. 视图被结构化成这个样子--先取出用户的资料的记录, 然后在它后面紧跟着的是用户的所有子域名的记录. 我们会使用用户资料数据和子域名信息来构建每个用户的配置. 这意味着在我们获取了所有当前用户的记录之前, 我们不能生成任何子域名的配置. 

									if (!subdomainOwnerRow) {
										subdomainOwnerRow = row;

这个判断只在第一个用户时成立. 我们仅仅只是设置一个初始化的状态.

					} else if (row.value.user != subdomainOwnerRow.value.user) {

这是结束的条件. 只有在当前用户的所有子域名记录都迭代完毕后它才会被调用. 它在某条记录的用户和当前用户不同时被触发, 表明我们已经得到了所有的记录.

						send(renderUserConf(subdomainOwnerRow, subdomainRows));

我们已经准备好生前当前用户的所有配置了, 所以我们传入用户的资料记录和所有的子域名记录到一个生成函数(这个函数隐藏了所有复杂的nginx配置细节). 结果被发送到HTTP客户端, 它会把结果写入配置文件.

						subdomainRows = [];
						subdomainOwnerRow = row;

因为我们已经生成好了这一个用户, 所以我们清空了子域名的记录然后开始下一个用户配置的生成工作.

					} else {
						subdomainRows.push(row);

继续收集子域名记录.

					}
				}
				send(renderUserConf(subdomainOwnerRow, subdomainRows));

最后的代码有一点技巧性--在循环结束以后(运行到了视图查询的结尾), 我们还需要生成最后一个用户的配置. 我们不会忘记这个的!

这段循环代码的作用是, 我们会收集属于某一特定用户的子域名记录, 直到找到一行属于另一个用户的记录, 这里我们就生成这一特定用户的配置, 清空状态, 然后开始一个新用户的工作. 像这样的技巧展示了list函数的迭代API有多么的灵活.

More uses along these lines include filtering rows that should be hidden from a particular result set, finding the top N grouped reduce values (e.g., to sort a tag cloud by popularity), and even writing custom reduce functions (as long as you don’t mind that reductions are not stored incrementally).

### 查询list ###

我们还没有仔细的讲过如何查询list函数. 和show函数一样, 它们是放在设计文档里的资源. 一个list函数的基本路径是这样的:

				/db/_design/foo/_list/list-name/view-name

因为list函数名和视图名两者都指定了, 这意味着一个list函数可以跑不止一个的视图. 比如, 你可能有一个把博客评论生成为Atom XML格式的list函数, 然后把它同时跑在一个最新评论的全局视图和一个某一篇日志的最新评论视图上. 这就让你可以使用同一个list函数来同时提供全网站评论的Atom feed以及每一篇日志的评论.

在list函数的路径之后跟的是视图查询参数. 就像普通的视图一样, 不带任何查询条件的调用list函数就会找出视图里的所有记录. 大多数时候, 你会想要带上查询参数来限制返回的数据量.

你应该已经熟悉了第六章, Finding Your Data with Views讲解的视图查询条件了. 相同的条件也适应于_list查询. 让我们来看看这些查询的URL; 请看示例1, "一个JSON视图查询".

				GET /db/_design/sofa/_view/recent-posts?descending=true&limit=10

Example 1. A JSON view query

这个视图只查询最近的10个日志. 当然, 这个查询可以包含像startkey这个的参数也可以忽略--为了简单, 我们选择忽略它. 要想通过list函数运行这个视图查询, 我们可以获取list资源, 像示例2, "HTML list查询"那样.

				GET /db/_design/sofa/_list/index/recent-posts?descending=true&limit=10

Example 2. The HTML list query

这个生成首页的list函数把JSON转化为HTML. 就和运行视图查询一样, 可以附加查询参数以便用于分页. 就像我们将在第三部分, "示例应用"里的那样, 当有有了一个list函数后, 要加入分布非常方便. 再来看示例3, "Atom list查询". 

				GET /db/_design/sofa/_list/index/recent-posts?descending=true&limit=10&format=atom

Example 3. The Atom list query

list函数还可以根据查询参数来转换输出格式. 你甚至可以把用户名作为查询参数(但是不推荐, 因为这样做会毁掉缓存效率).

### Lists, Etags, 和缓存

就像show函数和视图查询, list函数也会发送合适的HTTP Etags, 这使得它们都可以被中间代理缓存. 这意味着如果服务器运行着list函数代码, 那么就有可能通过像Squid这样的反向代理缓存来降低负载. 在这里, 我们不再深入讲解Etags和缓存. 因为它们在第8章, show函数中有着详细的讲解.
