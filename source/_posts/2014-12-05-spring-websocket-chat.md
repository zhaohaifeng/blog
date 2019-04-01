---
layout: post
title: "基于spring websocket开发聊天功能的研究实战"
date: 2014-12-05 10:00:00
description: ""
category : web 
tags: [websocket,聊天,chat,SockJS,Stomp,spring]
---

上一篇中我简单说了一下我们项目的构想，其中有一个环节是需要开发一个web聊天功能。这个功能是基于spring4.0 新推出的对websocket支持，和SOckJS以及Stomp开发的，这里简单讲解一下。

<!-- more -->

##首先说一下什么是websocket。

这个其实可以参见[使用 HTML5 WebSocket 构建实时 Web 应用](https://www.ibm.com/developerworks/cn/web/1112_huangxa_websocket/),因为不是本文的重点，这里我就不多说了。

##spring,websocket,SockJS和Stomp的关系

websocket规定了客户端和服务端的协议，所以客户端和服务端都必须支持这套协议。客户端自然是浏览器，下图可以看到各个浏览器对html5以及websocket的支持

![1](http://7tsz2d.com1.z0.glb.clouddn.com/websocket-browser.png)

吐槽一点的就是，chrome很早就准备支持html5了，但是Android的webkit内核到4.4才开始支持。。。。。。

不管怎么说，还是有大量的浏览器是不支持html5及websocket的，尤其是万恶的IE。为了能让我们的聊天服务兼容性更强，所以我们要使用SockJS作为客户端。SockJS 是一个浏览器上运行的 JavaScript 库，如果浏览器不支持 WebSocket，该库可以模拟对 WebSocket 的支持，实现浏览器和 Web 服务器之间低延迟、全双工、跨域的通讯通道。不过使用SockJs需要服务端的支持，这个后面讲。

服务端就是后台web容器的支持了。当websocket推出的时候，各个web容器的服务商都在开发新版本以提供支持。这里要提到的一点就是[JSR-356协议](http://www.oracle.com/technetwork/articles/java/jsr356-1937161.html)，WebSocket的Java API，规定了开发者把WebSockets 整合进他们的应用时可以使用的Java API — 包括服务器端和Java客户端。JSR 356是即将出台的Java EE 7标准中的一部分。这意味着所有Java EE 7兼容的应用服务器都将有一个遵守JSR 356标准的WebSocket协议的实现。开发者也可以在Java EE 7应用服务器之外使用JSR 356。Tomcat (7.0.47+), GlassFish (4.0+) and WildFly (8.0+)等web服务器都是基于此协议来支持websocket。但作为javaweb容器的急先锋-Jetty，早于JSR-356就在自己的9.0版本中推出了自己的api协议-Jetty WebSocket API，顺带一提，JSR-356很大一部分就是基于Jetty的websocket协议制订的。之后Jetty明确表示会兼容JSR-356，但还是继续发展自己的协议版本，应该是因为Jetty不仅需要支持Java，还要支持PHP等其它语言的缘故吧（也可能觉得JSR-356满足不了自己）。

spring最近推出的4.0版本中，对这些各种各样的websocket引擎容器封装了一套统一的api,可以更方便的使用。

但websocket的api毕竟是个很底层基于TCP的东东，所以Stomp协议对其进行了再封装，提供了一个可互操作的连接格式，允许STOMP客户端与任意STOMP消息代理(Broker)进行交互,Apache ActiveMQ就是基于此协议设计的。而spring也提供了对Stomp的支持。

## spring-websocket的配置

### 集成spring-websocket组件

好进入正题，使用spring的websocket功能，需要4.0以上版本，并且加载spring-websocket组件
pom.xml的配置:

``` xml
<dependency>
	<groupId>org.springframework</groupId>
	<artifactId>spring-websocket</artifactId>
	<version>4.1.2.RELEASE</version>
</dependency>
```

## 配置

在spring的xml中配置component-scan功能，这样可以直接在类文件中配置websocket

### 接收消息

``` java
@Configuration //这是一个配置文件
@EnableWebSocketMessageBroker //激活websocket消息代理
public class WebSocketConfig extends AbstractWebSocketMessageBrokerConfigurer {

	@Override
	public void configureMessageBroker(MessageBrokerRegistry config) {
        //匹配消息代理的url前缀
		config.enableSimpleBroker("/ws","/user");
        //
		config.setApplicationDestinationPrefixes("/ws");
	}

	@Override
	public void registerStompEndpoints(StompEndpointRegistry registry) {
        //配置sockjs连接的入口
		registry.addEndpoint("/ws").withSockJS();
	}
}
```

配置完代理之后，前端就可以通过sockJs来连接通道了。

``` js
function connect() {
    var socket= new SockJS('/ws');
    stompClient = Stomp.over(socket);
    var error_callback = function (error) {
        alert("连接关闭，刷新页面重新建立连接");
        // display the error's message header:
    };
    stompClient.connect({}, function (frame) {
        console.log('Connected: ' + frame);
        var dest = "/ws/web/chat/" + userId;
        stompClient.subscribe(dest, function (data) {
            showMessage(JSON.parse(data.body));
        });
        //TODO 断线重连
    }, error_callback);
}
```

### 发送消息

spring可以在controller中配置消息的处理controller:

``` java
@MessageMapping("/web/chat")
public String webChat(WebChatVo webChatVo) {
	LOGGER.info(JSON.toJSONString(webChatVo));
	LOGGER.info(
			"用户" + webChatVo.getFromUserId() + "向用户" + webChatVo.getToUserId() + "发送了一条消息:" +
					webChatVo.getMessage());
	simpMessagingTemplate
			.convertAndSend("/ws/web/chat/" + webChatVo.getToUserId(),
					webChatVo);

	return "success";
}
```

前端将封装好的json消息发送的服务器，服务器根据传入的toUserId发送给相应的接收方。

详细的代码参见：[webchat](https://github.com/zhaohaifeng/webchat)

运行起来的效果是:

![3](http://7tsz2d.com1.z0.glb.clouddn.com/0AA01275-07B8-42D1-BC38-D3C795D7E727.png)
![4](http://7tsz2d.com1.z0.glb.clouddn.com/D3CB0408-EAD4-4250-A8CD-22E2A42DBD11.png)
![5](http://7tsz2d.com1.z0.glb.clouddn.com/F1E29FB0-90E4-4342-9EA8-E6F0E4C34C06.png)

## 参考资料

* [spring websocket doc](http://docs.spring.io/spring/docs/current/spring-framework-reference/html/websocket.html)  

