---
title: JavaScript代码优化技巧
date: 2018-07-03 10:08:03
tags:
  - JS
categories:
  - JavaScript
---

## if-多条件判断

在 if 多条件判断的情况下建议使用 include 方法。

```javascript
const day = "星期二";
if (day === "星期二" || day === "星期三" || day === "星期四") {
  console.log(day);
}

// 优化
if (["星期二", "星期三", "星期四"].includes(day)) {
  console.log(day);
}
```

## 简写 if...else

当有不包含复杂逻辑的使用 if...else 条件时，可以三元运算符来简化。

```javascript
// 优化前
let test;
if (x > 100) {
  test = true;
} else {
  test = false;
}
// 优化后
let test = x > 10 ? true : false;
// 或简写如下
let test = x > 10;
```

## 合并变量声明

当要声明两个具有相同值或相同类型的变量时，可以使用简写方式。

```javascript
// 优化前
let number1;
let number2 = 1;

// 优化后
let number1,
  number2 = 1;
```

## 合并变量赋值

当处理多个变量并进行赋值的时候，使用“解构”可以使得代码更加友好。

```javascript
// 优化前
let test1, test2, test3;
test1 = 1;
test2 = 2;
test3 = 3;
// 优化后
let [test1, test2, test3] = [1, 2, 3];
```

## 运算符简写

在项目开发中，要处理大量的算术运算符，很多方式可以进行简写。

```javascript
// 优化前
test1 = test1 + 1;
test2 = test2 - 1;
test3 = test3 * 10;
// 长
if (test1) {
  callMethod();
}

// 优化后
test1++;
test2--;
test3 *= 10;
// 短
test1 && callMethod();
```

## 将数字四舍五入到固定的小数点

```javascript
const toFixed = (n, fixed) => ~~(Math.pow(10, fixed) * n) / Math.pow(10, fixed);
```

## for 循环的简写

```javascript
// 优化前
for (var i = 0; i < testData.length; i++)

// 优化后
for (let i in testData) or  for (let i of testData)
```

如果是数组，可以使用 **foreach**。

```javascript
[11, 24, 32].forEach((element, index, array) => {
  console.log("test[" + index + "] = " + element);
});
```

## 条件返回优化

```javascript
// 优化前
let test;
function callMe(val) {
  console.log(val);
}

function checkReturn() {
  if (!(test === undefined)) {
    return test;
  } else {
    return callMe("test");
  }
}
var data = checkReturn();
console.log(data);
// 优化后
function checkReturn() {
  return test || callMe("test");
}
```

## 多用箭头函数

在项目中还是建议多用箭头函数，简洁而且可以避免 this 的问题。
还可以隐式回调，使用箭头函数，可以直接返回值，而不必使用 return 语句。

```javascript
// 优化前
function add(a, b) {
  return a + b;
}
// 优化后
const add = (a, b) => a + b;

function callMe(name) {
  console.log("Hello", name);
}
// 箭头函数的简写
callMe = (name) => console.log("Hello", name);
```

## 条件调用

```javascript
// 优化前
function test1(title) {
  console.log(title);
}
function test2(title) {
  console.log("我的名称是：" + title);
}
var test3 = 1;
var title = "DevPoint";
if (test3 === 1) {
  test1(title);
} else {
  test2(title);
}

// 优化后
(test3 === 1 ? test1 : test2)(title);
```

## 可以少用 switch

优化的方式可以将条件保存在键值对象中，根据条件调用。可以少点判断，节省那一丢丢的性能。

```javascript
// 优化前
switch (data) {
  case 1:
    test1();
    break;

  case 2:
    test2();
    break;

  case 3:
    test();
    break;
}

// 优化后
var data = {
  1: test1,
  2: test2,
  3: test,
};

data[something] && data[something]();
```

## 十进制数值指数优化

这种感觉必要性不大

