## 在类Unix系统上安装 ##

### Debian GNU/Linux ###

你可以通过运行下面的命令来安装CouchDB包:

				sudo aptitude install couchdb

当这个命令完成后, 你应该就有一个可以运行的CouchDB了. 请仔细阅读与Debian相关的系统文档, 它们可以在/usr/share/couchdb目录下找到. 

从Ubuntu 9.10 ("Karmic")开始, CouchDB在每个桌面版本上都已经预装了.

### Ubuntu ###

你可以通过运行下面的命令来安装CouchDB包:

				sudo aptitude install couchdb

当这个命令完成后, 你应该就有一个可以运行的CouchDB了. 请仔细阅读与Ubuntu相关的系统文档, 它们可以在/usr/share/couchdb目录下找到. 

### Gentoo Linux ###

Enable the development ebuild of CouchDB by running:
通过下面的命令启用CouchDB的development ebuild:

				sudo echo dev-db/couchdb >> /etc/portage/package.keywords

取得CouchDB的ebuild:

				emerge -pv couchdb

编译安装CouchDB ebuild:

				sudo emerge couchdb

When this completes, you should have a copy of CouchDB running on your machine.

当这个命令完成后, 你应该就有一个可以运行的CouchDB了.

### 可能遇到的问题 ###

如果你的发行版本不包含CouchDB包, 请查看附录D, 从源代码安装.
