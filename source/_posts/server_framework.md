---
title: 协作半驻留式服务器程序开发框架
date: 2009-08-15 17:01:32
tags: 服务编程
categories: 服务编程
---

# 协作半驻留式服务器程序开发框架 --- 基于 Postfix 服务器框架改造

## 1. 概述
现在大家在和 Java, PHP, #C 写应用程序时，都会用到一些成熟的服务框架，所以开发效率是比较高的。而在用C/C++写服务器程序时，用的就五花八门了，有些人用ACE等等，这类服务器框架及库比较丰富，但入门门槛比较高，所以更多的人是自己直接写服务器程序，初始写时觉得比较简单，可时间久了，便会觉得难以扩展，性能低，容易出错。其实，Postfix 作者为我们提供了一个高效、稳定、安全的服务器框架模型，虽然Postfix主要用作邮件系统的 MTA，但其框架设计却非常具有通用性。ACL 库的作者将 Postfix 的服务器框架模型抽取出来，形成了更加通用的服务器程序开发框架，使程序员在编写服务器程序时可以达到事半功倍的效果。本文主要介 绍了 ACL 库中 acl_master 服务器程序（基于Postifx服务器程序框架）的设计及功能。

## 2. 框架设计图

<img src=/img/server_framework.webp />

master主进程为控制进程，刚启动时其负责监听所有端口服务，当有新的客户端连接到达时，master便会启动子进程进行服务，而自己依然监控服务端 口，同时监控子进程的工作状态；而提供对外服务的子进程在master启动时，若没有请求任务则不会被启动，只有当有连接或任务到达时才会被master 启动，当该服务子进程处理完某个连接服务后并不立即退出，而是驻留在系统一段时间，等待可能的新连接到达，这样当有新的连接到达时master就不会启动 新的子进程，因为已经有处于空闲的子进程在等待下一个连接请求；当服务子进程空闲时间达一定阀值后，就会选择退出，将资源全部归还操作系统（当然，也可以 配置成服务子进程永不退出的模式）。因此，可以称这种服务器框架为协作式半驻留式服务器框架，下面将会对协作式和半驻留作进一步介绍。

## 3. 协作方式
Postfix服务器框架设计的非常巧妙，因为master毕竟属于用户空间进程，不能象操作系统那样可以控制每个进程的运行时间片，所以master主进程必须与其服务子进程之间协作好，以处理好以下几个过程：
- 新连接到达时，master是该启动新的子进程接管该连接还是由空闲子进程直接接管
- master何时应该启动新的子进程
- 新连接到达，空闲子进程池中的子进程如何竞争接管该连接
- 子进程异常退出时，master如何处理新连接
- 空闲子进程如何选择退出时间（空闲时间或服务次数应决定子进程的退出）
- master如何知道各个子进程的工作状态（是死了还是活着？）
- 在不停止服务的前提下，服务子进程程序如果在线更新、如何添加新的服务、如何在线更新子进程配置
- 如何减少所有子进程与master之间的通讯次数从而降低master的负载

## 4. 流程图
### 4.1. master主进程流程图

<img src=/img/master_proc.webp />

Postifx 中的 master 主进程与各个子进程之间的IPC通讯方式为管道，所以管道的个数与子进程数是成正比的。如果管道中断，则 master 认为该管道所对应的子进程已经退出，如果是异常退出，master还需要标记该服务类子进程池以防止该类子进程异常退出频繁而启动也异常频繁（如果子进程 启动过于频繁则会给操作系统造成巨大负载）；另外，如果某类服务的子进程在服务第一个连接时就异常退出，则master认为该服务有可能是不可用的，所以 当有新的连接再到达时就会延迟启动该服务子进程。

当服务子进程池中有空闲子进程时，master便会把该服务端口的监听权让出，从而该服务 的空闲子进程在该服务端口上接收新的连接。当某个子进程获得新的连接后便会立即通知master其已经处于忙状态，master便立即查找该服务的子进程 进程池还有无空闲子进程，如果有则master依然不会接管该服务端口的监听任务；如果没有了，则master立即接管该服务端口的监听任务，当有新的连 接到达时，master先检查有没有该服务的空闲进程，若有便让出该服务端口的监听权，若没有便会启动新的子进程，然后让出监听权。


### 4.2. 服务子进程流程图

<img src=/img/child_proc.webp />

在master主进程刚启动时，因为没有任何服务请求，所以子进程是不随master一起启动的，此时所有服务端口的监控工作是由master统一负责， 当有客户端连接到达时，服务子进程才由master启动，进而接收该新连接，在进一步处理客户端请求前，子进程必须让master进程知道它已经开始" 忙"了，好由master来决定是否再次接管该服务端口的监控任务，所以子进程首先向master发送“忙”消息，然后才开始接收并处理该客户端请求，当 子进程完成了对该客户端的请求任务后，需向master发送“空闲”消息，以表明自己又可以继续处理新的客户端连接任务了。这一“忙”一“闲”两个消息， 便体现了服务子进程与master主进程的协作特点。

当然，服务子进程可以选择合适的退出时机：如果自己的服务次数达到配置的阀值，或自己空闲时间达到阀值，或与master主进程之间的IPC管道中断(一 般是由master停止服务要求所有服务子进程退出时或master要求所有服务子进程重读配置时而引起的)，则服务子进程便应该结束运行了。这个停止过 程，一方面体现了子进程与master主进程之间的协作特点，另一方面也体现了子进程半驻留的特点，从而体现子进程进程池的半驻留特性，这一特性的最大好 处就是：按需分配，当请求连接比较多时，所启动运行的子进程就多，当请求连接任务较少时，只有少数的子进程在运行，其它的都退出了。