```javascript
// 优化前
for (let i = 0; i < 10000; i++) {}

// 优化后
for (let i = 0; i < 1e4; i++) {}
```

## 参数默认值

```javascript
// 优化前
function add(test1, test2) {
  if (test1 === undefined) {
    test1 = 1;
  }
  if (test2 === undefined) {
    test2 = 2;
  }
  return test1 + test2;
}
// 优化后
add = (test1 = 1, test2 = 2) => test1 + test2;
add();
```

## ...运算符

展开运算符**_..._**是一个 es6 / es2015 特性，它提供了一种非常方便的方式来执行浅拷贝，这与 **Object.assign()**的功能相同。

```javascript
// 优化前
const data = [1, 2, 3];
const test = [4, 5, 6].concat(data);
// 优化后
const data = [1, 2, 3];
const test = [4, 5, 6, ...data];

// 数组对象浅拷贝
const map1 = {
    title: "DevPoint",
  },
  map2 = {
    info: "Develop",
  };
const newMap = { ...map1, ...map2 };
console.log(newMap); // { title: 'DevPoint', info: 'Develop' }
const test1 = [1, 2, 3];
const test2 = [...test1];
```

## 多用模板字符串

对于拼接字符串很实用，可以更加友好。

```javascript
// 优化前
const welcome = "Hi " + test1 + " " + test2 + ".";
// 优化后
const welcome = `Hi ${test1} ${test2}`;
```

多行字符串的拼接。

```javascript
// 优化前
const data =
  "abc abc abc abc abc abc\n\t" + "test test,test test test test\n\t";
// 优化后
const data = `abc abc abc abc abc abc
         test test,test test test test`;
```

## 对象属性赋值

```javascript
let test1 = "a";
let test2 = "b";
// 优化前
let obj = { test1: test1, test2: test2 };
// 优化后
let obj = { test1, test2 };
```

## 变量的解构赋值

```javascript
// 优化前
const test1 = this.data.test1;
const test2 = this.data.test2;
const test2 = this.data.test3;
// 优化后
const { test1, test2, test3 } = this.data;
```

> 变量的解构赋值是 es6/es2015 新增的能力，是非常有用的，特别是我曾经遇到过这样的面试题：
>
> 如何提取高度嵌套的对象里的指定属性？
>
> ```javascript
> // 有时会遇到一些嵌套程度非常深的对象：
> const school = {
>   classes: {
>     stu: {
>       name: "Bob",
>       age: 24,
>     },
>   },
> };
>
> // 更标准的做法 ，可以用一行代码来解决这个问题：
> const {
>   classes: {
>     stu: { name },
>   },
> } = school;
>
> console.log(name); // 'Bob'
> ```

## Array.find

当确实有一个对象数组并且我们想要根据对象属性查找特定对象时，find 方法确实很有用。

```javascript
const data = [
  {
    type: "test1",
    name: "abc",
  },
  {
    type: "test2",
    name: "cde",
  },
  {
    type: "test1",
    name: "fgh",
  },
];
function findtest1(name) {
  for (let i = 0; i < data.length; ++i) {
    if (data[i].type === "test1" && data[i].name === name) {
      return data[i];
    }
  }
}
// 优化后
filteredData = data.find(
  (data) => data.type === "test1" && data.name === "fgh"
);
console.log(filteredData); // { type: 'test1', name: 'fgh' }
```

## 条件执行

条件执行跟条件调用很像，如果有代码来检查类型，并且基于类型需要调用不同的方法，可以选择使用多个 else if 或进行切换，可以使用对象键值优化。

```javascript
// 优化前
if (type === "test1") {
  test1();
} else if (type === "test2") {
  test2();
} else if (type === "test3") {
  test3();
} else if (type === "test4") {
  test4();
} else {
  throw new Error("Invalid value " + type);
}
// 优化后
var types = {
  test1: test1,
  test2: test2,
  test3: test3,
  test4: test4,
};

var func = types[type];
!func && throw new Error("Invalid value " + type);
func();
```

