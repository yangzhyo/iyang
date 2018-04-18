---
layout: post
title: "进程间通信的解决方案"
date: 2017-10-31 16:21:00
categories: JAVA
author: lAOyANG
tags: [JAVA,IPC]
---

> 项目中经常会涉及到进程间通信（[IPC](https://en.wikipedia.org/wiki/Inter-process_communication)，Inter-Process Communication）的问题，比如一个任务调度器，对若干个 worker 进行调度和控制。想想，这样的系统模型是不是很多？PHP-fpm，nginx，各类 Application 容器，各类分布式系统……，笔者最近就在做一个类似的项目，需要通过调度器向各个 worker 发送工作指令，并收集从各 worker 反馈回来的信息。那该如何实现呢？

<!-- more -->

# 进程间有哪些通信方式

先来整理一下有哪些办法可以做到进程间的通信。

* 管道（Pipe）
* 命名管道（Named Pipe）
* 信号（Signal）
* 消息队列（Message Queue）
* 共享内存 (Shared Memory)
* 内存映射（Mapped Memory）
* 信号量（Semaphore）
* 套接口（Socket）

详细说明可参考：
* [https://my.oschina.net/hunglish/blog/761140](https://my.oschina.net/hunglish/blog/761140)
* [https://en.wikipedia.org/wiki/Inter-process_communication](https://en.wikipedia.org/wiki/Inter-process_communication)

## 尝试管道

主进程与子进程之间通过管道进行流访问，调度器（主进程）写数据到 worker（子进程）的标准输入流，worker 收到后作出响应，调度器再从 worker 的标准输出流读出反馈信息。

调度器：
```java
Runtime run = Runtime.getRuntime();
Process p = run.exec("java -jar worker-1.0.0.jar", null, new File("/dir"));

BufferedInputStream in = new BufferedInputStream(p.getInputStream());
BufferedOutputStream out = new BufferedOutputStream(p.getOutputStream());

BufferedReader reader = new BufferedReader(new InputStreamReader(in));
BufferedWriter writer = new BufferedWriter(new OutputStreamWriter(out));

writer.write("Hello world" + System.lineSeparator());
writer.flush();
writer.close();

String s;
while ((s = reader.readLine()) != null) {
	System.out.println(s);
}
```

Worker:
```java
BufferedInputStream in = new BufferedInputStream(System.in);
BufferedReader reader = new BufferedReader(new InputStreamReader(in));
while (true) {
	String strLine = reader.readLine();
	System.out.println("echo:" + strLine);
}
```

功能一切正常，代码也不复杂。那可行吗？有个大麻烦！我得自己实现一套指令协议，并且严格控制两个进程的读写同步，不然程序会失控。

## 其他方案

* 信号，数据承载量太小，不合适
* 消息队列，引入系统架构的复杂度，不符合我的简洁之道
* 共享内存&内存映射，非常高效，但是有管道方案同样的问题，需要同步各进程之间对内存读写的同步，非常麻烦
* 套接字，下面讨论

## 套接字

要说套接字，就要先说说 [Berkeley Sockets](https://en.wikipedia.org/wiki/Berkeley_sockets)，它是被专门设计用来做 IPC 的 API，除了支持本机进程间通信外（[Unix Domain Socket](https://en.wikipedia.org/wiki/Unix_domain_socket)），也可以支持跨机器的 IPC，也就是我们常说的 [Network Socket](https://en.wikipedia.org/wiki/Network_socket)。

在我的项目中，调度器与 worker 在同一个机器上，所以采用 Unix Domain Socket 看起来是最合理的，这种方式不走网络模型，是通过操作系统内核来完成交互的，基于文件作为通信信道。（是不是有点眼熟？fpm.sock，mysql.sock，……）

看起来可行？我们只需要定义一个简单的指定协议来格式化数据流，看起来很完美。

很不幸，Java 无法直接支持 Unix Domain Socket，需要第三方 Native Code 的支持。原因很简单，Java 是通用语言，而 Unix Domain Socket 是平台相关的技术。参考：

* [https://stackoverflow.com/questions/4099432/unix-domain-socket-in-java](https://stackoverflow.com/questions/4099432/unix-domain-socket-in-java)
* [https://github.com/jnr/jnr-unixsocket](https://github.com/jnr/jnr-unixsocket)
* [https://github.com/kohlschutter/junixsocket](https://github.com/kohlschutter/junixsocket)

那 Network Socket 可以吗？可以，一种基于 Network Socket 实现的，并且编程模型良好的技术方案，那就是 [RPC](https://en.wikipedia.org/wiki/Remote_procedure_call)。很多编程语言都支持 RPC，其支持的协议也不尽相同，这里我们就重点说一下 Java RPC 的方案：[RMI](https://en.wikipedia.org/wiki/Java_remote_method_invocation)（Java remote method invocation)，通过抽象屏蔽了底层网络传输的细节，程序员只关注高层代码的实现，对远程方法的调用就像调用本地方法一样的方便。具体的技术细节，可参考前述链接或者官方文档。

太赞了，这样我们连指令协议都不用制定了。

以下是简单的代码样例：

Worker 端作为 RMI 的服务端，接受调度器的调用
```java
JobService jobService = new JobServiceImpl(args[0]);
Registry registry = LocateRegistry.createRegistry(0);
int port = ((UnicastServerRef)((RegistryImpl)registry).getRef()).getLiveRef().getPort();
String name = "rmi://127.0.0.1:" + port + "/JobService";
Naming.rebind(name, jobService);
```

调度器调用 worker 的功能
```java
String endPoint = "rmi://127.0.0.1:" + port + "/JobService";
JobService jobService = (JobService) Naming.lookup(endPoint);
jobService.doSomething();
```

因为会有很多 worker，所以 worker 端的端口号分配就成为了问题，这里有个小窍门，```LocateRegistry.createRegistry``` 方法传入端口 0，系统会自动分配一个空闲的端口。worker 再通过标准输出流告诉调度器。（所以这里结合了“管道”技术，来协调整个系统的和谐运行）。

# 结束语
任何目标的达成，都有多种途径，我们应该选择合理、简单、实用、快捷的方式，在保证快速达成目标的同时，尽可能地确保系统后期的可维护性。