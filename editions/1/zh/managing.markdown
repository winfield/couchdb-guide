## 管理设计文档 ##

应用可以直接运行在CouchDB里--太好了. 只需要附加一些HTML和JavaScript文件到设计文档然后就可以运行了. 加上基于视图的查询以及可以从JSON文档中生成任何类型输出的显示函数, 你已经有了所有的, 可以写出自包含的CouchDB应用的工具了.

### 使用示例应用 ###

如果你想要在看接下来几章的过程中安装并修改Sofa, 会要使用CouchApp来上传源代码.

我们对于把应用部署到CouchDB的前景尤其感到兴奋. 因为, 在一个相同的环境里, 它鼓励用户不仅要控制数据还要控制源代码, 这就可以让更多的人构建自己的web应用了. 当你在空闲时间开发的web应用发展壮大的时候, CouchDB的扩展能力可以轻松的扩展到更大的硬件设备上.

在一个CouchDB设计文档中, 不同的开发语言(HTML, JS, CSS)放在不同的位置, 放在附件或者设计文档的属性里. 理想状态下, 你会想要开发环境能尽可能的帮助你减少工作量. 更重要的是, 你已经习惯于合适的语法高亮, 验证, 整合的文档, 宏, 辅助方法以及其他的帮助了. 在JSON对象里以字符串的形式编辑HTML和JavaScript代码不是现代人所做的工作.

幸运的是, 我们有一个解决方法. CouchApp. CouchApp让你可以在一个舒服的目录结构里开发CouchDB应用, 目录里views和shows是相分隔的, .js文件被清晰的管理; 静态文件(CSS, 图片)有自己的位置; 而且使用简单的couchapp push, 就可以把应用保存到CouchDB的设计文档里. 如果想做一个更改? 再做一次couchapp push就可以了.

这一章节会指导你CouchApp的安装过程和它的组件. 你将会学到它所拥有的那些可以帮助你工作的辅助方法. 当有了CouchApp后, 我们会用它来安装和部署Sofa到CouchDB数据库.

### 安装CouchApp ###

我们将要使用的CouchApp Python脚本和JavaScript框架是在设计这个示例应用时创建出来的. 它现在已经被使用于许多应用, 而且有了一个邮件列表, 一个wiki, 以及一个黑客社区. 在网上搜索"couchapp"来寻找最新的信息吧. 非常感谢Benoît Chesneau构建和维护这个库(以及其对CouchDB的Erlang代码库和许多Python库的贡献).

安装CouchApp最简单的办法是使用Python的easy_install脚本, 它是setuptools包的一部分. 如果你使用的是Mac, easy_install应该已经安装了. 如果easy_install还没有安装, 而你使用的是Debian的变种, 比如Ubuntu, 你可以使用下面的命令来安装它:

				sudo apt-get install python-setuptools

当你有了easy_install后, 安装CouchApp就简单了:

				sudo easy_install -U couchapp

一般这样就可以了, 你可以开始使用CouchApp了. 如果还是不能, 请继续看下去.

安装Couchapp时遇到的最普遍的问题是因为老版本的依赖, 特别是easy_install本身. 如果你遇到了一个安装错误, 最好的下一步做法就是尝试升级setuptools然后通过下面的命令升级CouchApp

				sudo easy_install -U setuptools
				sudo easy_install -U couchapp

如果你有其他的安装问题, 看一看setuptools的Python easy install问题解决, 或者访问CouchApp的邮件列表

### 使用CouchApp ###

通过easy_install安装CouchApp应该来说, 还是很简单的. 假设一切如愿, 它会管理所有的依赖并且把CouchApp的一些工具放到系统的PATH, 这样你就能马上开始使用它了:

				couchapp --help

我们会使用clone和push命令. clone会从一个云端的运行中的实例中拉取应用, 并把它保存在本地的一个目录结构里. push则会从本地文件系统中部署一个独立的CouchDB应用到任意一个你有管理权限的CouchDB里.

### 下载Sofa源代码 ###

