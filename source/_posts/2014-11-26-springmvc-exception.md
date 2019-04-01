---
layout: post
title: "springmvc 自定义异常处理 无法解析spring内部异常的分析"
description: ""
category : web 
tags: [spring,web]
---

一直在使用springmvc来当做后台应用的controller。最近发现一个问题，就是客户端做了一次数据处理，报了500异常，但是再服务端却没有任何异常输出。尽管我做了自定义异常处理：

<!-- more -->

``` xml      
<bean id="exceptionResolver" class="com.ruoshui.bethune.web.handler.BethuneMappingExceptionResolver">
	<property name="defaultErrorView" value="error/uncaughtException"/>
	<property name="exceptionMappings">
		<props>
			<prop key=".DataNotFoundException">error/dataNotFound</prop>
		</props>
	</property>
</bean>
```

``` java
/**
 * 覆盖doResolveException方法，记录错误日志
 */
@Override
protected ModelAndView doResolveException(HttpServletRequest request,
										  HttpServletResponse response, Object handler, Exception ex) {

	logger.error(ex.getMessage(), ex);
	.....balabala.......
}

```

这让我很困惑，查了半天，发现是spring mvc异常处理的一个漏洞。

## spring mvc 异常处理机制

在springmvc的主servlet:DispatcherServlet中定义里异常处理机制：

``` java
/**
 * Determine an error ModelAndView via the registered HandlerExceptionResolvers.
 * @param request current HTTP request
 * @param response current HTTP response
 * @param handler the executed handler, or {@code null} if none chosen at the time of the exception
 * (for example, if multipart resolution failed)
 * @param ex the exception that got thrown during handler execution
 * @return a corresponding ModelAndView to forward to
 * @throws Exception if no error ModelAndView found
 */
protected ModelAndView processHandlerException(HttpServletRequest request, HttpServletResponse response,
		Object handler, Exception ex) throws Exception {

	// Check registered HandlerExceptionResolvers...
	ModelAndView exMv = null;
	for (HandlerExceptionResolver handlerExceptionResolver : this.handlerExceptionResolvers) {
		exMv = handlerExceptionResolver.resolveException(request, response, handler, ex);
		if (exMv != null) {
			break;
		}
	}
	if (exMv != null) {
		if (exMv.isEmpty()) {
			return null;
		}
		// We might still need view name translation for a plain error model...
		if (!exMv.hasView()) {
			exMv.setViewName(getDefaultViewName(request));
		}
		if (logger.isDebugEnabled()) {
			logger.debug("Handler execution resulted in exception - forwarding to resolved error view: " + exMv, ex);
		}
		WebUtils.exposeErrorRequestAttributes(request, ex, getServletName());
		return exMv;
	}

	throw ex;
}
``` 
spring 会维护一个handlerExceptionResolvers列表，这其中包括我们的自定义exceptionHandler,也包括默认的DefaultHandlerExceptionResolver等等，处理异常时遍历列表中的所有handler，找到一个可以处理异常的handler：我们可以看下excpetionHandler的公共父类AbstractHandlerExceptionResolver

``` java
/**
 * Checks whether this resolver is supposed to apply (i.e. the handler matches
 * in case of "mappedHandlers" having been specified), then delegates to the
 * {@link #doResolveException} template method.
 */
@Override
public ModelAndView resolveException(HttpServletRequest request, HttpServletResponse response,
		Object handler, Exception ex) {

	if (shouldApplyTo(request, handler)) {
		// Log exception, both at debug log level and at warn level, if desired.
		if (logger.isDebugEnabled()) {
			logger.debug("Resolving exception from handler [" + handler + "]: " + ex);
		}
		logException(ex, request);
		prepareResponse(ex, response);
		return doResolveException(request, response, handler, ex);
	}
	else {
		return null;
	}
}
```

会调用shouldApplyTo来查看这个handler是否可以处理这个异常。
在这个handlerExceptionResolvers列中，我们的自定义异常是放在最后面的：

![1]({{ site.img_url }}/springmvc-exception1.png)

##有些异常被springmvc内部处理，不会再向外暴露

当有spring异常的时候，会优先由spring内部的异常处理handler来筛选（挑剩的才给我们。。。。）,而在DefaultHandlerExceptionResolver中