## 避免全局变量

创建全局变量被认为是糟糕的实践，尤其在团队开发的大背景下更是问题多多。随着代码量的增长，全局变量会导致一些非常重要的可维护性难题，全局变量越多，引入错误的概率会变得越高

一般而言，有如下三种解决办法

- 零全局变量：实现方法是使用一个立即调用函数 IIFE 并将所有脚本放置其中
- 单全局变量和命名空间
- 用模块

## 事件处理

将事件处理相关的代码和事件环境耦合在一起，导致可维护性很糟糕

1. 隔离应用逻辑
   将应用逻辑从所有事件处理程序中抽离出来是一种最佳实践，将应用逻辑和事件处理的代码拆分开来

```javascript
//不好的做法
function handleClick(event) {
  var popup = document.getElementById("popup");
  popup.style.left = event.clientX + "px";
  popup.style.top = event.clientY + "px";
  popup.className = "reveal";
}
addListener(element, "click", handleClick);

//好的做法
var MyApplication = {
  handleClick: function (event) {
    this.showPopup(event);
  },
  showPopup: function (event) {
    var popup = document.getElementById("popup");
    popup.style.left = event.clientX + "px";
    popup.style.top = event.clientY + "px";
    popup.className = "reveal";
  },
};
addListener(element, "click", function (event) {
  MyApplication.handleClick(event);
});
```

2. 不要分发事件对象
   应用逻辑不应当依赖于 event 对象来正确完成功能，方法接口应该表明哪些数据是必要的。代码不清晰就会导致 bug。最好的办法是让事件处理程序使用 event 对象来处理事件，然后拿到所有需要的数据传给应用逻辑

```javascript
//改进的做法
var MyApplication = {
  handleClick: function (event) {
    this.showPopup(event.clientX, event.clientY);
  },
  showPopup: function (x, y) {
    var popup = document.getElementById("popup");
    popup.style.left = x + "px";
    popup.style.top = y + "px";
    popup.className = "reveal";
  },
};
addListener(element, "click", function (event) {
  MyApplication.handleClick(event);
});
```

当处理事件时，最好让事件程序成为接触到 event 对象的唯一的函数。事件处理程序应当在进入应用逻辑之前针对 event 对象执行任何必要的操作，包括阻止事件冒泡，都应当直接包含在事件处理程序中

```javascript
//改进的做法
var MyApplication = {
  handleClick: function (event) {
    event.preventDefault();
    event.stopPropagation();
    this.showPopup(event.clientX, event.clientY);
  },
  showPopup: function (x, y) {
    var popup = document.getElementById("popup");
    popup.style.left = x + "px";
    popup.style.top = y + "px";
    popup.className = "reveal";
  },
};
addListener(element, "click", function (event) {
  MyApplication.handleClick(event);
});
```

## 配置数据

可能会变化的全局变量应该从代码中抽离出来

```javascript
//好的做法
var config = {
  MSG_INVALID_VALUE: "Invalid value",
  URL_INVALID: "/errors/invalid.php",
  CSS_SELECTED: "selected",
};
function validate(value) {
  if (!value) {
    alert(config.MSG_INVALID_VALUE);
    location.href = config.URL_INVALID;
  }
}
function toggleSelected(element) {
  if (hasClass(element, config.CSS_SELECTED)) {
    removeClass(element, config.CSS_SELECTED);
  } else {
    addClass(element, config.CSS_SELECTED);
  }
}
```

## 函数优化

在 javascript 开发中，大部分时间都在与函数打交道，所以希望这些函数有着良好的命名，函数体内包含的逻辑清晰明了。如果一个函数过长，不得不加上若干注释才能让这个函数显得易读一些，那这些函数就很有必要进行重构

如果在函数中有一段代码可以被独立出来，那最好把这些代码放进另外一个独立的函数中。这是一种很常见的优化工作，这样做的好处主要有以下几点：

