## 安全性 ##

之前我们提到过, CouchDB还处于开发之中. 并且在本书发行之后依然可能会有功能被添加进来. 这对于CouchDB的安全机制来讲, 尤其如此.  在目前版本(0.10.0)有着基本的安全性支持, 但在写作本书时, 我们正在讨论是否会加入更多的安全机制.

在本章节中, 我们会探讨CouchDB中几个基本的安全机制: the Admin Party, 基本认证, Cookie认证, 以及OAuth.

### The Admin Party ###

在刚安装完时, CouchDB允许任何人做出任何请求. 创建一个数据库? 没问题, 搞定. 删除一些文档? 也没问题. CouchDB把这个称为Admin Party. 任何人都有权限做任何事.

虽然对于上手CouchDB来说, 这是一种非常简单的方法, 但是很明显的, 把这种默认的配置放在网络上是十分冒险的. 任何一个恶意的客户端都可以连接上来并且删除数据库.

有一点让人安心的是: 默认的, CouchDB只会监听你的本地回路网络(127.0.0.1或者localhost), 因此只有你自己能够发送请求给CouchDB. 但是当你把CouchDB放到公众网络上时(把它设置成为绑定你机器的公网IP地点), 会想要限制访问以防有些坏人会搞些破坏.

在之前的讨论中, 我们忽略了在admin party模式之外应该怎么做的几个关键词. 首先, 是admin这个词本身, 它暗指了某类超级用户, 此外还有权限这个词. 让我们来更详细的探讨下这些术语. 

CouchDB有管理员用户(比如, 类似windows中的administrator, 超级用户, 或者root)的概念, 管理员帐号可以做任何事情. 默认的, 所有人都是管理员. 如果你不喜欢这样, 你可以创建特定的管理员用户, 设置一个用户名和密码作为认证.

CouchDB还定义了一个请求集合, 这些请求只有管理员用户才能执行. 如果你定义了一个或者更多个的管理员用户, CouchDB会对一些特定的请求要求认证:

* 创建一个数据库 (PUT /database)
* 删除一个数据库 (DELETE /database)
* 创建一个设计文档 (PUT /database/_design/app)
* 更新一个设计文档 (PUT /database/_design/app?rev=1-4E2)
* 删除一个设计文档 (DELETE /database/_design/app?rev=1-6A7)
* 触发compaction (POST /_compact)
* 读取任务状态列表 (GET /_active_tasks)
* 重启服务器 (POST /_restart)
* 读取激活中的配置 (GET /_config)
* 更新激活中的配置 (PUT /_config)

#### 创建新的管理员用户 #### 

让我们再用curl来走一遍API, 看看CouchDB是怎么添加管理员用户的.

				> HOST="http://127.0.0.1:5984"
				> curl -X PUT $HOST/database
				{"ok":true}

开始, 我们创建一个数据库. 这里没什么有意思的东西. 现在让我们来创建一个管理员用户. 我把叫她anna, 她的密码是secret. 注意下面的代码里的双引号; 对于配置API的字符串值来说(就和我们之前学到的那样)是必需的.

				curl -X PUT $HOST/_config/admins/anna -d '"secret"'
				""

对于每一个_config API请求, 我们都会得到这项配置之前所拥有的值. 因为我们的管理员用户还不存在, 所以得到一个空字符串.

此时如果我查看一下CouchDB的日志文件, 就会发现下面的两行:

				[debug] [<0.43.0>] saving to file '/Users/jan/Work/couchdb-git/etc/couchdb/local_dev.ini', Config: '{{"admins","anna"},"secret"}'

				[debug] [<0.43.0>] saving to file '/Users/jan/Work/couchdb-git/etc/couchdb/local_dev.ini', Config:'{{"admins","anna"}, "-hashed-6a1cc3760b4d09c150d44edf302ff40606221526,a69a9e4f0047be899ebfe09a40b2f52c"}'

第一行是我们的原始请求. 你可以看到我们的管理员用户被写入CouchDB的配置文件里了. 我们把CouchDB的日志等级设置为debug来看看发生了什么. 首先我们看到密码是以明文的方式请求过赤的, 然后产生经过哈希处理的密码.

#### 经过哈希处理的密码 ####

看到明文的密码让人很害怕, 是吧? 别担心; 正常情况下, 日志等级不会设置为debug, 明文的密码不会出现在任何地方. 它会马上就进行哈希处理. 处理后就是那串又大, 又丑, 又长的以-hashed-开头的字符串. 那么它是如何进行处理的呢?

