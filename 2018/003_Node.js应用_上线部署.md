
# Node.js应用——上线部署
Apr 16, 2018

## 写在前边

该项目使用的[阿里云服务器（ECS）](https://www.aliyun.com/)，系统为CentOS。
## 1. Linux必备命令

**1. 查看node进程信息(谁起的服务，谁在用)**

```javascript
$ ps aux | grep node
$ ps | grep node
```

**2. 查看端口占用情况**

【使用背景】

> 有时启动应用时会发现端口已经被占用，或者是感觉有些端口自己没有使用却发现是打开的。这时我们希望知道是哪个应用/进程在使用该端口。
>
> CentOS下可以用netstat或者lsof查看，Windows下也可以用netstat查看，不过参数会不同

【参考文档】

[Centos查看端口占用情况和开启端口命令](https://blog.csdn.net/carechere/article/details/52288058)

*linux*

```javascript
// 安装lsof
$ yum install  -y lsof 		// 如没有lsof命令可以通过yum安装

//检查端口被哪个应用所占用
$ lsof -i tcp:3306  	    // 例如：mysql默认端口

// 会列出所有正在使用的端口及关联的进程/应用
$ netstat -nap
$ netstat -nap | grep 3000

// 列出所有端口占用情况
$ netstat -ntlp	

// 检查端口被哪个进程占用
$ netstat -lnp | grep 3000   //-> 可以找到进程号，如：1777；

// 查看进程的详细信息
$ ps 1777(进程号)
```

**3. 杀死进程**

```javascript
$ kill -9 pid  				//杀掉编号为{{pid}}的进程
$ kill pid
```

**4. 远程上传资源**

```javascript
$ scp 上传文件 远程地址		  // $ scp helle.zip root@43.236.1.33
```

**5. 配置环境变量**

【参考文档】

[Linux下查看和添加环境变量](http://www.cnblogs.com/aaronLinux/p/5837702.html)

[Linux 基本命令不能用的解决方法](https://blog.csdn.net/houmou/article/details/51020709)

```javascript
// 1.打开~/.bashrc文件
$ vi ~/.bashrc

// 2.插入环境变量对应配置语句,以添加mysql命令到环境变量为例：
export PATH=/opt/lampp/bin:$PATH

// 3.保存退出执行该文件中的命令  
$ source ~/.bashrc
```

**6. 硬链接和软连接**

【参考文档】

[linux下创建和删除软、硬链接](http://www.cnblogs.com/xiaochaohuashengmi/archive/2011/10/05/2199534.html)

```javascript
// 查看软链列表
$ ls -il 

// 建立软链接
$ ln -s aaa alias 		//建立aaa的软链接

// 建立硬链接
$ ln aaa alias 			//建立aaa的硬链接

// 删除链接
$ rm -rf alias 			//删除建立的链接的别名，即删除链接.
```

```Txt
linux 软连接和硬链接的区别:
4点不同 ：
（1）软连接可以 跨文件系统 ，硬连接不可以 。
实践的方法就是用共享文件把windows下的 aa.txt文本文档连接到linux下/root目录 下 bb,cc . ln -s aa.txt
/root/bb 连接成功 。ln aa.txt /root/bb 失败 。

（2）关于 I节点的问题 。硬连接不管有多少个，都指向的是同一个I节点，会把 结点连接数增加 ，只要结点的连接数不是 0，文件就一直存在 ，不管你删除的是

源文件还是 连接的文件 。只要有一个存在 ，文件就 存在 （其实也不分什么 源文件连接文件的 ，因为他们指向都是同一个 I节点）。 当你修改源文件或者连接文件

任何一个的时候 ，其他的 文件都会做同步的修改 。软链接不直接使用i节点号作为文件指针,而是使用文件路径名作为指针。所以 删除连接文件 对源文件无影响，但

是 删除 源文件，连接文件就会找不到要指向的文件 。软链接有自己的inode,并在磁盘上有一小片空间存放路径名.

（3）软连接可以对一个不存在的文件名进行连接 。

（4）软连接可以对目录进行连接。
```

## 2. Node.js应用上线部署流程

### 2.1.  配置系统环境

1. 用[yum](http://man.linuxde.net/yum) 安装 [curl](https://man.linuxde.net/curl)

```javascript
$ yum -y install curl
```

2. 用`curl`安装[nvm](https://github.com/creationix/nvm#install-script)

```javascript
$ curl -o- https://raw.githubusercontent.com/creationix/nvm/v0.33.8/install.sh | bash
```

3. 用`nvm`安装`node`

```javascript
$ nvm use node 8.11.0
```

4. 安装解、压缩工具

```javascript
$ yum install -y unzip zip
```

### 2.2.  上传资源、部署项目

1. 上传本地生产环境应用到远程服务器

```javascript
$ scp node-app-1.1.0.zip  root@xx.xxx.x.xx:/root/
```

2. 解压文件

```javascript
$ unzip node-app-1.1.0.zip
```

3. 安装依赖

```javascript
$ npm i	
```

4. 启动服务

```
$ pm2 start pm2.json
```

### 2.3. 开启阿里云（ECS）对应的端口号（应用启动监听的端口号）

## 3. PM2 没有.5

> PM2 —— Advanced, production process manager for Node.js
>
> PM2 —— 强大的node进程管理器。 （认识中文吧，理解一下！非上文翻译哈~~~）

1. 安装

```javascript
npm install pm2@latest -g
```

2. 简单用法 —— 启动并监控应用程序

```javascript
pm2 start app.js

pm2 start pm2.json
```

3. 常用命令

```javascript
$ pm2 logs 									// 显示所有进程日志
$ sudo pm2 start pm2.json   			    // 启动一个进程，在app.json里设置选项 
$ sudo pm2 restart pm2.json  			    // 重启
$ sudo pm2 stop all						    // 停止所有进程
$ sudo pm2 kill 							// 杀死进程
$ sudo pm2 monit 						    // 系统监控（内核使用情况）	
$ sudo pm2 show 0(进程ID)					   
$ sudo pm2 delete [0 或 all]	     		   // 杀死全部进程
```

## 4. 参考文档

[pm2 进程管理工具使用指南](https://www.cnblogs.com/yangwenpeng/p/7267146.html)

[pm2 start命令中的json格式详解](https://newsn.net/say/pm2-start-json.html)

[pm2](http://pm2.keymetrics.io/)

[pm2配置](http://pm2.keymetrics.io/docs/usage/pm2-doc-single-page/)

[centos重启关机命令](http://www.cnblogs.com/lizhenlzlz/p/6391641.html)

## 5. 总结

通过这次部署node程序，学到了很多新东西。比如：文件远程上传、pm2进程管理、linux命令和系统配置。还有很多细节还需要继续深入学习，pm2的进程管理机制和广泛的应用场景、数据库、node各种机制、php等等。加油辣~~~。

[【demo了解一下】](http://47.104.183.145/)

