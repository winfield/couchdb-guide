## B-tree的威力 ##

CouchDB uses a data structure called a B-tree to index its documents and views. We’ll look at B-trees enough to understand the types of queries they support and how they are a good fit for CouchDB.
CouchDB使用了一个叫B-tree的数据结构来索引文档和视图. 
This is our first foray into CouchDB internals. To use CouchDB, you don’t need to know what’s going on under the hood, but if you understand how CouchDB performs its magic, you’ll be able to pull tricks of your own. Additionally, if you understand the consequences of the ways you are using CouchDB, you will end up with smarter systems.

If you weren’t looking closely, CouchDB would appear to be a B-tree manager with an HTTP interface.