1. 创建一个新的128位的UUID. 这是我们的salt.
2. 创建一个明文密码和salt组合的sha1哈希(sha1(password + salt)).
3. 加上前缀-hashed, 再加上后缀, salt.

进行认证时要比较明文密码和保存的哈希密码时, 也会进行同样的过程, 然后其结果再拿来和存储的哈希密码进行比较. 对于两个不同的密码来说, 产生两个完全一样的哈希基本上是不可能的事(c.f. Bruce Schneier). 那如果存储的哈希密码会落入攻击者的手中呢? 目前来讲, 要想从哈希中找出明文的密码不是一件那么容易的事(比如, 需要花费大量的金钱和时间).

那这个-hashed的前缀是做什么用的呢? 好吧, 还记得配置API是如何工作的吗? 当CouchDB启动时, 它会读取一组其中包含了配置信息的.ini文件. 它会把这些配置放入一个内部存储空间(并不是一个数据库里). 配置API允许你读取当前的配置, 同时也允许你改变或者创建新的配置项. CouchDB会把任何改变都写回.ini文件.

这些.ini文件也可以在CouchDB未启动时进行手工修改. 不像我们刚才展示的那样创建一个管理员用户, 你还可以停止CouchDB, 打开local.ini文件, 在其中的[admins]部分里加入anna = secret, 然后再启动CouchDB, 以这种方式来完成. 通过读取local.ini里的新加入行, CouchDB会运行哈希算法, 并把哈希值写回local.ini, 替换掉其中的明文密码. 为了确保CouchDB只会对那些明文密码进行哈希处理, 而不会对已经处理后的密码再次进行处理, 才在它们前面加上了-hashed-, 以便于区分明文密码和经过哈希处理的密码. 这意味着明文密码不能已经-hashed-开头, 但通常不会出现这种情况.

### 基本认证 ###

现在我们已经定义了一个管理员, CouchDB将不会允许我们创建新数据库了, 除非我们给出正确的管理员认证. 让我们来确认一下:

				> curl -X PUT $HOST/somedatabase
				{"error":"unauthorized","reason":"You are not a server admin."}

看起来是这么回事. 我们再试着使用正确的认证:

				> HOST="http://anna:secret@127.0.0.1:5984"
				> curl -X PUT $HOST/somedatabase
				{"ok":true}

如果你曾经访问过那些要求密码的网站或者FTP, username:password@这种形式应该很熟悉.

如果你对安全性很敏感, http://里少的那个s会让你觉得很不安. 我们用明文发送密码到CouchDB. 这点很不好, 是吧? 是的, 但是请考虑下这个场景: CouchDB在测试环境里, 监听在127.0.0.1, 这里我们是唯一的用户. 谁有可能会嗅探到我们的密码呢?

然而, 如果你在一个生产环境里, 就需要重新考虑了. 你的CouchDB实例是否会和另外一个公共网络进行通信? 即便是一个和他们共享的局域网也算是一个公共网络. 有多种访求可以加密你或者你的应用到CouchDB的通信, 这些已经超出了本书的范畴. 我们建议您阅读如何建立VPN以及如何把CouchDB设置在HTTP proxy(像Apache的mod_proxy, nginx, 或者varnish)之后, 它们会为你处理SSL. 目前来说, CouchDB不支持SSL的API. 不过, 它可以和其他处于SSL代理之后的CouchDB实例进行复制.

#### 再次更新验证函数 ####

还记得第7章, 验证函数吗? 我们有一个经过更新后的验证函数, 让我们可以用于验证文档的作者和认证的用户名是否相同.

				function(newDoc, oldDoc, userCtx) {
					if (newDoc.author) {
						if(newDoc.author != userCtx.name) {
							throw("forbidden": "You may only update documents with author " +
								userCtx.name});
						}
					}
				}

这个userCtx到底是什么? 它是一个包含了当前请求的认证数据的对象. 让我们来看看里面有什么. 我们来展示一个小技巧, 如何在写JavaScript时, 来查看一个具体对象.
 
				> curl -X PUT $HOST/somedatabase/_design/log -d '{"validate_doc_update":"function(newDoc, oldDoc, userCtx) { log(userCtx); }"}'
				{"ok":true,"id":"_design/log","rev":"1-498bd568e17e93d247ca48439a368718"}

