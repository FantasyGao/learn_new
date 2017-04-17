# 轻松排查线上Node内存泄漏问题

## I. 三种比较典型的内存泄漏

### 一. 闭包引用导致的泄漏

这段代码已经在很多讲解内存泄漏的地方引用了，非常经典，所以拿出来作为第一个例子，以下是泄漏代码：


```js
'use strict';
const express = require('express');
const app = express();

//以下是产生泄漏的代码
let theThing = null;
let replaceThing = function () {
    let leak = theThing;
    let unused = function () {
        if (leak)
            console.log("hi")
    };
    
    // 不断修改theThing的引用
    theThing = {
        longStr: new Array(1000000),
        someMethod: function () {
            console.log('a');
        }
    };
};

app.get('/leak', function closureLeak(req, res, next) {
    replaceThing();
    res.send('Hello Node');
});

app.listen(8082);

```

js中的闭包非常有意思，通过打印heapsnapshot，在chrome的dev tools中展示，会发现闭包中真正存储本作用域数据的是类型为 ```closure``` 的一个函数（其__proto__指向的function）的 ```context``` 属性指向的对象。

这个例子中泄漏引起的原因就是v8对上述的 ```context``` 选择性持有本作用域的数据的两个特点：

 * 父作用域的所有子作用域持有的闭包对象是同一个。
 * 该闭包对象是子作用域闭包对象中的 ```context``` 属性指向的对象，并且其中只会包含所有的子作用域中使用到的父作用域变量。


### 二. 原生Socket重连策略不恰当导致的泄漏

这种类型的泄漏本质上node中的events模块里的侦听器泄漏，因为比较隐蔽，所以放在第二个例子，以下是泄漏代码：

```js
const net = require('net');
let client = new net.Socket();

function connect() {
    client.connect(26665, '127.0.0.1', function callbackListener() {
    console.log('connected!');
});
}

//第一次连接
connect();

client.on('error', function (error) {
    // console.error(error.message);
});

client.on('close', function () {
    //console.error('closed!');
    //泄漏代码
    client.destroy();
    setTimeout(connect, 1);
});

```

泄漏产生的原因其实也很简单：```event.js``` 核心模块实现的事件发布/订阅本质上是一个js对象结构（在v6版本中为了性能采用了new EventHandles()，并且把EventHandles的原型置为null来节省原型链查找的消耗），因此我们每一次调用 ```event.on``` 或者 ```event.once``` 相当于在这个对象结构中对应的 ```type``` 跟着的数组增加一个回调处理函数。

那么这个例子里面的泄漏属于非常隐蔽的一种：```net``` 模块的重连每一次都会给 ```client``` 增加一个 ```connect事件``` 的侦听器，如果一直重连不上，侦听器会无限增加，从而导致泄漏。

### 三. 不恰当的全局缓存导致的泄漏

这个例子就比较简单了，但是也属于在失误情况下容易不小心写出来的，以下是泄漏代码

```js
'use strict';
const easyMonitor = require('easy-monitor');
const express = require('express');
const app = express();

const _cached = [];

app.get('/arr', function arrayLeak(req, res, next) {
	//泄漏代码
    _cached.push(new Array(1000000));
    res.send('Hello World');
});

app.listen(8082);
```

如果我们在项目中不恰当的使用了全局缓存：主要是指只有增加缓存的操作而没有清除的操作，那么就会引起泄漏。

这种缓存引用不当的泄漏虽然简单，但是我曾经亲自排查过：Appium自动化测试工具中，某一个版本的日志缓存策略有bug，导致搭建的server跑一段时间就重启。

## II. 常规排查方式

### 一. heapdump/v8-profiler + chrome dev tools

目前node上面用于排查内存泄漏的辅助工具也有一些，主要是：

* heapdump
* v8-profiler

这两个工具的原理都是一致的：调用v8引擎暴露的接口：
```v8::Isolate::GetCurrent()->GetHeapProfiler()->TakeHeapSnapshot(title, control)```
然后将获取的c++对象数据转换为js对象。

这个对象中其实就是一个很大的json，通过chrome提供的dev tools，可以将这个json解析成可视化的树或者统计概览图，通过多次打印内存结构，compare出只增不减的对象，来定位到泄漏点。

### 二. Easy-Monitor工具自动定位疑似泄漏点

我之前项目中遇到疑似的内存泄漏基本都是这样排查的，但是排查的过程中也遇到了几个比较困扰的问题：

* 只能在线下进行，而线上情况复杂，有些错误线下很难复现
* 总是需要多次插工具打印，然后对比，比较麻烦

所以后面花了点时间，详细解析了下v8引擎输出的heapsnapshot里面的json结构，做了一个轻量级的线上内存泄漏排查工具，也是之前的Easy-monitor性能监控工具的一个补完。

对如何测试自己项目线上js代码性能，以及找出js函数可优化点感兴趣的朋友可以参看这一篇：