有三种方法可以得到Sofa源代码. 这三种方法都可以; 取决于个人的喜好以及在得到源代码后你将如何使用它. 最简单的方法是使用CouchApp把它从一个运行的实例中clone过来. 如果在前一个部分中没有安装CouchApp, 你可以通过下载解压ZIP或者TAR的源代码包来阅读(但不能安装和运行)它们. 如果你对hacking Sofa或者想要加入开发社区, 最好的方法是从官方的Git库中获取源代码. 我们会逐一讲解这三种方法. 先来看看下面这张图片吧, 图1, "一只用来消除任何安装带来的沮丧的小鸟".
 
![一只用来消除任何安装带来的沮丧的小鸟](managing/01.png)

图1. 一只用来消除任何安装带来的沮丧的小鸟

#### CouchApp Clone ####

最简单的得到Sofa源代码的方法是使用CouchApp的clone命令直接从J. Chris的博客上下载设计文档以及其他一系列文件到本地硬盘上. clone命令带一个设计文档URL的参数, 它可以被托管在任何CouchDB数据库里. 想要clone J. Chris博客所运行版本的Sofa, 执行下面的命令:

				couchapp clone http://jchrisa.net/drl/_design/sofa

你应该会看到这样的输出:

				[INFO] Cloning sofa to ./sofa

现在你已经在本地有了一个Sofa, 你可以跳到"部署Sofa"这一部分, 作出一些小的本地改变然后把它push到你自己的CouchDB里.

#### ZIP and TAR Files ####

如果只是想在阅读本书的时候研读下源代码, Sofa有标准ZIP或者TAR格式的源代码包下载. 要得到ZIP版本的源代码, 在浏览器中访问以下的URL, 它会将你指向最新的ZIP文件: http://github.com/couchapp/couchapp/zipball/master. 如果你更喜欢TAR文件, 在这里:http://github.com/couchapp/couchapp/tarball/master.

#### 加入Github上的Sofa开发社区 ####

最新版本的Sofa永远会是在它的公开代码库上. 如果你对最新的开发以及贡献补丁感兴趣, 最好的方法是通过Git和GitHub.

Git是一个分布式的版本控制工具, 它可以让多组开发者跟踪和分享软件代码的变更. 如果你熟悉Git, 那通过它来使用Sofa没有任何问题. 如果你以前从来没有使用过Git, 则会有一点学习曲线. 所以根据自己对于新软件的容忍度, 你可能会想要节省下学习Git的时间或者会想要先学习一下Git! 关于更多的关于Git以及如何安装它的信息, 请查看Git的官方主页. 关于其他使用Git的技巧和帮助, 请查看Github的guides页面.

要通过Git得到Sofa(包括所有的开发历史), 运行下面的命令:

				git clone git://github.com/jchris/sofa.git

现在你已经有了源代码, 让我们来快速的浏览一下.

#### Sofa源代码的树结构 ####

当成功使用任一方法后, 你会在本地得到一份Sofa的源代码. 下面的文本是在Sofa目录上使用tree命令生成的, 它展示了Sofa包含的所有文件. 
				sofa/
				|-- README.md
				|-- THANKS.txt

源代码树包含了一些应用不需要的文件. 比如README和THANKS文件.

				|-- _attachments
				|   |-- LICENSE.txt
				|   |-- account.html
				|   |-- blog.js
				|   |-- jquery.scrollTo.js
				|   |-- md5.js
				|   |-- screen.css
				|   |-- showdown-licenese.txt
				|   |-- showdown.js
				|   |-- tests.js
				|   `-- textile.js

_attachments目录包含了作为附件保存到Sofa设计文档的文件. CouchDB服务器能让外部直接访问附件(而不是把它们包含在JSON包里), 所有这就是我们存储JavaScript, CSS和HTML文件的地方, 这样浏览器就能直接访问到它们了.

自己尝试一下修改Sofa的源代码, 你就会发现修改一个应用是多么的简单.

				|-- blog.json

blog.json文件包含了用于配置个人Sofa安装的JSON. 目前, 它只设置了一个值, 博客的标题. 现在你应该打开这个文件并个性化title这个域--可能并不想把你的博客命名为"Daytime Running Lights", 你脑袋里可以蹦出一些更加有趣的标题!

你可以加入其他的博客配置到这个文件里--像每一页要展现多少日志, "关于作者"页面的URL之类的. 在你看完接下来的几个章节后, 要加入这些改变到应用里会变得很简单.

				|-- couchapp.json

我们会在后面看到, 当运行couchapp push命令时, couchapp会打印出Sofa首页的链接. 原理很简单: CouchApp会在设计文档中寻找一个JSON域-design_doc.couchapp.index. 如果找到了, 它会把其中的值附加到设计文档本身的URL后面来生成URL. 如果没有指定CouchApp index, 但设计文档有一个附件叫index.html, 那么它就会被认为是首页. 在Sofa这个例子里, 我们使用了index域值来把首页指向一个最近日志的列表.

				|-- helpers
				|   `-- md5.js

