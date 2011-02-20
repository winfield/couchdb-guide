## 集群 ##

OK, you’ve made it this far. I’m assuming you more or less understand what CouchDB is and how the application API works. Maybe you’ve deployed an application or two, and now you’re dealing with enough traffic that you need to think about scaling. “Scaling” is an imprecise word, but in this chapter we’ll be dealing with the aspect of putting together a partitioned or sharded cluster that will have to grow at an increasing rate over time from day one.
好了, 你已经看到本书的这里了. 我相信你或多或少已经理解什么是CouchDB以及它的API怎么用了. 或许你已经部署了一两个应用, 并且已经有需要处理相当多的流量以至于开始考虑扩展性问题. "扩展性"是一个很不精确的词, 但是在本章节中, we’ll be dealing with the aspect of putting together a partitioned or sharded cluster that will have to grow at an increasing rate over time from day one.

We’ll look at request and response dispatch in a CouchDB cluster with stable nodes. Then we’ll cover how to add redundant hot-failover twin nodes, so you don’t have to worry about losing machines. In a large cluster, you should plan for 5–10% of your machines to experience some sort of failure or reduced performance, so cluster design must prevent node failures from affecting reliability. Finally, we’ll look at adjusting cluster layout dynamically by splitting or merging nodes using replication.
我们将会
