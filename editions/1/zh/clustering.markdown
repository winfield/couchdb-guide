## 集群 ##

好了, 你已经看到本书的这里了. 我相信你或多或少已经理解什么是CouchDB以及它的API是怎么用的了. 或许你已经部署了一两个应用, 并且已经有需要处理相当多的流量以至于开始考虑扩展的问题了. "扩展"是一个很不精确的词, 但是在本章节中, 我们将要处理的是, 如何创建一个分区的或者分片的集群. 这个集群会在上线后以某个速率增长.

我们将会来观察一个拥有稳定节点的CouchDB集群请求与响应的分发. 然后我们会讲解如何来增加多余的热备节点, 这个就不用担心某些机器崩溃会导致的问题了. 在一个大型的集群中, 你应该准备好会有5-10%的机器可能会崩溃或者经历性能低下, 所以集群的设计必须要阻止某些节点的崩溃会影响可靠性. 最后, 我们会来看看如何通过使用复制来分割或者合并节点, 从而动态的进行集布局的调整

### 介绍CouchDB Lounge ###

CouchDB Lounge是一个建立于代理之上的用于分区和建立集群的应用, 它最初是由Meebo开发出来的, Meebo是一个基于web的即时消息服务. Lounge有两个主要的组件: 一个用来处理简单的文档GET和PUT请求, 而另一个则用于处理视图的请求.

dubmproxy会处理任何不是CouchDB视图请求的简单请求. 它作为nginx的一个模块存在, nginx是一个高性能的反向HTTP代理. 得益于反向HTTP代理的工作方式, 这使得很多东西都变得可配置化, 安全性, 加密, 负载分发, 压缩, 当然还有数据库资源的大量缓存.

smartproxy则只会处理CouchDB视图请求, 并且会把它们分发到集群中的所有节点, 使集群的视图处理变得更加高效. 它作为Twisted的一个守护进程存在, Twisted是一个用Python写的高性能的事件驱动型网络编程框架.

### 一致性哈希处理 ###

CouchDB的存储模型使用唯一的ID来保存和读取文档. Lounge的核心有一个简单的方法来对文档ID进行哈希处理. 接着, Lounge会使用哈希的前几个字符来决定请求要分发到哪里去. 你可以通过编写一个共享匹配(shared map)来配置其行为, 它只是一个简单的文本配置文件.

因为Lounge会分配给每个节点一部分的哈希(这一部分的哈希被称为keyspace), 所以你可以加入任意数量的节点. 因为哈希函数产生的16进制字符串和文档ID完全没有关系, 而我们又是通过这些哈希的前几个字符来进行的请求分发, 所以可以保证所有的节点都会有相同的负载. 而且, 哈希函数拥有一致性, Lounge从HTTP请求URI中得到任意的文档ID都会生成同样的哈希, 所以每次都会指向同一个节点.

This idea of splitting a collection of shards based on a keyspace is commonly illustrated as a ring, with the hash wrapped around the outside. Each tic mark designates the boundaries in the keyspace between two partitions. The hash function maps from document IDs to positions on the ring. The ring is continuous so that you can always add more nodes by splitting a single partition into pieces. With four physical servers, you allocate the keyspace into 16 independent partitions by distributing them across the servers like so:

如果文档ID的哈希以0开头, 那么它就会被分发到分片A. 类似的, 以1, 2, 3开头的也是如此. 然而, 如果哈希以c, d, e或者f开头, 那么它就会被分发到分片D. 作为一个完整的例子, 哈希71db329b58378c8fa8876f0ec04c72e5会被分配到上面表格中节点B的数据库7. 在后端集群中, 你可以把它分配到http://B.couches.local/db-7/类似的地址. 通过这种方式, 哈希表就只是一个哈希到后端数据库URI的映射关系了. 如果这些听起来还是很复杂, 别急; 你所要做的只是提供一个分片到节点的映射, Lounge会合理创建哈希圈--所以如果你不想就没有必要去干这些脏活了.