helpers目录是否存在是随意的--CouchApp会把里面的任何文件和目录推送到设计文档. 在这个例子里, md5.js的源代码是JSON编码的, 它会被存放在design_document.helpers.md5元素里.

				|-- lists
				|   `-- index.js

lists目录包含了一个JavaScript函数, 这个函数会被CouchDB执行来生成Sofa的HTML首页和Atom Feed. 你可以通过在这个目录里添加新文件来加入新的list函数. 第14章, 显示博客日志列表里, 会深入讲解列表函数.

				|-- shows
				|   |-- edit.js
				|   `-- post.js

shows目录包含了CouchDB用来生成博客日志HTML的函数. 这个例子里有两种: 一个用来展现日志, 另一个用来编辑日志. 我们会在接下来的几个章节里来讲解这些函数. 

				|-- templates
				|   |-- edit.html
				|   |-- index
				|   |   |-- head.html
				|   |   |-- row.html
				|   |   `-- tail.html
				|   `-- post.html

templates目录更像helpers目录而不像lists, shows或者views目录, 在它里面存放的代码不会直接被CouchDB服务端执行. 而是当代码推送到服务器时, 通过执行CouchApp的宏被包含在list和show函数中. 这些CouchApp宏在第12章, 存储文档里有讲到. 关于templates目录关键的一点是目录名可以是任意的. 它不是设计文档的一个特殊域; 它只是一个用来存放我们模版文件的一个地方.

				|-- validate_doc_update.js

这个文件对应于Sofa使用的验证函数, 它用来保证只有博客所有者才能创建新的日志, 同时还保证评论都拥有合理的格式. 验证函数的细节在第12章, 存储文档里讲到.

				|-- vendor
				|   `-- couchapp
				|       |-- README.md
				|       |-- _attachments
				|       |   `-- jquery.couchapp.js
				|       |-- couchapp.js
				|       |-- date.js
				|       |-- path.js
				|       `-- template.js

vendor目录包含了相对于Sofa应用本身独立的代码. 在我们的例子里, 唯一使用到的vendor包就是couchapp, 它包含了用来链接list和show的URL以及生成模版之类工作的JavaScript代码.

在couchapp推送里, 在路径"vendor/**/_attachments/*"的文件会被放到设计文档的附件里. 在这个例子中, jquery.couchapp.js会被推送到叫"couchapp/jquery.couchapp.js的附件里(所以多个vendor包可以拥有相同的附件名, 而不用担心冲突)

				`-- views
						|-- comments
						|   |-- map.js
						|   `-- reduce.js
						|-- recent-posts
						|   `-- map.js
						`-- tags
								|-- map.js
								`-- reduce.js

views目录包含了MapReduce视图的定义, 每一个视图都由一个目录来代表, 包含了相应的map和reduce函数.

### 部署Sofa ###

你本地已经有了源代码, 还可以对blog.json做些小的修改. 现在是时候把博客部署到本地的CouchDB了. push命令的使用很简单而且第一次使用时应该会成功, 但还有其他两个需要做的步骤: 在CouchDB里以及CouchApp部署参数里设置你的管理员帐号. 在本章结束时, 你将会有一个运行中的Sofa.

#### 把Sofa推送到你的CouchDB

无论何时, 当你对本地的Sofa版本作了些改变而又想马上在浏览器里看到这些变动, 执行下面的命令:

				couchapp push . sofa

这条命令会把Sofa的源代码部署到CouchDB. 你应该会看到这样的输出:

				[INFO] Pushing CouchApp in /Users/jchris/sofa to design doc:
				http://127.0.0.1:5984/sofa/_design/sofa
				[INFO] Visit your CouchApp here:
				http://127.0.0.1:5984/sofa/_design/sofa/_list/index/recent-posts?descending=
				true&limit=5

