---
layout: post
title: '保障系统稳定性的一些常见手段和策略'
date: '2019-12-03 20:00:00'
category: 运维
tags: 运维
author: lework
---
* content
{:toc}


## 术语

**鲁棒性**

鲁棒是Robust的音译，也就是健壮和强壮的意思。它也是在异常和危险情况下系统生存的能力。比如说，计算机软件在输入错误、磁盘故障、网络过载或有意攻击情况下，能否不死机、不崩溃，就是该软件的鲁棒性。

**稳定系统**

当一个实际的系统处于一个[平衡](https://baike.baidu.com/item/平衡/5828678)的状态时（就相当于小球在木块上放置的状态一样）如果受到外来作用的影响时（相当于上例中对小球施加的力），系统经过一个过渡过程仍然能够回到原来的平衡状态，我们称这个系统就是稳定的，否则称系统不稳定。一个[控制系统](https://baike.baidu.com/item/控制系统/1051898)要想能够实现所要求的控制功能就必须是稳定系统。 




## 常见手段

### 1. 容量规划

根据以往业务的流量，估算出未来（通常是即将来临的大促，节假日）的流量。以整体流量为基础，估算出每个子系统需要满足的容量大小。然后根据子系统的容量来计算出需要的资源数量，对系统进行适当的扩容。计算方式可以简单的表示为如下公式：

机器数量 = 预估容量 / 单机能力 + Buffer （一定数量的冗余）

### 2. 压测

压测是为了把系统的瓶颈压出来，而不是把系统压出问题来。 知道了系统的瓶颈之后，就可以针对业务场景进行容量规划, 即压测的目的是为了能检验服务能力是否支持可水平扩展，即加机器就可以抵抗洪峰。 

1. 单机单应用压测
2. 应用依赖资源压测，如数据库，缓存类的。
3. 单链路压测，业务接口如下单，架构，支付等核心链路
4. 全链路压测

### 3. 监控

目的是为了建立业务的可用性。

对服务进行全方面的监控，实时掌控系统的状态，对系统中出现的问题及时预警，做到早发现，早治理。

1. 拔测法(站点监控)：按照各业务的应用、功能、模块进行周期性测试其运行状态是否正常的一种方法。常见的是周期性针对web应用进行http状态码检测，缺点是在周期缺口中无法探知服务状态。
2.  日志分析法(请求日志监控)： 是通过各业务的应用、功能、模块日志进行分析得到可用性的一种方法。 常见的是针对web应用分析请求状态2xx，5xx的状态分布，2xx状态视为可用，5xx状态视为不可用。缺点是如果错误集中在一段时间内发布，而其余时间则为正常，与实际业务情况有所偏差。
3.  日志分析阈值法(请求日志监控):  是在日志分析法的基础上添加了状态阈值判断的一种可用性计划方法。 常见的是针对web应用进行日志分析后，得出每分钟正常请求的次数为10万次，那我们可以设置 一个阈值为10次。这10次的意思就是指，我们容许在1分钟内发生万分之一以内的错误。如果1分钟内发生的错误在10次以内，我们就认为在过去1分钟的状态为正常，就标记为可用，否则不可用。 最后再统计Availability状态的占比即是这个模块的可用性。当然这个阈值需要根据业务的实际情况进行调整。  
4. 端到端的全链路检测(全链路监控)：通过对业务链路上所有的模块埋点监控，将埋点数据集中分析，在业务层面探知系统可用性，以便业务流程出现问题时，可快速知道哪些模块出现了问题。
5. 基础设施监控(计算资源监控): 通过对模块运行系统资源的监控，可以获知模块所需的资源情况，周期性的对比环比，同比数据进行判定是否需要扩容，以保障模块稳定运行的性能需求。

### 4. 预案演练

对系统可能面临的问题要进行全面的预演，结合断网，断电等等灾难模拟的手段来检验系统在灾难面前的表现。

### 5. 定期进行故障复盘

就是对故障发生及处理过程重新过一遍。对这个过程的进行和思考进行回顾，反思和探究，实现稳定性及能力的提升。具体步骤：(回顾),Analyze(分析),Summary(总结),Action(行动)

### 6. 灾备

一旦系统发生灾难性故障，需要将流量切换到容灾机房，避免对大量用户造成损失。

### 7. 流程

规范的工作流程，可以避免大部分的认为失误。最好的就是将运维日常工作平台化，以平台为流程导向。


## 常见的一些稳定性策略

### 1. 限流

第一点是防止系统高负荷运行，第二点是有效利用服务器资源。  常见限流的算法包括漏桶算法和令牌桶算法。 

### 2. 降级

第一个是保障服务器基本可用，第二个是保障服务的核心服务可用。  以社区案例为例，即便是 My SQL 挂掉，也要能够保证社区为用户提供基本的可读服务。  一般降级的每个策略都是针对一个场景，预想特定场景下需要要解决什么问题；然后再梳理在这个场景下需要保留哪些核心基本服务；最后才选定技术方案，系统化的进行实现 

### 3. 隔离

隔离目的非常简单，要限制住不稳定因素导致的风险，停止传播。 如秒杀场景一个高并发的场景，可能带来的问题也比较多，在高并发场景下秒杀的时候，需要和一些正常的业务区分开来。 慢 SQL 隔离，一个资源隔离。一条慢 SQL 会导致整个服务不稳定。每请求一次线程，慢 SQL 会一直耗着当前线程，所以资源占用非常大。 

### 4. 超时

合理的设置业务的超时时间，可有效避免因第三方原因导致的接口等待时间过久。 设置超时时间的大体思路：第一步，识别业务需要的服务响应时间。比如，需要 100 毫秒去响应数据，之后统计业务里面可能需要调多少服务。第二步，统计服务日常的响应时间。第三步，分清主次，即分出哪些是核心服务。因为核心服务一旦失败，整个链路便不可用，所以可以对核心服务的时间设置的宽松一些。 设置完超时之后需要验证。

### 5. 集群

集群的目的是为了解决单点的问题。集群的形式主要有主备，即同时只有一台机器提供整个服务，可以有一台或者多台提供备份，备份不仅要包含代码层面，整个服务运行所依赖的资源都要有备份。另外一个形式是主从。主是提供一个完整的服务，从是提供部分的服务。还有一种是多主，多主指的是每一台机器的决策是对等的，都会对外提供一些服务。 





## 相关资料

* [聊聊服务稳定性保障这些事](  https://yq.aliyun.com/articles/699892?spm=a2c4e.11157919.spm-cont-list.33.146c27aelCf6gR  )

* [发现运维稳定性问题的明眸: 可用性]( https://mp.weixin.qq.com/s?__biz=MzIzNjUxMzk2NQ==&mid=2247485139&idx=1&sn=e6bc1440ca5cf4ad9b858bc40c1c3b6c&chksm=e8d7f911dfa07007257ea44538795d558d239b624e2d504ac9b38b02f93c177a409d3ea98e9c&scene=21#wechat_redirect )

* [提升运维稳定性的利器：故障复盘]( https://mp.weixin.qq.com/s?__biz=MzIzNjUxMzk2NQ==&mid=2247484819&idx=1&sn=290e0d49f8b8f138398697cbafadb266&chksm=e8d7fa51dfa073475f7c201240a5dc539e6dbc965ba7fc21b83010edff23e29c33468b91af2d&scene=21#wechat_redirect )

* [业务高速发展的运维困局，如何保证系统稳定性？]( https://www.infoq.cn/article/development-and-operation-ensure-system-stability )

* [阿里巴巴运维保障体系的一种最佳实践]( http://www.gaowei.vip/info-NWYB7T.html )

* [备战 618，京东如何保障系统稳定性？]( https://www.infoq.cn/article/6iG6b5WkvoJ3AiY_Ydis )
* [有赞全链路压测实战]( https://tech.youzan.com/pressure-test/ )