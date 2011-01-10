## 用自定义的格式来显示文档 ##

CouchDB show函数的RESTful API是由Lotus Notes的一个类似的功能启发而来的. 简言之就是允许你提供给客户端以任何你想要格式的文档.

一个show函数, 根据存储的JSON文档, 对于任何的Content-Type都会构建一个HTTP响应. 对于Sofa来说, 我们会使用它来显示博客日志的页面. 这样就能保证这些页面可以被搜索引擎索引, 也可以让页面变得更容易访问. Sofa的show函数显示每个日志页面为HTML页面, 且链接有样式表以及其他资源(assets), 它们以附件的形式存储在Sofa的设计文档中.

哈, 这太棒了--我们已经生成了一个日志! 见图1, "一个生成好的日志".

Figure 1. A rendered post

完整的show函数和模版会生成一个静态的, 可缓存的资源. 这个资源不依赖于当前用户的各种细节或者任何除了被请求文档和Content-Type以外的因素. 从show函数生成HTML不会在数据库产生任何副作用, 这对于构建简单的可扩展的应用有着很积极的影响.

### 用show函数来展现文档 ###

让我们来看看源代码. 首先我们要看的是JavaScript函数的函数体, 它十分简单--只是运行了一个模版函数来生成HTML页面. 让我们把它分块来看看:

				function(doc, req) {
					// !json templates.post
					// !json blog
					// !code vendor/couchapp/template.js
					// !code vendor/couchapp/path.js

在第12章, 存储文档里我们对于!code和!json宏已经更加熟悉了. 在这个例子里, 我们会使用它们来导入一个模版和一些关于博客的源数据(作为JSON数据), 以及用来包含作为内嵌代码的链接和模版生成函数.

接下来, 我们生成模版:

					return template(templates.post, {
						title : doc.title,
						blogName : blog.title,
						post : doc.html,
						date : doc.created_at,
						author : doc.author,

The blog post title, HTML body, author, and date are taken from the document, with the blog’s title included from its JSON value. The next three calls all use the path.js library to generate links based on the request path. This ensures that links within the application are correct.
日志的标题, HTML体, 作者和日期是从文档里取出来的, 并且还带上了博客标题. 接下的三个调用都使用了path.js库, 根据请求的路径来生成链接. 这保证了应用里的链接都是正确的.

						assets : assetPath(),
						editPostPath : showPath('edit', doc._id),
						index : listPath('index','recent-posts',{descending:true, limit:5})
					});
				}

所以我们看到函数体本身只是计算了一些值(根据文档, 请求, 以及一些部署细节, 像数据库名字之类的), 然后就发送给模版来生成. 真正起作用的是在HTML模版里. 我们来看看.

#### 日志页面模版 ####

模版定义了输出的HTML, 其中有些标签用于替换动态内容. 在Sofa这个例子里, 动态标签是这个样子的: <%= replace_me %>, 这种表示在模版里很常见.

Sofa使用的模版引擎来自John Resig的博客日志, "JavaScript Micro-Templating". 因为不用修改就能简单的在服务器端使用而被用在Sofa里.

让我们来看看模版字符串. 记住, 它是用CouchApp的!json宏来包含在JavaScript中的, 这样CouchApp就能处理转义并且包含它以便模版引擎处理.

				<!DOCTYPE html>
				<html>
					<head>
						<title><%= title %> : <%= blogName %></title>

这是我们第一次实际的看到模版标签--日志的标题, 以及在blog.json中定义的博客名, 它们用于生成HTML标签<title>的内容.

						<link rel="stylesheet" href="../../screen.css" type="text/css">

因为show函数通过设计文档中的路径访问, 所以我们可以通过相对路径的URI来链接到附件. 这里我们链接的是screen.css, 一个存储在Sofa源代码_attachements目录的文件.

					</head>
					<body>
						<div id="header">
							<a id="edit" href="<%= editPostPath %>">Edit this post</a>
							<h2><a href="<%= index %>"><%= blogName %></a></h2>

再一次, 我们模版标签来代替内容. 在这个例子里, 我们链接到了这个日志的编辑页面, 以及到博客首页的链接.

						</div>
						<div id="content">
							<h1><%= title %></h1>
							<div id="post">
								<span class="date"><%= date %></span>

日志标题用于<h1>标签, 日期则放在一个class为date的<span>标签里. 为什么我们在这里放入的是静态的日志, 而不是更加用户友好的像是"三天前"之类的数据, 要请查看下面的"动态日期"部分

				<div class="body"><%= post %></div>
							</div>
						</div>
					</body>
				</html>

这模版的结尾, 我们生成日志的内容(as converted from Markdown and saved from the author’s browser).

### 动态日期 ###

如果CouchDB运行在一个缓存代理之后, 这就意味着每个show函数应该只会在每个更新的文档上运行一次. 这也解释了为什么时间戳会像是2008/12/25 23:27:17 +0000, 而不是"9天前"这样子的.

这也意味着如果想要根据当年时间来显示时间, 或者根据浏览页面的用户来显示时间, 我们就需要使用客户端的JavaScript来对最终的HTML页面作动态改变.

						$('.date').each(function() {
							$(this).text(app.prettyDate(this.innerHTML));
						});

我们包含了这个客户端JavaScript实现的细节并不是要教你Ajax, 而是因为这对于在如何在客户端展现文档有着代理意义. 根据客户端的请求, CouchDB可以提供任何格式的文档. 但是在从其他查询集成信息或者和其他web services一起实现实时展现时, 通过客户端来作这些工作, 可以把计算时间和内存消耗从CouchDB转向客户端. 因为客户端的数量远比CouchDB来的多, 把负载放到客户端就意味着每个CouchDB可以承受更多的用户.
