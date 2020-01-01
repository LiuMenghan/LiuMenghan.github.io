---
title: 如何写一个易扩展的模版引擎（1）：设计
tags:
- node.js
- 模版引擎
categories:
- javascript
---
作者：寒歌

项目地址：[轻量级高可扩展静态模板引擎Tirpitz](https://github.com/LiuMenghan/Tirpitz)

# 前言

这是一个四年前的项目，起因是当时跟风学习node.js，但是当了解到当时node.js单线程异步的奇怪设计之后，我就放弃了在生产环境服务端使用node.js的想法（就像现在我看到Rust不支持反射，就放弃了用Rust写业务逻辑的想法）。但是node.js不适合在大型互联网公司生产环境服务端使用，并不意味着node.js这门技术就没有合适的使用场景。node.js由于使用了javascript作为编程语言，非常适合作为前端转型全栈工程师的中间环节，一些重心在前端但是需要服务端技术支持的功能都非常适合用node.js开发，比如：

- 以展示为主、轻后端重前端的小型网站；
- 前端开发接口Mock;
- 模版引擎。

几个里面相对来说看上去比较有挑战性的就是模版引擎，所以就打算自己写个模版引擎来练练手。在阅读了FreeMarker、Velocity、Thymeleaf、StringTemplate、AngularJS等一些列前后端模版引擎的使用文档和源码之后，对模版引擎有了以下总结：

1. 模板引擎是用来生成文档的工具；
2. 大多数人对文档的理解都是一种类似目录的树形结构；
3. 模板引擎最大的作用是代替程序员进行重复劳动（如banner、Copyright）,让程序员可以更关注于整体结构和特殊的内容细节；
4. 性能其实没那么重要，如果由模版引擎实时生成文档难以满足生产环境的需求，最终不可避免的会改为预先由模版引擎生成静态文档、生产环境直接使用静态文档的技术方案，这种情况下模版引擎的性能只需要满足开发调试的需求就可以了；
5. 扩展性很重要，缺乏扩展性（无法扩展，或者开发扩展插件的代价较高）会促使程序员使用各种各样奇怪的hacker技术，反过来由于开源社区的反馈机制又会使模版引擎朝着奇怪的方向发展，最终使模版引擎难以使用。

所以最终定下的目标是基于node.js开发一个高度可扩展，但是性能（空间、时间）没那么好的模版引擎。

*由于编程习惯，设计上是按照强类型设计的。由于javascript是弱类型，所以部分代码通过java来进行描述，但实际都是通过弱类型的javascript实现的。*

# 总体设计

模版语言与普通程序语言不同，相对缺少时间上的先后顺序概念。大多数模版引擎以标签方式标记模版逻辑，每个模板文件的渲染过程都可以分为两步：

1. 将模板文件解析成一棵标签树；
2. 遍历每个标签节点并执行对应标签逻辑。

> 模版文件 -> 标签树 -> 遍历标签树处理标签逻辑 -> 生成文档

## 主要扩展点

上面四个节点，最好都具有一定的可扩展性。

模版文件是用户自己编写的，自然是容易扩展的。

标签树作为一种数据结构，应该允许存储用户自定义的一些参数。

用户应该可以方便的自定义标签及标签处理逻辑，标签需要有一些和标签处理逻辑相关的属性，同时用户应该也可以自定义标签符号、转义符号、属性的赋值符号。

由于文档用处不同（如输出到文件/控制台/http流），由处理后的标签树生成文档这步应该也是可以由用户根据需求自定义的。

## AOP扩展点

模版引擎处理流程中的三个箭头都可以作为AOP扩展点。

模版文件到标签树的切面，可以用来自定义添加标签，应作为一个扩展点保留。

标签树到遍历标签树处理标签逻辑的切面，实现的流程上应该是：

> 遍历标签树 -> 找到满足切面条件的扩展点 -> 执行AOP逻辑

这个逻辑与自定义标签的处理逻辑几乎完全相同，所以就不需要保留了。

处理标签逻辑到生成文档的切面，可以用来做一些最终文本的处理工作(如敏感词过滤)，应作为一个扩展点保留。

这样一共增加了两个扩展点，分别为前置拦截器和后置拦截器，处理流程变成下面这个样子：

> 模版文件 -> 前置拦截器 -> 标签树 -> 遍历标签树处理标签逻辑 -> 后置拦截器 -> 生成文档

# 详细设计

## 模版文件的标签设计

模板文件的内容包括两部分：普通文本和标签。标签的结构设计最好方便标签树的生成，同时标签需要存储一些和标签处理逻辑相关的属性。这种思路典型的实现就是XML格式。XML的标签有三组：

- `<`和`>`组成开始标签；
- `</`和`>`组成结束标签，和开始标签成对出现;
- `<`和`/>`组成独立标签。

鉴于模版文件肯定会广泛应用于类XML的HTML文件生成，所以最好不要直接使用XML的标签定义，模仿一下，修改成:

```
exports.starterTag = ["{%", "%}"];	//开始标签
exports.endTag = ["{%/", "%}"];	//结束标签
exports.singleTag = ["{%", "/%}"];	//独立标签
```

虽然标签的属性通常是结构化的，但是非结构化的需求可能也潜在存在，同时考虑到方便node.js解析，所以标签属性定义成json形式。

最终设计出来的标签是这个样子的：
```
{%variable={"key":"date"} /%}
```

## 前置拦截器

前置拦截器的输入和输出应该一致，都是字符串。前置拦截方法定义为：

```
String before(String context);
```

前置拦截器的执行顺序应该由用户自己决定。

## 解析器

由于标签参考了XML设计，实现上可以参考XML的解析设计。具体比较复杂，下一章单独讨论。

## 模版标签树节点

节点是模版引擎最重要的数据结构。从标签定义可以看出，至少需要有两个属性：标签名称和属性。同时，作为一棵树，考虑到遍历需求，需要有子节点数组、父节点引用。最后，这棵树中最多的节点肯定是普通文本节点，所以最好添加一个`text`属性保存文本。

模板标签树节点应该定义成这样:

```
class TemplateNode{
	String processorName;
	String text;
	TemplateNode parent;
	TemplateNode[] children;
	Map<String, Object> attribute;
}
```
## 标签处理器

最常见的标签毫无疑问是纯文本标签，通过将标签名称默认为空，不进行任何处理，就实现了纯文本标签的处理逻辑。其它标签处理器的抽象是其实是一件比较困难的事情，后面关于标签处理器的定义可能还要修改。开发的时候从常用标签处理器着手，实现了两个最常用的标签处理器：插值和继承。

### 插值处理器

插值处理器的作用是将变量替换为外部配置好的值，那么输入参数至少有两个：当前节点和外部字典表。根据`node.attribute.key`的值从字典表中取出对应的值并替换text。

```
exports.processorName = "variable";
exports.process = function(node, tplPath, parser, variables){

	var key = node.attribute.key;
	if(key !=  undefined && null != key){
		var value = variables[key];
		node.text = value == undefined || null == value ? "" : value + "";
	}
	node.processorName = "";
}
```

处理结束后将标签名称赋值为空，标识着标签处理逻辑结束，标签转换为纯被文本标签。

### 继承处理器

继承处理器比较复杂，它允许两个模版文件按一定规则组合起来，比如父模版文件为:
```
Super head
{%replacable={"id":"replacer0"} /%}
Super body
{%replacable={"id":"replacer1"} /%}
Super bottom
```
子模版文件为：
```
Child text before super.
{%super={"parent":"./super.tirpitz"}%}
	{%override={"id":"replacer0"}%}Text in child replacer0.{%/override%}
	{%override={"id":"replacer1"}%}Text in child replacer1.{%/override%}
{%/super%}
Child text after super.
```
子模版文件中的`super`块会被替换成父模版文件的内容，而且父模版文件当中的`replacable`块会被替换成和子模版文件中`override`块`id`一致的内容。编译子模版的输出内容如下：
```
Child text before super.
Super head
Text in child replacer0
Super body
Text in child replacer1.
Super bottom
Child text after super.
```
`super`标签处理器的参数至少需要有三个：

- 当前节点；
- 当前解析模版文件路径（用于查找父模版）；
- 解析器（用于解析父模版)

