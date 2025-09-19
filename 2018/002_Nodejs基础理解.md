# Node.js基础

 Apr 11, 2018

## 写在前边

### **本文大纲**

- 回调机制
- 事件驱动机制
- 模块化
- 路由（get、post）

## 1、Node.js回调机制

### 1.1、什么是回调

- **函数调用**方式有三种：`同步调用`、`异步调用`、`回调`
- **回调**是一种**双向调用**模式
- **回调**可以通过**回调函数**来实现

### 1.2、阻塞与非阻塞

- `阻塞`和`非阻塞`关注的是<u>**程序在等待调用结果**</u>（<u>消息</u>、<u>返回值</u>）时的`状态`‘

- 阻塞：回调函数的调用方**不执行完**<u>不准继</u>续执行后续程序。（通俗释义。。。）

- 非阻塞：回调函数调用方**执行过程**中，后续程序<u>继续可执行</u>。（通俗释义。。。）

- 贴代码，如下：

  ```Txt
  //data.txt
  hello block code， haha！
  ```

  ```Javascript
  // block.js 阻塞调用示例代码文件
  //阻塞代码
  let fs = require('fs');

  var data = fs.readFileSync('data.txt');

  // console.log(data);
  //<Buffer 68 65 6c 6c 6f 20 62 6c 6f 63 6b 20 63 6f 64 65 ef bc 8c 20 68 61 68 61 ef bc 81>
  console.log(data.toString());

  // -> 'hello block code， haha！'
  ```

  ```Javascript
  //non-blocking 非阻塞调用示例代码
  //非阻塞代码
  let fs = require('fs');

  fs.readFile('data.txt', (err, data) => {
      //容错
      if(err) {
          return console.error(err);
      }
      //输出文件内容
      console.log(data.toString());
  });
  //先执行下面的log内容，后输出回调中的文件内容；
  console.log('程序执行完毕！');

  // 1 -> 程序执行完毕！
  // 2 -> hello block code， haha！
  ```



## 2、Node.js事件驱动机制

### 2.1、事件驱动I/O模型 

#### 非阻塞的I/O模型：