### 4.3. 小结
以上从整体上介绍了Postfix服务器框架模型中master主进程与服务子进程的逻辑流程图，当然，其内部实现要点与细节远比上面介绍的要复杂，只是 这些复杂层面别人已经帮我们屏蔽了，我们所要做的是在此基础上写出自己的应用服务来，下面简要介绍了基于Postfix的服务器框架改造抽象的ACL库中 服务器程序的开发方式，以使大家比较容易上手。

## 5. 五类服务器编程模型
要想使用ACL服务器框架库开发服务器程序，首先介绍两个个小名词：acl_master服务器主进程与服务器模板。acl_master服务器主进程与 Postfix中的master主进程功能相似，它的主要作用是启动并控制所有的服务子进程，这个程序不用我们写，是ACL服务器框架中已经写好的；所谓 服务器模板，就是我们需要根据自己的需要从ACL服务器框架中选择一种服务器模型(即服务器模板)，然后在编写服务器时按该服务器模板的方式写即可。为什 么还要选择合适的服务器框架模板？这是因为Postfix本身就提供了三类服务器应用形式(单进程单连接进程池、单进程多连接进程池、触发器进程池)，这 三类应用形式便分别使用了Postfix中的三种服务器模板，此外，ACL中又增加了两种服务器应用形式(多线程进程池、单进程非阻塞进程池)。下面分别 就这五种服务器框架模板一一做简介：

- **多进程模型：** 由 acl_master 主进程启动多个进程组成进程池提供某类服务，但每个进程每次只能处理一个客户端连接请求；
- **多线程模型：** 由 acl_master 主进程启动多个进程组成进程池提供某类服务，而每个进程是由多个线程组成，每个线程处理一个客户端连接请求；
- **非阻塞模型：** 由 acl_master 主进程启动多个进程组成进程池提供某类服务，每个进程可以并发处理多个连接(类似于Nginx, Lighttpd, Squid, Ircd)，由于采用非阻塞技术，该模型服务器的并发处理能力大大提高，同时系统资源消耗也最小；当然，该模型与单进程多连接进程池采用的技术都是非阻塞 技术，但该模型进行更多的应用封装与高级处理，使编写非阻塞程序更加容易；
- **协程模型：** 该模型采用协程方式，将阻塞IO模式在底层转变为非阻塞模式，这样既可以象编写多线程一样简单，又可以象非阻塞程序一样高效，从而使技术人员快速地编写出高并发、高性能的网络服务程序；
- **触发器模型：** 由 acl_master 主进程启动多个进程组成进程池提供定时器类服务(类似于UNIX中的crond)，当某个定时器时间到达时，便由一个进程开始运行处理任务。

以上五种服务器方式中，由于可以根据需要配置成多个进程实例，所以可以充分地利用多核的系统。其中，第5种的运行效率是最高的，当然其编程的复杂度要比其 它的高，而第1种是执行效率最低的，其实它也是最安全的(在Postfix中的smtpd/smtp 等进程就属于这一类)，而相对来说， 第4种在运行效率与编写复杂度方面是一个比较好的折衷，所以在写一般性服务器时，该服务器模型是作者推荐的方案，此外，第4种方案还有一个好处，可以做到 对于空连接不必占用线程，这样也大大提供了并发度(即线程数可以远小于连接数)。

## 6. 使用简介
本节暂不介绍具体的编程过程，只是介绍一些配置使用过程。假设已经写好了服务器程序：echo_server, 该程序可以采用上面的 1), 2), 4), 5) 中的任一服务器模型来写，假设采用了第4)种；另外，还假设：acl_master等所有文件安装的根目录为 /opt/soft/acl-master/, 主进程程序 acl_master 及 echo_server 安装在 /opt/soft/acl-master/libexec/,  acl_master 主程序配置文件安装在 /opt/soft/acl-master/conf/，echo_server 配置文件安装在 /opt/soft/acl-master/conf/service/, 日志文件目录为 /opt/soft/acl-master/var/log/, 进程号文件目录为 /opt/soft/acl-master/var/pid/。比如，你让 echo_server 的服务端口为 6601，服务地址为 127.0.0.1, 它只是提供简单的 echo 服务。

你可以运行 /opt/soft/acl-master/sh/start.sh 脚本来启动 acl_master 主进程(用 ps -ef|grep acl_master 会看到 acl_master 进程存在 ，但 ps -ef|grep echo_server 却没有发现它的存在)，然后你在一个Unix终端上 telnet 127.0.0.1 6601, 在另一个终端上 ps -ef|grep echo_server 就会发现有一个 echo_server子进程了，然后在 telnet 6601 的终端上随便输入一些字符串，便会立即得到回复，关闭该 telnet 连接，用 ps -ef|grep echo_server 会发现该进程依然存在，当再次 telnet 127.0.0.1 6601 时，该echo_server进程又继续为新连接提供服务了。可以试着多开几个终端同时 telnet 127.0.0.1 6601，看看运行效果如何？

注意，服务子进程的配置文件格式需要与其所采用的模板类型一致；进程池中最大进程个数、进程中线程池最大线程个数、进程最大服务次数、空闲时间等都是可以以配置文件中指定的。
