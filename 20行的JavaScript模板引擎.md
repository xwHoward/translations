# 20行的JavaScript模板引擎

*原文链接[http://krasimirtsonev.com/blog/article/Javascript-template-engine-in-just-20-line](http://krasimirtsonev.com/blog/article/Javascript-template-engine-in-just-20-line)*

我（作者 Krasimir）一直在完善基于JavaScript的预处理器 - [AbsurdJS](http://krasimirtsonev.com/blog/article/AbsurdJS-fundamentals)。它最开始只是一个CSS预处理器，但后来被扩展为CSS / HTML预处理器。很快，它实现了从JavaScript到CSS / HTML的转换。当然，正因为它被用来生成HTML代码，所以自然就有模板引擎的功能，也就是用数据填充HTML。

所以，我想写一段简单的模板引擎逻辑，它与我目前的成果很好地契合。AbsurdJS主要作为NodeJS模块分发，但也被移植到了客户端。考虑到这一点，我知道我不能从一些现有的模板引擎获得支持。这是因为它们中的大多数只支持Node环境，在浏览器中难以复用它们。我需要一轻量，且用纯粹的JavaScript编写的代码。最后我发现了[John Resig](http://ejohn.org/)的这篇[博文](http://ejohn.org/blog/javascript-micro-templating/)，它看起来正是我需要的。我做了点改动，精简到20行内。我认为这段代码是非常有趣的。在本文中，我将逐步重新创建模板引擎，以便你可以看到最初来自John Resig的绝妙idea。

一开始我们有这样的一段代码：

```js
var TemplateEngine = function(tpl, data) {
    // magic here ...
}
var template = '<p>Hello, my name is <%name%>. I\'m <%age%> years old.</p>';
console.log(TemplateEngine(template, {
    name: "Krasimir",
    age: 29
}));
```

很简单的一个函数，接收模板template和数据对象data作为参数，很明显，我们想要得到这样一个输出：

```js
<p>Hello, my name is Krasimir. I'm 29 years old.</p>
```

第一件事需要取到模板里的动态代码块，然后我们用传递进来的数据来替换这些代码块。我决定用正则来实现这个操作，正则不是我的擅长项，如果你有更好的正则建议，欢迎提出改进。

```js
var re = /<%([^%>]+)?%>/g;
```

我们先提取所有的包裹在```<%```和```%>```中的片段，正则匹配的原生方法很多，这里我们需要一个结果数组，所以我们用[```exec```](http://www.w3schools.com/jsref/jsref_regexp_exec.asp)。

```js
var re = /<%([^%>]+)?%>/g;
var match = re.exec(tpl);
```

console.log结果：

```js
[
    "<%name%>",
    " name ",
    index: 21,
    input:
    "<p>Hello, my name is <%name%>. I\'m <%age%> years old.</p>"
]
```

返回的数组只有一个匹配元素，所以我们需要在while循环里处理所有的匹配结果：

```js
var re = /<%([^%>]+)?%>/g, match;
while(match = re.exec(tpl)) {
    console.log(match);
}
```

运行上面的代码就会发现```<%name%>```和```<%age%>```被打印出来了。
有意思的地方来了，我们需要用参数data里的真实数据来替换这些片段，最简单的方式就是用repalace()，就像这样：

```js
var TemplateEngine = function(tpl, data) {
    var re = /<%([^%>]+)?%>/g, match;
    while(match = re.exec(tpl)) {
        tpl = tpl.replace(match[0], data[match[1]])
    }
    return tpl;
}
```

OK，这样可以达到我们想要的效果，但是还不够。我们的参数```data```非常简单，所以可以使用```data["property"]```的方式，但在实践中我们可能会遇到复杂的嵌套对象，比如像下面这样：

```js
{
    name: "Krasimir Tsonev",
    profile: { age: 29 }
}
```

遇到这种情况前述的代码就有问题了，当模板中存在```<%profile.age%>```的时候我们就会取到```data["profile.age"]```，就会在页面中显示undefined，因此replace()就不适用于我们的方案。能够在```<%```和```%>```中放置真实的JavaScript代码片段是最完美的方式，如果这些片段是基于传递进来的data上下文是最好的(if it is evaluated against the passed data)，举个例子：

```js
var template = '<p>Hello, my name is <%this.name%>. I\'m <%this.profile.age%> years old.</p>';
```

如何实现呢？John利用了new Function这种语法来实现，也就是从字符串构造一个函数，下面是一个简单的例子：

```js
var fn = new Function("arg", "console.log(arg + 1);");
fn(2); // 输出 3
```

fn是一个接收两个参数的函数实例，函数体是```console.log(arg + 1);```。上面的代码等价于：

```js
var fn = function(arg) {
    console.log(arg + 1);
}
fn(2); // 输出 3
```

从简单的字符串构造函数，就是我们所需要的。在构造函数之前我们需要先构造函数体，并且这个函数需要返回编译后的模板字符串。我们用之前步骤的字符串来设想一下最终返回的字符串应该是什么样：

```js
return "<p>Hello, my name is " + this.name + ". I\'m " + this.profile.age + " years old.</p>";
```

当然，我们需要事先分解模板为普通文本和JavaScript代码片段，像上面一样用简单的字符串拼接的方式并不能100%契合我们的需求，因为最终我们是要做基于真实JavaScript代码的模板引擎，我们需要有循环：

```js
var template = 'My skills:'
+ '<%for(var index in this.skills) {%>'
+ '<a href=""><%this.skills[index]%></a>'
+ '<%}%>';
```

如果使用字符串拼接的方式，结果就是下面这样：

```js
return
'My skills:'
+ for(var index in this.skills) {
+ '<a href="">'
+ this.skills[index]
+ '</a>'
+ }
```

这肯定会抛错，所以我用来自John的博文里的这段逻辑，即将待拼接的字符串存到数组中，最后再来```join```。

```js
var r = [];
r.push('My skills:');
for(var index in this.skills) {
r.push('<a href="">');
r.push(this.skills[index]);
r.push('</a>');
}
return r.join('');
```

接下来的步骤是收集自定义生成函数的不同行。我们已经从模板中提取了一些信息。我们知道占位符的内容及其位置。所以，通过使用帮助变量（cursor），我们能够产生期望的结果。

```js
var TemplateEngine = function(tpl, data) {
    var re = /<%([^%>]+)?%>/g,
        code = 'var r=[];\n',
        cursor = 0, match;
    var add = function(line) {
        code += 'r.push("' + line.replace(/"/g, '\\"') + '");\n';
    }
    while(match = re.exec(tpl)) {
        add(tpl.slice(cursor, match.index));
        add(match[1]);
        cursor = match.index + match[0].length;
    }
    add(tpl.substr(cursor, tpl.length - cursor));
    code += 'return r.join("");'; // <-- return the result
    console.log(code);
    return tpl;
}
var template = '<p>Hello, my name is <%this.name%>. I\'m <%this.profile.age%> years old.</p>';
console.log(TemplateEngine(template, {
    name: "Krasimir Tsonev",
    profile: { age: 29 }
}));
```