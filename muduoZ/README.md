# 0. **写在前面**

这是一个基于C++11的muduo网络库。

# 1. 项目概述

在阅读完muduo的源码，并通过单步调试对其有一定理解后，出于学习和更深入地理解muduo，笔者重写了muduoZ网络库，其相比muduo主要有如下特点

1.C++11。使用了C++11的std::mutex \...\...

2.去掉了Muduo库中的Boost依赖，完全使用C++标准

3.没有单独封装Thread，使用C++11引入的std::thread搭配lambda表达式实现工作线程，没有直接使用pthread库。类似的直接使用C++11/17的还有std::atomic，std::any等

4\. We use writev instead of fwrite for less system call.

5\. We don\'t put cookie\_ inside FixedBuffer, so others can freely use FixedBuffer without having same cookie as AsyncLogging, which may confuse us

6.相比muduo，muduoZ只重写了核心部分。没有实现诸如[TimeZone](https://github.com/chenshuo/muduo/blob/master/muduo/base/TimeZone.cc)的辅助类。

## 1.1 总览

muduo是由<span class="underline">chen shuo</span>所写基于Reactor模式的C++网络库。UML类图如下所示（为突出重点，只画出核心类及各类的核心成员）。

如下图，我将构成muduo的框架划分为三块内容：

1）由EventLoop、Poller、Channel构成的Reactor框架

2）由TcpServer、Acceptor、TcpConnection构成的Server三件套

3）由TcpClient、Connector、(TcpConnection)构成的Client三件套

其中由EventLoop、Poller、Channel构成的Reactor框架是muduo的核心，EventLoop的loop函数会循环往复地进行"调用poll/epoll(由Poller封装)，处理事件"。Server和Client三件套则通过Channel来接入Reactor框架（Channel==黏合剂 部分对如何接入进行了介绍）。

![Image text](https://raw.githubusercontent.com/a504644805/resources/master/muduoZ/UML_Class_Graph.png)

# 2. 几点心得

关于muduo，陈硕以及众多博客都有了很多见解，在此就不重复前人的发言了。挑几点我认为比较重要但很少见人（至今没看到，或者说讲得不够具体丰富）说起的理解：

## 2.1 Channel==黏合剂

要深刻理解这三部分之间的关联，关键是把握Channel类作为**\"黏合剂\"**的本质。

只看Channel，会发现其负责某个socket fd的事件分发（handleEvent()函数），不负责socket fd的生命周期，好像是个非常普通的类。

但一旦从整体来看，则会发现其和Poller属于聚合关系，和Acceptor，Connector，TcpConnection属于组合关系，EventLoop也用到了Channel类来进行唤醒操作。简直无所不在。这本质上是因为muduo采用Channel到Poller的注册和注销机制实现I/O事件的动态增删。

实现机制如下：（用代码来具体介绍下？）

1\. Channel及Channel负责的socket fd的生命周期和其拥有者一致（Acceptor，Connector，TcpConnection etc)

2.Channel的拥有者负责Channel的创建、配置、注册、注销这一系列动作。

## 2.2 如何优雅地结束

如陈硕和evpp所言，优雅的结束往往比endless的loop更难。

更确切得说：要实现优雅的结束，需要在endless loop的基础上，考虑更多的东西。介绍**状态机**

## 2.3 一条日志消息都别想跑

First flush then Destruct buffer !!! So cookie\_=bufferEnd means invalid buffer

## 2.4 别被回调绕晕了

这个本来没打算写的，无意中看见知乎的一个评论，仔细想想初看muduo确实花了一点时间捋清楚里面的回调逻辑，所以一并写了吧

## 2.5 为何要tie

在重写muduo时，我有一条原则：遇见自己难以理解的手法，在查阅资料无果后，先标注，然后按照自己的理解去写而不是原样照搬。这虽然让我在吃了不少苦头（调试程序的时间更久了，刚写完作性能测试发现比muduo差了三倍，花了几天才找到所有病症），但收获也不少，理解为何要tie就是收获之一。

# 3. 性能评测

性能优化

1\.

2\.

## 3.1 和muduo比

单线程

![image-20230306153018332](https://raw.githubusercontent.com/a504644805/resources/master/muduoZ/Performance_Test.png)

多线程

![image-20230306153030257](https://raw.githubusercontent.com/a504644805/resources/master/muduoZ/Performance_Test.png)

## 3.2 和nginx比

使用了Apache Benchmark做了压测，**与nginx对比**

```
Concurrency Level:      1000
Time taken for tests:   14.852 seconds
Complete requests:      1000000
Failed requests:        0
Keep-Alive requests:    1000000
Total transferred:      118000000 bytes
HTML transferred:       14000000 bytes
Requests per second:    67330.24 [#/sec] (mean)
Time per request:       14.852 [ms] (mean)
Time per request:       0.015 [ms] (mean, across all concurrent requests)
Transfer rate:          7758.76 [Kbytes/sec] received
```

```
Concurrency Level:      1000
Time taken for tests:   48.376 seconds
Complete requests:      1000000
Failed requests:        0
Keep-Alive requests:    1000000
Total transferred:      118000000 bytes
HTML transferred:       14000000 bytes
Requests per second:    20671.60 [#/sec] (mean)
Time per request:       48.376 [ms] (mean)
Time per request:       0.048 [ms] (mean, across all concurrent requests)
Transfer rate:          2382.08 [Kbytes/sec] received
```

```
#user  nobody;
worker_processes  4;
events {
    worker_connections  10240;
}
http {
    include       /usr/local/openresty/nginx/conf/mime.types;
    default_type  application/octet-stream;
    access_log  off;
    sendfile       on;
    tcp_nopush     on;
    keepalive_timeout  65;
    server {
        listen       8001;
        server_name  localhost;
        location / {
            root   html;
            index  index.html index.html;
        }
        location /hello {
          default_type text/plain;
          echo "hello, world!";
        }
    }
}
```

# 总结

感谢硕哥，如采访所说，muduo用意

<https://www.oschina.net/question/28_61182>
