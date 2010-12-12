## 存储文档 ##

Documents are CouchDB’s central data structure. To best understand and use CouchDB, you need to think in documents. This chapter walks you though the lifecycle of designing and saving a document. We’ll follow up by reading documents and aggregating and querying them with views. In the next section, you’ll see how CouchDB can also transform documents into other formats.
文档是CouchDB的中心数据结构. 想要最好的理解和使用CouchDB, 你需要以文档的方式来思考. 这一章节

Documents are self-contained units of data. You might have heard the term record to describe something similar. Your data is usually made up of small native types such as integers and strings. Documents are the first level of abstraction over these native types. They provide some structure and logically group the primitive data. The height of a person might be encoded as an integer (176), but this integer is usually part of a larger structure that contains a label ("height": 176) and related data ({"name":"Chris", "height": 176}).

How many data items you put into your documents depends on your application and a bit on how you want to use views (later), but generally, a document roughly corresponds to an object instance in your programming language. Are you running an online shop? You will have items and sales and comments for your items. They all make good candidates for objects and, subsequently, documents.