处理过程如下（其中`Util.traverseNodeTree`是一个深度遍历树的工具方法）：

1. 获取可以替换的`override`子节点;
2. 获取并解析父模版；
3. 将父模版中的`replacable`子节点替换为`override`节点；
4. 将当前节点的子节点替换为父模版节点的子节点；
5. 将标签名称赋值为空，标识着标签处理逻辑结束，标签转换为纯被文本标签。。

```
var Util = require('../util.js');
exports.processorName = "super";
exports.process = function(node, tplPath, parser, variables){	
	var children = node.children;
	var replacer = {};
	for(var i = 0; i < children.length ; i++){
		var child = children[i];
		Util.traverseNodeTree(child, tplPath, function(node, superPath){
			if("override" == node.processorName){
				replacer[node.attribute.id] = node;
			}else if("super" == node.processorName){
				node.attribute.parent = Path.resolve(Path.dirname(tplPath), node.attribute.parent);
			}
		});
	}
	
	//获取父模版
	var superPath = Path.resolve(Path.dirname(tplPath), node.attribute.parent);
	console.log(superPath);
	var superContext = Fs.readFileSync(superPath, parser.encoding);
	var superNode = parser.parse2node(superContext, superPath);
	
	Util.traverseNodeTree(superNode, superPath, function(node, superPath){
		if("super" == node.processorName){
			var relativePath = node.attribute.parent;
			//路径改为绝对路径，以解决多级嵌套的问题
			node.attribute.parent = Path.resolve(Path.dirname(superPath), relativePath);
		}else if(node.processorName == "replacable"){
			//替换
			if(undefined != replacer[node.attribute.id]){
				node.processorName = "";
				node.children = replacer[node.attribute.id].children;			
			}else{
				node.processorName = "";				
			}			
		}
	});
	node.parentNode.children[node.parentIdx].children = superNode.children;
	node.parentNode.children[node.parentIdx].processorName = "";
}
```

## 后置拦截器

后置拦截器的输入是标签树根节点，因为多个后置拦截器都是在同一棵树上进行连续操作，所以不需要返回值。后置拦截方法定义为：

```
void after(TemplateNode root);
```

后置拦截器的执行顺序应该由用户自己决定。

## 生成文档

生成文档的过程实际上通过深度遍历树，将各节点的`text`值输出到控制台/文件/http response的过程。生成文档的过程也应该可以由用户自己定义，以输出到控制台为例：
```
exports.handle = function(node, tplPath, logger, starttime){
	var wholeText = "";
	util.traverseNodeTree(node, logger, function(textNode, logger){
		wholeText += textNode.text;
	});
	logger.log(wholeText);
	var endtime = new Date().getTime();			
	console.log("Parse " + tplPath + " to console.(" + (endtime - starttime) + " milliseconds)");
}
```

# 封装及文档撰写

模板解析器里面可以由用户自定义的部分很多，加载用户自定义的内容应遵循“约定优于配置”的规则。用户使用大体步骤分两步：

1. 初始化模板引擎;
2. 通过引擎渲染模板文件。

常用配置可以通过在初始化模版引擎的时候配置，并在文档中注明，方便新手用户使用。

项目地址：[轻量级高可扩展静态模板引擎Tirpitz](https://github.com/LiuMenghan/Tirpitz)
