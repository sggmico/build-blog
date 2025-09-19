# Linux使用笔记_nginx篇

Jul 20, 2018
## 常用技巧

### 1. 安装

**方式一：使用`yum`安装`nginx`，及相关操作。**

> 好处：安装方式简单，不易出错。

```javascript
============== 安装、卸载、查找 ============
    
// 安装命令
$  yum -y install nginx

// 查看安装路径
$ rpm -ql nginx

// 卸载nginx
$ rpm -e nginx
$ rpm -e --nodeps nginx  // 对于其他软件，如提示有依赖问题，可使用该命令完成卸载。


============== 使用nginx服务 =======
    
// nginx 启动 / 停止 / 重启
// 启动 nginx 服务
$ service nginx start  

// 停止 nginx 服务
$ service nginx stop 

// 重启 nginx 服务
$ service nginx restart

```

**方式二： 使用`源码包`安装`nginx`。**

> 好处：在自己系统上编译的，更符合自己系统的性能，执行nginx服务性能效果更好。

```Javascript

============== 安装成功 ============
    
// 先安装缺少的依赖包
$ yum -y install gcc gcc-c++ make libtool zlib zlib-devel openssl openssl-devel pcre pcre-devel

// 下载源码包
$ wget https://nginx.org/download/nginx-1.14.0.tar.gz

// 解压
$ tar -zxvf nginx-1.10.1.tar.gz

// 进入 解压得到的 nginx文件目录；
$ cd nginx-1.14.0

// 在解压后的目录下，执行配置和make命令
$ ./configure --prefix=/usr/local/webserver/nginx --with-http_stub_status_module --with-http_ssl_module --with-pcre

$ make && make install


============== 使用nginx服务 =======
    
// 启动 nginx 服务
$ /usr/local/webserver/nginx/sbin/nginx 

// 停止 nginx 服务
$ /usr/local/webserver/nginx/sbin/nginx -s stop 
```



## 坑遇

### 1. nginx为新项目配置完后，访问项目，报`403` 错误，解决方案？

> **方案一：修改nginx.conf配置，设置用户权限为root。

![image-20180706232311722](https://i.loli.net/2018/07/20/5b51ef04daee5.jpg)



> 方案二：把项目放在`非`/root/目录下，并给项目目录为755权限，即可。

![image-20180706233006384](https://i.loli.net/2018/07/20/5b51f02f34dca.jpg)

```Javascript
$ chmod -R 755 noroot(非root目录名)
```



未完待续...

## 参考文档

[ nginx服务器详细安装过程（使用yum 和 源码包两种安装方式，并说明其区别）](https://segmentfault.com/a/1190000007116797)

[HyperApp 用户文档](https://www.hyperapp.fun/zh/)