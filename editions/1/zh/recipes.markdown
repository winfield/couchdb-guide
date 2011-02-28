## Recipes ##

本章会展示一些一般的任务, 它们在CouchDB里处理的最佳实践. 以及它们是如何实现的easy-to-follow, step-by-step的指导.

### 银行 ###

银行的业务是重要的. 它们需要重要的数据库来存储重要的交易和重要的帐户信息. 它们不能丢失任何钱. 他们同时也不能造出钱来. 一个银行必须在所有时候都处于平衡状态.

传统的观点认为一个数据库需要支持事务才能算可以在重要的地方使用. CouchDB不支持传统意义上的事务(虽然它是以事务的方式工作的), 所以你会认为CouchDB并不适合于存储银行数据. 另外, 你会认为把钱放在沙发里安全吗? 好吧, 我们会. 本章会解释为什么.

### 会计不会使用橡皮擦 ### Accountants Don’t Use Erasers

Say you want to give $100 to your cousin Paul for the New York cheesecake he sent to you. Back in the day, you had to travel all the way to New York and hand Paul the money, or you could send it via (paper) mail. Both methods were considerably inconvenient, so people started looking for alternatives. At one point, banks offered to take care of the money and make sure it arrived at Paul’s bank safely without headaches. Of course, they’d charge for the convenience, but you’d be happy to pay a little fee if it could save a trip to New York. Behind the scenes, the bank would send somebody with your money to give it to Paul’s bank—the same procedure, but another person was dealing with the trouble. Banks could also batch money transfers; instead of sending each order on its own, they could collect all transfers to New York for a week and send them all at once. In case of any problems—say, the recipient was no longer a customer of the bank (remember, it used to take weeks to travel from one coast to the other)—the money was sent back to the originating account.