![img](https://i.loli.net/2018/04/16/5ad40a0b0f358.jpg)

（上图为事件处理流程）

#### Node.js运行机制：

- node是**单进程**，**单线程**，不能同时并发去完成更多的事情，**<u>只能通过`事件`和`回调`来实现并发的效果。</u>**也正因为node不是多线程，所以不需要同时处理很多额外的工作。**<u>单进程、单线程的node性能比较高</u>**。
- node的API都是**异步执行**的，都作为一个**独立的线程**在运行，node利用这种**异步函数调用机制**来实现**并发处理**。
- node中几乎所有**事件机制**都是使用设计模式中的**观察者模式**来实现的。
- 上图中的Events是事件队列。

### <u>2.2</u>、事件与事件绑定

#### 事件处理代码流程

1. 引入events对象，创景eventEmitter对象
2. 绑定时间处理程序
3. 触发事件

```Javascript
//引入Events模块并创建eventsEmitter对象
let events = require('events');
let eventEmitter = new events.EventEmitter();

//绑定事件处理函数
let connectHandler = () => {
    console.log('connectHandler 被调用');
}
eventEmitter.on('connection', connectHandler);  // 进行事件绑定

//触发事件
eventEmitter.emit('connection');

console.log('程序执行完毕……');

// -> connectHandler 被调用
// -> 程序执行完毕……

```

## 3、Node.js模块化

### 3.1、模块化的概念和意义

1. **为了让Node.js的文件可以相互调用**，Node.js提供了一个简单的模块系统
2. **模块**是Node.js应用程序的**基本组成**部分
3. **文件**和**模块**是**一一对应**的。<u>一个Node.js文件就是一个模块</u>
4. 这个文件可能是**Javascript代码**、**JSON**或者**编译过的C/C++**扩展
5. Node.js中存在<u>4类模块</u>（**原生模块**和**3种文件模块**）

### 3.2、Node.js的模块加载流程

![img](https://i.loli.net/2018/04/16/5ad40cba8e41d.jpg)

（模块加载流程图）

- 文件缓存区、原始模块缓存区：<u>**可以防止某一个模块反复的加载到内存中，这样可以`节省内存`，`提高模块的加载速度`**</u>
- 上图可以了解到：Node.js模块可以从 `文件模块缓存区`、`原始模块缓存区`、`原始模块`、`文件模块`加载

### 3.3、贴代码，模块化示例；

```Javascript
//say.js 文件模块
class Say {
    constructor(something) {
        this.doWhat = something || '';
    }

    hello(name) {
        console.log('hello ' + name + ', ' + this.doWhat);
    }
}

//导出Say模块
module.exports = Say;
```

```Javascript
// main.js
//引入Say文件模块
let Say = require('./say');

let say = new Say('我们去看场电影，怎么样？');
say.hello('小张');
// -> 'hello 小张, 我们去看场电影，怎么样?'

```

## 4、Node.js路由

### 4.1、贴代码：nodejs程序中的router示例

```Javascript
//server.js

//引入node原生模块 http、url
let http = require('http');
let url = require('url');

//web服务启动函数
function start(router) {
    function requestServer(req, res) {
        //解析请求地址；
        let pathname = url.parse(req.url).pathname;
        console.log('请求来自于' + pathname + '路由地址');
		
        //设置响应头；
        res.writeHead(200, { "Content-Type": "text/html;charset=utf8" });
        
        //设置路由
        router(pathname, res);
        // res.write('Hello World');
        // res.end();
    }
	
    //创建web服务
    http.createServer(requestServer).listen(8888);
	
    console.log('server is starting ...');
}

//导出该函数
exports.start = start;
```

```javascript
//router.js
//路由设置函数
function route (pathname, res) {
    if(pathname == '/') {
        res.write('访问根路由');
        res.end();
    } else if(pathname == '/home') {
        res.write('访问/home路由，<a href="https://www.baidu.com" target="_blank">去百度</a>');
        res.end();
    } else {
        res.end('404');
    }
}

//导出该函数
exports.route = route;
```

```Javascript
//index.js
// 引入node 文件模块 server、router
let server = require('./server');
let router = require('./router');

//启动web服务
server.start(router.route);
```

### 4.2、get请求处理

```Javascript
//node 处理get 请求

let http = require('http');
let url = require('url');
let util = require('util');

http.createServer((req, res) => {
    res.writeHead(200, {'Content-Type': 'text/plain; charset=utf8'});
    //简单输出
    // res.end(util.inspect(url.parse(req.url, true)));
    //复杂输出
    let params = url.parse(req.url, true).query;
    res.write('查询姓名：' + params.name);
    res.write('\n');
    res.write('查询年龄：' + params.age);
    res.end();    
}).listen(8886);

console.log('server is starting……');
```

### 4.3、post请求处理

- 模拟前端服务

  ```html
  <!-- post.html -->

  <!DOCTYPE html>
  <html lang="en">
  <head>
      <meta charset="UTF-8">
      <meta name="viewport" content="width=device-width, initial-scale=1.0">
      <meta http-equiv="X-UA-Compatible" content="ie=edge">
      <title>统计个人信息</title>
  </head>
  <body>
      <!-- <form> -->
          姓名：
          <input type="text" name="name" id="name" placeholder="请填写真实姓名"><br>
          年龄：
          <input type="text" name="age" id="age" placeholder="请填写真实年龄"><br>
          
          <input type="submit" id="btn"> <br>
      <!-- </form> -->

          后端返回数据： <span id="data"></span>
      <script src="https://cdn.bootcss.com/jquery/3.3.1/jquery.min.js"></script>
      <script src="./index.js"></script>
  </body>
  </html>

  ```

  ```Javascript
  //index.js

  $('#btn').click(function (e) {
      e.stopPropagation();
      $.ajax({
          method: 'post',
          url: 'http://localhost:8886/add-user',
          data: {
              name: $('#name').val(),
              age: $('#age').val()
          },
          success: function(res) {
              $('#data').html(res);
          },
          fail: function(res) {
              $('#data').html('error:' + res);
          }
      });
  });
  ```

- 模拟后端node服务

  ```javascript
  //postReq.js
  //node 处理post 请求

  let http = require('http');
  let url = require('url');
  let querystring = require('querystring');
  let util = require('util');
 
  http.createServer((req, res) => {
      res.writeHead(200, { 'Content-Type': 'text/html; charset=utf8', 'Access-Control-Allow-Origin': '*'});
      
      let pathname = url.parse(req.url).pathname;
      let postData = '';
      if(pathname == '/add-user') {
          req.on('data', (chunk) => {
              postData += chunk;
          })
          req.on('end', () => {
              postData = querystring.parse(postData);
              // res.write(util.inspect(postData));
              // res.write('<br>');
              if (postData.name && postData.age) {
                  res.write('接口返回姓名：' + postData.name + '; ');
                  // res.write('<br>');
                  res.write('接口返回年龄：' + postData.age);
              } else {
                  res.write('404');
              }
              res.end();
          })
      } else {
          res.end('request address error');
      }

      // res.end('hello');
  }).listen(8886);

  console.log('server is starting……');
  ```


- 启动前后端服务，并在前端添加表单，经过node的web服务处理后 返回给前端显示。



## 参考文档

- [Node.js路由](http://www.runoob.com/nodejs/nodejs-router.html)
- [Node.js GET/POST请求](http://www.runoob.com/nodejs/node-js-get-post.html)

- [结合源码分析 Node.js 模块加载与运行原理](https://zhuanlan.zhihu.com/p/35238127)
- [深入浅出Node.js（三）：深入Node.js的模块机制](http://www.infoq.com/cn/articles/nodejs-module-mechanism)
- [Node.js挖掘之五：浅析模块加载流程,我们的应用如何启动以及如何加载依赖模块](https://cnodejs.org/topic/55b044399594740e76ab3e91)
- [NodeJs模块机制](http://www.qbian.me/#/article/NodeJs%E6%A8%A1%E5%9D%97%E6%9C%BA%E5%88%B6)