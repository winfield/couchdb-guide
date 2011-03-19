## The Core API ##

本章节将仔细的来探索CouchDB. 我们会讲解所有的关于CouchDB的重要的话题以及一些明智的解决方法. 我们会讲解一些最佳实践并在一些常见问题上进行指导.

让我们从回顾在前几个章节的操作开始, 看看这些操作的背后在做什么. 我们还会讲解Futon在它的底层需要做些什么来提供给我们先前我们看到的那些美妙的特性.

这个章节同时是一个关于核心CouchDB API的介绍和参考. 如果你记不起来如何执行一个特殊请求或者忘记了为什么需要某个参数, 你总是可以回到这里来查找(我们自己可能是使用这一章节最多的用户.)

当我们在探索API时, 有时候需要绕个弯路来解释某个特定请求的原因. 这对我们来说, 是一个告诉你为什么CouchDB这么工作的好机会.

API可以被分成下面的几个部分. 我们会分别来看它们:

* 服务器
* 数据库
* 文档
* 复制

### 服务器 ###

这部分是基础的也是简单的. 它可以检查CouchDB是否正在工作. 也可以在有些需要特定版本CouchDB的软件库里检查CouchDB版本来做为安全保障. 我们会再一次用到curl这个工具.

        curl http://127.0.0.1:5984/

CouchDB响应:

				{"couchdb":"Welcome","version":"0.10.1"}

你会得到一个JSON字符串, 这个字符串, 如果把它作为原生对象或者你所使用的编程语言里的数据结构进行解析, 会得到一个welcome字符串和版本信息.

这些并不是非常有用, 但是它很好的展示了与CouchDB交互的一种方法. 发送一个HTTP请求然后就会在HTTP响应里收到一个JSON字符串作为结果.

### 数据库 ###

现在让我们做些更加有用的: 创建数据库. 严格的来说, CouchDB是一个数据库管理系统(DMS). 这意味着它可以有多个数据库. 一个数据库相当于一个桶, 这个桶里保存有一些"相互联系的数据". 我们会在后面解释这到底意味着什么. 而在实际中, 这个术语的意思有些重叠了, 人们经常把DMS当成一个数据库, 也同时把一个在DMS里数据库当成DMS. 我们可能会不去关心这种奇怪的现象, 所以请不要被它搞混了. 通常情况下, 通过上下文, 还是能的分清楚我们在讲的到底是整个CouchDB还是一个在CouchDB里的数据库的.

现在让我们来创建一个! 我们想要存储我们最喜欢的音乐专辑, 把数据库命名为albums. 注意, 我们又一次使用了-X这个选项来告诉curl来发送一个PUT请求而不是默认的GET请求.

				curl -X PUT http://127.0.0.1:5984/albums

CouchDB响应:

				{"ok":true}

这样. 你创建了一个数据库, CouchDB告诉你一切正常. 如果你试图创建一个已经存在的数据库会发生什么事? 我们来试试再创建同一个数据库:

				curl -X PUT http://127.0.0.1:5984/albums

CouchDB响应: 

				{"error":"file_exists","reason":"The database could not be created, the file already exists."}

我们得到了一个错误. 这相当的方便. 我们同时学到了一点关于CouchDB是如何工作的知识. CouchDB里每个数据库都存储在一个单一的文件里. 非常简单. This has some consequences down the road, but we skip on details for now and explore the underlying storage system the The Power of B-Trees appendix.

让我们来创建另一个数据库, 这次带上curl的-v("verbose"的简写)选项. verbose这个选项告诉curl不仅仅只显示必要的信息---HTTP的回复体, 还要显示请求与回复的细节:

				curl -vX PUT http://127.0.0.1:5984/albums-backup

