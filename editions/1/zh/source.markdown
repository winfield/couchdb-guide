## 从源代码安装 ##

一般来说, 应该避免从源代码安装. 许多操作系统都提供了包管理工具, 可以让你使用一个命令, 就可以下载和安装好CouchDB. 这些包管理工具通常会帮你进行正确的配置, 处理安全性问题, 并确保CouchDB能被系统正常的启动和停止. 前面的几个附录向你展示了如何在Unix-like, Mac OS X, 以及Windows上安装CouchDB软件包. 如果那些方式在你的机器上不起作用, 或者你因为其他一些原因想要自己手工安装, 那么这一章节就是为你准备的.

### 依赖 ###

要编译和安装CouchDB, 你需要安装一系列CouchDB所依赖于的其他软件. 如果这些软件没有被正常的安装在你的系统上, CouchDB将不能正常的工作. 你需要下载和安装以下的软件:

* Erlang OTP (>=R12B)
* ICU
* OpenSSL
* Mozilla SpiderMonkey
* libcurl
* GNU Make
* GNU Compiler Collection

如果可以的话, 建议安装Erlang OTP R12B-5或更新版本.

这些软件应该都有提供自定义安装的指导, 不是在它们的网站上就是在你下载的压缩包里. 如果幸运的话, 那么你可能可以通过包管理工具来安装这些依赖.

#### 基于Debian的系统(包括Ubuntu) ####

你可以通过执行下面的命令来安装依赖:

				apt-get install build-essential erlang libicu-dev libmozjs-dev libcurl4-openssl-dev

如果你遇到了错误提示, 记得检查下你的发行版系统是否提供了这个版本的包. 也许是有更新的版发布了, 而它的包名被改变了. 比如, 你可以通过下面的命令来搜索最新版本的ICU包:

				apt-cache search libicu

选择并安装列表中最高版本的包.

#### Mac OS X ####

你需要安装Xcode工具, 可以通过以下命令:

				open /Applications/Installers/Xcode\ Tools/XcodeTools.mpkg

如果你的系统上不存在这个软件, 那就需要从Mac OS X的安装CD中来安装Xcode. 另外, 你还可以下载一份Xcode的拷贝.

然后你就可以通过MacPorts来安装其他的依赖了:

				port install icu erlang spidermonkey curl

请查看附录B, 在Mac OS X上安装, 了解更多的信息.

### 安装 ###

当你安装完所有的依赖后, 你可以下载一份CouchDB源代码. 它是一个压缩包, 你需要进行解压. 打开终端, 进到你解压后的目录里.

通过下面的命令配置源代码:

				./configure

我们将会把CouchDB安装到目录/usr/local, 这个目录是用户自定义安装软件的默认目录. 这个命令有一大堆的可用选项, 你可以自定义任何东西, 从指定安装目录位置, 比如安装到你的家目录, 到指定Erlang或者SpiderMonkey的安装位置.

要想看看有哪些可用的选项, 你可以使用下面的命令:

				./configure --help

一般来说, 如果第一次运行时你没有遇到任何错误, 那么以后就可以忽略这一步. 如果安装有点问题, 比如找不到某一个之前安装的依赖, 你只需要传入附加的选项就可以了.

如果一切顺利, 应该会看到下面的消息:

				You have configured Apache CouchDB, time to relax.

放松下.

编译和安装:

				make && sudo make install

如果把安装目录改到了其他的位置, 你可能不想要在这里使用sudo. 如果在执行make时遇到了问题, 你可以试试执行gmake作为替代, 如果你的系统上有这个命令的话. 你可以在INSTALL文件里找到更多的选项.

### 安全性考虑 ###

用超级用户来运行CouchDB是不建议的. 如果CouchDB服务器被攻击者侵入了, 而它又是由超级用户运行的, 那么这个攻击者就得到了你整个系统的访问权限. 这不会是我们想要看到的!

我们强烈推荐你为CouchDB创建一个指定的用户. 这个用户应该有尽可能小的权限, 最好只需要有足够运行CouchDB服务器的权限, 可以读取配置文件, 可以写入数据和日志目录就可以了. 

你可通过任何你的系统所提供的工具来创建一个couchdb用户.

在许多Unix-like系统, 你可以这样做:

				adduser --system --home /usr/local/var/lib/couchdb --no-create-home --shell /bin/bash --group --gecos "CouchDB" couchdb

Mac OS X在系统选项里提供了标准的帐户配置选项, 或者你也可以使用用户组管理应用, 这个应用可以作为服务器管理工具的一部分下载下来.

你应该确保, couchdb这个用户拥有一个可以工作的登录shell(login shell). 你可通过在终端里用couchdb用户登录来验证这一点. 你还应该确保把couchdb用户的家目录设置成为/usr/lcoal/var/lib/couchdb, 这个是CouchDB的数据库目录.

执行下面的命令来改变CouchDB目录的所有权:

				chown -R couchdb:couchdb /usr/local/etc/couchdb
				chown -R couchdb:couchdb /usr/local/var/lib/couchdb
				chown -R couchdb:couchdb /usr/local/var/log/couchdb
				chown -R couchdb:couchdb /usr/local/var/run/couchdb