因为CouchDB使用HTTP协议, 要在web架构上实现相同的概念, 代理服务器可以根据请求URL来对文档进行分区, 而不需要查看请求体. 这个REST架构背后的一个核心原则, 也是HTTP协议提供给我们的众多好处中的一个. 在实际中, Lounge会对请求URI进行哈希处理, 再将其比对结果, 最后找出它所属的那一部分密钥空间. 然后, Lounge会在配置表格中查找这个哈希的关联分片, 再把HTTP请求转发到后端的CouchDB服务器

一致性哈希处理是一个简单的办法来保证你总是可以找到保存的文档, 即使是在跨分区的进行存储负载均衡的情况下. 因为哈希函数很简单(它建立于CRC32), 所以你可以实现自己的HTTP中间作或者客户端, 它们可以类似的把请求分发到数据所在的正确物理位置.

#### 冗余的存储 ####

一致性哈希处理解决了如何把一个单一逻辑数据库保存于一组分区的问题, 这些分区可以分布于多个服务器. 但它不讨论如何在硬件或软件故障时, 保证数据安全性的问题. 如果你很关心数据的安全性, 除非有至少两份的相同数据拷贝, 最好还是在不同的地理位置, 否则你是不会安心的.

CouchDB replication makes maintaining hot-failover redundant slaves or load-balanced multi-master databases relatively painless. The specifics of how to manage replication are covered in Chapter 16, Replication. What is important in this context is to understand that maintaining redundant copies is orthogonal to the harder task of ensuring that the cluster consistently chooses the same partition for a particular document ID.
CouchDB的复制功能使得维护hot-failover冗余slaves或者multi-master负载均衡变得相对不那么痛苦. 关于如何管理复制的细节在第16章, 复制中有详细讲解. 在这里, 重要的是理解

For data safety, you’ll want to have at least two or three copies of everything. However, if you encapsulate redundancy, the higher layers of the cluster can treat each partition as a single unit and let the logical partitions themselves manage redundancy and failover.
为了数据安全, 你可以会想要拥有至少两到三份的所有数据的拷贝. 然而, 如果你封装了冗余, 集群的更上层可能会把每一个分区都当成是一个单一的单元, 并且

#### 冗余的代理 ####

就像我们因为不能接受硬件故障可能导致的数据丢失, 我们同样需要运行多个代理节点的实例, 来避免因为某个代理节点崩溃而导致集群的部分不可用. 通过运行冗余的代理实例, 并且在它们之间作负载均衡, 我们可以增加集群的处理能力和可靠性.

#### 视图合并 ####

一致性哈希可以使得文档处于合适的节点, 但文档仍然可以用emit()函数产生任意的key. 增量MapReduce的关键在于把这种能力带到数据层面, 所以我们不应该重复分配产生的key; 而是应该通过HTTP代理来发送查询到CouchDB节点, 然后再使用Twisted Python Smartproxy合并结果.

Smartproxy会发送视图请求到每一个节点, 所以它需要在返回给客户端前把响应数据进行合并. 谢天谢地, 这个操作不是一个很耗资源的操作, 因为合并可以在固定内存空间里完成, 不管有多少记录返回了. Smartproxy会接收每个节点的第一条记录并比较它们. 我们使用CouchDB的校对规则, 根据节点返回记录的key来对它们进行排序. Smartproxy会取出排序节点中的最上面一条记录, 然后把它返回给客户端.

这个过程可以重复进行, 只要客户端还继续要求更多的记录, 但是如果客户端加上了一个限制条件, 那么Smartproxy就必须提早终止响应, 扔掉任何多余的节点记录.

这种布局简单而松散. 我的优势在于其简单性, 这有利于理解拓扑结构及故障判断. 我们正在把这个工作转到Erlang上, 这样就可以管理动态的集群, 并且可以把集群控制集成到CouchDB运行时.

#### Growing the Cluster ####

