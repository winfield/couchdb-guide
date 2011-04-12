## 前言 ## 

感谢购买本书! 如果这是一份礼物, 那么恭喜你. 如果, 你是没付钱直接从网上下载的, 那么好吧, 其实, 我们对此同样感到很高兴. 本书是在一个自由许可协议下发行的, 这很重要, 因为我们想要把它作为社区的文档--而文档应当是免费的. 

那么, 为什么要付钱买一本免费的书呢? 额, 也许你喜欢舒服的坐在沙发上, 喝着咖啡, 把书本拿在手上阅读那种无法言传的感觉. 不管原因是什么, 购买本书都帮助, 支持了我们, 这让我们有了更多的时间可以来更好的改进这本书以及CouchDB. 所以, 非常感谢你!

我们打算把最好的最容易理解的关于CouchDB的信息编纂进本书, 而我们知道我们还是失败了. CouchDB还要快速的演进中, 并且在我们写作本书的时候还增加了一些重大的变化. 我们可以快速适应, 保持内容的更新, 但如果还想要本书可以出版的话, 我们还是需要在某一个地方划下一条分割线.

在写作本书的时候, CouchDB 0.10.1是最新的版本, 但是现在你可能已经看到了0.10.2, 甚至0.11.0, 或者可能1.0版本都已经出现了. 虽然我们知道未来的CouchDB版本会是什么样的, 但是我们对那些可能性并不确定, 也不想去做出任何大胆的猜测. CouchDB是一个开源社区项目, 所以最终, 它要由你--我们的读者--来帮助构建项目的未来.

从好的方面看, 很多人已经成功的在生产环境运行了CouchDB 0.10了, 所以CouchDB已经足够用来构建一个可靠的项目了. 未来版本的CouchDB会让事情变得更加简单, 但是其核心应当会保持不变. 此外, 学习其核心特性可以帮助你理解领会一些捷径, 从而使你可以想出适合自己的解决方案.

写作一本开放的书是一个极大的乐趣. 我们很高兴, O'Reilly用了各种可能的办法来支持我们的决定. 最好的一点--除了让CouchDB社区可以提早接触各种材料--是本书网站上的评论系统. 它使得任何人都可以在任何的段落上留下评论, 只需要简单的点击下鼠标. 我们使用了一些简单的JavaScript和Google Groups实现了简单的评论系统. 结果是惊人的. 截至今天, 已经有866人发送了越过1100条的消息到我们这个小讨论组. 各种提交范围很广泛, 从指出小的拼写错误, 到深度的技术讨论. 第一章节最初版本收到的反馈意见使我们完全把它重写了一遍, 为了确保想要表达的观点是真正的被表达出来了. 这个系统可以让我们用你--我们的读者--容易理解的方式, 进行清楚而系统的阐述.

总的来说, 因为有了数百个志愿者花费了他们时间来提交各种建议, 这本书已经好了很多. 我们明白这一模型所拥有的巨大价值, 所以想要继续保留它. CouchDB的新特性应该可以直接进入本书, 而不必需要每三个月重印一次. 出版业还没准备好这么做, 但我们想要继续发布新的修订后的内容, 并且仔细聆听各种反馈意见. 我们要如何做到这些的细节现在还没有定下来, 但我们知道后会马上把信息在本书的网站上公布出来. 这是一个承诺! 所以记得访问本书的网站http://books.couchdb.org/relax, 来获取最新的信息.

在深入本书之前, 我们想要确保你已经准备好了. CouchDB是用Erlang写的, 但是使用CouchDB本身并不需要了解任何关于Erlang的知识. CouchDB还大量的信赖于像HTTP和JavaScript这样的web技术, 这些技术的经历可以更好的帮助你理解书中所讲到的例子. 如果你创建过一个网站--不管简单还是复杂--那么你应该已经准备好了.

If you are an experienced developer or systems architect, the introduction to CouchDB should be comforting, as you already know everything involved—all you need to learn are the ways CouchDB puts them together. Toward the end of the book, we ramp up the experience level to help you get as comfortable building large-scale CouchDB systems as you are with personal projects.
如果你是一个资深的开发者或者系统架构师, CouchDB的介绍应该会让你觉得很舒服, 因为你已经知道所有涉及到的技术--所有你需要学习的只有CouchDB是如何把它们结合到一起的. 接近本书末尾时, 

如果你是一个web开发的初学者, 别着急--当你看到本书后面几部分的时候, 你应该已经可以跟上那些更难的内容了.

现在, 休息, 放松, 然后开始在CouchDB的美妙世界里寻找快乐吧.

### 致谢 ###

#### J.Chris ####

我想要感谢所有的CouchDB开发者, 那些提交补丁的人们, 以及社区里其他的人们. 没有我妻子, Amy, 帮我构思大体的框架; 没有本书的共同作者们和O'Reilly的耐心和支持; 没有所有那些在邮件列表上帮助我们推敲本书内容的人们, 我不可能完全这项工作. 我还要向编辑们高呼, 你们太棒了! 

#### Jan ####

I would like to thank the CouchDB community. Special thanks go out to a number of nice people all over the place who invited me to attend or talk at a conference, who let me sleep on their couches (pun most definitely intended), and who made sure I had a good time when I was abroad presenting CouchDB. There are too many to name, but all of you in Dublin, Portland, Lisbon, London, Zurich, San Francisco, Mountain View, Dortmund, Stockholm, Hamburg, Frankfurt, Salt Lake City, Blacksburg, San Diego, and Amsterdam: you know who you are—thanks!
我想要感谢CouchDB社区. 特别感谢

To my family, friends, and coworkers: thanks you for your support and your patience with me over the last year. You won’t hear, “I’ve got to leave early, I have a book to write” from me anytime soon, promise!

Anna, you believe in me; I couldn’t have done this without you.


