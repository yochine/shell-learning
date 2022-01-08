

本篇文章介绍一下OAuth2.0相关的知识点，并且手把手带大家搭建一个认证授权中心、资源服务进行OAuth2.0四种授权模式的验证，案例源码详细，一梭子带大家了解清楚。本篇文章的案例源码项目架构为：Spring Boot + Spring Cloud Alibaba + Spring Security ，Spring Cloud Alibaba有不了解可以看下陈某的往期[《Spring Cloud 进阶》](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzU3MDAzNDg1MA==&action=getalbum&album_id=2042874937312346114#wechat_redirect)文章：

- [五十五张图告诉你微服务的灵魂摆渡者Nacos究竟有多强？](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247493854&idx=1&sn=4b3fb7f7e17a76000733899f511ef915&scene=21#wechat_redirect)
- [openFeign夺命连环9问，这谁受得了？](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247496653&idx=1&sn=7185077b3bdc1d094aef645d677ec472&scene=21#wechat_redirect)
- [阿里面试这样问：Nacos、Apollo、Config配置中心如何选型？这10个维度告诉你！](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247496772&idx=1&sn=8a88b998920bb9b665f52320cf94d9c7&scene=21#wechat_redirect)
- [阿里面试败北：5种微服务注册中心如何选型？这几个维度告诉你！](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247497371&idx=1&sn=df5aa872452970f5f46efff5fc777b34&scene=21#wechat_redirect)
- [阿里限流神器Sentinel夺命连环 17 问？](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247498039&idx=1&sn=3a3caee655ff015b46249bd51aa4dc79&scene=21#wechat_redirect)
- [对比7种分布式事务方案，还是偏爱阿里开源的Seata，真香！\(原理+实战\)](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247499421&idx=1&sn=a55797652284bafd9216ea981f4125e0&scene=21#wechat_redirect)
- [Spring Cloud Gateway夺命连环10问？](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247499894&idx=1&sn=f1606e4c00fd15292269afe052f5bca2&chksm=fcf71fbbcb8096ad349e6da50b0b9141964c2084d0a38eba977fe8baa3fbe8af3b20c7591110&token=1887105114&lang=zh_CN&scene=21#wechat_redirect)
- [Spring Cloud Gateway 整合阿里 Sentinel网关限流实战！](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247500540&idx=1&sn=2967bf1f9fa2c4d5b94b7fe291b7869b&chksm=fcf71d31cb8094271b5bbeca85cc03cf7d7c8bbf7c4d5b83a539e39c956e3f4015f91e7742b6&token=2077958771&lang=zh_CN&scene=21#wechat_redirect)
- [分布式链路追踪之Spring Cloud Sleuth夺命连环9问？](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247500757&idx=1&sn=ef71f10d5736029c92287f7894c842ed&chksm=fcf71c18cb80950ec3144d66957a9d2914b51b8af22cf59a1ca3f9e2b561a11aca359d50d5a1&token=498818367&lang=zh_CN&scene=21#wechat_redirect)


文章目录如下：

![](https://mmbiz.qpic.cn/mmbiz_png/19cc2hfD2rCpPJB83SvgzosiboTJxftAhnqnhKLMN9ia07tCDnA9o0ohHfAR9fRboOV0uib3HauIYpToOe3Er6ibxw/640?wx_fmt=png)

## 为什么需要OAuth2.0？

编码永远都是为了解决生产中的问题，想要理解为什么需要OAuth2，当然要从实际生活出发。举个例子：小区的业主点了一份外卖，但是小区的门禁系统不给外卖人员进入，此时想要外卖员进入只能业主下来开门或者告知门禁的密码。密码告知外卖员岂不是每次都能凭密码进入小区了，这明显造成了安全隐患。那么有没有一种方案：既能不泄露密码，也能让外卖小哥进入呢？于是此时就想到了一个授权机制，分为以下几个步骤：

1.  门禁系统中新增一个授权按钮，外卖小哥只需要点击授权按钮呼叫对应业主
2.  业主收到小哥的呼叫，知道小哥正在要求授权，于是做出了应答授权
3.  此时门禁系统弹出一个密码（类似于access\_token），有效期30分钟，在30分钟内，小哥可以凭借这个密码进入小区。
4.  小哥输入密码进入小区

另外这个授权的密码不仅可以通过门禁，还可以通过楼下的门禁，这就非常类似于网关和微服务了。

## 令牌和密码的区别？

上述例子中令牌和密码的作用是一样的，都可以进入小区，但是存在以下几点差异：

1.  时效不同：令牌一般都是存在过期时间的，比如30分钟后失效，这个是无法修改的，除非重新申请授权；而密码一般都是永久的，除非主人去修改
2.  权限不同：令牌的权限是有限的，比如上述例子中，小哥获取了令牌，能够打开小区的门禁、业主所在的楼下门禁，但是可能无法打开其它幢的门禁；
3.  令牌可以撤销：业主可以撤销这个令牌的授权，一旦撤销了，这个令牌也就失效了，无法使用；但是密码一般不允许撤销。

## 什么是OAuth2？

OAuth 是一个开放标准，该标准允许用户让第三方应用访问该用户在某一网站上存储的私密资源（如头像、照片、视频等），而在这个过程中无需将用户名和密码提供给第三方应用。实现这一功能是通过提供一个令牌（token），而不是用户名和密码来访问他们存放在特定服务提供者的数据。采用令牌（token）的方式可以让用户灵活的对第三方应用授权或者收回权限。OAuth2 是 OAuth 协议的下一版本，但不向下兼容 OAuth 1.0。传统的 Web 开发登录认证一般都是基于 session 的，但是在前后端分离的架构中继续使用 session 就会有许多不便，因为移动端（Android、iOS、微信小程序等）要么不支持 cookie（微信小程序），要么使用非常不便，对于这些问题，使用 OAuth2 认证都能解决。对于大家而言，我们在互联网应用中最常见的 OAuth2 应该就是各种第三方登录了，例如 QQ 授权登录、微信授权登录、微博授权登录、GitHub 授权登录等等。

## OAuth2.0的四种模式？

OAuth2.0协议一共支持 4 种不同的授权模式：

1.  授权码模式：常见的第三方平台登录功能基本都是使用这种模式。
2.  简化模式：简化模式是不需要客户端服务器参与，直接在浏览器中向授权服务器申请令牌（token），一般如果网站是纯静态页面则可以采用这种方式。
3.  密码模式：密码模式是用户把用户名密码直接告诉客户端，客户端使用说这些信息向授权服务器申请令牌（token）。这需要用户对客户端高度信任，例如客户端应用和服务提供商就是同一家公司，自己做前后端分离登录就可以采用这种模式。
4.  客户端模式：客户端模式是指客户端使用自己的名义而不是用户的名义向服务提供者申请授权，严格来说，客户端模式并不能算作 OAuth 协议要解决的问题的一种解决方案，但是，对于开发者而言，在一些前后端分离应用或者为移动端提供的认证授权服务器上使用这种模式还是非常方便的。

### 1、授权码模式

这种方式是最常用的流程，安全性也最高，它适用于那些有后端的 Web 应用。授权码通过前端传送，令牌则是储存在后端，而且所有与资源服务器的通信都在后端完成。这样的前后端分离，可以避免令牌泄漏。令牌获取的流程如下：

![授权码模式](https://mmbiz.qpic.cn/mmbiz_png/19cc2hfD2rCpPJB83SvgzosiboTJxftAhibgOZffmU9RnmNUusomvBtoUKaxEXIU1df2icbUZOwSUeG4G0DxWgjtQ/640?wx_fmt=png)

上图中涉及到两个角色，分别是客户端、认证中心，客户端负责拿令牌，认证中心负责发放令牌。但是不是所有客户端都有权限请求令牌的，需要事先在认证中心申请，比如微信并不是所有网站都能直接接入，而是要去微信后台开通这个权限。至少要提前向认证中心申请的几个参数如下：

1.  client\_id：客户端唯一id，认证中心颁发的唯一标识
2.  client\_secret：客户端的秘钥，相当于密码
3.  scope：客户端的权限
4.  redirect\_uri：授权码模式使用的跳转uri，需要事先告知认证中心。

1、请求授权码客户端需要向认证中心拿到授权码，比如第三方登录使用微信，扫一扫登录那一步就是向微信的认证中心获取授权码。请求的url如下：

`/oauth/authorize?client_id=&response_type=code&scope=&redirect_uri=  
`

上述这个url中携带的几个参数如下：

- client\_id：客户端的id，这个由认证中心分配，并不是所有的客户端都能随意接入认证中心
- response\_type：固定值为code，表示要求返回授权码。
- scope：表示要求的授权范围，客户端的权限
- redirect\_uri：跳转的uri，认证中心同意或者拒绝授权跳转的地址，如果同意会在uri后面携带一个`code=xxx`，这就是授权码

2、返回授权码第1步请求之后，认证中心会要求登录、是否同意授权，用户同意授权之后直接跳转到`redirect_uri`（这个需要事先在认证中心申请配置），授权码会携带在这个地址后面，如下：

`http://xxxx?code=NMoj5y  
`

上述链接中的`NMoj5y`就是授权码了。3、请求令牌客户端拿到授权码之后，直接携带授权码发送请求给认证中心获取令牌，请求的url如下：

`/oauth/token?  
 client_id=&  
 client_secret=&  
 grant_type=authorization_code&  
 code=NMoj5y&  
 redirect_uri=  
`

相同的参数同上，不同参数解析如下：

- grant\_type：授权类型，授权码固定的值为authorization\_code
- code：这个就是上一步获取的授权码

4、返回令牌认证中心收到令牌请求之后，通过之后，会返回一段JSON数据，其中包含了令牌access\_token，如下：

`{      
  "access_token":"ACCESS_TOKEN",  
  "token_type":"bearer",  
  "expires_in":2592000,  
  "refresh_token":"REFRESH_TOKEN",  
  "scope":"read",  
  "uid":100101  
}  
`

access\_token则是颁发的令牌，refresh\_token是刷新令牌，一旦令牌失效则携带这个令牌进行刷新。

### 2、简化模式

这种模式不常用，主要针对那些无后台的系统，直接通过web跳转授权，流程如下图：

![简化模式](https://mmbiz.qpic.cn/mmbiz_png/19cc2hfD2rCpPJB83SvgzosiboTJxftAhxGsEsTxPIovmxbYqEqregcCE7o0h7fvcjkGSrdtXqUxFfs4EwqbegQ/640?wx_fmt=png)

这种方式把令牌直接传给前端，是很不安全的。因此，只能用于一些安全要求不高的场景，并且令牌的有效期必须非常短，通常就是会话期间（session）有效，浏览器关掉，令牌就失效了。1、请求令牌客户端直接请求令牌，请求的url如下：

`/oauth/authorize?  
  response_type=token&  
  client_id=CLIENT_ID&  
  redirect_uri=CALLBACK_URL&  
  scope=  
`

这个url正是授权码模式中获取授权码的url，各个参数解析如下：

- client\_id：客户端的唯一Id
- response\_type：简化模式的固定值为token
- scope：客户端的权限
- redirect\_uri：跳转的uri，这里后面携带的直接是令牌，不是授权码了。

2、返回令牌认证中心认证通过后，会跳转到redirect\_uri，并且后面携带着令牌，链接如下：

`https://xxxx#token=NPmdj5  
`

`#token=NPmdj5`这一段后面携带的就是认证中心携带的，令牌为NPmdj5。

### 3、密码模式

密码模式也很简单，直接通过用户名、密码获取令牌，流程如下：

![密码模式](https://mmbiz.qpic.cn/mmbiz_png/19cc2hfD2rCpPJB83SvgzosiboTJxftAhxGsEsTxPIovmxbYqEqregcCE7o0h7fvcjkGSrdtXqUxFfs4EwqbegQ/640?wx_fmt=png)

1、请求令牌认证中心要求客户端输入用户名、密码，认证成功则颁发令牌，请求的url如下：

```
/oauth/token?  grant_type=password&  username=&  password=&  client_id=&  client_secret=
```

参数解析如下：

- grant\_type：授权类型，密码模式固定值为password
- username：用户名
- password：密码
- client\_id：客户端id
- client\_secret：客户端的秘钥

2、返回令牌上述认证通过，直接返回JSON数据，不需要跳转，如下：

`{      
  "access_token":"ACCESS_TOKEN",  
  "token_type":"bearer",  
  "expires_in":2592000,  
  "refresh_token":"REFRESH_TOKEN",  
  "scope":"read",  
  "uid":100101  
}  
`

access\_token则是颁发的令牌，refresh\_token是刷新令牌，一旦令牌失效则携带这个令牌进行刷新。

### 4、客户端模式

适用于没有前端的命令行应用，即在命令行下请求令牌。这种方式给出的令牌，是针对第三方应用的，而不是针对用户的，即有可能多个用户共享同一个令牌。流程如下：

![客户端模式](https://mmbiz.qpic.cn/mmbiz_png/19cc2hfD2rCpPJB83SvgzosiboTJxftAhxGsEsTxPIovmxbYqEqregcCE7o0h7fvcjkGSrdtXqUxFfs4EwqbegQ/640?wx_fmt=png)

1、请求令牌请求的url为如下：

`/oauth/token?  
grant_type=client_credentials&  
client_id=&  
client_secret=  
`

参数解析如下：

- grant\_type：授权类型，客户端模式固定值为client\_credentials
- client\_id：客户端id
- client\_secret：客户端秘钥

2、返回令牌认证成功后直接返回令牌，格式为JSON数据，如下：

`{  
    "access_token": "ACCESS_TOKEN",  
    "token_type": "bearer",  
    "expires_in": 7200,  
    "scope": "all"  
}  
`

## OAuth2.0的认证中心搭建

为了方便测试OAuth2的四种授权模式，这里为了方便测试，简单搭建一个认证中心，后续会逐渐完善。

### 1、案例架构

陈某使用的是Spring Boot + Spring Cloud Alibaba 作为基础搭建，新建一个`oauth2-auth-server-in-memory`模块作为认证中心，目录如下：

![](https://mmbiz.qpic.cn/mmbiz_png/19cc2hfD2rCpPJB83SvgzosiboTJxftAhE9rgwic3jBnUwuruUiamWjb7rSrxaEic3YHzO7pzwfdfj9mWsnDybxVicg/640?wx_fmt=png)

### 2、添加依赖

Spring Boot 和 Spring Cloud 的相关依赖这里陈某就不再说了，直接上Spring Security和OAuth2的依赖，如下：

`<!--spring security的依赖-->  
<dependency>  
   <groupId>org.springframework.boot</groupId>  
 <artifactId>spring-boot-starter-security</artifactId>  
</dependency>  
  
<!--OAuth2的依赖-->  
<dependency>  
 <groupId>org.springframework.cloud</groupId>  
 <artifactId>spring-cloud-starter-oauth2</artifactId>  
</dependency>  
`

### 3、Spring Security安全配置

这里主要涉及到Spring Security的配置，有不清楚的可以陈某第一篇文章：[实战！Spring Boot Security+JWT前后端分离架构登录认证！](https://mp.weixin.qq.com/s?__biz=MzU3MDAzNDg1MA==&mid=2247502546&idx=1&sn=bfb6fd9d96d8c5bf107a4981ba5e1547&chksm=fcf7151fcb809c09b7ae29de8c0af0d00976539a46ee5f9bf583a6a7b196ea82f26ce98fd982&token=869584969&lang=zh_CN&scene=21#wechat_redirect)`SecurityConfig`这个配置类中主要设置有4块内容，如下：1、加密方式采用BCryptPasswordEncoder加密，如下：

![](https://mmbiz.qpic.cn/mmbiz_png/19cc2hfD2rCpPJB83SvgzosiboTJxftAhvLNdeRwoZXLcV6QCmPcyJJgbs589o1CV5McNoFcvlkjgUd7j6BCgiaQ/640?wx_fmt=png)

2、配置用户这里为了方便测试，直接将用户信息存储在内存中，后续完善，代码如下：

![](https://mmbiz.qpic.cn/mmbiz_png/19cc2hfD2rCpPJB83SvgzosiboTJxftAht66ZPibEH2AsBicQHKaibWickD8nuEMWmQbLND242scvrD68iaxOpPJu1KA/640?wx_fmt=png)

上述代码配置了两个用户，如下：

- 用户名admin，密码123，角色admin
- 用户名user，密码123，角色user

3、注入认证管理器AuthenticationManager`AuthenticationManager`在密码授权模式下会用到，这里提前注入，如果你用的不是密码模式，可以不注入，代码如下：

![](https://mmbiz.qpic.cn/mmbiz_png/19cc2hfD2rCpPJB83SvgzosiboTJxftAhYpKwjQxlarnhcMMDR9NZZpTxyo8ib938yia9icSiangl0h7qSTLueNysng/640?wx_fmt=png)

4、配置安全拦截策略由于需要验证授权码模式，因此开启表单提交模式，所有url都需要认证，代码如下：

![](https://mmbiz.qpic.cn/mmbiz_png/19cc2hfD2rCpPJB83SvgzosiboTJxftAhQ4YxVZu4TLV4GWBavLH028LFIShMuylePiaYhkS2je5R1033YVbKkicQ/640?wx_fmt=png)

### 4、令牌存储策略配置

令牌支持多种方式存储，比如内存方式、Redis、JWT，比较常用的两种则是Redis、JWT。这里暂时使用内存存储的方式，一旦服务器重启令牌将会失效。代码如下：

![](https://mmbiz.qpic.cn/mmbiz_png/19cc2hfD2rCpPJB83SvgzosiboTJxftAhWoEicrFQZHIkkGhjmfsHWrxeSOAAqsMIuicVfBWKBZcNBNTzfvA7nz2g/640?wx_fmt=png)

### 5、OAuth2.0的配置类

不是所有配置类都可以作为OAuth2.0认证中心的配置类，需要满足以下两点：

1.  继承AuthorizationServerConfigurerAdapter
2.  标注 \@EnableAuthorizationServer 注解

代码如下：

![](https://mmbiz.qpic.cn/mmbiz_png/19cc2hfD2rCpPJB83SvgzosiboTJxftAhqx6iafMj4hvl7WQt37aZvyRR3kCBicic7fUB5EKNohy0Fx6zDxAsWiaz9w/640?wx_fmt=png)

AuthorizationServerConfigurerAdapter需要实现的三个方法如下：

![](https://mmbiz.qpic.cn/mmbiz_png/19cc2hfD2rCpPJB83SvgzosiboTJxftAhuzwusRJYgquibYKPJA0mBQLHVicm76E3xTXibx2TUAcUcs1bFXiaPY68HQ/640?wx_fmt=png)

下面便是围绕这三个方法进行OAuth2的详细配置。

### 6、客户端配置

在介绍OAuth2.0 协议的时候介绍到，并不是所有的客户端都有权限向认证中心申请令牌的，首先认证中心要知道你是谁，你有什么资格？因此一些必要的配置是要认证中心分配给你的，比如客户端唯一Id、秘钥、权限。客户端配置的存储也支持多种方式，比如内存、数据库，对应的接口为：org.springframework.security.oauth2.provider.ClientDetailsService，接口如下：

![](https://mmbiz.qpic.cn/mmbiz_png/19cc2hfD2rCpPJB83SvgzosiboTJxftAhkpFKgtFJONrsToKAWdYguWfqoCK3YjxYKR9xuqwj2PNoNZ6Td4iaDbA/640?wx_fmt=png)

同样这里为了方便测试，依然是加载在内存中，后续完善，完整的配置如下：

![](https://mmbiz.qpic.cn/mmbiz_png/19cc2hfD2rCpPJB83SvgzosiboTJxftAhXKicEmnzwKwoPqzicnEH0K3hIgN1otvsFQrmI3KuVyaBLN2zN9IfTtnw/640?wx_fmt=png)

几个重要参数说一下，如下：

- `.withClient("myjszl")`：指定客户端唯一ID为myjszl
- `.secret()`：指定秘钥，使用加密算法加密了，秘钥为123
- `.resourceIds("res1")`：给客户端分配的资源权限，对应的是资源服务，比如订单这个微服务就可以看成一个资源，作为客户端肯定不是所有资源都能访问。
- `authorizedGrantTypes()`：定义认证中心支持的授权类型，总共支持五种

  - 授权码模式：authorization\_code
  - 密码模式：password
  - 客户端模式：client\_credentials
  - 简化模式：implicit
  - 令牌刷新：refresh\_token，这并不是OAuth2的模式，定义这个表示认证中心支持令牌刷新

- `scopes()`：定义客户端的权限，这里只是一个标识，资源服务可以根据这个权限进行鉴权。
- `autoApprove`：是否需要授权，设置为true则不需要用户点击确认授权直接返回授权码
- `redirectUris`：跳转的uri

### 7、授权码服务配置

使用授权码模式必须配置一个授权码服务，用来颁布和删除授权码，当然授权码也支持多种方式存储，比如内存，数据库，这里暂时使用内存方式存储，代码如下：

![](https://mmbiz.qpic.cn/mmbiz_png/19cc2hfD2rCpPJB83SvgzosiboTJxftAhDwRGytTsAibxhJ823oUMJe5XK1cty21GG7ibggsicXDSIIMo1ZnmicuSjA/640?wx_fmt=png)

### 8、令牌服务的配置

除了令牌的存储策略需要配置，还需要配置令牌的服务`AuthorizationServerTokenServices`用来创建、获取、刷新令牌，代码如下：

![](https://mmbiz.qpic.cn/mmbiz_png/19cc2hfD2rCpPJB83SvgzosiboTJxftAhuAjNia6zJWK4O1sXYUYx5Fxq9qqTSwibI8YUKgym7sjZAtgp728N4Atg/640?wx_fmt=png)

### 9、令牌访问端点的配置

目前这里仅仅配置了四个，分别如下：

- 配置了授权码模式所需要的服务，AuthorizationCodeServices
- 配置了密码模式所需要的AuthenticationManager
- 配置了令牌管理服务，AuthorizationServerTokenServices
- 配置`/oauth/token`申请令牌的uri只允许POST提交。

详细代码如下：

![](https://mmbiz.qpic.cn/mmbiz_png/19cc2hfD2rCpPJB83SvgzosiboTJxftAhuR05ogu7veesYbzC9mntvswZmLabJJrXR3pWJSAceaQ2quWukFzCJA/640?wx_fmt=png)

spring Security框架默认的访问端点有如下6个：

- /oauth/authorize：获取授权码的端点
- /oauth/token：获取令牌端点。
- /oauth/confifirm\_access：用户确认授权提交端点。
- /oauth/error：授权服务错误信息端点。
- /oauth/check\_token：用于资源服务访问的令牌解析端点。
- /oauth/token\_key：提供公有密匙的端点，如果你使用JWT令牌的话。

当然如果业务要求需要改变这些默认的端点的url，也是可以修改的，`AuthorizationServerEndpointsConfigurer`有一个方法，如下：

`public AuthorizationServerEndpointsConfigurer pathMapping(String defaultPath, String customPath)  
`

第一个参数：需要替换的默认端点url第二个参数：自定义的端点url

### 10、令牌访问安全约束配置

主要对一些端点的权限进行配置，代码如下：

![](https://mmbiz.qpic.cn/mmbiz_png/19cc2hfD2rCpPJB83SvgzosiboTJxftAhn4ajkggMp7baMMCCTmWgibIkiazPice7Fmk7JLWajjssuMibrTlQlvicn9w/640?wx_fmt=png)

## OAuth2.0的资源服务搭建

客户端申请令牌的目的就是为了访问资源，当然这个资源也是分权限的，一个令牌不是所有资源都能访问的。在认证中心搭建的第6步配置客户端详情的时候，一行代码`.resourceIds("res1")`则指定了能够访问的资源，可以配置多个，这里的res1则是唯一对应一个资源。

### 1、案例架构

陈某使用的是Spring Boot + Spring Cloud Alibaba 作为基础搭建，新建一个`oauth2-auth-resource-in-memory`模块作为认证中心，目录如下：

![](https://mmbiz.qpic.cn/mmbiz_png/19cc2hfD2rCpPJB83SvgzosiboTJxftAhOgK0Yh6I4K9qSfjfNWV1IrvDt0yGYysHzLptlOpP2BfB0Bj7aBiaaEA/640?wx_fmt=png)


### 2、OAuth2.0的配置类

作为资源服务的配置类必须满足两个条件，如下：

- 标注注解`@EnableResourceServer`
- 继承`ResourceServerConfigurerAdapter`

代码如下：

![](https://mmbiz.qpic.cn/mmbiz_png/19cc2hfD2rCpPJB83SvgzosiboTJxftAhKYxFeHnXTVfhunduVmgGJjIUBVDTNF5ibLrb1EEia8WLhjibA2RYfHIdQ/640?wx_fmt=png)

### 3、令牌校验服务配置

由于认证中心使用的令牌存储策略是在内存中的，因此服务端必须远程调用认证中心的校验令牌端点\*\*/oauth/check\_token\*\*进行校验。代码如下：

![](https://mmbiz.qpic.cn/mmbiz_png/19cc2hfD2rCpPJB83SvgzosiboTJxftAhMU8IHEQ1NObyF8zxQaFasDAiaBVN79Ho2A3WER5fiag3crxWOibE2RgPw/640?wx_fmt=png)

> “注意：远程校验令牌存在性能问题，但是后续使用JWT令牌则本地即可进行校验，不必远程校验了。”

### 4、配置客户端唯一id和令牌校验服务

上文说到客户端有一个唯一标识，因此需要配置上，代码如下：

![](https://mmbiz.qpic.cn/mmbiz_png/19cc2hfD2rCpPJB83SvgzosiboTJxftAhpBIqSFWMfTS3OqtLWzOk8icPZ3ribbUfloBF6h0hSX7OeuKKdHyVx5iaA/640?wx_fmt=png)

### 5、配置security的安全机制

上文在认证中心的第6步配置客户端详情那里，有一行代码`.scopes("all")`则是指定了客户端的权限，资源服务可以根据这个scope进行url的拦截。拦截方式如下：

`.access("#oauth2.hasScope('')")  
`

详细配置代码如下：

![](https://mmbiz.qpic.cn/mmbiz_png/19cc2hfD2rCpPJB83SvgzosiboTJxftAhwcW61B9d5sjmB2OyqyZv4eiaKhvr1E37YCcwOibfR3RyxltxeeHFg9zQ/640?wx_fmt=png)

这里陈某配置了所有路径都需要all的权限。

### 6、新建测试接口

新建了两个接口，如下：

- `/hello`：认证成功都可以访问
- `/admin`：只有具备ROLE\_admin角色的用户才可以访问

![](https://mmbiz.qpic.cn/mmbiz_png/19cc2hfD2rCpPJB83SvgzosiboTJxftAh1H4IoFqePz9Nk47tChS7M37hnuA1aVtiaZQx0A8LO2IYrrYSzgKwDQQ/640?wx_fmt=png)

## OAuth2.0的四种模式测试

下面结合认证中心、资源服务对OAuth2.0的四种服务进行测试。启动上述搭建的认证中心和资源服务，如下图：

![](https://mmbiz.qpic.cn/mmbiz_png/19cc2hfD2rCpPJB83SvgzosiboTJxftAhTicj2cERwHQurUEtBXDZwN2vicfpeJKX9VJE6ictckWAdr0Ig2rFiaJ7cg/640?wx_fmt=png)

### 授权码模式

1、获取授权码请求的url如下：

`http://localhost:2003/auth-server/oauth/authorize?client_id=myjszl&response_type=code&scope=all&redirect_uri=http://www.baidu.com  
`

浏览器访问，security需要登录，如下：

![](https://mmbiz.qpic.cn/mmbiz_png/19cc2hfD2rCpPJB83SvgzosiboTJxftAho7zkicykkZBuyCES6XcMOqKoW74Y9KLfytAZRD2r8YRqqzzY2D5XoEA/640?wx_fmt=png)

输入用户名user，密码123，成功登录。此时来到了确认授权的页面，如下：

![](https://mmbiz.qpic.cn/mmbiz_png/19cc2hfD2rCpPJB83SvgzosiboTJxftAhGVHWvJH2vwy5xXq8K1gbFG1TIofmGWQEPX2r2vic6CN89YMVKlP9gFQ/640?wx_fmt=png)

选择Apporove、确认授权，成功跳转到了百度页面，并且携带了授权码，如下：

![](https://mmbiz.qpic.cn/mmbiz_png/19cc2hfD2rCpPJB83SvgzosiboTJxftAhu4zr9QkXLWDa0TMLXbsjia2V3OqxEkOEBGaaHJibicTdSeDmfiaakVsdEA/640?wx_fmt=png)

这里的6yV2bF就是获取到的授权码。2、获取token

`http://localhost:2003/auth-server/oauth/token?code=jvMH5U&client_id=myjszl&client_secret=123&redirect_uri=http://www.baidu.com&grant_type=authorization_code  
`

注意：/oauth/token获取token的接口请求允许的方式要配置在授权服务器中，比如配置POST方式，代码如下：

`.allowedTokenEndpointRequestMethods(HttpMethod.POST)  
`

POSTMAN请求如下图：

![](https://mmbiz.qpic.cn/mmbiz_png/19cc2hfD2rCpPJB83SvgzosiboTJxftAhfqLCGB0bBcnibWULDnabW4YhK1xFSc8I9micmgDia1fxt2icPjbWtbmlKQ/640?wx_fmt=png)

3、访问资源服务拿着令牌访问资源服务的\*\*/hello\*\*接口，请求如下：

![](https://mmbiz.qpic.cn/mmbiz_png/19cc2hfD2rCpPJB83SvgzosiboTJxftAhK2VR78aWEPdUWt7jbOSCIEPbMlwia0gvnw9uoTen3JTooxqjWJAxMYQ/640?wx_fmt=png)

请求头需要添加Authorization，并且值为Bearer+" "+access\_token的形式。注意：Bearer后面一定要跟一个空格。

### 密码模式

密码模式比较简单，不用先获取授权码，直接使用用户名、密码获取token。POSTMAN请求如下：

![](https://mmbiz.qpic.cn/mmbiz_png/19cc2hfD2rCpPJB83SvgzosiboTJxftAhibYpSPiaYMBLpGOoPdcIWfzICcRDylxKNEfBP7OibmnZ85EpqGcr1O0jA/640?wx_fmt=png)

PS：访问资源自己拿着获取到的令牌尝试下.....

### 简化模式

简化模式就很简单了，拿着客户端id就可以获取token，请求的url如下：

`http://localhost:2003/auth-server/oauth/authorize?response_type=token&client_id=myjszl&redirect_uri=http://www.baidu.com&scope=all  
`

这个过程和获取授权码一样，需要登录，同意授权最终跳转到百度，链接后面直接携带了令牌，如下：

![](https://mmbiz.qpic.cn/mmbiz_png/19cc2hfD2rCpPJB83SvgzosiboTJxftAhBbiatq9wuaY30ATokEOR1OvibqMtFcHxIpDeb2hHGOYKx0fSYOKTMsmg/640?wx_fmt=png)

上图中的0d5ecf06-b255-4272-b0fa-8e51dde2ce3e则是获取的令牌。PS：访问资源自己尝试下..........

### 客户端模式

请求的url如下：

`http://localhost:2003/auth-server/oauth/token?client_id=myjszl&client_secret=123&grant_type=client_credentials  
`

POSTMAN请求如下：

![](https://mmbiz.qpic.cn/mmbiz_png/19cc2hfD2rCpPJB83SvgzosiboTJxftAhoA7Wrl4B7yE3nZofKqCA0yo2diadic3tcY4tlFJFhcibV60RVc0GhHUzw/640?wx_fmt=png)

PS：访问资源自己尝试下..........

## OAuth2.0 其他端点的测试

Spring Security OAuth2.0还提供了其他的端点，下面来逐一测试一下。

### 1、刷新令牌

OAuth2.0提供了令牌刷新机制，一旦access\_token过期，客户端可以拿着refresh\_token去请求认证中心进行令牌的续期。请求的url如下：

`http://localhost:2003/auth-server/oauth/token?client_id=myjszl&client_secret=123&grant_type=refresh_token&refresh_token=  
`

POSTMAN请求如下：

![](https://mmbiz.qpic.cn/mmbiz_png/19cc2hfD2rCpPJB83SvgzosiboTJxftAhgETibtRmSPPOQXt1uofhuF7jn0JftLgGVZefL0OA2JnYFO2hyqWwz6g/640?wx_fmt=png)

### 2、校验令牌

OAuth2.0还提供了校验令牌的端点，请求的url如下：

`http://localhost:2003/auth-server/oauth/check_token?toke=  
`

POSTMAN请求如下：

![](https://mmbiz.qpic.cn/mmbiz_png/19cc2hfD2rCpPJB83SvgzosiboTJxftAhrretO6oe783icmSvpUjlt6diaC8dzQ4K51oPNsBofLbFaaFqUxib6DkJg/640?wx_fmt=png)

## 总结

本文介绍了OAuth2.0协议原理、四种授权模式，并且搭建了认证授权中心、资源服务进行了四种模式的测试。作为OAuth2.0入门教程已经非常详细了...........
