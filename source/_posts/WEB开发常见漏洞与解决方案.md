---
title: WEB开发常见漏洞与解决方案
date: 2018-09-03 10:34:40
tags: 安全
---

## SQL注入
定义：通过把SQL命令插入到Web表单提交或输入域名或页面请求的查询字符串，最终达到欺骗服务器执行恶意的SQL命令

防范：SQL注入只对SQL语句的编译过程有破坏作 用，而PreparedStatement已经准备好了，执行 阶段只是把输入串作为数据处理，而不再对语句进 行解析，因此也就避免了SQL注入问题。（在Mybatis中，#{}：相当于JDBC中的PreparedStatement，经过预编译的，是安全的。${}：是输出变量的值，是未经过预编译的，仅仅是取变量的值，是非安全的，存在SQL注入。）<!--more-->

## XSS漏洞
定义：XSS是一种经常出现在web应用中的计算机安全漏洞，它允许恶意web用户将代码植入到提供给其它用户使用的页面中。比如这些代码包括HTML代码和客户端脚本。

防范： `<script>alert()</script>`可以执行任意脚本，可 以从输入、输出两个角度去考虑防御。
- 输入不处理，输出时根据应用场景统一工具类进行转义
- CSP 全称为 Content Security Policy，即内容安全策略，是一种开发者定义的安全策略声明，可以阻止恶意内容在受信 web 页面上下文中的执行，减少 XSS、clickjacking、code inject 等攻击。其主要以“白名单”的形式指定可信的内容来源，或是控制一些安全相关的选项。

## 任意文件上传/下载

定义：
- 上传：文件上传漏洞是指网络攻击者上传了一个可执行的文件到服务器并执行。这里上传的文件可以是木马，病毒，恶意脚本或者WebShell等。
- 下载：一些网站由于业务需求，可能提供文件查看或下载的功能，如果对用户查看或下载的文件不做限制，则恶意用户就能够查看或下载任意的文件，可以是源代码文件、敏感文件等，就会造成任意文件下载漏洞。

防范：
- 上传：限制目录不可执行、上传文件类型检查、上传文件大小检查
- 下载：禁止客户端自定义下载路径

## CSRF

定义：跨站请求伪造(Cross- site request forgery)，能够在用户不知情的情况下发起请求。与跨网站脚本(XSS) 相比，XSS利用的是用户对指定网站的信任， CSRF利用的是网站对用户网页浏览器的信任 。

防范：
- 请求带随机数：每次请求带有一次有效随机数(隐藏input)
- 避免跨域请求：1.校验origin、referer 2.post发送Json数据
- 浏览器的跨域策略：Double Submit Cookie

## 水平越权

定义：水平权限漏洞一般出现在一个用户对象关联多个其他对象（订单、地址等）、并且要实现对关联对象的CRUD的时候。开发容易习惯性的在生成CRUD表单（或AJAX请求）的时候根据认证过的用户身份来找出其有权限的被操作对象id，提供入口，然后让用户提交请求，并根据这个id来操作相关对象。在处理CRUD请求时，往往默认只有有权限的用户才能得到入口，进而才能操作相关对象，因此就不再校验权限了。可悲剧的是大多数对象的ID都被设置为自增整型，所以攻击者只要对相关id加1、减1、直至遍历，就可以操作其他用户所关联的对象了。

防范：把权限的控制转移到数据接口层中，要求web层在调用数据接口层的接口时额外提供userid，而这个userid在web层看来只能通过seesion来取到。这样在面对水平权限攻击时，web层的开发者不用额外花精力去注意鉴权的事情，也不需要增加一个SQL来专门判断权限——如果权限不对的话，那个and条件就满足不了，SQL自然就找不到相关对象去操作。而且这个方案对于一个接口多个地方使用的情况也比较有利，不需要每个地方都鉴权了。

## cors配置不当

定义：
1. 网站A存在cors配置不当
2. 攻击者在自己的网站B构造ajax,请求A网站的接口
3. 当用户访问B网站页面时，触发跨域
4. 请求A网站接口，返回的数据存入 攻击者网站。

防范：例如www.api.com仅允许www.example.com发起的跨域请求: httpResponse.setHeader("Access-Control-Allow-Origin","www.example.com");

## 其他风险

URL传输token：
1. 访问日志泄露用户认证凭证 
2. referer泄露用户认证凭证

日志审计:
1. 禁止记录：密码、CVV、有效期、session等C4类敏 感信息禁止打印日志。
2. 需要记录日志:时间、地点、人物、起因、经过、结果 时间、IP、username、url、内容、成功/失败
3. 日志管理:保存至少6个月以上、异地备份、管理员禁止删除日志