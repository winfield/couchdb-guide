## JSON Primer ##

CouchDB uses JavaScript Object Notation (JSON) for data storage, a lightweight format based on a subset of JavaScipt syntax. One of the best bits about JSON is that it’s easy to read and write by hand, much more so than something like XML. We can parse it naturally with JavaScript because it shares part of the same syntax. This really comes in handy when we’re building dynamic web applications and we want to fetch some data from the server.
CouchDB使用JavaScript Object Notation (JSON) 作为数据存储, 它是在JavaScript语法子集的基础上的一个轻量级的格式. JSON最好的一点就是它很容易阅读, 很容易手工来写, 比像XML之类的要简单的多得多. 我们可以自然的用JavaScript来解析它, 因为它和JavaScript共用一部分的语法. 这在我们构建动态的web应用并且想要从服务器端获取数据时变得非常方便, 

这是一个示例的JSON文档:

				{
						"Subject": "I like Plankton",
						"Author": "Rusty",
						"PostedDate": "2006-08-15T17:30:12-04:00",
						"Tags": [
								"plankton",
								"baseball",
								"decisions"
						],
						"Body": "I decided today that I don't like baseball. I like plankton."
				}

You can see that the general structure is based around key/value pairs and lists of things.

