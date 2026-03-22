---
title: 游戏引擎Phaser使用总结
date: 2020-02-27 16:31:46
tags:
  - 游戏
  - canvas
categories:
  - 2D图形处理
---

> Phaser 在渲染方面支持 webGL/canvas，架构方面 Phaser 直接内嵌了 3 个物理引擎（Arcade Physics，Ninja，p2.js）
> 提供了动画，预加载，设备适配方案，粒子系统等功能。兼容方面移动端优先。这边之介绍我们最常用的一些方法/对象跟自己踩过的一些坑的经验分享。

## 初始化游戏

`new Phaser.Game(width, height, renderer, parent, state, transparent, antialias)`

- width：画布宽度
- height：画布高度
- renderer：渲染方式
  - Phaser.AUTO
  - Phaser.CANVAS
  - Phaser.WEBGL
- parent：游戏 canvas 挂载的 DOM ID 或者 DOM 节点
- state：游戏场景对象(按执行顺序：init、preload、create、update、render)
- transparent：是否为透明
- antialias：贴图是否抗锯齿

```javascript
let game = new Phaser.Game(
  800,
  600,
  Phaser.AUTO,
  "phaser-container",
  {
    init: init,
    preload: preload,
    create: create,
    update: update,
    render: render,
  },
  false,
  true
);
```

## 移动端适配

以 750\*1336 设计稿为例。

```javascript
let game = new Phaser.Game(750, 1336, Phaser.CANVAS, "game", {}, true, true);
// 推荐 SHOW_ALL：
// 比例不变，缩放不变形（通常使用该类型）
game.scale.scaleMode = Phaser.ScaleManager.SHOW_ALL;
```

## 场景的生命周期

可以理解为类型 vue 哪有的生命周期，在一个场景从创建到渲染都会走的几个钩子函数

```javascript
// 初始化
function init() {}
// 预加载
function preload() {}
// 场景创建
function create() {}
// 画布更新
function update() {}
// 渲染设置
function render() {}
```

### 资源的预加载 Preload

写在 preload 函数中

1. game.load.image(key, url, overwrite)

### 场景创建 create

### 画布更新 update

### 渲染设置 render

## 场景的管理

### 场景的切换

### 场景的销毁

```javascript
// 销毁整个游戏
game.destroy();
```

## 常见对象

Game 对象

- World 对象
  - 设置游戏世界范围

```javascript
// 不设默认为canvas画布大小
game.world.setBounds(x, y, width, height);
```

Stage 对象
Camare 对象  
State  
Sprite

## 文本：Text

### 自定义字体

```css
/* CSS中先设置好字体 */
@font-face {
  font-family: "WEBFONT";
  font-display: swap;
  src: url("../font/webfont.eot"); /* IE9*/
  src: url("../font/webfont.eot?#iefix") format("embedded-opentype"), /* IE6-IE8 */
      url("../font/webfont.woff2") format("woff2"),
    url("../font/webfont.woff") format("woff"), /* chrome、firefox */
      url("../font/webfont.ttf") format("truetype"),
    /* chrome、firefox、opera、Safari, Android, iOS 4.2+*/
      url("../font/webfont.svg#思源黑体-细") format("svg"); /* iOS 4.1- */
}
```

```javascript
// Phaser中添加字体的设置
let style = {
  font: "30px WEBFONT",
};
game.add.text(0, 0, "你好世界", style);
```

### 字体预加载

- 自定义字体常见的坑：
  - 当字体<font color='red'> 没有加载完毕 </font>，Phaser 还是会渲染系统默认当字体
- 解决方案：
  - 使用第三方库<font color='red'> fontfaceobserver </font>来做字体的预加载

```javascript
// 在vue的钩子函数mounted中做预加载
async mounted () {
  // 要加载的字体
  let fontOptions = [
    'HELVETI1-1',
    'WEBFONT'
  ];
  // 加载完成后返回的promise队列
  let fontLoadList = [];
  fontOptions.forEach((item) => {
    let font = new FontFaceObserver(item);
    fontLoadList.push(font.load(null, 5000));
  })
  // 等待所有promise完成
  await Promise.all(fontLoadList).then((resolve) => {
    // 游戏初始化
    init();
  }).catch((reason) => {
    console.log('字体加载失败：', reason);
  });
},
```

### 字体的行距

```javascript
text.lineSpacing = 40;
```

### 字符的间距

很遗憾 Phaser 不支持，也不是 Phaser 的问题而是 canvas 对文字排版这块支持比较弱。可以使用一些 hack 手段来实现，但是会比较繁琐，可以看张鑫旭老师的这篇博文介绍：
[绘制字符间距](https://www.zhangxinxu.com/wordpress/2018/02/canvas-text-break-line-letter-spacing-vertical/)。

也可以用通过`BitmapText`这种图片字体来实现，不过种方式也不太适合中文（文字太多工作量太大）

## 位图字体：BitmapText

## 时间：Time

基本时间事件

```javascript
// Phaser.Timer.SECOND 每秒的毫秒数(1000)
game.time.events.add(Phaser.Timer.SECOND * 4, function () {}, this);
```

### 定时器：Timer

### 定时器事件：TimerEvent

使用`Phaser.Timer.add` / `Phaser.Timer.repeat` / `Phaser.Timer.loop` 方法来新建事件

## 动画

## 事件绑定
