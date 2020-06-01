---
layout: post
title:  "springboot 接入actuator"
date:   2020-06-01 17:19:00 +0800-- 
---
## 引入pom
```xml
<dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```
如果对不在防火墙后的话最好再引入spring-security

```xml
<dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```
## 修改yaml

建议先禁用所有的endpoint,然后再按需打开需要的endpoint
```yaml
#启用spring actuator
management:
  server:
    #管理端口
    port: 18899
  endpoints:
    enabled-by-default: false
    web:
      #管理context-path
      base-path: /management
      exposure:
        include: health, info, beans, loggers, heapdump, threaddump
        exclude: shutdown
  endpoint:
    health:
      enabled: true
    info:
      enabled: true
    beans:
      enabled: true
    loggers:
      enabled: true
    heapdump:
      enabled: true
    threaddump:
      enabled: true
```
## 使用例子

### 1、动态修改日志级别
```shell
curl 'http://localhost:18899/management/loggers/ROOT' -i -X POST \
    -H 'Content-Type: application/json' \
    -d '{"configuredLevel":"debug"}'
```
返回结果
```shell
{
    "levels":[
        "OFF",
        "FATAL",
        "ERROR",
        "WARN",
        "INFO",
        "DEBUG",
        "TRACE"
    ],
    "loggers":{
        "ROOT":{
            "configuredLevel":"INFO",
            "effectiveLevel":"INFO"
        },
        "com.xiaomi.browser":{
            "configuredLevel":"INFO",
            "effectiveLevel":"INFO"
        },
        "com.xiaomi.browser.newsfeed.mapper":{
            "configuredLevel":"INFO",
            "effectiveLevel":"INFO"
        },
        "com.xiaomi.infra.galaxy.lcs.log.log4j2.LCSLogger":{
            "configuredLevel":"INFO",
            "effectiveLevel":"INFO"
        },
        "com.xiaomi.infra.galaxy.lcs.log.log4j2.appender.LCSThriftAppender":{
            "configuredLevel":"INFO",
            "effectiveLevel":"INFO"
        },
        "com.xiaomi.infra.pegasus":{
            "configuredLevel":"INFO",
            "effectiveLevel":"INFO"
        }
    }
}
```
修改ROOT日志级别为debug

### 2、线程dump
```shell
curl 'http://localhost:8080/actuator/threaddump' -i -X GET \
    -H 'Accept: application/json'
```
返回结果