执行下面的命令来改变CouchDB目录的权限:

				chmod -R 0770 /usr/local/etc/couchdb
				chmod -R 0770 /usr/local/var/lib/couchdb
				chmod -R 0770 /usr/local/var/log/couchdb
				chmod -R 0770 /usr/local/var/run/couchdb

This isn’t the final word in securing your CouchDB setup. If you’re deploying CouchDB on the Web, or any place where untrusted parties can access your sever, it behooves you to research the recommended security measures for your operating system and take any additional steps needed. Keep in mind the network security adage that the only way to properly secure a computer system is to unplug it from the network.

### 手动运行 ###

你可以通过下面的命令来运行CouchDB服务器:

				sudo -i -u couchdb couchdb -b

它通过sudo命令, 来以couchdb这个用户运行服务器.

当CouchDB启动后, 它应该会显示下面的消息:

				Apache CouchDB has started, time to relax.

放松吧.

要检查是否一切正常, 可以在浏览器里打开:

				http://127.0.0.1:5984/_utils/index.html

这就是Futon, CouchDB基于web的管理工具. 我们在本书早先的章节里讲解过Futon的一些基础. 当载入完毕后, 你应该在右侧菜单中选择执行CouchDB测试套件. 它可以用来检查一切是否如期工作, 而且如果有什么运行不大正常的, 它能发现出来, 帮你省去一大堆头痛的事情.

### 作为守护进程运行 ###

当你已经可以正常的启动CouchDB后, 可能会想把它作为守护进程来运行. 一个守护进程是指, 不间断运行在后台的, 等待请求进行处理的一个软件应用. 这也是大多数数据库服务器的运行方式, 当然你也可以把CouchDB配置为这种方式.

当CouchDB作为守护进程运行时, 它会在多个一系列文件里写入日志, 你需要时不时的进行清理. 如果想要让服务器崩溃, 那么让日志文件塞满整个磁盘会是个好办法! 有些操作系统自带有软件, 可以帮助清理. 而对于你来说, 最重要的是事先作好准备, 进行必要的运作来确保这不会成为一个问题. CouchDB自身带有一个logrotate配置, 它也许能够帮上你.

#### SysV/BSD风格的操作系统 ####

根据你的操作系统, couchdb守护进程脚本可能会被安装到/usr/local/etc目录下的init.d(如果是SysV风格的系统), 或者rc.d(如果是BSD风格的系统). 下面的例子使用了[init.d|rc.d]这种形式来表示这种选择, 你需要根据自己的系统进行选择.

你可以通过运行下面的命令来启动CouchDB守护进程:

				sudo /usr/local/etc/[init.d|rc.d]/couchdb start

你可以通过运行下面的命令来停止CouchDB守护进程:

				sudo /usr/local/etc/[init.d|rc.d]/couchdb stop

你可以通过运行下面的命令来获取CouchDB守护进程的状态:

				sudo /usr/local/etc/[init.d|rc.d]/couchdb status

如果你想要配置守护进程的工作方式, 可以在/usr/local/etc/default/couchdb文件里找到一堆的选项.

如果你不想要通过sudo来运行这个脚本, 那么就需要在这个文件去掉COUCHDB_USER这个设置.

你的操作系统可能会提供一种方式, 可以自动的来控制CouchDB守护进程, 把它作为一个系统服务来启动和停止. 要做到这样, 你需要把守护进程脚本拷贝到系统的/etc/[init.d|rc.d]目录里, 然后运行像下面这样的一个命令:

				sudo update-rc.d couchdb defaults

请查看你的系统文档了解更多的信息.

#### Mac OS X #####

你可以通过launchd系统来控制CouchDB守护进程.

你可以通过下面的命令来载入launchd配置

				sudo launchctl load /usr/local/Library/LaunchDaemons/org.apache.couchdb.plist

你可以通过下面的命令来卸载launchd配置

				sudo launchctl unload /usr/local/Library/LaunchDaemons/org.apache.couchdb.plist

你可以通过下面的命令来启动CouchDB守护进程:

				sudo launchctl start org.apache.couchdb

你可以通过下面的命令来停止CouchDB守护进程:

				sudo launchctl stop org.apache.couchdb

lauchd系统可以自动的控制CouchDB守护进程, 作为一个系统服务进行启动与停止. 要做到这样, 你需要把plist文件拷贝到系统的/Library/LaunchDaemons目录

请查看launchd的文档了解更多的信息.

### 故障处理 ###

软件就是软件, 它们有时候就是会发生什么错误. 不要惊慌; CouchDB有一个很棒的社区, 里面有很多人能够回答你的问题, 帮你入门. 下面是你的学习之路上可能会帮助的一些资源:

* 如果你遇到的一个奇怪的错误提示, 可以查看Error Messages wiki page.
* 一般的故障处理, 可以试试查看Troubleshooting steps页面.
* 对于其他的一般性技术支持, 你应该访问邮件列表.

在判断问题时, 别忘了使用你最喜欢的搜索引擎. 如果你在搜索引擎里找一圈, 通常可以找到一些有用的东西. 很可能有一大堆人正好遇到了和你相同的问题, 而这个问题的解决方法已经在某个地方贴出来了. 祝你好运, 记得要放松!
