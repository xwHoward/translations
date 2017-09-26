# 探索EcmaScript装饰器

*原文链接[https://medium.com/google-developers/exploring-es7-decorators-76ecb65fb841](https://medium.com/google-developers/exploring-es7-decorators-76ecb65fb841)*

从迭代器([Iterators](http://jakearchibald.com/2014/iterators-gonna-iterate/))， 生成器（[generators](http://2ality.com/2015/03/es6-generators.html)） 到数组推导式（[array comprehensions](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Array_comprehensions)）, JavaScript和Python之间的相似特性越来越多，我（作者Addy Osmani）很高兴看到这样的趋势。我们将探讨由Yehuda Katz提供的ECMAScript下一个Python特性的提案——装饰器(Decorators)。

## 装饰器模式

究竟什么是装饰器呢？在Python中，装饰器为调用[高阶函数](https://en.wikipedia.org/wiki/Higher-order_function)提供了一个非常简单的语法。**一个Python装饰器是一个函数，它接收一个函数作为参数，扩展这个函数的行为，而不需要明确地修改它。**一个最简单的Python装饰器看起来是这样的：

```python
@mydecorator
def myfunc():
    pass
```

最上面的`@mydecorator`是一个装饰器，但跟ES2016(ES7)里的语法不一样，注意这一点:)。`@`向解释器表明我们正在使用装饰器，并用`mydecorator`这个名称来引用这个装饰器函数。装饰器接受一个参数（被装饰的函数），并返回这个额外添加了某些功能的函数。装饰器对于任何你想要进行透明包装的功能来说都非常有用。这些功能包括记忆函数，强制访问控制(enforcing access control)和授权验证，仪器和时间函数，日志，速率限制等，并且支持的功能越来越多。

> 装饰器是用来修饰 JavaScript 类的函数。 Angular 有很多装饰器，它们负责把元数据附加到类上，以了解那些类的设计意图以及它们应如何工作。— [Angular架构](https://www.angular.cn/guide/architecture)

## ES5和ES2015（ES6）中的装饰器

在ES5中，执行命令式的装饰器（纯函数）是非常琐碎的。在ES2015(ES6)中，类开始支持扩展，当多个类需要共享一部分功能时，我们就需要更好的分配方式。

Yehuda的装饰器提案旨在设计阶段启用注释(annotating)和修改的JavaScript类，属性和对象字面量，同时保留声明式的语法。

让我们来看看一些ES2016装饰器！

## ES2016装饰器

记住我们从Python学到的东西。一个ES2016装饰器是一个返回值为函数的表达式，它可以将目标，名称和属性描述符作为参数。你可以通过将装饰器加上前缀字符`@`来使用它，并将其放置在你要进行装饰的目标的顶部。也可以为类或属性定义装饰器。

### 装饰属性

我们来看一个基本的Cat类：

```javascript
class Cat {
    meow() { return `${this.name} says Meow!`;}
}
```

这个类将`meow`函数挂载到`Cat.prototype`上，大致如下：

```js
Object.defineProperty(Cat.prototype, 'meow', {
    value: specifiedFunction,
    enumerable: false,
    configurable: true,
    writable: true
});
```

假设我们想将一个属性或者方法标记为不可写，装饰器优先于定义属性的语法。我们可以这样定义一个`@readonly`装饰器，如下所示：

```js
function readonly(target, key, descriptor) {
    descriptor.writable = false;
    return descriptor;
}
```

并将其添加到我们的`meow`属性中：

```js
class Cat {
    @readonly
    meow() { return `${this.name} says Meow!`;}
}
```

装饰器只是一个将被评估(evaluated)的表达式，必须返回一个函数。这就是为什么`@readonly`和`@something(parameter)`都能生效。

现在，在将描述符挂载到`Cat.prototype`之前，解释器首先调用装饰器：

```js
let descriptor = {
    value: specifiedFunction,
    enumerable: false,
    configurable: true,
    writable: true
};

// 装饰器和Object.defineProperty有一样的签名
// 并且有机会在相关的defineProperty生效之前执行
// (and has an opportunity to intercede before the relevant 'defineProperty' actually occurs)
```

这导致`meow`现在只读。我们可以验证一下：

```js
var garfield = new Cat();
garfield.meow = function() {
    console.log('I want lasagne!');
};

// Exception: Attempted to assign a readonly property
```

太棒了，对吧？我们接下来快速看一下装饰类（而不仅仅是属性），但在那之前让我们来谈谈装饰器有关的库。尽管装饰器相关库出现得较晚，但在2016年装修器的库就已经开始出现，比如Jay Phelps的[https://github.com/jayphelps/core-decorators.js](https://github.com/jayphelps/core-decorators.js)。

类似于上述对于只读属性的尝试，它包含自己的“@readonly”实现，仅仅需要一个导入：

```js
import { readonly } from 'core-decorators';

class Meal {
    @readonly
    entree = 'steak';
}

var dinner = new Meal();
dinner.entree = 'salmon';

// Cannot assign to readonly property 'entree' of [object Object]
```

它还包括其他装饰器工具，如“@deprecate”，当API需要在内部方法改变时输出提示信息：

> 调用具有deprecation消息类型的console.warn()。提供自定义消息来覆盖默认的消息。你还可以提供一个带有url的选项哈希，以供进一步阅读。

```js
import { deprecate } from 'core-decorators';

class Person {
    @deprecate
    facepalm() {}

    @deprecate('We stopped facepalming')
    facepalmHard() {}

    @deprecate('We stopped facepalming', { url: 'http://knowyourmeme.com/memes/facepalm' })
    facepalmHarder() {}
}

let captainPicard = new Person();
captainPicard.facepalm();
// DEPRECATION Person#facepalm: This function will be removed in future versions.

captainPicard.facepalmHard();
// DEPRECATION Person#facepalmHard: We stopped facepalming

captainPicard.facepalmHarder();
// DEPRECATION Person#facepalmHarder: We stopped facepalming
//
// see http://knowyourmeme.com/memes/facepalm for more details.
//
```

### 装饰器作用于类

接下来我们来看看装饰器如何作用在类上。在这种情况下，根据提出的规范，装饰器将接受目标的构造函数作为参数。对于虚构的“MySuperHero”类，我们使用“@superhero”装饰器来为它定义一个简单的装饰器：

```js
function superhero(isSuperhero) {
    return function(target) {
        target.isSuperhero = isSuperhero
    }
}

@superhero(true)
class MySuperheroClass {}
console.log(MySuperheroClass.isSuperhero) // true

@superhero(false)
class MySuperheroClass {}
console.log(MySuperheroClass.isSuperhero) // false
```

ES2016装饰器作用于属性描述符和类。它会自动获得传递进来的属性名称和目标对象，后文会做介绍。对于描述符的可访问性允许装饰器做一些类似于改变属性以使用getter的事情，从而使得我们获得一些在之前会非常麻烦的功能，例如在首次访问属性时自动绑定一些方法到当前实例。

### ES2016装饰器和混入(Mixins)

> 在设计类的继承关系时，通常，主线都是单一继承下来的，例如，Ostrich继承自Bird。但是，如果需要“混入”额外的功能，通过多重继承就可以实现，比如，让Ostrich除了继承自Bird外，再同时继承Runnable。这种设计通常称之为Mixin。—— [多重继承-廖雪峰](https://www.liaoxuefeng.com/wiki/001374738125095c955c1e6d8bb493182103fac9270762a000/0013868200511568dd94e77b21d4b8597ede8bf65c36bcd000)
>
> Mixin 实质上是利用语言特性（比如 Ruby 的 include 语法、Python 的多重继承）来更简洁地实现组合模式。—— [知乎](https://www.zhihu.com/question/20778853)

我非常喜欢Reg Braithwaite最近的文章[ES2016 Decorators as mixins](http://raganwald.com/2015/06/26/decorators-in-es7.html)以及之前的[Functional Mixins](http://raganwald.com/2015/06/17/functional-mixins.html)。Reg提出了一个将行为混入到任何目标（类的原型或独立类）中的助手，并再次基础上描述了一个特定的基于类的版本。将实例行为混合到类的原型中的函数式混入(functional mixin)如下所示：

```js
function mixin(behaviour, sharedBehaviour = {}) {
    const instancekeys = Reflect.ownKeys(behaviour);
    // Reflect.ownKeys(target) 方法返回target的属性key组成的数组
    const sharedkeys = Reflect.ownKeys(sharedBehaviour);
    const typeTag = Symbol('isa');

    function _mixin(clazz) {
        for (let property of instancekeys) {
            Object.defineProperty(clazz.prototype, property, { value: behaviour[property] });
        }
        Object.defineProperty(clazz.prototype, typeTag, { value: true });
        return clazz;
    }
    for (let property of sharedkeys) {
        Object.defineProperty(_mixin, property, {
            value: sharedBehaviour[property],
            enumerable: sharedBehaviour.propertyIsEnumerable(property)
        });
    }
    Object.defineProperty(_mixin, Symbol.hasInstance, {
        value: (i) => !!i[typeTag]
    });
    return _mixin;
}
```

非常好。现在我们可以定义一些mixins，并尝试使用它们来装饰一个类。我们假设我们有一个简单的`ComicBookCharacter`类：

```js
class ComicBookCharacter {
    constructor(first, last) {
        this.firstName = first;
        this.lastName = last;
    }
    realName() {
        return this.firstName + ' ' + this.lastName;
    }
};
```

这可能是世界上最无聊的角色了，但是我们可以定义一些Mixins，它们将授予`SuperPowers`和`UtilityBelt`的行为。让我们使用Reg的mixin助手：

```js
const SuperPowers = mixin({
    addPower(name) {
        this.powers().push(name);
        return this;
    },
    powers() {
        return this._powers_pocessed || (this._powers_pocessed = []);
    }
});
const UtilityBelt = mixin({
    addToBelt(name) {
        this.utilities().push(name);
    },
    utilities() {
        return this._utility_items || (this._utility_items = []);
    }
});
```

有了这个，我们现在可以使用带有我们的mixin函数名称的`@`语法，来装饰`ComicBookCharacter`，使它具备我们所期望的行为。注意我们如何使用多个装饰器语句对该类进行前缀：

```js
@SuperPowers
@UtilityBelt
class ComicBookCharacter {
    constructor(first, last) {
        this.firstName = first;
        this.lastName = last;
    }
    realName() {
        return this.firstName + ' ' + this.lastName;
    }
};
```

现在，使用我们之前的定义来创建一个蝙蝠侠的角色。

```js
const batman = new ComicBookCharacter('Bruce', 'Wayne');
console.log(batman.realName());
// Bruce Wayne

batman
    .addToBelt('batarang')
    .addToBelt('cape');
console.log(batman.utilities());
// ['batarang', 'cape']

batman
    .addPower('detective')
    .addPower('voice sounds like Gollum has asthma');
console.log(batman.powers());
// ['detective', 'voice sounds like Gollum has asthma']
```

这些类的装饰器相对紧凑，我自己把它们作为函数调用的替代方法，或者作为高阶组件的助手。

*注意：@WebReflection在本节中使用的mixin模式有一些替代方法，您可以在[这里](https://gist.github.com/addyosmani/a0ccf60eae4d8e5290a0#comment-1489585)的评论中找到。*

## 通过Babel启用装饰器

装修器（在本文写作时）仍然是一个提案。他们还没有被完全支持。b㘝，感谢Babel在实验模式下支持语法转换，所以这篇文章的大多数样例都可以通过babel的转换直接尝试。

如果使用Babel CLI，您可以按照以下方式启用装饰器：

`$ babel --optional es7.decorators`

或者，你可以使用transformer开启支持：

`babel.transform("code", { optional: ["es7.decorators"] });`

还有一个在线转换的[online Babel REPL](https://babeljs.io/repl/)，勾选"Experimental"就可以启用，试一下！

## 有趣的实验

幸运的是，坐在我旁边的保罗·刘易斯（Paul Lewis）仍然一直在[实验](https://github.com/GoogleChrome/samples/tree/gh-pages/decorators-es7/read-write)将装饰器作为重新组织读写DOM的代码的途径。它借鉴了Wilson Page的FastDOM的想法，但提供了另一个小型的API。如果你在@write下调用方法或属性（或在@read下更改DOM）触发布局重绘，Paul的读/写装饰器也可以通过控制台通知你。

以下是Paul实验中的一个示例，尝试在@read中改变DOM，导致将一个警告抛到控制台中：

```js
class MyComponent {
    @read
    readSomeStuff() {
        console.log('read');

        // Throws a warning.
        document.querySelector('.button').style.top = '100px';
    }

    @write
    writeSomeStuff() {
        console.log('write');

        // Throws a warning.
        document.querySelector('.button').focus();
    }
}
```

## 现在去尝试装饰器！

在短期内，ES2016装饰器可用于声明性装饰和注释，类型检查以及解决将装饰器应用于ES2015类的挑战。从长远来看，他们对于静态分析非常有用（可以让位给编译时的类型检查或自动补全的工具）。

它们与经典OOP中的装饰器没有什么不同，这种模式允许对象以静态或动态方式对行为进行装饰，而不会影响同一类的对象。我认为这是一个很好的补充。类属性上的装饰器的语义仍然存在，请留意Yehuda的仓库更新。

库作者们目前正在讨论装饰器可能会在哪里取代mixins，而且肯定会有一些方式可以用于React中的高阶组件。

通过他们的使用增加了装饰器的实验，我感到很兴奋，希望你借也助Babel尝试使用装饰器，发现更多可重用的装饰器，也许你甚至会像Paul一样分享你的工作:)

## 进一步阅读和参考

- [https://github.com/wycats/javascript-decorators](https://github.com/wycats/javascript-decorators)
- [https://github.com/jayphelps/core-decorators.js](https://github.com/jayphelps/core-decorators.js)
- [http://blog.developsuperpowers.com/eli5-ecmascript-7-decorators/](http://blog.developsuperpowers.com/eli5-ecmascript-7-decorators/)
- [http://elmasse.github.io/js/decorators-bindings-es7.html](http://elmasse.github.io/js/decorators-bindings-es7.html)
- [http://raganwald.com/2015/06/26/decorators-in-es7.html](http://raganwald.com/2015/06/26/decorators-in-es7.html)
- [Jay的功能表达式ES2016 Decorators的例子](https://babeljs.io/repl/#?experimental=true&evaluate=true&loose=false&spec=false&playground=true&code=class%20Foo%20%7B%0A%20%20%40function%20%28target%2C%20key%2C%20descriptor%29%20%7B%20%20%20%0A%20%20%20%20descriptor.writable%20%3D%20false%3B%20%0A%20%20%20%20return%20descriptor%3B%20%0A%20%20%7D%0A%20%20bar%28%29%20%7B%0A%20%20%20%20%0A%20%20%7D%0A%7D&stage=0)