---
title: 学习RESTful API 设计
author: 梅溪
comments: true
layout: post
slug: 
tags : [WEB]
date: 2015-09-24 09:29:54
---


# REST: What and Why

## REST含义

REST的全称是Representational State Transfer。字面上的意义看起来很难懂。

不过不要担心，这正是计算机中很典型的“字面看上去比实际含义要复杂100万倍”现象。

我们还是先从字面看，

-   首先representational是具象的而非抽象的，"Re"强调的是对资源的表现；
-   State是状态，强调资源是有状态的；
-   Transfer是状态迁移。

RESTful API字面的含义，就是强调应用或者服务API的调用，体现的是资源状态的迁移。
<!--more-->
概念很枯燥，举例就很容易理解了。一个关于销售情况的REST API的形式可能是

    http://foo.bar/sales

当浏览器对这个URL进行HTTP GET请求，就是完成了一次REST API的获取调用。

当浏览器对它进行HTTP DELETE请求，就是完成了一次REST API的删除调用。

以此类推。

这其中，销售情况作为一种资源，浏览器的请求是调用，体现的是销售这种资源的有状态的迁移。

## 为什么人人都爱REST？

软件应用在服务化，笨重的接口形式早已是过去式，我们早已习惯了打开任何终端上的任一个浏览器都能使用我们
想要的应用或者服务。

REST为我们提供了一种轻量的，松散的，容易维持不变的连接应用的形式。

在我看来，使用RESTful API有这些好处：

-   简单   API形式本身就是一个很直观的URL。而HTTP协议有几个动作？GET，POST，DELETE，PUT而已。
-   统一   不论什么终端，什么软件系统，什么数据库，什么前后端技术……RESTful API可以做到始终如一
-   低耦   因为形式的简单和HTTP协议的标准化，REST可以带来应用间的低耦合。功能扩展，版本升级;
-   高效   得益于现有WEB技术的发展和形式上的简单，RESTful API的调用过程可以非常高效

下面我们讨论怎样设计更REST的API。RESTful 是一种风格，而不是具体技术或架构。

# URL不要有动词

这一点是所有RESTful 的教程都首先提到的一点。

和传统SOAP API不同，REST的接口形式本身不强调行为，只表示资源本身。

调用行为仅限于HTTP协议本身所提供的有限几种。

所以一个合理的RESTful API从形式上应该是看不到动词的。

可是很遗憾，我们还是经常能看到带着动词REST API。

比如我现在的项目中就有同学写了这样的代码(略作改动)：

    @RequestMapping(value = "/mr/getMRJob", method = RequestMethod.GET)

他期望的是一个获得MapReduce Job的功能。
很奇怪不是么？同时出现了getMRJob和GET method，重复的get语义。
更RESTful的写法是下面这样。

    @RequestMapping(value = "/mr/MRJob", method = RequestMethod.GET)

如果你觉得上面的例子还好的话，请看看这个：

    @RequestMapping(value = "/mr/deleteMRJob", method = RequestMethod.GET)

啊，一个看起来像是删除的URL，一个GET方法……

考虑各种行为，一系列更REST的API调用应该是像这样：

-   GET /MRJobs : 返回所有的MapReduce作业
-   POST /MRJobs : 加入MapReduce作业
-   GET /MRJobs/Messi : 获得名为Messi的MapReduce作业
-   PATCH/PUT /MRJobs/Messi : 更新名为Messi的MapReduce作业

总之，虽然API形式中带着行为符合传统应用接口的习惯，但REST并不该是这样。REST表现的是资源本身，要尽量避免动词的出现。

# 服务的无状态

严格意义上，一个好的REST 的API服务应该是无状态的。这里的无状态指的是服务不需要去维护客户端的状态
(REST所表示资源本身当然是有状态的，不然就没有"S"了),每次客户端的请求都是独立的，服务不需要记录请求的
相关信息。

无状态的服务意味着没有羁绊，一样的输入就有一样的输出，是可以被简单地迁移或扩充的，负载分担和高可靠性
都好做。这也是无状态被提倡的原因。

# 返回数据

## 资源的表现形式

REST的API不应该拘泥于特定的资源表现形式。

foo.bar/userinfo.html 就不是一个REST该有的样子。

foo.bar/userinfo 更REST一些。返回的资源可以是Json串，可以是Html，也可以是XML。

不要把一切都框死。

更进一步地，如果你的服务可以做到根据HTTP请求中的Accept属性要求返回不同格式就更酷了。

## 对正确和错误返回统一的结构

这可能是一种好的习惯，就是对正确和错误情况给出相同结构的返回数据，便于客户端的处理。
正确时返回data，没有错误信息：

    {
         "meta":{
          "code":200
       },
       "data":{
          "MRJob":"FooMR Job"
       }
    }

出错时data为空：

    {
         "meta":{
          "code":500,
          "error":"Foo bar internal error",
          "info":"com.dtdream.FooMapper"
       },
       "data":{
       }
    }

# 错误码

HTTP协议规定了错误码，比如200是成功，500是服务器错误，401告诉客户端需要登录……

如果使用HTTP的错误码，请不要偏离它本来的含义。

[HTTP错误码定义请看这里](http://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html)

如果使用自定义错误码（业务往往复杂到HTTP那几个错误码不够用，这很正常），最好不要和HTTP错误码冲突。

# 版本

世界在变。我们的服务会有变化，API的版本可能也会有变化。我们当然希望我们的新版本可以自动兼容旧版本的
功能，但是这并非永远可以做到。

关于REST API，很多人提倡在URI中体现版本号。

    http://foo.bar/v1/MRjob

或者作为参数传递，这其实没有不同：

    http://foo.bar/v1/MRjob?version=1

很显然，根据前面我们介绍过的内容，你大概也能推测出，在URI或参数中带着版本号这不够”REST“。

的确，更REST的做法，还是在HTTP请求中带着Accept的版本信息，URI是恒定的。

不过，这对客户端有更多要求，我个人觉得反而还是在URI中指定版本更清晰。

在实际生产中，很可能不同版本的APIAPI在URI形式、参数上都存在很大差异。

比如国内某互联网巨头(A)的认证系统，获得AccessKey的API，v1和v2版本就大相径庭：

v1版本:

    GET /user HTTP/1.1
    Authorization: UMM 15B4D3461F177624206A:xQE0diMbLRepdf3YB+FIEXAMPLE=
    Host: SERVER IP
    Date: Wed, 01 Mar 2011 12:00:00 GMT
    METHOD: GET_ACCESS_KEY
    ACCESS_ID: kdbl1xexsbxi8at0cov84gif

v2 版本:

    GET /accesskeys/accesskeyid?fields=status HTTP 1.1
    Date: date
    Authorization: signature

这显然就给客户端实现带来极大困难，而且他们自身的一个复杂系统中，有的地方用v1版本，有的地方用v2版本，
也没有显式说明，很容易出现问题（别问我是怎么知道的）。

# 参考资料

-   [最佳实践：更好的设计你的 REST API](http://www.ibm.com/developerworks/cn/web/1103_chenyan_restapi/)
-   [对于REST中无状态(stateless)的一点认识](http://developer.51cto.com/art/200906/129424.htm)
-   [RESTful API版本控制策略](http://ningandjiao.iteye.com/blog/1990004)