在web扩展方面使用CouchDB很可能会需要CouchDB集群进行动态的扩展. 做大网站必须连结的增加存储能力, 所有我们需要一个策略, 在不下线网站的情况下, 增加集群的大小.
 有些操作可能会导致数据量的临时增大, 这种情况下, 我们还需要一个过程来缩减集群而不中断服务.

在这一部分里, 我们会看到如何使用CouchDB的复制过滤器来把一个数据库分割到几个分区, 以及如何使用这项技术来在不下线的情况下增长集群. 你可以使用几个简单的步骤来避免增长集群时的数据库分区.

Oversharding这项技术的意思是: 对集群进行分区, 使得在每个物理节点上都有多个分片. 要把一个分区从一台机器移动到另一台, 要比把它分割为更小的分区来得简单, 因为代理服务器所使用的集群映射配置只需要改变指向的分片的位置就行了, 而不需要增加新的逻辑分片. 移动分区也比分割成更小的多个分片占用更少的资源.

有一个我们需要回答的问题是, "我们应该overshard多少?" 答案取决于你的应用程序和部署方式, but there are some forces that push us in one direction over another. 如果我们找出了正确的所需分片数量, 就会得到一个最优化的增长集群.

在"视图合并"一节里, 我们讨论了合并是如何在一个固定空间里完全的, 不管返回有多少记录. 然而, 用于合并视图的内存空间和网络资源, 以及文档ID到分区的映射, 的的确确是随特定代理下的分区线性增长的. 因为这个原因, 我们想要限制每个代理下的分区数量. 然而, 我们不能接受分区大小有一个上限. 解决方法是使用一个代理树, 根代理分区到一些中间代理, 这些中间代理再映射到数据库节点.

决定每个代理需要管理多少个分区的因素有: 每个独立服务器节点的存储大小, 项目的数据增长速率, 代理可用的网络和内存资源, 以及集群可接受的延时.

Assuming a conservative 64 shards per proxy, and 1 TB of data storage per node (including room for compaction, these nodes will need roughly 2 TB of drive space), we can see that with a single proxy in front of CouchDB data nodes, we’ll be able to store at maximum 64 TB of data (on 128 or perhaps 192 server nodes, depending on the level of redundancy required by the system) before we have to increase the number of partitions.
保守的假设每个代理有64个分片, 每个节点有1TB的数据存储空间(如果包括压缩所需要占用的空间, 这些节点大约会需要2TB的空间). 那么我们可以发现, 一个处于CouchDB数据节点的代理, 能够存储最大64TB的数量(在)

By replacing database nodes with another proxy, and repartitioning each of the 64 partitions into another 64 partitions, we end up with 4,096 partitions and a tree depth of 2. Just as the initial system can hold 64 partitions on just a few nodes, we can transition to the 2-layer tree without needing thousands of machines. If we assume each proxy must be run on its own node, and that at first database nodes can hold 16 partitions, we’ll see that we need 65 proxies and 256 database machines (not including redundancy factors, which should typically multiply the cluster size by two or three times). To get started with a cluster that can grow smoothly from 64 TB to 4 PB, we can begin with roughly 600 to 1,000 server nodes, adding new ones as data size grows and we move partitions to other machines.
通过把数据库节点换成另外一个代理服务器, 并且把这64个分区每个再分区为64个分区, 我们就得了一个有4096个分区, 深度为2的树. 

我们已经看过了, 只是一个深度为2的集群就可以存储巨大量的数据了. 简单的算法就可以告诉我们, 通过相同的方法创建一个3层代理的集群, 我们就能够在万千上万台机器上管理262拍的数据. 保守的估计每层会产生的延时是100ms, 这样即使没有做性能优化, 即便是在一个3层的树里, 总体上我们还能拥有300ms的响应时间. 并且我们还是应该能够在小于1秒的时间内, 在巨大的数据集里管理查询.

通过使用oversharding和迭代的把完整分片(只有一个分区的那些数据库节点)替换为指向另一组oversharding分区的代理节点, 我们可以在保持最小延时的同时, 把集群增长到一个非常大的规模.

