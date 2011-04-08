## JSON初步 ##

CouchDB使用JavaScript Object Notation (JSON) 作为数据存储, 它是在JavaScript语法子集的基础上的一个轻量级的格式. JSON最好的一点就是它很容易阅读, 很容易手工来写, 比像XML之类的要简单的多得多. 我们可以很自然的用JavaScript来解析它, 因为它和JavaScript共用一部分的语法. 这在我们构建动态的web应用并且想要从服务器端获取数据时变得非常方便. 

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

可以看到一般的结构就是基于key/value对以及一些列表的.

### 数据类型 ###

JSON有几个基本的数据类型. 我们在这里把它们都讲解一下.

#### 数字 ####

可以是一个正整数: "Count": 253

或者负整数: "Score": -19

或者一个浮点数: "Area": 456.31

在浮点数和十进制小数之间有一个微妙的但却很重要的区别. 当你使用一个数字, 比如15.7时, 它会被大多数的客户端解析成为699999999999999, 这有时可能会导致问题. 因为这个原因. 货币值通常在JSON以字符串的形式表示. "15.7"这个字符串会被所有的JSON客户端解析为"15.7".

或者是科学计数法: "Density": 5.6e+24

#### 字符串 ####

可以使用字符串作为值:

				"Author": "Rusty"

有些特殊字符必须要转义, 比如tab或者newline:

				"poem": "May I compare thee to some\n\tsalty plankton."

JSON网站有关于哪些字符需要进行转义的细节.

#### Booleans ####

可以是为true:

				"Draft": true

或者是为false:

				"Draft": false

#### 数组 #### 

数组是一个值的列表:

				"Tags": ["plankton", "baseball", "decisions"]

数组可以包含其他任何数据类型, 包括数组:

				"Context": ["dog", [1, true], {"Location": "puddle"}]

#### 对象 ####

对象一个key/value对的列表:

				{"Subject": "I like Plankton", "Author": "Rusty"}

#### Nulls ####

你还可以使用null值:

				"Surname": null

