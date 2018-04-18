---
layout: post
title: "ZooKeeper运维之命令行接口&Rest接口"
date: 2018-4-17 23:00:00
categories: 运维
author: lAOyANG
tags: [运维,ZooKeeper]
---

> 项目运维的原因，需要手动对线上 [ZooKeeper](https://zookeeper.apache.org/) 的部分节点进行修改，设置一串预置的JSON字符串。看似很简单的一件事，却着实让我折腾了一番，写一篇记录一下。

<!-- more -->

# ZooInspector

自然想到的第一个方案，是 ZooInspector，它是一个 GUI 版本的 ZooKeeper 管理器，可以通过它来浏览和管理节点。

![ZooInspector](https://i.imgur.com/emSRNjl.png)

下载地址：
[https://issues.apache.org/jira/secure/attachment/12436620/ZooInspector.zip](https://issues.apache.org/jira/secure/attachment/12436620/ZooInspector.zip)

源代码：
[https://github.com/apache/zookeeper/tree/master/src/contrib/zooinspector](https://github.com/apache/zookeeper/tree/master/src/contrib/zooinspector)

## 遇到的问题
这个方案的问题是，需要线上的 ZooKeeper 服务器对外开放 2181 端口，然而由于 ACL 控制的弱点，运维不允许对外开放这个端口。

# zkCli

方案二，ZooKeeper 运行包自带命令行工具，可以在服务器上通过命令行来管理节点。

## 如何使用

命令行工具位于 `{zk_root}/bin/zkCli.sh`

运行后，进入到交互模式，支持以下命令：

```
ZooKeeper -server host:port cmd args
	stat path [watch]
	set path data [version]
	ls path [watch]
	delquota [-n|-b] path
	ls2 path [watch]
	setAcl path acl
	setquota -n|-b val path
	history 
	redo cmdno
	printwatches on|off
	delete path [version]
	sync path
	listquota path
	rmr path
	get path [watch]
	create [-s] [-e] path data acl
	addauth scheme auth
	quit 
	getAcl path
	close 
	connect host:port
```

通过字面值，就能明白命令的含义。我可以通过 `set path data [version]` 来完成我的需求，完整的命令：

`set /a/b/c json_str`

## 遇到的问题

但实际上，待写入的 JSON 串很复杂，同时含有空格、单引号、双引号等字符，导致如下问题：

1. 空格导致 data 参数被截断，引起 data 不完整，以及 version 参数被错误解析；
2. 将参数包装一对单引号或者双引号，可以解决空格的问题，但 JSON 中的单引号和双引号同样会引起截断，从而引起第1点中的问题。
3. 是否可以对 JSON 中的单引号和双引号进行转义？根据经验试了几种可能的转义方法，都不行。

咱们只能去研究 ZooKeeper 源代码一探究竟了。

通过 zkCli.sh 脚本，可以定位到源代码起始于 org.apache.zookeeper.ZooKeeperMain 类，我们进一步跟踪到参数解析的关键代码片段，如下所示：

```java
    /**
     * A storage class for both command line options and shell commands.
     *
     */
    static class MyCommandOptions {

        .....

        public static final Pattern ARGS_PATTERN = Pattern.compile("\\s*([^\"\']\\S*|\"[^\"]*\"|'[^']*')\\s*");
        public static final Pattern QUOTED_PATTERN = Pattern.compile("^([\'\"])(.*)(\\1)$");

        ......

        /**
         * Breaks a string into command + arguments.
         * @param cmdstring string of form "cmd arg1 arg2..etc"
         * @return true if parsing succeeded.
         */
        public boolean parseCommand( String cmdstring ) {
            Matcher matcher = ARGS_PATTERN.matcher(cmdstring);

            List<String> args = new LinkedList<String>();
            while (matcher.find()) {
                String value = matcher.group(1);
                if (QUOTED_PATTERN.matcher(value).matches()) {
                    // Strip off the surrounding quotes
                    value = value.substring(1, value.length() - 1);
                }
                args.add(value);
            }
            if (args.isEmpty()){
                return false;
            }
            command = args.get(0);
            cmdArgs = args;
            return true;
        }
    }
```

可以看到，核心规则是两个正则表达式。

第一个表达式 `\\s*([^\"\']\\S*|\"[^\"]*\"|'[^']*')\\s*`，用于提取参数，规则有三：

1. 如果参数不是以单引号或双引号开头，则以空格作为参数的结束标志
2. 如果参数以双引号开头，则以下一个双引号作为结束符
3. 如果参数以单引号开头，则以下一个单引号作为结束符

第二个表达式 `^([\'\"])(.*)(\\1)$`，用于当参数被单引号或双引号包围时，擦除单引号和双引号。

基于以上结论，可以确定无法支持参数中同时包含空格、单引号、双引号。

[ZooKeeper源码](https://github.com/apache/zookeeper)

# ZooKeeper Rest API

方案三，在 ZooKeeper contrib 包中，有一个 Rest 工具，支持以 Rest API 形式的接口对 ZooKeeper 进行管理。最重要的是，它支持以文件的形式对 ZK 节点进行更新。

```
#get the root node data
curl http://localhost:9998/znodes/v1/

#set a node (data.txt contains the ascii text you want to set on the node)
curl -T data.txt -w "\n%{http_code}\n" "http://localhost:9998/znodes/v1/cluster1/leader?dataformat=utf8"
```

更多用法，可参考 [https://github.com/apache/zookeeper/tree/master/src/contrib/rest](https://github.com/apache/zookeeper/tree/master/src/contrib/rest)

## 如何使用

1. 方法一，如果服务器有安装 Ant 构建工具，可进入到源码目录手动构建、运行
   ```
   cd {zk_root}/src/contrib/rest/
   ant run
   ```

2. 方法二，如果没有 Ant，也可以使用已经构建好的版本
   ```
   cp {zk_root}/src/contrib/rest/rest.sh {zk_root}/contrib/rest/
   cp {zk_root}/zookeeper-3.4.10.jar {zk_root}/contrib/rest/lib/
   cd {zk_root}/contrib/rest/
   ./rest.sh start
   ```
   其中 rest.sh 支持 start,stop,restart 参数。

通过检查 zkrest.log 日志文件，可以检查服务是否已经启动。

通过方案三，问题终于顺利解决。