Now we need to look at the mechanics of the two processes that allow the cluster to grow: moving a partition from an overcrowded node to an empty node, and splitting a large partition into many subpartitions. Moving partitions is simpler, which is why it makes sense to use it when possible, running the more resource-intensive repartition process only when partitions get large enough that only one or two can fit on each database server.
现在我们需要来看看那两个使得集群可以增长的过程的机制了: 把一个分区从一个过于拥挤的节点移动到一个空节点, 以及把一个大分区分割成为许多小的子分区. 移动分区更简单些, 这也为什么移动如果需要就应该这么做, 而运行更加消耗资源的再分区进程只应该在分区过于庞大, 以至于只有一两个.....的原因

#### 移动分区 ####

就像我们先前提到的那样, 每个分区是由N个冗余CouchDB数据库组成的, 每个CouchDB数据库话在不同的物理服务器上. 为了更加概念化, 假设任何操作都应该会被自动的作用于所有的冗余备份. 为了讨论方便, 我们只讲抽象的分区, 但你应该了解冗余节点拥有同样的大小, 所以在分区进行增长时需要同样的操作.

把一个分区从一个节点移动到另一个的最简单的方法是, 在目标节点上创建一个空的数据库, 然后使用CouchDB的复制功能把老节点数据复制到新节点. 当新分区和原分区一样新后, 代理节点可以重新配置, 指向到新机器. 代理节点指向新分区后, 再进行最后一次的复制, 确保新分区和原分区数据相同, 之后, 老的分区就可以退休了, 可以把原来那台机器上的空间释放掉.

Another method for moving partition databases is to rsync the files on disk from the old node to the new one. Depending on how recently the partition was compacted, this should result in efficient, low-CPU initialization of a new node. Replication can then be used to bring the rsynced file up-to-date. See more about rsync and replication in Chapter 16, Replication.
另一个移动分区数据库的方法是通过rsync, 把文件从老节点备份到新的上面. 

#### 分割分区 ####

The last major thing we need to run a CouchDB cluster is the capability to split an oversized partition into smaller pieces. In Chapter 16, Replication, we discussed how to do continuous replication using the _changes API. The _changes API can use filters (see Chapter 20, Change Notifications), and replication can be configured to use a filter function to replicate only a subset of a total database. Splitting partitions is accomplished by creating the target partitions and configuring them with the range of hash keys they are interested in. They then apply filtered replication to the source partition database, requesting only documents that meet their hash criteria. The result is multiple partial copies of the source database, so that each new partition has an equal share of the data. In total, they have a complete copy of the original data. Once the replication is complete and the new partitions have also brought their redundant backups up-to-date, a proxy for the new set of partitions is brought online and the top-level proxy is pointed at it instead of the old partition. Just like with moving a partition, we should do one final round of replication after the old partition is no longer reachable by the cluster, so that any last second updates are not lost. Once that is done, we can retire the old partition so that its hardware can be reused elsewhere in the cluster.

运行一个CouchDB集群需要的最后一件主要武器是分割过大分区的能力. 在第16章, 复制里, 我们讨论了如何使用_changes API来做连续复制. _changes API可以使用过滤器(请查看第20章, 变更通知), 复制可以配置一个过滤器函数, 来复制一个数据库总数据的一个子数据集. 分割分区是通过创建目标分区, 并把它们配置需要的一个哈希值范围来完成的. 然后, 它们运行经过过滤的复制, 从源分区中只复制那些满足它们要求的数据. 结果就是一个源数据的多个部分拷贝, 这样每个新分区都得到了同样大小的一份数据. 完全组成起来, 就是原来数据的一个完整拷贝. 当复制完成后, 新分区也完成了冗余备份后, 一个这些新分区的代理就可以上线了, 而最高层的代理则指这个新代理, 替代掉原来的分区. 就像移动一个分区那样, 我们应该在集群不再需要老分区后再做一次复制, 防止有潜在的更新丢失. 当这个复制完成后, 老的分区就可以退休了, 而它的硬件则可以被用于分区的其他地方.