1. 避免出现超大函数
2. 独立出来的函数有助于代码复用
3. 独立出来的函数更容易被覆写
4. 独立出来的函数如果拥有一个良好的命名，它本身就起到了注释的作用

比如在一个负责取得用户信息的函数里面，还需要打印跟用户信息有关的 log，那么打印 log 的语句就可以被封装在一个独立的函数里：

```javascript
var getUserInfo = function () {
  ajax("http:// xxx.com/userInfo", function (data) {
    console.log("userId: " + data.userId);
    console.log("userName: " + data.userName);
    console.log("nickName: " + data.nickName);
  });
};
//改成：
var getUserInfo = function () {
  ajax("http:// xxx.com/userInfo", function (data) {
    printDetails(data);
  });
};
var printDetails = function (data) {
  console.log("userId: " + data.userId);
  console.log("userName: " + data.userName);
  console.log("nickName: " + data.nickName);
};
```

### 尽量减少参数数量

如果调用一个函数时需要传入多个参数，那这个函数是让人望而生畏的，必须搞清楚这些参数代表的含义，必须小心翼翼地把它们按照顺序传入该函数。在实际开发中，向函数传递参数不可避免，但应该尽量减少函数接收的参数数量。下面举个非常简单的示例。有一个画图函数 draw，它现在只能绘制正方形，接收了 3 个参数，分别是图形的 width、heigth 以及 square：

```javascript
var draw = function (width, height, square) {};
```

但实际上正方形的面积是可以通过 width 和 height 计算出来的，于是我们可以把参数 square 从 draw 函数中去掉：

```javascript
var draw = function (width, height) {
  var square = width * height;
};
```

假设以后这个 draw 函数开始支持绘制圆形，需要把参数 width 和 height 换成半径 radius，但图形的面积 square 始终不应该由客户传入，而是应该在 draw 函数内部，由传入的参数加上一定的规则计算得来。此时，可以使用策略模式，让 draw 函数成为一个支持绘制多种图形的函数

### 传递对象参数代替过长的参数列表

有时候一个函数有可能接收多个参数，而参数的数量越多，函数就越难理解和使用。使用该函数的人首先得搞明白全部参数的含义，在使用的时候，还要小心翼翼，以免少传了某个参数或者把两个参数搞反了位置。如果想在第 3 个参数和第 4 个参数之中增加一个新的参数，就会涉及许多代码的修改，代码如下：

```javascript
var setUserInfo = function (id, name, address, sex, mobile, qq) {
  console.log("id= " + id);
  console.log("name= " + name);
  console.log("address= " + address);
  console.log("sex= " + sex);
  console.log("mobile= " + mobile);
  console.log("qq= " + qq);
};
setUserInfo(1314, "xiaohuochai", "beijing", "male", "150********", 121631835);
```

这时可以把参数都放入一个对象内，然后把该对象传入 setUserInfo 函数，setUserInfo 函数需要的数据可以自行从该对象里获取。现在不用再关心参数的数量和顺序，只要保证参数对应的 key 值不变就可以了：

```javascript
var setUserInfo = function (obj) {
  console.log("id= " + obj.id);
  console.log("name= " + obj.name);
  console.log("address= " + obj.address);
  console.log("sex= " + obj.sex);
  console.log("mobile= " + obj.mobile);
  console.log("qq= " + obj.qq);
};
setUserInfo({
  id: 1314,
  name: "xiaohuochai",
  address: "beijing",
  sex: "male",
  mobile: "150********",
  qq: 121631835,
});
```

> 但是在在一个函数里，改变传入的对象本身是不好的，因为对象是引用数据类型，如果外部意外修改了对象的属性，会引发一些不必要的 bug。

函数的优化知识点还有很多，像惰性思想、递归尾调用等，后续在拿出来单独的篇章讲解。
