# Claude Code 安装&注册&升级简述

Sep 19, 2025

> **关键词：** `Claude Code` · `安装` · `注册` · `升级` · `科学上网` · `接码平台`

---

**阅读时间：** 约 5 分钟

**难度等级：** ⭐ 入门级

## 前提条件

- 科学上网
    - 美区节点
    - 开启tunnel模式
    
    ```bash
    # 查看出口ip是否为美区节点ip
    $ curl ipinfo.io
    ```
    
## 注册

1. gmail邮箱
2. [接码平台](https://sms-activate.guru/cn/buy)，获取 claude 手机验证码。
3. 完成claude账户注册


## 购买升级
### ios（亲测版）

1. 注册**尼日利亚**Appid账户
    1. 推荐使用chrome浏览器操作，其他浏览器可能会出现，手机验证码验证不通过的问题。
2. 购买AppStore尼日利亚礼品卡
    1. 咸鱼购买
3. 将礼品卡充值到 AppStore账户中
4. iphone或ipad上下载 claude ，并在app内，完成购买升级（pro、max..）
    1. 若需二次验证地址，可使用[虚拟地址生成器](https://cn.americaaddress.com/nigeria-address/)

[参考方案](https://www.youtube.com/watch?v=XZA4NjSIO_4)

### Android（非亲测）
[参考方案](https://www.youtube.com/watch?v=MV0bvyGVOkM)

## 可能的坑？ 

### **使用Pro或Max套餐访问Claude Code时遇到问题？**

如果您没有看到使用首选账户进行身份验证的选项，请按照以下步骤更新Claude Code：

1. 使用`/logout`命令完全退出您的活动会话。
2. 运行`claude update`。
3. 完全重启您的终端以使更改生效。
4. 运行`claude`并选择正确的账户来使用Claude Code。


至此，我们就可以拥有当下非常nice的AI编程助手了~ 试试吧！