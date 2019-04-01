---
layout: post
title: "HttpClient相关优化"
description: ""
tags: [Android,tool]
---

在测试应用的时候，发现当连接很多的时候，多线程下的http连接并不比单线程的快多少，很疑惑，于是就研究了一下httpClient。
<!-- more -->

1.httpClient默认并不是支持多线程的，必须使用

``` java 
    ThreadSafeClientConnManager connManager = new ThreadSafeClientConnManager(
    		httpParams, schemeRegistry);
    AbstractHttpClient httpClient = new DefaultHttpClient(connManager,
    		httpParams);
```
实现单例httpClient并发请求。

2.ThreadSafeClientConnManager默认使用了连接池，
``` java 
    /**
     * Creates a new thread safe connection manager.
     *
     * @param params    the parameters for this manager.
     * @param schreg    the scheme registry.
     */
    public ThreadSafeClientConnManager(HttpParams params,
    		SchemeRegistry schreg) {
    	if (params == null) {
    		throw new IllegalArgumentException("HTTP parameters may not be null");
    	}
    	if (schreg == null) {
    		throw new IllegalArgumentException("Scheme registry may not be null");
    	}
    	this.schemeRegistry = schreg;
    	this.connOperator   = createConnectionOperator(schreg);
    	//创建连接池
    	this.connectionPool = createConnectionPool(params);
    }
    
    /**
     * Hook for creating the connection pool.
     *
     * @return  the connection pool to use
     */
    protected AbstractConnPool createConnectionPool(final HttpParams params) {
    	return new ConnPoolByRoute(connOperator, params);
    }
    
    /**
     * Gets the maximum number of connections allowed.
     *
     * @param params HTTP parameters
     *
     * @return The maximum number of connections allowed.
     *
     * @see ConnManagerPNames#MAX_TOTAL_CONNECTIONS
     * 获取连接池最大数量，可以设置，如果不设值，取默认值，DEFAULT_MAX_TOTAL_CONNECTIONS=20
     */
    public static int getMaxTotalConnections(
    		final HttpParams params) {
    	if (params == null) {
    		throw new IllegalArgumentException
    			("HTTP parameters must not be null.");
    	}
    	return params.getIntParameter(MAX_TOTAL_CONNECTIONS, DEFAULT_MAX_TOTAL_CONNECTIONS);
    }
``` 
3.在多线程使用httpClient时，在moma中是用线程池来使用，最好加上
``` java 
    //设置最大连接数
    ConnManagerParams.setMaxTotalConnections(httpParams, 10);
    //设置最大路由连接数
    // 重要，这个参数的默认值为2，如果 不设置这个参数值默认情况下对于同一个目标机器的最大并发连接只有2个！
    // 这意味着如果你正在执行一个针对某一台目标机器的抓取任务的时候，哪怕你设置连接 池的最大连接数为200，
    // 但是实际上还是只有2个连接在工作，其他剩余的198个连接都在等待，都是为别的目标机器服务的。
    ConnPerRouteBean connPerRoute = new ConnPerRouteBean(10);
    ConnManagerParams.setMaxConnectionsPerRoute(httpParams, connPerRoute);
``` 
经实验，当线程请求很多时，加这个比不加要快1.5以上。同时开启的线程越多，效果越明显，但因为是移动端，所以线程不宜过多，线程池设的是5个，那么这个最大连接数就不用设那么大了。但抓取数据的时候一定要设大一点。 

