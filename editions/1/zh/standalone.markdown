## 独立的应用 ##

CouchDB在一个应用的多个方面都很有用. 因为它的增量MapReduce和备份特性, 使其尤为适合那些在线交互文档和数据管理类的任务. 这些是大多数web应用都会经历的. 搭配CouchDB的HTTP接口让它自然而然的适合于web.

在这个部分里, 我们会来看一个面向文档的web应用--一个博客的实现.  As a lowest common denominator, 我们将会使用古老的纯HTML和JavaScript. 这里学到的东西可以同样应用于Django/Rails/Java风格的中间件应用, 甚至是专一的MapReduce数据挖掘任务. CouchDB的API也是一样, 不管你是跑在一个小型单一安装或者一个独立的集群上.

对于你应该使用哪个应用开发框架来搭配CouchDB, 没有一个标准答案. 我们已经看到过用几乎所有常用语言和框架实现的成功应用了. 在这个示例应用里, 我们使用一个两层的架构: CouchDB作为数据层, 浏览器来显现用户界面. 我们认为这是对于许多面向文档应用的一个可行的模型, 而且用于CouchDB教学, 它也是一个好方法. 因为我们可以简单的认为所有人都会装有一个浏览器, 且不需要你熟悉某一种特定的服务器端脚本语言.

### 使用正确的版本 ###

本书的这部分内容是交互式的, 所以准备好你的笔记本和一个运行中的CouchDB数据库. 我们已经把完整的示例程序和源代码放在了网上, 你要做的是把正确版本的示例应用下载下来并把它安装进CouchDB实例里.

写作和发行本书的一个很大挑战是CouchDB本身在以一个很快的速度发展进化. 那些基本概念在很长一段时间里不会改变, 甚至在更为遥远的未来也不会改变很多, 但是那些边边角角的东西却在接近CouchDB的1.0版本的过程中快速的演变.

本书的发行版将对应于CouchDB的0.10.0版本. 大多数的代码写于版本0.9.1和将要成为0.10.0的开发分支上. 在这一部分中, 我们会使用其他两个软件包: CouchApp, 它是一个包含编辑和分享CouchDB应用代码的工作集; 以及Sofa, 示例博客应用本身.

See http://couchapp.org for the latest information about the CouchApp model.

读者需要自己寻找这些软件包的正确版本. 对于CouchApp来说, 正确的版本永远是最新的那个版本. 正确版本的Sofa取决于你使用的CouchDB版本. 要知道你使用的CouchDB是什么版本, 运行下面的代码:

				curl http://127.0.0.1:5984

你应该会看到类似下面3行的输出:

				{"couchdb":"Welcome","version":"0.9.1"}

				{"couchdb":"Welcome","version":"0.10.0"}

				{"couchdb":"Welcome","version":"0.11.0a858744"}

这三行对应于版本0.9.1, 0.10.0和trunk. 如果你安装的CouchDB是0.9.1或更早的版本, 你应该升级到至少0.10.0, 因为Sofa使用到了只有0.10.0才有的一些特性. 也有一个老版本的Sofa可以在老版本中运行, 但本书涉及到的特性和API是CouchDB 0.10.0版本的一部分. 在你读到这里时, 可以想像到还会有0.9.2, 0.10.1甚至0.10.2版本. 请使用你喜欢的任一的最新版本.

Trunk代表了在Apache Subversion库中最新CouchDB开发版本. 我们推荐你使用一个已经正式发行的CouchDB版本, 但作为CouchDB的开发者, 我们经常使用的是trunk. Sofa的主分支将会在trunk上开发, 所以如果你想要跑在最前沿, 这会是一个正确的方法.

### Portable JavaScript ###

如果你不熟悉JavaScript, 我们希望示例代码能给你足够的上下文和解释来让你跟上. 如果你熟悉JavaScript, 你可能已经对于CouchDB支持视图和模版生成JavaScript函数很兴奋了.

另一个创建可以跑在一个标准CouchDB实例的应用的优势是它们可以通过备份实现移动性. 这意味着你的应用, 如果你将它直接开发在CouchDB里, "免费"的得到了一个离线的模式. 本地数据对于不同用户有着很大的不同, 我们在这里就不展开了. 我们把那些可以放入一个标准CouchDB的应用叫做CouchDB CouchApps.

CouchApps对于CouchDB教学来说一个极好的容器, 因为我们不需要去担心选择某一语言或框架之灰的问题; 我们直接使用CouchDB, 这样读者很快就能得到一个熟悉的应用模式的整体. 当你走完整个示例应用, 你将掌握足够的知识来把CouchDB应用到你自己的问题中去. 如果你不了解太多的Ajax开发, 你会需要学习一点jQuery. 最后我们希望你发现整个过程很轻松.

