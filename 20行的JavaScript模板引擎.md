# 20行的JavaScript模板引擎

*原文链接[http://krasimirtsonev.com/blog/article/Javascript-template-engine-in-just-20-line](http://krasimirtsonev.com/blog/article/Javascript-template-engine-in-just-20-line)*

我（作者 Krasimir）一直在完善基于JavaScript的预处理器 - [AbsurdJS](http://krasimirtsonev.com/blog/article/AbsurdJS-fundamentals)。它最开始只是一个CSS预处理器，但后来被扩展为CSS / HTML预处理器。很快，它实现了从JavaScript到CSS / HTML的转换。当然，正因为它被用来生成HTML代码，所以自然就有模板引擎的功能，也就是用数据填充HTML。

所以，我想写一段简单的模板引擎逻辑，它与我目前的成果很好地契合。AbsurdJS主要作为NodeJS模块分发，但也被移植到了客户端。考虑到这一点，我知道我不能从一些现有的模板引擎获得支持。这是因为它们中的大多数只支持Node环境，在浏览器中难以复用它们。我需要一段轻量，且用纯粹的JavaScript编写的代码。最后我发现了[John Resig](http://ejohn.org/)的这篇[博文](http://ejohn.org/blog/javascript-micro-templating/)，它看起来正是我需要的。我做了点改动，精简到20行内。我认为这段代码是非常有趣的。在本文中，我将逐步重新创建模板引擎，以便你可以看到最初来自John Resig的绝妙idea。

一开始我们有这样的一段代码：

```js
var TemplateEngine = function(tpl, data) {
    // magic here ...
}
var template = "<p>Hello, my name is <%name%>. I\'m <%age%> years old.</p>";
console.log(TemplateEngine(template, {
    name: "Krasimir",
    age: 29
}));
```

很简单的一个函数，接收模板```template```和数据对象```data```作为参数，很明显，我们想要得到这样一个输出：

```html
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

```console.log```结果：

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
有意思的地方来了，我们需要用参数data里的真实数据来替换这些片段，最简单的方式就是用```repalace()```，就像这样：
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

遇到这种情况前述的代码就有问题了，当模板中存在```<%profile.age%>```的时候我们就会取到```data["profile.age"]```，就会在页面中显示```undefined```，因此```replace()```就不适用于我们的方案。能够在```<%```和```%>```中放置真实的JavaScript代码片段是最完美的方式，如果这些片段是基于传递进来的```data```上下文是最好的(*if it is evaluated against the passed data*)，举个例子：

```js
var template = '<p>Hello, my name is <%this.name%>. I\'m <%this.profile.age%> years old.</p>';
```

如何实现呢？John利用了```new Function```这种语法来实现，也就是从字符串构造一个函数，下面是一个简单的例子：

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
var template = 'My skills:' +
    '<%for(var index in this.skills) {%>' +
    '<a href=""><%this.skills[index]%></a>' +
    '<%}%>';
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

接下来的步骤是收集我们自定义生成的函数的不同行。我们已经从模板中提取了一些信息，知道了片段的内容及其位置。所以，通过使用辅助变量（```cursor```），我们就能够得到期望的结果。

```js
var TemplateEngine = function(tpl, data) {
    var re = /<%([^%>]+)?%>/g,
        code = 'var r=[];\n',
        cursor = 0,
        match;
    var add = function(line) {
        code += 'r.push("' + line.replace(/"/g, '\\"') + '");\n';
    }
    while (match = re.exec(tpl)) {
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

```code```变量保存了函数体，它以数组定义开始。```cursor```变量表示当前模板字符串处理位置，我们需要它来遍历模板字符串并跳过代码片段。额外声明了一个```add```函数，作用是给```code```变量不断加入代码行。然后是比较tricky的写法，我们需要特殊处理双引号，否则生成的字符串形式的代码就有问题（注：这里为什么是```\\"```的写法，因为这里先替换```"```为```\\"```，后续的正则匹配时会将```\\```转义为```\```，这样code变量最终才会正确保留模板字符串中的```"```为```\"```）。运行这段代码，可以看到打印出：

```js
var r = [];
r.push("<p>Hello, my name is ");
r.push("this.name");
r.push(". I'm ");
r.push("this.profile.age");
return r.join("");
```

Hm...还不是我们想要的结果。```this.name```和```this.profile.age```不应该被处理为字符串，我们还需要修改一下```add```函数：

```js
var add = function(line, js) {
    js ? code += 'r.push(' + line + ');\n' :
        code += 'r.push("' + line.replace(/"/g, '\\"') + '");\n';
}
var match;
while (match = re.exec(tpl)) {
    add(tpl.slice(cursor, match.index));
    add(match[1], true); // <-- 表明这是一段js代码
    cursor = match.index + match[0].length;
}
```

这样代码片段传参时就会带着一个布尔变量，可以解析出正确的结果了：

```js
var r = [];
r.push("<p>Hello, my name is ");
r.push(this.name);
r.push(". I'm ");
r.push(this.profile.age);
return r.join("");
```

现在只需要用工厂模式构造一个函数并执行它就可以了，在我们的模板引擎函数的最后，不返回```tpl```了：

```js
return new Function(code.replace(/[\r\t\n]/g, '')).apply(data);
```

我们都不需要传递任何参数进去，而是使用```apply()```来调用，它会自动绑定作用域，这就是```this.name```能够正确替换数据的原因——```this```就指向```data```。
到这里我们的模板引擎就实现得差不多了，但还有最后一件事要做。我们需要支持更多复杂的操作，比如```if/else```条件和循环语句。我们用目前的成果来检验一下：

```js
var template =
    'My skills:' +
    '<%for(var index in this.skills) {%>' +
    '<a href="#"><%this.skills[index]%></a>' +
    '<%}%>';
console.log(TemplateEngine(template, {
    skills: ["js", "html", "css"]
}));
```

得到的结果是一个"Uncaught SyntaxError: Unexpected token for"的错误，我们打印一下```code```变量就能看出问题出在哪儿：

```js
var r = [];
r.push("My skills:");
r.push(
    for (var index in this.skills) {); r.push("<a href=\"\">"); r.push(this.skills[index]); r.push("</a>"); r.push(
    });
r.push("");
return r.join("");
```

包含```for```循环的代码行不应该被```push```到数组里，而是应该直接在函数体内执行，我们需要在将函数体代码连接到```code```变量前再做一次检查。

```js
var re = /<%([^%>]+)?%>/g,
    reExp = /(^( )?(if|for|else|switch|case|break|{|}))(.*)?/g,
    code = 'var r=[];\n',
    cursor = 0;
var add = function(line, js) {
    js ? code += line.match(reExp) ? line + '\n' : 'r.push(' + line + ');\n' :
        code += 'r.push("' + line.replace(/"/g, '\\"') + '");\n';
}
```

新增了一个正则，如果JavaScript片段以```if, for, else, switch, case, break, {```或者```}```开头，就直接将这个片段作为函数体执行，否则就作为字符串```push```到数组中。现在得到的结果是这样的：

```js
var r = [];
r.push("My skills:");
for (var index in this.skills) {
    r.push("<a href=\"#\">");
    r.push(this.skills[index]);
    r.push("</a>");
}
r.push("");
return r.join("");
```

这下我们的模板引擎就可以正确编译出结果了：

```html
My skills:<a href="#">js</a><a href="#">html</a><a href="#">css</a>
```

最后的这个改进让我们的模板引擎有了可以处理复杂逻辑的能力，举个例子：

```js
var template =
    'My skills:' +
    '<%if(this.showSkills) {%>' +
    '<%for(var index in this.skills) {%>' +
    '<a href="#"><%this.skills[index]%></a>' +
    '<%}%>' +
    '<%} else {%>' +
    '<p>none</p>' +
    '<%}%>';
console.log(TemplateEngine(template, {
    skills: ["js", "html", "css"],
    showSkills: true
}));
```

再做一点小调整，最终版本的模板引擎就完成了：

```js
var TemplateEngine = function(html, options) {
    var re = /<%([^%>]+)?%>/g,
        reExp = /(^( )?(if|for|else|switch|case|break|{|}))(.*)?/g,
        code = 'var r=[];\n',
        cursor = 0,
        match;
    var add = function(line, js) {
        js ? (code += line.match(reExp) ? line + '\n' : 'r.push(' + line + ');\n') :
            (code += line != '' ? 'r.push("' + line.replace(/"/g, '\\"') + '");\n' : '');
        return add;
    }
    while (match = re.exec(html)) {
        add(html.slice(cursor, match.index))(match[1], true);
        cursor = match.index + match[0].length;
    }
    add(html.substr(cursor, html.length - cursor));
    code += 'return r.join("");';
    return new Function(code.replace(/[\r\t\n]/g, '')).apply(options);
}
```

这下只有15行了。

原文下一些相关的讨论：
