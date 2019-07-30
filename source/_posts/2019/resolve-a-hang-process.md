---
layout: post
title: "记一次进程 Hang 住的问题"
date: 2019-07-28 16:19:00
categories: 经验总结
tags: []
---

最近有个线上进程总是在运行中 Hang 住，不得不重启进程。没有规律，测试环境也未复现，解决这类问题需要一点耐心和技巧，无非是要理清思路，不要有畏难情绪，利用合适的工具，再加上代码静态分析，一定能搞定问题。

<!-- more -->

# 定位问题

首先得找到是哪里卡住了。日志输出的粒度一般都比较粗，很难精确的定位到问题。这里推荐两个工具，结合日志可以有效地定位代码。

## pstack / gstack

`gstack - print a stack trace of a running process`

这个工具用于查看正在运行中的进程的调用栈信息，不限进程的编程语言。

```
$ pstack pid

Thread 5 (Thread 0x7efe181f3700 (LWP 16871)):
#0  0x00007efe3085db8b in recv () from /lib64/libpthread.so.0
#1  0x00007efe11a4d64d in NET_Read () from /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.181-3.b13.el7_5.x86_64/jre/lib/amd64/libnet.so
#2  0x00007efe11a4cf56 in Java_java_net_SocketInputStream_socketRead0 () from /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.181-3.b13.el7_5.x86_64/jre/lib/amd64/libnet.so
#3  0x00007efe19cdb8e1 in ?? ()
#4  0x0000000000000000 in ?? ()
```