```shell
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 7535
 
{
  "threads" : [ {
    "threadName" : "Thread-2985",
    "threadId" : 3598,
    "blockedTime" : -1,
    "blockedCount" : 0,
    "waitedTime" : -1,
    "waitedCount" : 1,
    "lockName" : "java.util.concurrent.CountDownLatch$Sync@2a0c822c",
    "lockOwnerId" : -1,
    "inNative" : false,
    "suspended" : false,
    "threadState" : "WAITING",
    "stackTrace" : [ {
      "methodName" : "park",
      "fileName" : "Unsafe.java",
      "lineNumber" : -2,
      "className" : "sun.misc.Unsafe",
      "nativeMethod" : true
    }, {
      "methodName" : "park",
      "fileName" : "LockSupport.java",
      "lineNumber" : 175,
      "className" : "java.util.concurrent.locks.LockSupport",
      "nativeMethod" : false
    }, {
      "methodName" : "parkAndCheckInterrupt",
      "fileName" : "AbstractQueuedSynchronizer.java",
      "lineNumber" : 836,
      "className" : "java.util.concurrent.locks.AbstractQueuedSynchronizer",
      "nativeMethod" : false
    }, {
      "methodName" : "doAcquireSharedInterruptibly",
      "fileName" : "AbstractQueuedSynchronizer.java",
      "lineNumber" : 997,
      "className" : "java.util.concurrent.locks.AbstractQueuedSynchronizer",
      "nativeMethod" : false
    }, {
      "methodName" : "acquireSharedInterruptibly",
      "fileName" : "AbstractQueuedSynchronizer.java",
      "lineNumber" : 1304,
      "className" : "java.util.concurrent.locks.AbstractQueuedSynchronizer",
      "nativeMethod" : false
    }, {
      "methodName" : "await",
      "fileName" : "CountDownLatch.java",
      "lineNumber" : 231,
      "className" : "java.util.concurrent.CountDownLatch",
      "nativeMethod" : false
    }, {
      "methodName" : "lambda$jsonThreadDump$0",
      "fileName" : "ThreadDumpEndpointDocumentationTests.java",
      "lineNumber" : 56,
      "className" : "org.springframework.boot.actuate.autoconfigure.endpoint.web.documentation.ThreadDumpEndpointDocumentationTests",
      "nativeMethod" : false
    }, {
      "methodName" : "run",
      "lineNumber" : -1,
      "className" : "org.springframework.boot.actuate.autoconfigure.endpoint.web.documentation.ThreadDumpEndpointDocumentationTests$$Lambda$2935/334838288",
      "nativeMethod" : false
    }, {
      "methodName" : "run",
      "fileName" : "Thread.java",
      "lineNumber" : 748,
      "className" : "java.lang.Thread",
      "nativeMethod" : false
    } ],
    "lockedMonitors" : [ ],
    "lockedSynchronizers" : [ {
      "className" : "java.util.concurrent.locks.ReentrantLock$NonfairSync",
      "identityHashCode" : 269684612
    } ],
    "lockInfo" : {
      "className" : "java.util.concurrent.CountDownLatch$Sync",
      "identityHashCode" : 705462828
    }
  }, {
    "threadName" : "http-nio-auto-28-Acceptor",
    "threadId" : 3550,
    "blockedTime" : -1,
    "blockedCount" : 0,
    "waitedTime" : -1,
    "waitedCount" : 0,
    "lockOwnerId" : -1,
    "inNative" : true,
    "suspended" : false,
    "threadState" : "RUNNABLE",
    "stackTrace" : [ {
      "methodName" : "accept0",
      "fileName" : "ServerSocketChannelImpl.java",
      "lineNumber" : -2,
      "className" : "sun.nio.ch.ServerSocketChannelImpl",
      "nativeMethod" : true
    }, {
      "methodName" : "accept",
      "fileName" : "ServerSocketChannelImpl.java",
      "lineNumber" : 419,
      "className" : "sun.nio.ch.ServerSocketChannelImpl",
      "nativeMethod" : false
    }, {
      "methodName" : "accept",
      "fileName" : "ServerSocketChannelImpl.java",
      "lineNumber" : 247,
      "className" : "sun.nio.ch.ServerSocketChannelImpl",
      "nativeMethod" : false
    }, {
      "methodName" : "serverSocketAccept",
      "fileName" : "NioEndpoint.java",
      "lineNumber" : 469,
      "className" : "org.apache.tomcat.util.net.NioEndpoint",
      "nativeMethod" : false
    }, {
      "methodName" : "serverSocketAccept",
      "fileName" : "NioEndpoint.java",
      "lineNumber" : 71,
      "className" : "org.apache.tomcat.util.net.NioEndpoint",
      "nativeMethod" : false
    }, {
      "methodName" : "run",
      "fileName" : "Acceptor.java",
      "lineNumber" : 95,
      "className" : "org.apache.tomcat.util.net.Acceptor",
      "nativeMethod" : false
    }, {
      "methodName" : "run",
      "fileName" : "Thread.java",
      "lineNumber" : 748,
      "className" : "java.lang.Thread",
      "nativeMethod" : false
    } ],
    "lockedMonitors" : [ {
      "className" : "java.lang.Object",
      "identityHashCode" : 968329834,
      "lockedStackDepth" : 2,
      "lockedStackFrame" : {
        "methodName" : "accept",
        "fileName" : "ServerSocketChannelImpl.java",
        "lineNumber" : 247,
        "className" : "sun.nio.ch.ServerSocketChannelImpl",
        "nativeMethod" : false
      }
    } ],
    "lockedSynchronizers" : [ ]
  }, {
    "threadName" : "http-nio-auto-28-ClientPoller",
    "threadId" : 3549,
    "blockedTime" : -1,
    "blockedCount" : 0,
    "waitedTime" : -1,
    "waitedCount" : 0,
    "lockOwnerId" : -1,
    "inNative" : true,
    "suspended" : false,
    "threadState" : "RUNNABLE",
    "stackTrace" : [ {
      "methodName" : "epollWait",
      "fileName" : "EPollArrayWrapper.java",
      "lineNumber" : -2,
      "className" : "sun.nio.ch.EPollArrayWrapper",
      "nativeMethod" : true
    }, {
      "methodName" : "poll",
      "fileName" : "EPollArrayWrapper.java",
      "lineNumber" : 269,
      "className" : "sun.nio.ch.EPollArrayWrapper",
      "nativeMethod" : false
    }, {
      "methodName" : "doSelect",
      "fileName" : "EPollSelectorImpl.java",
      "lineNumber" : 93,
      "className" : "sun.nio.ch.EPollSelectorImpl",
      "nativeMethod" : false
    }, {
      "methodName" : "lockAndDoSelect",
      "fileName" : "SelectorImpl.java",
      "lineNumber" : 86,
      "className" : "sun.nio.ch.SelectorImpl",
      "nativeMethod" : false
    }, {
      "methodName" : "select",
      "fileName" : "SelectorImpl.java",
      "lineNumber" : 97,
      "className" : "sun.nio.ch.SelectorImpl",
      "nativeMethod" : false
    }, {
      "methodName" : "run",
      "fileName" : "NioEndpoint.java",
      "lineNumber" : 709,
      "className" : "org.apache.tomcat.util.net.NioEndpoint$Poller",
      "nativeMethod" : false
    }, {
      "methodName" : "run",
      "fileName" : "Thread.java",
      "lineNumber" : 748,
      "className" : "java.lang.Thread",
      "nativeMethod" : false
    } ],
    "lockedMonitors" : [ {
      "className" : "sun.nio.ch.Util$3",
      "identityHashCode" : 2026559947,
      "lockedStackDepth" : 3,
      "lockedStackFrame" : {
        "methodName" : "lockAndDoSelect",
        "fileName" : "SelectorImpl.java",
        "lineNumber" : 86,
        "className" : "sun.nio.ch.SelectorImpl",
        "nativeMethod" : false
      }
    }, {
      "className" : "java.util.Collections$UnmodifiableSet",
      "identityHashCode" : 976911721,
      "lockedStackDepth" : 3,
      "lockedStackFrame" : {
        "methodName" : "lockAndDoSelect",
        "fileName" : "SelectorImpl.java",
        "lineNumber" : 86,
        "className" : "sun.nio.ch.SelectorImpl",
        "nativeMethod" : false
      }
    }, {
      "className" : "sun.nio.ch.EPollSelectorImpl",
      "identityHashCode" : 143151187,
      "lockedStackDepth" : 3,
      "lockedStackFrame" : {
        "methodName" : "lockAndDoSelect",
        "fileName" : "SelectorImpl.java",
        "lineNumber" : 86,
        "className" : "sun.nio.ch.SelectorImpl",
        "nativeMethod" : false
      }
    } ],
    "lockedSynchronizers" : [ ]
  } ]
}
```
### 3、堆dump(谨慎使用)
```shell
curl 'http://localhost:18899/management/heapdump' -O
```

