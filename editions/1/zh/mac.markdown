## 在Mac OS X上安装 ##

### CouchDBX ###

在Mac OS X上最简便的方法就是下载CouchDBX. 这个非官方的应用不会在你的系统上安装任何东西, 只需要双击运行就可以了. 请注意, 如果你要在更重要的地方使用CouchDB的话, 还是建议你用类似Homebrew的工具做一个传统的安装.

### Homebrew ###

Homebrew是最近出现的一个Mac OS X上的软件管理工具. 它的理念是零配置, 高优化, 并且它是开源的. 可以从 http://github.com/mxcl/homebrew 得到Homebrew. 安装很容易. 安装设置完成后, 运行:

				brew install couchdb

等待其运行完毕. 要想启动CouchDB, 只需要简单的运行下面的命令:

				couchdb

要想看到所有可用的启动选项, 可以运行:

				couchdb -h

这个命令会告诉你如何在后台运行CouchDB, 以及一些其他可用的提示.

要验证CouchDB已经真正的在运行了, 打开你的浏览器然后试着访问 http://127.0.0.1:5984/_utils/index.html.
 
### MacPorts ###

MacPorts被认为是Mac OS X上实际的包管理工具. 虽然并非操作系统正式的一部分, 它可以用来简化安装软件的过程. 在你可以使用MacPorts安装CouchDB之前, 你需要先下载和安装MacPorts.

运行下面的命令以确保你安装的MacPorts是最新版本:

				sudo port selfupdate

可以通过下面的命令来CouchDB:

				sudo port install couchdb

这条命令会安装所有CouchDB所需要的依赖包. 如果一个依赖包已经有安装了, MacPorts不会去升级它到最新的版本. 要确保所有的依赖包都是最新的版本, 你还需要运行:

				sudo port upgrade couchdb

Mac OS X有一个服务管理框架叫做launchd, 它可以用来启动, 停止, 以及管理系统守护程序. 你可以使用它在系统启动时自动的启动CouchDB. 如果你想要把CouchDB加入launchd配置, 应该运行下面的命令:
 
				sudo launchctl load -w /opt/local/Library/LaunchDaemons/org.apache.couchdb.plist

运行完毕后, CouchDB应该就已经启动了, 可以在下面的URL访问到:

				 http://127.0.0.1:5984/_utils/index.html

以后, CouchDB会和操作系统一起启动和停止.