``` java
@Override
protected ModelAndView doResolveException(HttpServletRequest request, HttpServletResponse response,
		Object handler, Exception ex) {

	try {
		if (ex instanceof NoSuchRequestHandlingMethodException) {
			return handleNoSuchRequestHandlingMethod((NoSuchRequestHandlingMethodException) ex, request, response,
					handler);
		}
		else if (ex instanceof HttpRequestMethodNotSupportedException) {
			return handleHttpRequestMethodNotSupported((HttpRequestMethodNotSupportedException) ex, request,
					response, handler);
		}
		//......balabala..........
		else if (ex instanceof TypeMismatchException) {
			return handleTypeMismatch((TypeMismatchException) ex, request, response, handler);
		}
		else if (ex instanceof HttpMessageNotReadableException) {
			return handleHttpMessageNotReadable((HttpMessageNotReadableException) ex, request, response, handler);
		}
		else if (ex instanceof HttpMessageNotWritableException) {
			return handleHttpMessageNotWritable((HttpMessageNotWritableException) ex, request, response, handler);
		}
		else if (ex instanceof MethodArgumentNotValidException) {
			return handleMethodArgumentNotValidException((MethodArgumentNotValidException) ex, request, response, handler);
		}
		else if (ex instanceof MissingServletRequestPartException) {
			return handleMissingServletRequestPartException((MissingServletRequestPartException) ex, request, response, handler);
		}
		else if (ex instanceof BindException) {
			return handleBindException((BindException) ex, request, response, handler);
		}
		else if (ex instanceof NoHandlerFoundException) {
			return handleNoHandlerFoundException((NoHandlerFoundException) ex, request, response, handler);
		}
	}
	catch (Exception handlerException) {
		logger.warn("Handling of [" + ex.getClass().getName() + "] resulted in Exception", handlerException);
	}
	return null;
}

/**
 * Handle the case where a {@linkplain org.springframework.http.converter.HttpMessageConverter message converter}
 * cannot write to a HTTP request.
 * <p>The default implementation sends an HTTP 500 error, and returns an empty {@code ModelAndView}.
 * Alternatively, a fallback view could be chosen, or the HttpMediaTypeNotSupportedException could be
 * rethrown as-is.
 * @param ex the HttpMessageNotWritableException to be handled
 * @param request current HTTP request
 * @param response current HTTP response
 * @param handler the executed handler
 * @return an empty ModelAndView indicating the exception was handled
 * @throws IOException potentially thrown from response.sendError()
 */
protected ModelAndView handleHttpMessageNotWritable(HttpMessageNotWritableException ex,
		HttpServletRequest request, HttpServletResponse response, Object handler) throws IOException {

	sendServerError(ex, request, response);
	return new ModelAndView();
}

```

在处理一些异常的时候只是返回错误,但却没有任何log.error级别的输出,只有在log4j中将springmvc的异常级别调成debug，才会有输出。
后来查到我们后台就是在从对象转json的时候有一个空指针异常，被HttpMessageNotWritableException异常封装，所以才会没有日志输出。

``` xml
20:50:37,923 DEBUG DefaultHandlerExceptionResolver:134 - Resolving exception from handler [public com.xx.JsonResponse com.xx.xx.putUserDetail(com.xx.xx,org.springframework.validation.BindingResult,javax.servlet.http.HttpServletRequest)]:
 org.springframework.http.converter.HttpMessageNotWritableException: Could not write JSON: (was java.lang.NullPointerException) (through reference chain: com.xx.JsonResponse["data"]->com.xx.xx["xx"]->com.xx.xx["xx"]); nested exception is com.fasterxml.jackson.databind.JsonMappingException: (was java.lang.NullPointerException) (through reference chain: com.xx.JsonResponse["data"]->com.xx.xx["xx"]->com.xx.model.xx.xx["xx"])
20:50:39,092 DEBUG DispatcherServlet:1012 - Null ModelAndView returned to DispatcherServlet with name 'springServlet': assuming HandlerAdapter completed request handling

```
（后来查到我们后台就是在从对象转json的时候有一个空指针异常，被HttpMessageNotWritableException异常封装，所以才会没有日志输出。）

## 解决方案

虽然可以通过debug来查出异常，但肯定不是我想要的。如何才能在生产环境也能输出异常信息呢？

很简单，只要让我们的自定义exceptionHandler放在handlerExceptionResolvers列的第一位就可以,所以的exceptionHandler都实现了Ordered接口，可以定义优先级，spring默认的几个handler，都比较良心的设置成最低

``` java
/**
 * Sets the {@linkplain #setOrder(int) order} to {@link #LOWEST_PRECEDENCE}.
 */
public DefaultHandlerExceptionResolver() {
	setOrder(Ordered.LOWEST_PRECEDENCE);
}
```

我们只要在自定义exceptionHandler的构造函数中，把优先级设置成最高，就可以了。下图是设置好之后DispathServlet处理异常时的顺序

![2]({{ site.img_url }}/springmvc-exception2.png)

这样所有的异常都由我们监控，输出异常日志也就简单了。
