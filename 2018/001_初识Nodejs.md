# 初识Node.js
Apr 8, 2018 
## 1、什么是Node.js

- **Node.js**是一个`js解释器`、一个`js运行环境`、一个`服务器程序`
- **Node.js**使用`V8引擎`
- **Node.js**`不是`**web服务器**
- **Node.js**可以`读取磁盘文件`、`操作DOM`、`搭建web服务器`……

## 2、为什么要用Node.js

- 为了提供**高性能的web服务**
- **IO性能强大**
- **事件处理**机制完善
- 天然能够**处理DOM**
- 社区非常活跃、生态圈日趋完善

## 3、Node.js的优势在哪里

- 处理**大流量数据**
- 适合**实时交互**的应用（eg：<u>*在线聊天系统等*</u>）
- 完美支持**对象型数据库**（eg：<u>*MongoDB*</u>）
- **异步**处理**大量并发连接**（并发连接可以用来<u>*衡量服务器的性能*</u>）

## 4、学习Node.js的前置知识

- **JavaScript**（<u>基于面向对象</u>的语言）
- **ES6**（<u>面向对象</u>的语言，nodejs天生支持ES6）
- 一些**服务器**相关的知识（<u>Apache</u>……）
- 最好在**Linux系统**下进行开发

## 5、npm知多少

- 从NPM服务器下载别人编写的**<u>第三方包</u>**到本地使用

- 从NPM服务器下载并安装别人编写的**<u>命令行程序</u>**到本地使用

- 将自己编写的**<u>包</u>**或**<u>命令行程序</u>**上传到NPM服务器供别人使用

- 安装cnpm（速度快）

  ```Javascript
  $ npm install cnpm //安装cnpm镜像
  ```


- 更新npm版本

  ```javascript
  $ npm install npm -g  //更新npm版本
  ```

## 6、"Hello World"

```Javascript
// hello.js
console.log('hello world');

// 使用 node 运行 hello.js
// 输出 'hello world'
```

## 7、使用Node.js搭建一个简单web服务器

```Javascript
// server.js
let http = require('http');

http.createServer((req, res) => {
    //定义HTTP头
    res.writeHead(200, {'Content-Type': 'text/plain'});

    //发送相应数据；
    res.end('Hello World!\n');
}).listen(8000);

//服务运行后输出一行信息；
console.log('server is running...')

// 使用 node 运行 server.js
// 启动服务器
```

## 8、Node.js环境

### 8.1、REPL环境——交互式解释器

#### 8.1.1、在该环境中可以进行**<u>`简单计算`</u>**  、 <u>`代码验证`</u>

```Javascript
$ node // 回车 进入Node的 REPL环境
> 1 + 3			// 简单的计算
  4
> console.log('helle REPL');  //代码执行
  'helle REPL'
> let x = 10	//复杂些的计算，声明变量、使用变量计算
  undefined
> let y = 2
  undefined
> let z = 3
  undefined
> x + y * z
  16
```

#### 8.1.2、REPL下的常用命令

```Javascript
$ ctrl + c  	//退出当前终端。比如 npm、http-server……
$ ctrl + c		//退出Node REPL环境， 需要按两次
$ ctrl + d 		//退出Node REPL环境， 当前REPL命令行没有可执行的命令时，可用
$ 向上/向下 键	 //查看输入的历史命令
$ tab 键  	    //补全记不清的命令(列出当前命令)
$ .exit			 //退出当前终端
$ .help			//列出使用的命令
$ .break		//退出多行表达式
$ .clear		//退出多行表达式
$ .save filename //保存当前的Node REPL 会话到指定文件
$ .load filename //载入当前 Node REPL 会话的文件内容
```

## 10、参照资料

- [Node.js英文官网](https://nodejs.org/en/)
- [Nodejs中文官网](http://nodejs.cn/)
- [Github——Node.js](https://github.com/nodejscn/node-api-cn)
- [NPM官网](https://www.npmjs.com/)
- [淘宝NPM镜像](https://npm.taobao.org/)
- [狼叔：如何正确的学习](https://i5ting.github.io/How-to-learn-node-correctly/)

## 写在后边
如有问题，欢迎指正。[微笑]