如果你得到了一个错误, 可以通过做一个简单的HTTP请求, 来检查确保你的目标CouchDB实例是在运行中:

				curl http://127.0.0.1:5984

响应应该是像这个样子的:

				{"couchdb":"Welcome","version":"0.10.1"}

如果CouchDB还没有运行起来, 退回到第3章, 新手上路并参照那里的"Hello World"指示.

#### 访问应用 ####

如果CouchDB已经运行了, 那么couchapp push命令应该已经给你指明了应用了首页URL. 访问这个URL应该会把你带到如图2, "空白的首页"所示的页面.

![空白的首页](managing/02.png)

图2. 空白的首页

在得到一个功能完全的Sofa实例前, 我们还有些没有做完, 还有几个步骤留下.

### 创建你的管理员帐号 ###

Sofa是一个单用户应用. 你, 博客的作者, 就是管理员, 是唯一可以添加和修改日志的人. 为了确保没有其他人可以登录在你的博客里乱写, 你必须在CouchDB里创建一个管理员帐号. 这是一个很简单的任务. 找到你的local.ini文件, 用编辑器打开. (默认地, 它存储在目录/usr/local/etc/couchdb/local.ini.) 如果你还没有创建过管理员帐号, 打开文件末尾[admmins]那部分的注释, 在[admins]下面加入一行, 内容是你想要的用户名和密码:

				[admins]
				jchris = secretpass

编辑好了local.ini配置文件后, 你需要重启CouchDB来让配置生效. 根据你是如何启动CouchDB的, 重启CouchDB也有不同的方式. 如果你是在一个命令行里启动的, 那么按下Ctrl-C然后重新执行同样的启动命令就行了.

如果你不喜欢密码明文的放在纯文本文件里, 别担心. 当CouchDB启动时, 会读取这个文件, 它会把你的密码加密成一个安全的哈希值, 就像这样:

				[admins]
				jchris = -hashed-207b1b4f8434dc604206c2c0c2aa3aae61568d6c,96406178007181395cb72cb4e8f2e66e

现在当你试图创建数据库或者改变文档里, CouchDB会向你询问密码.

#### 部署到一个有了安全机制的CouchDB ####

现在我们已经创建一个管理员认证, 我们需要在couchapp push命令行里提供这个认证. 让我们来试一试:

				couchapp push . http://jchris:secretpass@localhost:5984/sofa

请把jchris和密码替换为你自己的, 不然你会得到一个"permission denied"的错误. 如果一切顺利, 所有的都已经在CouchDB里设置完毕, 你可以开始使用你的博客了.

到了这里, 从技术上来说, 我们可以继续讲解其他内容了, 但如果可以使用接下来要讲的.couchapprc文件, 你会更加开心.

### 通过.couchapprc来配置CouchApp

如果你不想每次push时都要在命令行里输入完整的URL(可能还要包括认证的参数), 你可以使用.couchapprc文件来存储部署设置. 这个文件的内容不会像其他文件那样被push, 所以当你要部署到一个有安全机制的服务器时, 把认证信息放在这里是安全的.

.couchapprc文件是在你应用源代码的目录中, 所以你应该看看它是不是在目录/path/to/the/directory/of/sofa/.couchapprc里(如果不存在就创建一个). 点文件(文件名以点号开头的文件)在大多数的文件管理器里不会被列出来. 使用你操作系统的各种技巧来"显示隐藏文件". 最简单的方法就是用在标准命令行shell里使用ls -a, 这条命令会显示所有的隐藏文件和普通文件.

						{
							"env": {
								"default": {
									"db": "http://jchris:secretpass@localhost:5984/sofa"
								},
								"staging": {
									"db": "http://jchris:secretpass@jchrisa.net:5984/sofa-staging"
								},
								"drl": {
									"db": "http://jchris:secretpass@jchrisa.net/drl"
								}
							}
						}

当这个文件设置好后, 你可以直接用命令"couchapp push"推送你的CouchApp, 它会把应用推送到"default"数据库. CouchApp也支持不同的环境. 要把你的应用推送到一个开发用数据库, 你可以使用命令"couchapp push dev". 以我们的经验来说, 花点时间来设置一个好的.couchapprc文件总是值得的. 另一个好处是在你工作时, 它可以让密码不用显示在屏幕上.
