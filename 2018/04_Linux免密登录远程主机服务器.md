# Linux免密登录远程主机服务器
Jul 13, 2018 
## 1. 免密登录原理分析

密钥对

私钥：相当于钥匙——🔑，用来解密。

公钥：相当于锁——🔐，用来加密。

## 2. 免密登录实现过程

### 2.1 生成密钥对

```Javascript
$ mkdir keys
$ cd keys
//-t rsa 指定一种加密算法；-C alias(指明身份标识：邮箱、花名...)；-f 文件名 (指定输出加密文件的文明名)
$ ssh-keygen -t rsa -C 'zjy' -f 'zjy_rsa' 
```

### 2.2 上传并配置公钥

```Javascript
// 上传公钥到服务器对应账号的home路径下的.ssh/中的authorized_keys文件中(写入)
$ ssh-copy-id -i '公钥文件名' 用户名@服务器IP或域名

或// scp上传——比较麻烦
$ scp '公钥文件名' 用户名@服务器IP或域名
// 把 上传后的公钥文件内容写入到authorized_keys文件中
$ cat zjy_rsa.pub >> authorized_keys //追加进去
$ cat zjy_rsa.pub > authorized_keys //覆盖进去

// 配置公钥文件访问权限为600 (不允许其他用户写)
$ chmod 600 authorized_keys(在.ssh/目录下，默认就是这个权限，不设置也可以。先查看一下)
	
// check公钥是否上传并写入成功
$ ssh 用户名@target服务器
$ cd ~ 或 cd root/
$ ll
$ cd .ssh/
$ ll
$ cat authorized_keys // 查看是否写入公钥内容
```

### 2.3 配置本地私钥

```javascript
//把第一步生成的私钥复制到.ssh/目录下，并设置私钥文件的访问权限为600(其他用户不可写入)
$ pwd
> /root/keys
$ cp zjy_rsa /root/.ssh
$ chmod 600 zjy_rsa
// 本地私钥配置完毕后，即可以初步尝试免密登录(显示的传输私钥)
$ pwd
> /root/.ssh
$ ssh -i zjy_rsa 用户名@服务器地址 (-i 私钥名：指定私钥名)
$ ssh -i .ssh/zjy_rsa 用户名@服务器地址 (只要指定对私钥地址就可以，不限制那个目录去登录)
// 免密登录目标服务器完成
```

### 2.4 免密登录功能的本地配置文件

```Javascript
//添加本地config文件
$ touch config
//设置配置文件的访问权限为600(其他用户不可写入)
$ chmod 600 config
// 在config中添加登录远程服务器配置
$ vim config
// config内容
User root
Host szj
HostName 目标服务器(域名或IP)
//Port 80
StrictHostKeyChecking no
IdentityFile ~/.ssh/zjy_rsa
IdentitiesOnly yes
Protocol 2
Compression yes
ServerAliveInterval 60
ServerAliveCountMax 20
LogLevel INFO
// 再次尝试登录另一台远程服务器(eg：szj)
$ ssh szj (在任意目录下 执行 ssh 目标服务器配置的别名)
// 完成登录；cool~~ :D
```

