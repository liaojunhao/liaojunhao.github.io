---
title: JavaScript函数重载
date: 2017-09-21 14:56:19
categories:
  - JavaScript 系列
tags:
  - JavaScript
---

> 译者按: jQuery 之父 John Resig 巧妙地利用了闭包，实现了 JavaScript 函数重载。

而所谓**函数重载**：就是函数名称一样，输入不一样，输出也就不一样。或者说，允许某个函数有各种不同输入，根据不同的输入，调用不同的函数，然后返回不同的结果。

但是 JS 中函数不能像传统编程那样重载。在其他语言比如 Java 中，一个函数可以有两个定义， 只要签名(接收参数的类型和数量)不同就行。如前所述，ECMAScript 函数没有签名，因为参数是由 包含零个或多个值的数组表示的。没有函数签名，自然也就没有重载。**所以 JavaScript 中没有真正意义上的函数重载**

如果在 ECMAScript 中定义了两个同名函数，则后定义的会覆盖先定义的。来看下面的例子:

```javascript
function addSomeNumber(num) {
  return num + 100;
}
function addSomeNumber(num) {
  return num + 200;
}
let result = addSomeNumber(100); // 300
```

这里，函数 addSomeNumber()被定义了两次。第一个版本给参数加 100，第二个版本加 200。最 后一行调用这个函数时，返回了 300，因为第二个定义覆盖了第一个定义。

## 1.模拟函数重载

一般来说 JS 可以通过 if…else 或者 switch 来区分不同的逻辑实现模拟函数重载的效果。

```javascript
function overload() {
  if (arguments.length === 1) {
    console.log("一个参数");
  }
  if (arguments.length === 2) {
    console.log("两个参数");
  }
}

overload(1); //一个参数
overload(1, 2); //两个参数
```

凭直觉，函数重载可以通过 if…else 或者 switch 实现，这就不去管它了。但是有一次面试中，被说到不要使用这么多的 if…else，当时觉得很奇怪也不知道是性能问题还是阅读性问题，不让使用这么多，于是就好奇怎么可以不通过 if…else 来实现函数的重载。

## 2.模拟函数重载 - 利用闭包

直到看到 jQuery 之父 John Resig 提出了一个非常巧(bian)妙(tai)的方法，利用了闭包。

这个例子是 John Resig 写他的书 《secrets of the JavaScript ninja》第一版中都有提到过，在书中的第 4 章中也有讲解 Function overloading，文中的 addMethod 函数 就是书中的例子 4.15，感兴趣的朋友可以去看看，但是它很代码量很少、很简单，巧妙地使用了 JavaScript 的特性。

```javascript
function addMethod(object, name, fn) {
  // 先把原来的object[name] 方法，保存在old中
  var old = object[name];
  // 重新定义 object[name] 方法
  object[name] = function () {
    // 如果函数需要的参数 和 实际传入的参数 的个数相同，就直接调用fn
    if (fn.length == arguments.length) return fn.apply(this, arguments);
    // 如果不相同,判断old 是不是函数，
    // 如果是就调用old，也就是刚才保存的 object[name] 方法
    else if (typeof old == "function") return old.apply(this, arguments);
  };
}

// 不传参数时，返回所有name
function find0() {
  return this.names;
}

// 传一个参数时，返回firstName匹配的name
function find1(firstName) {
  var result = [];
  for (var i = 0; i < this.names.length; i++) {
    if (this.names[i].indexOf(firstName) === 0) {
      result.push(this.names[i]);
    }
  }
  return result;
}

// 传两个参数时，返回firstName和lastName都匹配的name
function find2(firstName, lastName) {
  var result = [];
  for (var i = 0; i < this.names.length; i++) {
    if (this.names[i] === firstName + " " + lastName) {
      result.push(this.names[i]);
    }
  }
  return result;
}

function Users() {
  addMethod(Users.prototype, "find", find0);
  addMethod(Users.prototype, "find", find1);
  addMethod(Users.prototype, "find", find2);
}

var users = new Users();
users.names = ["John Resig", "John Russell", "Dean Tom"];

console.log(users.find()); // 输出[ 'John Resig', 'John Russell', 'Dean Tom' ]
console.log(users.find("John")); // 输出[ 'John Resig', 'John Russell' ]
console.log(users.find("John", "Resig")); // 输出[ 'John Resig' ]
console.log(users.find("John", "E", "Resig")); // 输出undefined
```

从效果上来说，users 对象的 find 方法允许 3 种不同的输入: 0 个参数时，返回所有人名；1 个参数时，根据 firstName 查找人名并返回；2 个参数时，根据完整的名称查找人名并返回。

难点在于，users.find 事实上只能绑定一个函数，那它为何可以处理 3 种不同的输入呢？它不可能同时绑定 3 个函数 find0,find1 与 find2 啊！这里的关键在于 old 属性。

由 addMethod 函数的调用顺序可知，users.find 最终绑定的是 find2 函数。然而，在绑定 find2 时，old 为 find1；同理，绑定 find1 时，old 为 find0。3 个函数 find0,find1 与 find2 就这样通过闭包链接起来了。

根据 addMethod 的逻辑，当 fn.length 与 arguments.length 不匹配时，就会去调用 old，直到匹配为止。

## 重载的好处

重载其实是把多个功能相近的函数合并为一个函数，重复利用了函数名。减少了代码的数量，还提高代码阅读性和代码质量，假如 jQuery 中的 css()方法不使用重载，那么就要有 5 个不同的函数，来完成功能，那我们就需要记住 5 个不同的函数名，和各个函数相对应的参数的个数和类型，显然就麻烦多了。

## 总结

> 虽然 JavaScript 并没有真正意义上的重载，但是重载的效果在 JavaScript 中却非常常见，比如 数组的 splice( )方法，一个参数可以删除，两个参数可以删除一部分，三个参数可以删除完了，再添加新元素。
> 再比如 parseInt( )方法 ，传入一个参数，就判断是用十六进制解析，还是用十进制解析，如果传入两个参数，就用第二个参数作为数字的基数，来进行解析。
> 文中提到的实现重载效果的方法，本质都是对参数进行判断，不管是判断参数个数，还是判断参数类型，都是根据参数的不同，来决定执行什么操作的。
> 虽然，重载能为我们带来许多的便利，但是也不能滥用，不要把一些根本不相关的函数合为一个函数，那样并没有什么意义。