curl详细显示:

				* About to connect() to 127.0.0.1 port 5984 (#0)
				*   Trying 127.0.0.1... connected
				* Connected to 127.0.0.1 (127.0.0.1) port 5984 (#0)
				> PUT /albums-backup HTTP/1.1
				> User-Agent: curl/7.16.3 (powerpc-apple-darwin9.0) libcurl/7.16.3 OpenSSL/0.9.7l zlib/1.2.3
				> Host: 127.0.0.1:5984
				> Accept: */*
				>
				< HTTP/1.1 201 Created
				< Server: CouchDB/0.9.0 (Erlang OTP/R12B)
				< Date: Sun, 05 Jul 2009 22:48:28 GMT
				< Content-Type: text/plain;charset=utf-8
				< Content-Length: 12
				< Cache-Control: must-revalidate
				<
				{"ok":true}
				* Connection #0 to host 127.0.0.1 left intact
				* Closing connection #0

满满的一屏幕. 让我们一行行的来看, 搞明白具体是在做什么并且找出哪些是重要的. 当看过几次这样的输出后, 你会更加容易的找出哪些是重要的.

				* About to connect() to 127.0.0.1 port 5984 (#0)

curl告诉我们它正在向我们的请求URI中的CouchDB服务器建立一个TCP连接. 这里没什么重要的东西, 只在调试网络问题的时候有用.

				*   Trying 127.0.0.1... connected
				* Connected to 127.0.0.1 (127.0.0.1) port 5984 (#0)

curl告诉我们成功的连接到了CouchDB. 这些也不重要, 如果没有发现什么网络问题的话.

下面的几行有一个">"或者"<"的前缀. ">"的意思是这几行被逐字发送到CouchDB(不包括">"). "<"的意思是这些是CouchDB发送回给curl的.

				> PUT /albums-backup HTTP/1.1

这行初始化一个HTTP请求. 它的方法是PUT, URI是 /albums-backup, HTTP版本HTTP/1.1. 还有一个HTTP/1.0的版本, 它在某些情况下更简单, 但是因为各种现实原因, 我们应该使用HTTP/1.1. 

接下来, 我们看到一些请求头. 这些是用来提供到CouchDB的请求的附加细节的.

				> User-Agent: curl/7.16.3 (powerpc-apple-darwin9.0) libcurl/7.16.3 OpenSSL/0.9.7l zlib/1.2.3

User-Agent头告诉CouchDB哪种客户端软件在作HTTP请求. 这里没什么新奇的东西, 我们用的就是curl. 在web开发中这个头经常很有用, 当服务器响应某个客户端实现的请求出现问题时. 它也可以帮助我们知道, 用户是在哪个平台上的. 这个信息可以被用于某些技巧和数据统计. 对于CouchDB来说, 这个头是无关紧要的.

				> Host: 127.0.0.1:5984

这个头是HTTP1.1需要的, 它告诉服务器请求的主机名.

				> Accept: */*

Accept头告诉CouchDB, curl接受任何媒体类型. 我们会在后面来深入了解为什么这很有用.

				>

一个空行表示请求头已经结束了, 剩下的请求包含我们要发送给数据库的数据. 在这个例子里, 我们不发送任何数据, 所以剩下的curl输出是HTTP响应的了.

				< HTTP/1.1 201 Created

CouchDB的HTTP响应的第一行包含了HTTP版本信息(也让我们知道了, 我们使用的HTTP版本可以被处理), HTTP状态码和状态码信息. 不同的请求会触发不同的返回状态码. 存在有一系列的状态码来告诉客户端(这个例子里是curl), 它作出的请求在服务器上起了什么作用. 或者, 如果有错误发生了, 告诉客户端什么错误发生了. RFC 2612(the HTTP 1.1 specification)清楚了定义了返回状态码的行为. CouchDB完全遵守这个RFC.

201 Created状态码告诉客户端, 请求创建的资源被成功的创建了. 这里没有什么值得惊奇的事, 但如果你还记得当我们试图两个创建这个数据库时得到了一个错误, 就明白这时会有一个不同的返回码. 根据返回码做出处理是一个常见的做法. 比如, 所有400或400以上的返回码告诉你有什么错误发生了. 如果你想要简化逻辑并即时处理错误, 可以只检查大于400的返回码.

				< Server: CouchDB/0.9.0 (Erlang OTP/R12B)

Server头对于诊断很有用, 它告诉我们是再和哪个版本的CouchDB和底层的哪个版本的Erlang打交道. 通常来说, 可以忽略这个头, 但当你需要它的时候, 要知道它在哪里.

				< Date: Sun, 05 Jul 2009 22:48:28 GMT

Date头告诉你服务器的时间. 因为客户端和服务器端的时间没有要求一定要保持同步, 这个头只是纯粹告诉你服务器时间这一信息而已. 你不应该根据这个信息为逻辑构建任何关键应用.

				< Content-Type: text/plain;charset=utf-8

这个头告诉你HTTP响应体是什么Content-Type和用的编码. 我们已经知道CouchDB返回JSON字符串. 合适的Content-Type是application/json. 为什么我们看到的是text/plain呢? 这就是实践战胜纯粹理论的地方了. 发送一个applicaion/json的Content-Type头会使浏览器把返回的JSON提供给你下载而不是显示它. 因为可以在浏览器里测试CouchDB非常重要, CouchDB发送了一个text/plain的Content-Type, 这样浏览器就把JSON以文本的形式显示出来.

有一些浏览器插件可以让你的浏览器认得出JSON, 但是它们并不是默认安装的.

你还记得Accept请求头吗, 它被设置成\*/\* -> */*, 表示它接受任何的Content-Type. 如果你在你的请求里发送Accept: application/json, CouchDB认为你可以处理纯JSON响应, 就会返回正确的Conten-Type头, 而不是text/plain.

				< Content-Length: 12

这仅仅告诉我们响应体有多少字节.

				< Cache-Control: must-revalidate

这告诉你, 或者任何在CouchDB和你之间的代理服务器, 不要缓存这个响应.

				<

这个空行告诉我们响应头已经完了, 接下来的是响应体了.

				{"ok":true}

我们以前已经看到过这个了.

				* Connection #0 to host 127.0.0.1 left intact
				* Closing connection #0

最后两行是curl告诉我们它会保持TCP连接打开一会, 但是在接收完整个响应后会关闭它.

贯穿于整书中, 我们会讲解更多的带-v选项的请求, 但会忽略掉一些我们已经在这里看过的头, 只讲解那些对于某个特定请求来说重要的头.

我们已经知道如何创建数据库了, 但是如何删除一个呢? 简单, 只要改变HTTP方法.

				curl -vX DELETE http://127.0.0.1:5984/albums-backup

这会删除一个CouchDB数据库. 这个请求会删除存储数据库内容的文件. 删除数据库时, 没有"你确定吗"这样的提醒或者"清空垃圾箱"之类的魔法. 请谨慎的使用这个命令. 你的数据会被删除, 并且如果你没有做复制, 就没有机会再轻易的恢复回来了.

这部分深入的讲解了HTTP并且为讲解剩下的CouchDB API建立了基础. 下一站: 文档.

### 文档 ###

文档是CouchDB的核心数据结构. 文档背后的观念是, 不出意料的, 就是真实世界的文档. 像帐单, 食谱或者名片一样的小纸片. 我们已经知道了, CouchDB使用JSON格式存储文档. 让我们看看这种存储是底层是如何工作的. 

CouchDB里的每个文档都会一个id. 每个数据库里这个id都是唯一的. 你可以选择任何字符串作为id, 但是最好的, 我们推荐使用UUID(或者GUID). Universally (or Globally) Unique IDentifier. UUID是一些极小概率可能重复的数字, 小到即使每个人每分钟产生成千上万个UUID, 持续几百万年都不会有重复产生. 这是一种非常棒的方法来保证两个不同的人不会产生相同id的文档. 为什么你要关心其他人在干什么? 第一个原因, 那个其他人可能会是以后某个时间某台不同电脑上的你自己; 第二个原因, CouchDB让你可以和其他人分享文档, 它使用UUID来保证其正常工作. 呆会我们再详细解释, 现在先来创建几个文档.

				curl -X PUT http://127.0.0.1:5984/albums/6e1295ed6c29495e54cc05947f18c8af -d 
				'{"title":"There is Nothing Left to Lose","artist":"Foo Fighters"}'

CouchDB响应:

				{"ok":true,"id":"6e1295ed6c29495e54cc05947f18c8af","rev":"1-2902191555"}

这个curl命令看起来有些复杂, 我们来分解一下. 首先-X PUT告诉curl作一个PUT请求. 它后面跟着一个URL来指定你的CouchDB的IP地址和端口. URL的资源部分/albums/6e1295ed6c29495e54cc05947f18c8af指定了我们的albums数据库中文档的位置. 那串乱七八糟的数字和字母集合是一个UUID. 这个UUID是你的文档的id. 最后, -d标志告诉curl用后面跟着的字符串来做PUT请求的body. 这个字符串是一个简单的JSON结构, 包括了标题和艺术家以及它们相应的值.


如果你手头上没有UUID, 你可以让CouchDB给你一个(实际上, 这正是我们刚才做的, 只不过没有向你展示出来). 仅仅需要发送一个GET请求到 /_uuids.

				curl -X GET http://127.0.0.1:5984/_uuids

CouchDB响应:

				{"uuids":["6e1295ed6c29495e54cc05947f18c8af"]}

如果你需要多于一个的UUID, 你可以传入?count=10 的HTTP参数来请求10个UUID, 或者任何你想要的数字.

为了确认CouchDB没有撒谎说它已经保存了你的文档, 实际上却并没有(通常它不会撒谎的), 试着用一个GET请求来得到这个文档.

				curl -X GET http://127.0.0.1:5984/albums/6e1295ed6c29495e54cc05947f18c8af

我们希望你能看出来这种模式. CouchDB里的一切东西都有一个地址, 一个URI; 你使用不同的HTTP方法来操作这些URI.

CouchDB响应:

				{"_id":"6e1295ed6c29495e54cc05947f18c8af","_rev":"1-2902191555","title":"There 
				is Nothing Left to Lose","artist":"Foo Fighters"}

这和你要CouchDB保存的文档很相似, 很好. 但是你应该注意到了, CouchDB在JSON结构中加了两个域. 第一个是_id, 它的值是我们要求CouchDB保存的文档的UUID. 请求一个文档时总是能得到文档的ID, 这很方便.

第二个是_rev. 它代表修订号.

#### 修订号 ####

如果你想更改CouchDB里的一个文档, 不是去找那个文档中的某个域然后插入一个新值. 而是从CouchDB载入整个文档, 在得到的JSON结构里作改变(或者是一个对象, 如果你在使用某个编程语言), 然后把整个新修订的文档存回CouchDB. 每个修订由一个新的_rev值标识.

如果你想要更新或者删除一个文档, CouchDB会期望你提供一个_rev域来标识你要改变的那个修订. 当CouchDB接受了一个更改以后, 它会产生一个新的修订号. 这种机制保证了, 万一有人在你对文档做更新之前做了一个你并不知情的更新, CouchDB不会接受你的更新因为你可能会覆盖你以为并不存在的数据. 或者简单点的说: 谁先保存了对一个文档的改变, 谁就赢了. 让我们来看看如果我们不提供一个_rev域会发生什么(这和提供一个过时的值是一样的).

				curl -X PUT http://127.0.0.1:5984/albums/6e1295ed6c29495e54cc05947f18c8af -d 
				'{"title":"There is Nothing Left to Lose","artist":"Foo Fighters","year":"1997"}'

CouchDB响应:

				{"error":"conflict","reason":"Document update conflict."}

如果你看到了这个, 在JSON结构里加上你的文档的最新修订号:

				curl -X PUT http://127.0.0.1:5984/albums/6e1295ed6c29495e54cc05947f18c8af -d 
				'{"_rev":"1-2902191555","title":"There is Nothing Left to Lose","artist":"Foo Fighters","year":"1997"}'

现在你发现为什么在作初始请求时CouchDB会返回_rev是件很方便的事了吧. 

CouchDB响应:

				{"ok":true,"id":"6e1295ed6c29495e54cc05947f18c8af","rev":"2-2739352689"}

CouchDB接受了你的写请求并且它也产生了一个新的修订号. 修订号是文档的md5散列, 加上一个N-的前缀表示文档被更新的次数. 这对复制很有用. 具体查看第17章, 冲突管理.

为什么CouchDB使用这种修订系统, 也被叫作多版本并发控制(MVCC), 有多个原因. 我们来解释其中的一些.

CouchDB使用的HTTP协议一个的特性便是它的无状态性. 这是什么意思? 要和CouchDB交流, 你需要做出请求. 做一个请求包括了打开一个到CouchDB的网络连接, 交换字节然后关闭连接. 这些事情在你每做一个请求时都会重复一遍. 其他协议允许你打开一个连接, 交换字节, 保持这个连接打开, 然后在此后交换更多的字节--可能是根据你一开始交换字节里包含的内容--最后关闭连接. 保持一个连接用于今后使用要求服务器做额外的工作. 在一个连接的生命周期里, 常见的模式是, 客户端会有一个持久的, 静态的服务器端的数据视图. 管理巨大量的并行连接是一项极大工作量的工作. HTTP连接通常是短生命周期的, 作出同样的保障会轻松很多. 结果就是, CouchDB可以处理更多的并发连接.

另外一个原因是这个模型在概念上更简单, 因此更加容易编程. CouchDB使用了更少的代码来达到目标, 而使用更少的代码总是好事, 因为固定行数代码的缺陷比例是固定的.

修订系统对于复制和存储机制也有积极的作用, 但我们将在本书的后面章节来讲到它们.

术语版本和修订听起来似乎很熟悉(如果你编程时不使用版本控制, 现在就赶紧把本书扔了, 然后找个流行的版本控制系统学习一下). 使用文档的新版本看赶来很像版本控制, 但是它们有一个很重要的区别: CouchDB不保证老版本一定不会丢失.

#### 文档的细节 ####

现在让我们用curl的-v选项来仔细的看看文档创建请求, 这在之前我们探索数据库API的时候很有用. 这也是一个创建更多文档的好机会, 以便我们在今后的例子中使用.

我们会增加一些喜欢的音乐专辑. 从/_uuids这个URI资源得到一个新的UUID. 如果你不记得这是怎么做了, 把书翻回去几页找找.

				curl -vX PUT http://127.0.0.1:5984/albums/70b50bfa0a4b3aed1f8aff9e92dc16a0 -d 
				'{"title":"Blackened Sky","artist":"Biffy Clyro","year":2002}'

顺便提一下, 如果你正好知道更多的关于最喜爱专辑的信息的话, 不要犹豫添请加上这些属性. 也不要着急如果你不知道所有这些专辑的所有信息, CouchDB的无模式文档可以包含任何你知道的. 总之, 你应该放松, 不要去担心数据.

带着-v选项, CouchDB响应的重要部分看起来应该像是这样:

				> PUT /albums/70b50bfa0a4b3aed1f8aff9e92dc16a0 HTTP/1.1
				>
				< HTTP/1.1 201 Created
				< Location: http://127.0.0.1:5984/albums/70b50bfa0a4b3aed1f8aff9e92dc16a0
				< Etag: "1-2248288203"
				<
				{"ok":true,"id":"70b50bfa0a4b3aed1f8aff9e92dc16a0","rev":"1-2248288203"}

在返回头中, 我们得到了一个201"创建" HTTP状态码, 这在之前我们创建数据库时我们也见过了. Location头告诉我们最新创建文档的完整URL. 而且有一个新的头; 来看看Etag先生. 在HTTP里, 一个Etag标识了一个资源的特定版本. 在这个例子里, 它标识了我们的新文档的一个特定版本. 听起来很熟悉? 是的, 从概念上讲, 一个Etag就是CouchDB文档的一个修订号, 所以CouchDB使用修订号作为Etag也没有什么可以惊讶的了. Etag在缓存系统中很有用, 我们会在第五部分, 扩展CouchDB中学会如何使用.

#### 附件 ####

CouchDB文档可以有附件, 就像email可以带附件一样. 一个附件由一个名字和它的源类型(或者Content-Type)以及它的字节数来标识. 附件可以是任何数据. 最简单的理解是附件就是附加在文档上的文件. 这些文件可以是文本, 图像, Word文档, 音乐或者电影文档. 让我们来创建一个.

附件有它们自己的URL, 你可以把数据上传到那. 假设我们想要把一张专辑的封面添加到文档6e1295ed6c29495e54cc05947f18c8af, 并且假设封面文档是当前目录下的artwork.jpg.

				> curl -vX PUT http://127.0.0.1:5984/albums/6e1295ed6c29495e54cc05947f18c8af/
				artwork.jpg?rev=2-2739352689 --data-binary @artwork.jpg -H "Content-Type: image/jpg"

-d@ 选项告诉 curl 去读取文档的内容放到HTTP请求体. 我们使用-H选项告诉CouchDB我们上传的是一个JPG文件. CouchDB会保存这个信息并且当我们请求这个文档的时候, 返回合适的头; 比如像这样的一个图像, 一个浏览器会显示这个图像而不会要你下载这个数据. 这在今后会变得很方便. 注意, 你需要提供你想到附加到的文档的当前修订号, 就和你更新一个文档时一样. 因为, 不管怎么样, 附加一些数据也是在改变这个文档.

如果你把你的浏览器指向http://127.0.0.1:5984/albums/6e1295ed6c29495e54cc05947f18c8af/artwork.jpg, 你应该会看到你的封面图片.

如果你再一次请求文档, 你会看到一个新的成员_attachments:

				curl http://127.0.0.1:5984/albums/6e1295ed6c29495e54cc05947f18c8af

CouchDB响应:

				{"_id":"6e1295ed6c29495e54cc05947f18c8af","_rev":"3-131533518","title":"There 
				is Nothing Left to Lose","artist":"Foo Fighters","year":"1997","_attachments":
				{"artwork.jpg":{"stub":true,"content_type":"image/jpg","length":52450}}}

_attachments是一个key和value的列表, value是包含附件原数据JSON对象. stub=true告诉我们, 这个附件只是一个元数据. 如果我们在请求一个文档时, 使用?attachments=true这个HTTP选项, 我们会得到一个包含附件数据的base64编码数据.

在我们探索CouchDB特性时, 会看到更多的文档请求选项. 比如复制, 我们的下一个主题.

### 复制 ###

CouchDB复制是一个用于数据库同步的机制. 很像rsync在本地或者网络上同步两个目录, 复制也在本地或者远程同步两个数据库.

在一个简单的POST请求里, 告诉CouchDB复制的源和目标, CouchDB会在源上找出有哪些文档和哪些新文档修订是目标上没有的并且会把它们移到目标上.

我们会在本书的后面深入的探索复制; 在本章节中, 我们只是展示如何使用它.

首先, 我们创建一个目标数据库. 注意, CouchDB不会自动的为你创建一个目标数据库而是会返回一个复制失败, 如果目标不存在的话(少了源的话也一样, 不过这个错误很不容易犯:)

				curl -X PUT http://127.0.0.1:5984/albums-replica

现在我们可以使用数据库album-replica作为一个复制目标:

				curl -vX POST http://127.0.0.1:5984/_replicate -d '{"source":"albums","target":"albums-replica"}'

在版本0.11中, CouchDB在POST到复制目标URL的JSON里支持了选项"create_target":true. 如果目标数据库不存在, 它会非显式的创建数据库.

CouchDB响应(这次我们对输出做了格式化, 这样你可以更加简单的读它):

				{
					"history": [
						{
							"start_last_seq": 0,
							"missing_found": 2,
							"docs_read": 2,
							"end_last_seq": 5,
							"missing_checked": 2,
							"docs_written": 2,
							"doc_write_failures": 0,
							"end_time": "Sat, 11 Jul 2009 17:36:21 GMT",
							"start_time": "Sat, 11 Jul 2009 17:36:20 GMT"
						}
					],
					"source_last_seq": 5,
					"session_id": "924e75e914392343de89c99d29d06671",
					"ok": true
				}

CouchDB会维护一份复制的历史. 一个复制请求的响应会包含这个复制的历史复制. 复制请求会一直保持打开直到复制结束. 如果你有很多的文档, 这会花点时间, 直到它们都被复制了. 而且在它们都被复制之前, 你不会得到复制的响应. 有一点很重要的需要注意的是, 复制只会复制复制开始时这个点上的数据库数据. 所以, 任何在复制开始后的添加, 更改或者删除都不会被复制.

最后的"ok":true告诉我们一切顺利. 如果现在你看下albums-replica数据库, 你应该会看到所有你在albums数据库创建的文档. 

刚才所做的在CouchDB的术语里叫做本地复制. 你创建了一个本地的数据库的副本. 这对于备份, 或者保留一份在某个特定时间的快照的数据用于日后使用来说是很有用的. 如果你在开发一个应用, 但是想在需要的时候可以返回到稳定的代码和数据版本, 你可能会想要这么做.

还有其他种类的复制, 在其他状况下有用. 我们复制的源和目标实际上是链接(就像在HTML里的那种), 而到目前为止, 我们看到的链接是指向我们正在工作的(就是本地的). 你也可以指定一个远程数据库作为目标:

				curl -vX POST http://127.0.0.1:5984/_replicate -d 
				'{"source":"albums","target":"http://127.0.0.1:5984/albums-replica"}'

使用一个本地源和一个远程目标数据库被叫做推送复制. 我们把改变推到远程服务器.

这里因为我们没有第二个CouchDB服务器, 我们就使用了本地单一服务器的绝对地址来演示. 但是从这里你应该可以看出来, 一个远程的服务器也是可以这样工作的.

想和远程服务器或者对门的哥们共享数据, 这方法好极了.

你也可以使用一个远程源和一个本地目标做拉取复制. 想要拿到别人数据库上作的最新改变, 这是个好办法.

				curl -vX POST http://127.0.0.1:5984/_replicate -d 
				'{"source":"http://127.0.0.1:5984/albums-replica","target":"albums"}'

最后, 你可以作远程复制, 这在进行管理操作时比较有用:

				curl -vX POST http://127.0.0.1:5984/_replicate -d 
				'{"source":"http://127.0.0.1:5984/albums","target":"http://127.0.0.1:5984/albums-replica"}'

CouchDB和REST

CouchDB对于拥有一个REST化的API感到很自豪, 但是复制请求看起来并不是很REST化. 这里出了什么问题? CouchDB的核心数据库, 文档, 以及附件API是REST化的, 但并不是所有的CouchDB API都是. 复制API就是其中的一个例子. 还有更多的非REST形式的API, 我们会在本书的后面章节里看到.

为什么这些REST化的非REST化的API混在了一起呢? 是这些开发人员懒的使这些API都REST化吗? 记住, REST是一种架构风格以用来建立一种特定的架构(比如CouchDB的文档API). 但它不能解决所有的问题, 同一尺寸大小的满足不了所有的, 你懂的. 触发像复制这样的事件在REST世界中并没有什么非常的意义. 它更像是一种传统的远程过程调用. 所以CouchDB这样做并没有什么不妥.

我们非常相信"使用合适的工具来工作"的哲学, 而REST并不合适于所有的工作. 为了得到支持, 我们参考了Leonard Richardson和Sam Ruby的意见, 他们写了RESTful Web Services(O'Reilly)这本书, 他们和我们有着同样的观点.

### 收尾 ###

这仍然不是完整的CouchDB API, 但是我们仔细的讨论了必要的部分. 我们会在下面的章节里把剩下的慢慢补完. 现在我们相邻你已经准备好构建CouchDB应用了.