### 应用就是文档 ###

应用被存储为设计文档(图1, "CouchDB运行保存在设计文档里的应用"). 你可以备份设计文档就你备份其他任何在CouchDB里的东西. 因为设计文档可以被备份, 整个CouchApps就能被备份. CouchApps可以通过备份来更新, 也能轻易的被用户"forked", 他们可以随意的修改源代码.

Figure 1. CouchDB executes application code stored in design documents

因为应用只是一个特殊类型的文档, 就很容易编辑和分享.

J. Chris says: Thinking of peer-based application replication takes me back to my first year of high school, when my friends and I would share little programs between the TI-85 graphing calculators we were required to own. Two calculators could be connected via a small cable and we’d share physics cheat sheets, Hangman, some multi-player text-based adventures, and, at the height of our powers, I believe there may have been a Doom clone running.

The TI-85 programs were in Basic, so everyone was always hacking each other’s hacks. Perhaps the most ridiculous program was a version of Spy Hunter that you controlled with your mind. The idea was that you could influence the pseudorandom number generator by concentrating hard enough, and thereby control the game. Didn’t work. Anyway, the point is that when you give people access to the source code, there’s no telling what might happen.

如果有人不喜欢你的应用的审美, 他们可以修改CSS. 如果有人不喜欢你的界面设计, 他们可以改进HTML. 如果有人想要修改某些功能, 他们可以编辑JavaScript. 更为极端的, 他们可以完全改变你的应用来适合他们自己的目标. 当他们把修改的版本展示给同学和同事, 并且很有可能的, 会展示给你看时, 这意味着更多的人会想要做出改进.

作为最初的开发者, 你拥有版本的控制, 可以接受或者拒绝你认为合理的改变. 如果有人在本地应用里搞乱了源代码, 并且超出了可以修复的范围, 他们可以从你的服务器上备份一份最初的代码, 就像图2, "Replicating application changes to a group of friends"所示的那样.

Figure 2. Replicating application changes to a group of friends

当然, 这也许并不是你想要的. 别担心; 在CouchDB里能做到的限制这里也可以. 你可以用随意的限制数据的访问, 但要了解你可能失去的机会. 在开放协作和限制访问之间可以找到一个折中点.

当你完全安装后, 你就能看到完全的Sofa的代码了, 不仅仅是在文本编辑器里, 还可以在Futon展示的设计文档里.

### Standalone ###

如果添加了一个HTML文件作为文档的附件会怎么样? 和之前完全一样, 我们可以在CouchDB里直接server网页. 当然, 我们可能也需要图片, 样式, 或者脚本. 没问题; 只要把这些资源作为文档的附件添加进来, 然后用相对的URI来链接就行.

让我们后退一步. 到目前为止, 我们有了什么? 一种可以在Web上serve HTML文档和其他静态文件的方法. 这意味着我们可以使用CouchDB构建和serve传统的网站. Fantastic! 但是这是不是有点重复发明轮子的味道? 好吧, 一个好重要的不同之处在于, 在后台我们还有一个文档数据库. 我们可以使用我们served的网页中的JavaScript和这个数据库交互. 现在我们真的可以开工了!

CouchDB的特性是构建一个带有一个强大数据库的standalone web应用的基础. 作为这一概念的证明, 不用看别的, 只要看看CouchDB内建的管理界面就行. Futon是一个完整的数据库管理应用, 它由HTML, CSS和JavaScript构建. 没别的了. CouchDB和web应用是天生一对.

### In the Wild ###

======

### 收尾 ###

J. Chris 决定把他的博客从Ruby on Rails移植到CouchDB. 他开始把Rails的ActiveRecord对象导出为JSON文档, 并在他转换为HTML和JavaScript的过程中, 减去了一些特性, 加入了一些其他的特性.

最后的博客引擎拥有的功能有: 登录发帖, 避免滥用的开放评论, Atom feeds, Markdown格式支持, 以及一些其他的小功能. 这本书不是关于jQuery的, 所以虽然我们使用了这一JavaScript库, 我们不会过多的讨论它. 熟悉使用异步XMLHTTPRequest(XHR)的读者应该会觉得这些代码很熟悉. Keep in mind that the figures and code samples in this part omit many of the bookkeeping details.

我们将会研究这个应用, 并且学习它是如何使用所有的CouchDB特性的. 在这一部分里所学的技能可以被广泛的应用于任何CouchDB应用, 不管你是否准备构建一个self-hosted  CouchApp