来看看validate_doc_update函数:

				function(newDoc, oldDoc, userCtx) {
					log(userCtx);
				}

这个函数在以后每次文件更新时都会被调用, 它不做任何事情, 但是会在CouchDB的日志文件中进行记录. 如果我们现在创建一个新文档:

				> curl -X POST $HOST/somedatabase/ -d '{"a":1}'
				{"ok":true,"id":"36174efe5d455bd45fa1d51efbcff986","rev":"1-23202479633c2b380f79507a776743d5"}

我们就会在couch.log文件中看到这个:

				[info] [<0.9973.0>] OS Process :: {"db": "somedatabase","name": "anna","roles":["_admin"]}

我们格式化一下它:

				{
					"db": "somedatabase",
					"name": "anna",
					"roles": ["_admin"]
				}

我们可以看到当前的数据库, 认证用户的名字, 以及一个角色的数组, 其中有一个角色"_admin". 我们可以得知CouchDB的管理员用户只是一个普通的用户, 只是带有一个管理员的角色.

通过把用户和角色相分离, 认证系统就有了更大的灵活性. 目前而言, 我们只关注于管理员用户.

### Cookie认证 ### 

基本认证使用了明文的密码, 这很方便, 但是如果没有另外的措施, 这并不是很安全. 此外, 用户体验也很糟糕. 如果你使用基本认证来认证管理员, 应用程序的用户就需要使用一个丑陋的, 浏览器对话框, 会让人觉得很不专业.

会了消除这些顾虑, CouchDB支持cookie认证. 使用cookie认证, 你的应用就不必再包含丑陋的浏览器自带的对话框了. 你可以使用正常的HTML表单来向CouchDB提交登录请求. 收到请求后, CouchDB会产生一个标记(token), 客户端在下次发送请求至CouchDB时还可以使用. 当在接下来的请求过来时, CouchDB会根据这个标记来进行用户认证, 而不需要再次验证密码了. 默认, 一个标记的有效时间是10分钟.

要想取得标记并首先认证用户, 用户名和密码必须先发送至_session API. 这个API很聪明, 能够自己从表单提交中解码HTML, 所以你不需要在应用程序中做任何处理.

如果你并没有使用HTML表单来登录, 你需要发送一个HTTP请求, 这个请求像是一个HTML表单产生的. 幸运的是, 这非常简单:

				> HOST="http://127.0.0.1:5984"
				> curl -vX POST $HOST/_session -H 'application/x-www-form-urlencoded' -d 'username=anna&password=secret'

CouchDB响应, 下面就是一些细节:

				< HTTP/1.1 200 OK
				< Set-Cookie: AuthSession=YW5uYTo0QUIzOTdFQjrC4ipN-D-53hw1sJepVzcVxnriEw;
				< Version=1; Path=/; HttpOnly
				> ...
				<
				{"ok":true}

一个200的返回码告诉我们一切正常, 一个Set-Cookie头包括一个标记让我们可以在下次请求中使用, 标准的JSON响应则再次告诉我们请求成功了.

现在我们就可以使用这个标记以同一个用户来作另一次请求, 而不必再发送用户名和密码了:

				> curl -vX PUT $HOST/mydatabase --cookie AuthSession=YW5uYTo0QUIzOTdFQjrC4ipN-D-53hw1sJepVzcVxnriEw -H "X-CouchDB-WWW-Authenticate: Cookie" -H "Content-Type: application/x-www-form-urlencoded"
				{"ok":true}

默认情况下, 你可以不断的使用这个标记分钟. 10分钟以后, 你需要重新认证用户. 标记的存活时间可以在配置的couch_httpd_auth部分的timeout(以秒计算)里设置.

请注意, 要使用cookie认证, 你需要在local.ini里启用cookie_authentication_handler.

[httpd]
authentication_handlers = {couch_httpd_auth, cookie_authentication_handler}, {couch_httpd_oauth, oauth_authentication_handler}, {couch_httpd_auth, default_authentication_handler}

另外, 你还需要定义一个服务器密码:

[couch_httpd_auth]
secret = yours3cr37pr4s3

### 网络服务器安全性 ###

CouchDB是一个网络服务器, 一些如何进行安全设置的最佳实践超出了本书要讨论的范畴. 附录D, 从源代码安装包含了一些最佳实践. 请确保自己了解其中的含义.