* [轻量级易部署Node性能监控工具：Easy-Monitor](https://cnodejs.org/topic/58d0dd8b17f61387400b7de5)

本文下一节主要是以第I节中的三种非常典型的内存泄漏状况，来使用新一版的Easy-Monitor进行简单的定位排查。

## III. 使用[Easy-Monitor](https://github.com/hyj1991/easy-monitor)快速定位泄漏点

### 一. 安装&嵌入项目

Easy-Monitor的使用非常简单，安装启动总共三步

#### 1.安装模块

```
npm install easy-monitor
```

#### 2.引入模块

```
const easyMonitor = require('easy-monitor');
easyMonitor('你的项目名称');
```

#### 3.访问监控页面

打开你的浏览器，输入以下地址，即可看到进程相关信息：

```
http://127.0.0.1:12333
```

### 二. 内存泄漏排查使用方式

Easy-Monitor可以实时展示内存分析信息，所以在线上使用也是没有问题的，下面就来使用此工具分析第I节中出现的问题。

#### 1.闭包泄漏

在闭包泄漏的代码中，按照上面的步骤引入easy-monitor，然后不停在浏览器中访问：

```
http://127.0.0.1:8082/leak
```

那么几次后通过top或者别的自带内存监控工具能看到内存明显上升：

![closure_mem_stat.jpeg](//dn-cnode.qbox.me/Fgmg1RSn5JvskDIAdmW43zHygZLX)

这里我本地访问多次后，已经飙升到211MB。

此时，我们可以在Easy-Monitor的首页，点击对应Pid后面的 ```MEM``` 链接，即可自动进行当前业务进程的堆内内存快照打印以及泄漏点分析：

![index_mem.jpeg](//dn-cnode.qbox.me/FnFrVXTtIRlneHnfJWuK7X-0bgy7)

大约等待10s左右，页面即会呈现出解析的结果。最上面的 ```Heap Status``` 一栏呈现的内容是一个对当前堆内内存解析后的概览，大概看看就行了，比较重要的泄漏点定位在下面的 ```Memory Leak``` 一栏。

我对疑似的内存泄漏点推测是从计算得到的 ```retainedSize``` 着手的：**泄漏的感知首先是内存无故增加，且只增不减，那么当前堆内内存结构中从 ```(GC roots)``` 节点出发开始，占据的 ```retainedSize``` 最大的就可能是疑似泄漏点的起始。**

遵循这个规则，```Memory Leak``` 第一个子栏目得到的是疑似泄漏点的概览：

![closure_mem_point_index.jpeg](//dn-cnode.qbox.me/Fk_Ujb1sH0Ah5vCIjugez7f8wPt_)

这里按照 ```retainedSize``` 大小做了从大到小的排序，可以看到，这几个点基本上占据了90%以上的堆内内存大小。

好了，下面的子栏目则是对这里面的5个疑似泄漏点构建 **引力图**，来找出泄漏链条，原理和前面一样：**占据总堆内内存 ```retainedSize``` 最大的对象下面一定也有占据其 ```retainedSize``` 最大的节点**：

![closure_mem_force.jpeg](//dn-cnode.qbox.me/FmGdeY1c_5QEMjmOFaePN7fU-DRD)

根据引力图可以很清晰看到 ```retainedSize``` 最大的疑似泄漏链条，颜色和大小的一部分含义：

* 蓝色表示疑似的泄漏节点
* 紫色表示普通节点
* 最大的节点表示的是当前疑似泄漏链条的根节点

这里的展示用了Echarts2，所有的节点都可以点击展开/折叠。当我们把鼠标移动到疑似泄漏链条的最后一个子节点时，引力图下面会用文字显示出当前的泄漏链条的详细指向信息 ```Reference List``` ，这里简单的解析下其内容：

```bash
[object] (Route::@122187) ' stack 
---> [object] (Array::@124261) ' [0] 
---> [object] (Layer::@124265) ' handle 
---> [closure] (closureLeak::@124169) ' context 
---> [object] (system / Context::@84427) ' theThing 
---> [object] (Object::@122271) ' someMethod 
---> [closure] (someMethod::@122275) ' context 
---> [object] (system / Context::@122269) ' leak 
---> [object] (Object::@122113) ' someMethod 
---> [closure] (someMethod::@122117) ' context 
---> [object] (system / Context::@122111)
```

每一行表示一个节点：**[类型] \(名称::节点唯一id\) ' 属性名称或者index**。
因为测试代码用了Express框架，熟悉Express框架源码的小伙伴都能看出来了：

* 根节点是初始化express时构造的 ```Route``` 的实例。
* 该 ```Route``` 实例的 ```stack``` 属性对应的数组的第一个元素，即这里的 ```[0]``` 对应的元素，其实也就是一个中间件，所以是 ```Layer``` 的一个实例。
* 该中间件的 ```handle``` 属性指向 **```closureLeak``` 函数**，这里开始出现我们自己编写的Express框架外的代码了，简单分析下也很容易明白这个中间件其实就是我们编写的 ```app.get``` 部分。
* ```closureLeak``` 函数持有了上级作用域产生的闭包对象，这个闭包对象中 ```retainedSize``` 最大的变量为 ```theThing```
* ```theThing``` 持有了 ```someMethod``` 的引用，**```someMethod``` 又通过上级作用域的闭包对象持有了 ```leak``` 变量，```leak``` 变量又指向 ```theThing``` 变量指向的上一次的老对象，这个老对象中依旧包含了 ```someMethod``` ...**

通过这个引力图和下面提供的 ```Reference List``` 分析，其实很容易发现泄漏点和泄漏原因：正是因为第I节中提到的v8引擎作用域生成和持有闭包引用的规则，那么 ```unused``` 函数的存在，导致了 ```leak``` 变量被 ```replaceThing``` 函数作用域生成的闭包对象存储了，那么 ```theThing``` 每一次指向的新对象里面的 ```someMethod``` 函数持有了这个闭包对象，因此间接持有了上一次访问 ```theThing``` 指向的老对象。所以每一次访问后，老对象永远因为被持有永远无法得到释放，从而引起了泄漏。

这里也把关键词整理出来，方便大家项目全局搜索排查：**Leak Key**

#### 2.Socket重连泄漏

同样的方式，第I节中的代码保存后执行，注意 ```connect``` 操作的端口填写一个本地不存在的端口，来模拟触发客户端的断线重连。

那么这段代码跑大概一分钟左右，即开始产生比较明显的泄漏现象。同样打开easy-monitor监控页面进行堆内存分析，得到如下结果：

![socket_mem_index.jpeg](//dn-cnode.qbox.me/FskCsNhsPFQnxMPquniyvJNovA-2)

这个图很容易看出来，占据 ```retainedSize``` 最大的对象正是 ```socket``` 对象，几乎占到了堆内总内存的 50% 以上。

接着往下看引力图，如下所示：

![socket_mem_force.jpeg](//dn-cnode.qbox.me/Fm62hb6Ga3inyqI8eJugQvj-sE62)

其中的 ```Reference List``` 如下：

```bash
[object] (Socket::@97097) ' _events
---> [object] (EventHandlers::@97101) ' connect 
---> [object] (Array::@102511)
```
这里熟悉Node核心模块 ```events``` 的小伙伴就能感到熟悉，```_events``` 正是存储订阅事件/事件回调函数的属性，那么这边很显然是原生的socket触发断线重连时，会不停增加 ```connect``` 事件的处理，如果服务器一直挂掉，即客户端无法断线重连成功，那么内存就会不断增加导致泄漏。

题外插一句，我翻了下net.js的代码，这里的 ```connect``` 事件是以 ```once``` 的方式添加的，所以只要重连过程中能够连上一次，这部分侦听器增加的内存就能够被回收掉。

#### 3.全局缓存泄漏

这个是最简单的原因了，大家可以使用Easy-Monitor自行尝试一番~

## IV. 如何修改避免泄漏

### 一. 断掉闭包中的泄漏变量引用链条

根据第III节中的解析，明白了这种泄漏的原理，就比较容易对代码进行修改了，断掉 ```unused``` 函数对 ```leak``` 变量的引用，那么 ```replaceThing``` 函数作用域的闭包对象中就不会有 ```leak``` 变量了，这样 ```someMethod``` 即不会再对老对象间接产生引用导致泄漏，修改后代码如下：

```js
'use strict';
const express = require('express');
const app = express();
const easyMonitor = require('easy-monitor');
easyMonitor('Closure Leak');

let theThing = null;
let replaceThing = function () {
    let leak = theThing;
    //断掉leak的闭包引用即可解决这种泄漏
    let unused = function (leak) {
        if (leak)
            console.log("hi")
    };

    theThing = {
        longStr: new Array(1000000),
        someMethod: function () {
            console.log('a');
        }
    };
};

app.get('/leak', function closureLeak(req, res, next) {
    replaceThing();
    res.send('Hello Node');
});

app.listen(8082);

```

### 二. 断线重连时去掉老侦听器

修改主要目的是在重连时去掉连接失败时添加的 ```connect``` 事件，修改后代码如下：

```js
const net = require('net');
const easyMonitor = require('easy-monitor');
easyMonitor('Socket Leak');
let client = new net.Socket();

function callbackListener() {
    console.log('connected!');
});

function connect() {
    client.connect(26665, '127.0.0.1', callbackListener}

connect();

client.on('error', function (error) {
    // console.error(error.message);
});

client.on('close', function () {
    //console.error('closed!');
    //断线时去掉本次侦听的connect事件的侦听器
    client.removeListener('connect', callbackListener);
    client.destroy();
    setTimeout(connect, 1);
});

```

### 三.

修改和测试大家可以自行尝试一番。

## V. 结语

做这个工具也让自己对于v8的内存管理有了更深入的认识，收获挺大的，下一步的计划是优化代码逻辑和前台呈现界面，提高易用性和开发者的体验。

Easy-Monitor新版本下依旧支持线上部署和多项目cluster部署，最后项目的git地址在：

[Easy-Monitor](https://github.com/hyj1991/easy-monitor)

如果大家觉得有帮助或者不错，欢迎给个star 💕~