可以看到，进程卡在 `recv()` 这个系统调用上了，一直在等 socket 返回数据。参考官方文档 [http://man7.org/linux/man-pages/man2/recv.2.html](http://man7.org/linux/man-pages/man2/recv.2.html)

但是这个信息只能说明进程阻塞在网络 IO 上了，并不能定位到具体的网络连接。

## jstack

因为这个进程是由 Java 编写，所以还可以利用 JDK 提供的工具进行分析。我们使用 `jstack` 来输出 Java 进程的调用栈。

```
$ jstack pid

"pool-1-thread-1" #12 prio=5 os_prio=0 tid=0x00007f99f4f82000 nid=0x4509 runnable [0x00007f99c4172000]
   java.lang.Thread.State: RUNNABLE
	at java.net.SocketInputStream.socketRead0(Native Method)
	at java.net.SocketInputStream.socketRead(SocketInputStream.java:116)
	at java.net.SocketInputStream.read(SocketInputStream.java:171)
	at java.net.SocketInputStream.read(SocketInputStream.java:141)
	at java.io.BufferedInputStream.fill(BufferedInputStream.java:246)
	at java.io.BufferedInputStream.read1(BufferedInputStream.java:286)
	at java.io.BufferedInputStream.read(BufferedInputStream.java:345)
	- locked <0x00000000ecb19000> (a java.io.BufferedInputStream)
	at sun.net.www.http.HttpClient.parseHTTPHeader(HttpClient.java:735)
	at sun.net.www.http.HttpClient.parseHTTP(HttpClient.java:678)
	at sun.net.www.protocol.http.HttpURLConnection.getInputStream0(HttpURLConnection.java:1587)
	- locked <0x00000000ecb0ca58> (a sun.net.www.protocol.http.HttpURLConnection)
	at sun.net.www.protocol.http.HttpURLConnection.getInputStream(HttpURLConnection.java:1492)
	- locked <0x00000000ecb0ca58> (a sun.net.www.protocol.http.HttpURLConnection)
	at java.net.HttpURLConnection.getResponseCode(HttpURLConnection.java:480)
	at com.KGitextpdf.text.pdf.security.OcspClientBouncyCastle.getOcspResponse(OcspClientBouncyCastle.java:142)
	at com.KGitextpdf.text.pdf.security.OcspClientBouncyCastle.getBasicOCSPResp(OcspClientBouncyCastle.java:152)
	at com.KGitextpdf.text.pdf.security.OcspClientBouncyCastle.getEncoded(OcspClientBouncyCastle.java:176)
	at com.KGitextpdf.text.pdf.security.MakeSignature.signDetached(MakeSignature.java:179)
	at com.kinggrid.pdf.KGPdfHummer.doSignature(KGPdfHummer.java:428)
	at com.kinggrid.pdf.KGPdfHummer$$EnhancerByCGLIB$$45c3c60d.CGLIB$doSignature$42(<generated>)
	at com.kinggrid.pdf.KGPdfHummer$$EnhancerByCGLIB$$45c3c60d$$FastClassByCGLIB$$c56dd4ae.invoke(<generated>)
	at net.sf.cglib.proxy.MethodProxy.invokeSuper(MethodProxy.java:228)
	at com.kinggrid.authorization.KGFacadeCglib.intercept(KGFacadeCglib.java:58)
	at com.kinggrid.pdf.KGPdfHummer$$EnhancerByCGLIB$$45c3c60d.doSignature(<generated>)
	at com.xxxx.sign(xxxx.java:62)
	at com.xxxx.doSeal(xxxx.java:89)
	at com.xxxx.seal(xxxx.java:58)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at org.springframework.scheduling.support.ScheduledMethodRunnable.run(ScheduledMethodRunnable.java:65)
	at org.springframework.scheduling.support.DelegatingErrorHandlingRunnable.run(DelegatingErrorHandlingRunnable.java:54)
	at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:511)
	at java.util.concurrent.FutureTask.runAndReset(FutureTask.java:308)
	at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.access$301(ScheduledThreadPoolExecutor.java:180)
	at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.run(ScheduledThreadPoolExecutor.java:294)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	at java.lang.Thread.run(Thread.java:748)
```

这下就很明确了，按照调用栈的指示，定位到问题是由于电子签章软件在请求 CA 证书的 [OCSP](https://zh.wikipedia.org/wiki/%E5%9C%A8%E7%BA%BF%E8%AF%81%E4%B9%A6%E7%8A%B6%E6%80%81%E5%8D%8F%E8%AE%AE) 服务时发生了 IO 阻塞。

这里有个技巧：通过代码的静态分析，基本可以定位到发生问题的位置。但是如果想要进一步追踪问题的根源，还需要配合动态调试，获取到更多的运行时信息。例如，我可以通过在本机运行 IDE 调试工具，跟踪到发生网络阻塞的服务地址是 [http://ocsp.cfca.com.cn/ocsp](http://ocsp.cfca.com.cn/ocsp)；以及可以知道用于网络请求的实现类。

# 解决方案

对于发生 IO 阻塞原因的调查，需要服务端配合一起进行分析，但这次并不具备条件，因为服务端由外部供应商提供，我们只能反馈问题，以期得到进一步的支持。

## 可能的问题

1. 可以确定的是，客户端和服务端都未设置超时时间。
2. 挂起的原因，有可能是服务端架设了反向代理，而代理的 Upstream 一直未回调。深入和确切的原因，有待进一步查证。

## 基于客户端方案解决问题

通过为 Socket 设置 ReadTimeout 可以解决问题。
比如，我们通过 URLConnection 请求一个服务，
```
public abstract class URLConnection {
    
    ...

    /**
     * Sets the read timeout to a specified timeout, in
     * milliseconds. A non-zero value specifies the timeout when
     * reading from Input stream when a connection is established to a
     * resource. If the timeout expires before there is data available
     * for read, a java.net.SocketTimeoutException is raised. A
     * timeout of zero is interpreted as an infinite timeout.
     *
     *<p> Some non-standard implementation of this method ignores the
     * specified timeout. To see the read timeout set, please call
     * getReadTimeout().
     *
     * @param timeout an {@code int} that specifies the timeout
     * value to be used in milliseconds
     * @throws IllegalArgumentException if the timeout parameter is negative
     *
     * @see #getReadTimeout()
     * @see InputStream#read()
     * @since 1.5
     */
    public void setReadTimeout(int timeout) {
        if (timeout < 0) {
            throw new IllegalArgumentException("timeout can not be negative");
        }
        readTimeout = timeout;
    }
```

但坑的是，服务调用的代码是由供应商的 SDK 提供，我无法修改。那只能暂时粗暴解决问题了，设置了一个全局的超时配置。

```
System.setProperty("sun.net.client.defaultReadTimeout", "30000");
System.setProperty("sun.net.client.defaultConnectTimeout", "30000");
```

参考：[https://docs.oracle.com/javase/6/docs/technotes/guides/net/properties.html](https://docs.oracle.com/javase/6/docs/technotes/guides/net/properties.html)

```
sun.net.client.defaultConnectTimeout (default: -1)
sun.net.client.defaultReadTimeout (default: -1)
These properties specify the default connect and read timeout (resp.) for the protocol handler used by java.net.URLConnection.
sun.net.client.defaultConnectTimeout specifies the timeout (in milliseconds) to establish the connection to the host. For example for http connections it is the timeout when establishing the connection to the http server. For ftp connection it is the timeout when establishing the connection to ftp servers.

sun.net.client.defaultReadTimeout specifies the timeout (in milliseconds) when reading from input stream when a connection is established to a resource.
```

问题解决。