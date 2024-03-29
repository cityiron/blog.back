---
title: 【转载】蚂蚁金服 Service Mesh 深度实践
date: 2019-11-07 11:50:58
tags: [转载, Service Mesh]
categories: Service Mesh
---

[ServiceMesh] 敖小剑大佬在 QCon 上关于 Service Mesh 的分享

<!-- more --> 

作者丨敖小剑

> 2019 年，蚂蚁金服在 Service Mesh 领域继续高歌猛进，进入大规模落地的深水区。本文整理自蚂蚁金服高级技术专家敖小剑在 QCon 全球软件开发大会（上海站）2019 上的演讲，他介绍了 Service Mesh 在蚂蚁金服的落地情况和即将来临的双十一大考，以及大规模落地时遇到的困难和解决方案，助你了解 Service Mesh 的未来发展方向和前景。 

# 前   言
大家好，我是敖小剑，来自蚂蚁金服中间件团队，今天带来的主题是“诗和远方：蚂蚁金服 Service Mesh 深度实践”。
在过去两年，我先后在 QCon 做过两次 Service Mesh 的演讲：

- 2017 年，当时 Service Mesh 在国内还属于蛮荒时代，我当时做了一个名为“Service Mesh: 下一代微服务”的演讲，开始在国内布道 Service Mesh 技术；
- 2018 年，做了名为“长路漫漫踏歌而行：蚂蚁金服 Service Mesh 实践探索”的演讲，介绍蚂蚁金服在 Service Mesh 领域的探索性的实践，当时蚂蚁金服刚开始在 Service Mesh 探索。
今天，有幸第三次来到 QCon，给大家带来的依然是蚂蚁金服在 Service Mesh 领域的实践分享。和去年不同的是，今年蚂蚁金服进入了 Service Mesh 落地的深水区，规模巨大，而且即将迎来双十一大促考验。

> 备注：现场做了一个调研，了解听众对 Servicve Mesh 的了解程度，结果不太理想：在此之前对 Service Mesh 有了解的同学目测只有 10% 多点（肯定不到 20%）。Service Mesh 的技术布道，依然任重道远。

今天给大家带来的内容主要有三块：

- 蚂蚁金服落地情况介绍：包括大家最关心的双十一落地情况；
- 大规模落地的困难和挑战：分享一下我们过去一年中在大规模落地上遇到的问题；
- 是否采用 Service Mesh 的建议：这个问题经常被人问起，所以借这个机会给出一些中肯的建议供大家参考；

## 蚂蚁金服落地情况介绍

发展历程和落地规模

![ppt-5-1.jpg](https://res.cloudinary.com/dogbaobao/image/upload/v1573108860/blog/servermesh%E6%B7%B1%E5%BA%A6%E5%AE%9E%E8%B7%B5/05d1b1cbafe6153923c87e895f5766b4_zi7svw.jpg)

Service Mesh 技术在蚂蚁金服的落地，先后经历过如下几个阶段：

**技术预研 阶段**：2017 年底开始调研并探索 Service Mesh 技术，并确定为未来发展方向；

**技术探索 阶段**：2018 年初开始用 Golang 开发 Sidecar SOFAMosn，年中开源基于 Istio 的 SOFAMesh；

**小规模落地 阶段**：2018 年开始内部落地，第一批场景是替代 Java 语言之外的其他语言的客户端 SDK，之后开始内部小范围试点；

**规模落地 阶段**：2019 年上半年，作为蚂蚁金融级云原生架构升级的主要内容之一，逐渐铺开到蚂蚁金服内部的业务应用，并平稳支撑了 618 大促；

**全面大规模落地 阶段**：2019 年下半年，在蚂蚁金服内部的业务中全面铺开，落地规模非常庞大，而且准备迎接双十一大促；

目前 ServiceMesh 正在蚂蚁金服内部大面积铺开，我这里给出的数据是前段时间（大概 9 月中）在云栖大会上公布的数据：应用数百个，容器数量（pod 数）超过 10 万。当然目前落地的 pod 数量已经远超过 10 万，这已经是目前全球最大的 Service Mesh 集群，但这仅仅是一个开始，这个集群的规模后续会继续扩大，明年蚂蚁金服会有更多的应用迁移到 Service Mesh。

## 主要落地场景

![ppt-6-1.jpg](https://res.cloudinary.com/dogbaobao/image/upload/v1573108859/blog/servermesh%E6%B7%B1%E5%BA%A6%E5%AE%9E%E8%B7%B5/5c89467385df755e41556e92ebed4a50_zo6ypd.jpg)

目前 Service Mesh 在蚂蚁金服内部大量落地，包括支付宝的部分核心链路，落地的主要场景有：

多语言支持：目前除了支持 Java 之外，还支持 Golang，Python，C++，NodeJS 等语言的相互通信和服务治理；

应用无感知的升级：关于这一点我们后面会有特别的说明；

流量控制：经典的 Istio 精准细粒度流量控制；

RPC 协议支持：和 Istio 不同，我们内部使用的主要是 RPC 协议；

可观测性；

## Service Mesh 的实际性能数据
之前和一些朋友、客户交流过，目前在 Service Mesh 方面大家最关心的是 Service Mesh 的性能表现，包括对于这次蚂蚁金服 Service Mesh 上双十一，大家最想看到的也是性能指标。
为什么大家对性能这么关注？

![ppt-7-1.png](https://res.cloudinary.com/dogbaobao/image/upload/v1573108860/blog/servermesh%E6%B7%B1%E5%BA%A6%E5%AE%9E%E8%B7%B5/773cc2680c5bde9dc1bde04bba5c0f0a_zrmncj.png)

因为在 Service Mesh 工作原理的各种介绍中，都会提到 Service Mesh 是将原来的一次远程调用，改为走 Sidecar（而且像 Istio 是客户端和服务器端两次 Sidecar，如上图所示），这样一次远程调用就会变成三次远程调用，对性能的担忧也就自然而然的产生了：一次远程调用变三次远程调用，性能会下降多少？延迟会增加多少？

下图是我们内部的大促压测数据，对比带 SOFAMosn 和不带 SOFAMosn 的情况（实现相同的功能）。其中 SOFAMosn 是我们蚂蚁金服自行开发的基于 Golang 的 Sidecar/ 数据平面，我们用它替代了 Envoy，在去年的演讲中我有做过详细的介绍。

> SOFAMosn：
https://github.com/sofastack/sofa-mosn

![ppt-8.png](https://res.cloudinary.com/dogbaobao/image/upload/v1573119950/blog/servermesh%E6%B7%B1%E5%BA%A6%E5%AE%9E%E8%B7%B5/63d50a0b24c66db7f7c50aacf4ba56f5_tza198.png)

- CPU：CPU 使用在峰值情况下增加 8%，均值约增加 2%。在最新的一次压测中，CPU 已经优化到基本持平（低于 1%）；
- 内存：带 SOFAMosn 的节点比不带 SOFAMosn 的节点内存占用平均多 15M；
- 延迟：延迟增加平均约 0.2ms。部分场景带 SOFAMosn 比不带 SOFAMosn RT 增加约 5%，但是有部分特殊场景带 SOFAMosn 比不带 SOFAMosn RT 反而降低 7.5%；

这个性能表现，和前面"一次远程调用变三次远程调用"的背景和担忧相比有很大的反差。尤其是上面延迟的这个特殊场景，居然出现带 SOFAMosn（三次远程调用）比不带 SOFAMosn（一次远程调用） 延迟反而降低的情况。
是不是感觉不科学？

## Service Mesh 的基本思路
我们来快速回顾一下 Service Mesh 实现的基本思路：

![ppt-9.png](https://res.cloudinary.com/dogbaobao/image/upload/v1573119949/blog/servermesh%E6%B7%B1%E5%BA%A6%E5%AE%9E%E8%B7%B5/848ecf748dd573d2fe91b7e271d1ba7a_fibk70.png)

在基于 SDK 的方案中，应用既有业务逻辑，也有各种非业务功能。虽然通过 SDK 实现了代码重用，但是在部署时，这些功能还是混合在一个进程内的。

在 Service Mesh 中，我们将 SDK 客户端的功能从应用中剥离出来，拆解为独立进程，以 Sidecar 的模式部署，让业务进程专注于业务逻辑：

业务进程：专注业务实现，无需感知 Mesh；

Sidecar 进程：专注服务间通讯和相关能力，与业务逻辑无关；

我们称之为"关注点分离"：业务开发团队可以专注于业务逻辑，而底层的中间件团队（或者基础设施团队）可以专注于业务逻辑之外的各种通用功能。

通过 Sidecar 拆分为两个独立进程之后，业务应用和 Sidecar 就可以实现“独立维护”：我们可以单独更新 / 升级业务应用或者 Sidecar。

## 性能数据背后的情景分析

我们回到前面的蚂蚁金服 Service Mesh 落地后的性能对比数据：从原理上说，Sidecar 拆分之后，原来 SDK 中的各种功能只是拆分到 Sidecar 中。整体上并没有增减，因此理论上说 SDK 和 Sidecar 性能表现是一致的。由于增加了应用和 Sidecar 之间的远程调用，性能不可避免的肯定要受到影响。

首先我们来解释第一个问题：为什么性能损失那么小，和"一次远程调用变三次远程调用"的直觉不符？

![ppt-10.png](https://res.cloudinary.com/dogbaobao/image/upload/v1573119949/blog/servermesh%E6%B7%B1%E5%BA%A6%E5%AE%9E%E8%B7%B5/60600553e98ddc1196adb6882c7e4942_me36uh.png)

所谓的“直觉”，是将关注点都集中到了远程调用开销上，下意识的忽略了其他开销，比如 SDK 的开销、业务逻辑处理的开销，因此：

![ppt-10-2.png](https://res.cloudinary.com/dogbaobao/image/upload/v1573119949/blog/servermesh%E6%B7%B1%E5%BA%A6%E5%AE%9E%E8%B7%B5/d0defac5020222d694db0db8ffcd10a3_maqpt3.png)

推导出来的结果就是有 3 倍的开销，性能自然会有非常大的影响。

但是，真实世界中的应用不是这样：

1. 业务逻辑的占比很高：Sidecar 转发的资源消耗相比之下要低很多，通常是十倍百倍甚至千倍的差异；
2. SDK 也是有消耗的：即使不考虑各种复杂的功能特性，仅仅就报文（尤其是 Body）序列化的编解码开销也是不低的。而且，客户端和服务器端原有的编解码过程是需要处理 Body 的，而在 Sidecar 中，通常都只是读取 Header 而透传 Body，因此在编解码上要快很多。另外应用和 Sidecar 的两次远程通讯，都是走的 Localhost 而不是真实的网络，速度也要快非常多；

因此，在真实世界中，我们假定业务逻辑百倍于 Sidecar 的开销，而 SDK 十倍于 Sidecar 的开销，则：

![ppt-10-3.png](https://res.cloudinary.com/dogbaobao/image/upload/v1573119949/blog/servermesh%E6%B7%B1%E5%BA%A6%E5%AE%9E%E8%B7%B5/cd554734662867b192ea576cf5511fcc_nimk98.png)

推导出来的结果，性能开销从 111 增加到 113，大约增加 2%。这也就解释了为什么我们实际给出的 Service Mesh 的 CPU 和延迟的性能损失都不大的原因。当然，这里我是刻意选择了 100 和 10 这两个系数来拼凑出 2% 这个估算结果，以迎合我们前面给出“均值约增加 2%”的数据。这不是准确数值，只是用来模拟。

## 情理当中的意外惊喜

前面的分析可以解释性能开销增加不多的情景，但是，还记得我们的数据中有一个不科学的地方吗：“部分特殊场景带 SOFAMosn 比不带 SOFAMosn RT 反而降低 7.5%”。

理论上，无论业务逻辑和 SDK 的开销比 Sidecar 的开销大多少，也就是不管我们怎么优化 Sidecar 的性能，其结果也只能接近零。无论如何不可能出现多两个 Sidecar，CPU 消耗和延迟反而降低的情况。

这个“不科学”是怎么出现的？
我们继续来回顾这个 Service Mesh 的实现原理图：

![ppt-11.png](https://res.cloudinary.com/dogbaobao/image/upload/v1573119949/blog/servermesh%E6%B7%B1%E5%BA%A6%E5%AE%9E%E8%B7%B5/4827ce40f2090065f3b2ffd366e2926a_m1lobk.png)

出现性能大幅提升的主要的原因，是我们在 SOFAMosn 上做了大量的优化，特别是路由的缓存。在蚂蚁金服内部，服务路由的计算和处理是一个异常复杂的逻辑，非常耗资源。而在最近的优化中，我们为服务路由增加了缓存，从而使得服务路由的性能得到了大幅提升。因此：

![ppt-11-2.png](https://res.cloudinary.com/dogbaobao/image/upload/v1573119949/blog/servermesh%E6%B7%B1%E5%BA%A6%E5%AE%9E%E8%B7%B5/f686d573939ab6f138707e3d57c5662c_xt0ym4.png)

> 备注：这里我依然是刻意拼凑出 -7% 这个估算结果，请注意这不是准确数值，只是用来模拟示意。

也许有同学会说，这个结果不“公平”：这是优化了的服务路由实现在 PK 没有优化的服务路由实现。的确，理论上说，在 Sidecar 中做的任何性能优化，在 SDK 里面同样可以实现。但是，在 SDK 上做的优化需要等整个调用链路上的应用全部升级到优化后的 SDK 之后才能完全显现。而在传统 SDK 方案中，SDK 的升级是需要应用配合，这通常是一个漫长的等待过程。很可能代码优化和发版一周搞定，但是让全站所有应用都升级到新版本的 SDK 要花费数月甚至一年。

此时 Service Mesh 的优点就凸显出来了：Service Mesh 下，业务应用和 Sidecar 可以“独立维护” ，我们可以很方便的在业务应用无感知的情况下升级 Sidecar。因此，任何 Sidecar 的优化结果，都可以非常快速的获取收益，从而推动我们对 Sidecar 进行持续不断的升级。

前面这个延迟降低 7% 的例子，就是一个非常经典的故事：在中秋节前后，我们开发团队的同学，不辞辛苦加班加点的进行压测和性能调优，在一周之内连续做了多次性能优化，连发了多个性能优化的小版本，以“小步快跑”的方式，最后拿到了这个令大家都非常开心的结果。
总结：**持续不断的优化 + 无感知升级 = 快速获得收益**

这是一个意外惊喜，但又在情理之中：这是 SDK 下沉到基础设施并具备独立升级能力后带来的红利。

也希望这个例子，能够让大家更深刻的理解 Service Mesh 的基本原理和优势。

# 大规模落地的困难和挑战

当 Service Mesh 遇到蚂蚁金服的规模，困难和挑战也随之而来：当规模达到一定程度时，很多原本很小的问题都会急剧放大。后面我将在性能、容量、稳定性、可维护性和应用迁移几个方面给大家介绍我们遇到的挑战和实践。

![ppt-13.png](https://res.cloudinary.com/dogbaobao/image/upload/v1573119949/blog/servermesh%E6%B7%B1%E5%BA%A6%E5%AE%9E%E8%B7%B5/f686d573939ab6f138707e3d57c5662c_xt0ym4.png)

## 数据平面的优化

在数据平面上，蚂蚁金服采用了自行研发的基于 Golang 的方案：SOFAMosn。关于为什么选择全新开发 SOFAMosn，而不是直接使用 Envoy 的原因，在去年 QCon 的演讲中我有过详细的介绍，有兴趣可以了解。
前面我们给出的性能数据，实际上主要是数据平面的性能，也就是作为 Sidecar 部署的 SOFAMosn 的性能表现。从数据上看 SOFAMosn 目前的性能表现还是很不错的，这背后是我们在 SOFAMosn 上做了非常多的性能优化。

CPU 优化：在 SOFAMosn 中我们进行了 Golang 的 writev 优化，将多个包拼装一次写以降低 syscall 调用。测试中发现，Golang 1.9 的时候 writev 有内存泄露的 bug。当时 debug 的过程非常的辛苦...... 详情见我们当时给 Golang 提交的 PR：

> https://github.com/golang/go/pull/32138；

内存优化：在内存复用，我们发现报文直接解析会产生大量临时对象。SOFAMosn 通过直接复用报文字节的方式，将必要的信息直接通过 unsafe.Pointer 指向报文的指定位置来避免临时对象的产生；

延迟优化：前面我们谈到 Sidecar 是通过只解析 Header 而透传 Body 来保证性能的。针对这一点，我们进行了协议升级，以便快速读取 Header。比如我们使用的 TR 协议请求头和 Body 均为 hessian 序列化，性能损耗较大。而 Bolt 协议中 Header 是一个扁平化 map，解析性能损耗小。因此我们升级应用改走 Bolt 协议来提升 Sidecar 转发的性能。这是一个典型的针对 Sidecar 特性而做的优化；

此外还有前面特意重点介绍的路由缓存优化（也就是那个不科学的延迟降低 7% 的场景）。由于蚂蚁金服内部路由的复杂性（一笔请求经常需要走多种路由策略最终确定路由结果目标），通过对相同条件的路由结果做秒级缓存，我们成功将某核心链路的全链路 RT 降低 7%。

这里我简单给出了上述几个典型案例，双十一之后会有更多更详细的 SOFAMosn 资料分享出来，有兴趣的同学可以多关注。
在双十一过后，我们也将加大 SOFAMosn 在开源上的投入，将 SOFAMosn 做更好地模块化地抽象，并且将双十一中经过考验的各种优化放进去，预计在 2020 年的 1 月底可以发布第一个优化后的版本。

## Mixer 的性能优化

vMixer 的性能优化是个老生常谈的话题，基本上只要谈及 Istio 的性能，都避无可避。

**Mixer 的性能问题，一直都是 Istio 中最被人诟病的地方。**

尤其在 Istio 1.1/1.2 版本之后，引入 Out-Of-Process Adapter 之后，更是雪上加霜。

![ppt-15.png](https://res.cloudinary.com/dogbaobao/image/upload/v1573178175/blog/servermesh%E6%B7%B1%E5%BA%A6%E5%AE%9E%E8%B7%B5/9a4469ed722e58f706e0a5ad90eed36b_j05aiy.png)

原来 Sidecar 和 Mixer 之间的远程调用已经严重影响性能，在引入 Out-Of-Process Adapter 之后又在 Traffic 流程中引入了新的远程调用，性能更加不可接受。

从落地的角度看，**Mixer V1** 糟糕至极的性能，已经是“生命无法承受之重”。对于一般规模的生产级落地而言，Mixer 性能已经是难于接受，更不要提大规模落地……

Mixer V2 方案则给了社区希望：将 Mixer 合并进 Sidecar，引入 web assembly 进行 Adapter 扩展，这是我们期待的 Mixer 落地的正确姿势，是 Mixer 的未来，是 Mixer 的"诗和远方"。然而社区望穿秋水，但 Mixer V2 迟迟未能启动，长期处于 In Review 状态，远水解不了近渴。

因此在 Mixer 落地上，我们只能接受妥协方案，所谓"眼前的苟且"：一方面我们弃用 Mixer v1，改为在 SOFAMosn 中直接实现功能；另一方面我们并没有实现 Mixer V2 的规划。实际的落地方式是：我们只在 SOFAMosn 中提供最基本的策略检查功能如限流，鉴权等，另外可观测性相关的各种能力也都是从 SOFAMosn 直接输出。

## Pilot 的性能优化

在 Istio 中，Pilot 是一个被 Mixer 掩盖的重灾区：长期以来大家的性能关注点都在 Mixer，表现糟糕而且问题明显的 Mixer 一直在吸引火力。但是当选择放弃 Mixer（典型如官方在 Istio 新版本中提供的关闭 Mixer 的配置开关）之后，Pilot 的性能问题也就很快浮出水面。

这里简单展示一下我们在 Pilot 上做的部分性能优化：

- 序列化优化：我们全面使用 types.Any 类型，弃用 types.Struct 类型，序列化性能提升 70 倍，整体性能提升 4 倍。Istio 最新的版本中也已经将默认模式修改为 types.Any 类型。我们还进行了 CR(CustomResource) 的序列化缓存，将序列化时机从 Get/List 操作提前至事件触发时，并缓存结果。大幅降低序列化频率，压测场景下整体性能提升 3 倍，GC 频率大幅下降；
- 预计算优化：支持 Sidecar CRD 维度的 CDS /LDS/RDS 预计算，大幅降低重复计算，压测场景下整体性能提升 6 倍；支持 Gateway 维度的 CDS / LDS / RDS 预计算；计算变更事件的影响范围，支持局部推送，减少多余的计算和对无关 Sidecar 的打扰；
- 推送优化：支持运行时动态降级，支持熔断阈值调整，限流阈值调整，静默周期调整，日志级别调整；实现增量 ADS 接口，在配置相关处理上，Sidecar cpu 减少 90%，Pilot cpu 减少 42%；

这里简单解释一下，Pilot 在推送数据给 Sidecar 时，代码实现上的有些简单：Sidecar 连接上 Pilot 时；Pilot 就给 Sidecar 下发 xDS 数据。假定某个服务有 100 个实例，则这 100 个实例的 Sidecar 连接到 Pilot 时，每次都会进行一次下发数据的计算，然后进行序列化，再下发给连上来的 Sidecar。下一个 Sidecar 连接上来时，重复这些计算和序列化工作，而不管下发的数据是否完全相同，我们称之为“千人千面”。

而实际中，同一个服务往往有多个实例，Pilot 下发给这些实例的 Sidecar 的数据往往是相同的。因此我们做了优化，提前做预计算和序列化并缓存结果，以便后续重复的实例可以直接从缓存中取。因此，“千人千面”就可以优化为“千人百面”或者“千人十面”，从而大幅提高性能。

另外，对于整个 Service Mesh 体系，Pilot 至关重要。因此 Pilot 本身也应该进行保护，也需要诸如熔断 / 限流等特性。

## Service Mesh 的运维

在 Service Mesh 的运维上，我们继续坚持“线上变更三板斧”原则。这里的变更，包括发布新版本，也包括修改配置，尤其特指修改 Istio 的 CRD。

![ppt-17.png](https://res.cloudinary.com/dogbaobao/image/upload/v1573178175/blog/servermesh%E6%B7%B1%E5%BA%A6%E5%AE%9E%E8%B7%B5/04960fe09a6f9be96a24b957e64b83c4_qktckn.png)

线上变更“三板斧”指的是：

- 可灰度：任何变更，都必须是可以灰度的，即控制变更的生效范围。先做小范围内变更，验证通过之后才扩大范围；
- 可监控：在灰度过程中，必须能做到可监控，能了解到变更之后对系统的应用。如果没有可监控，则可灰度也就没有意义了；
- 可回滚：当通过监控发现变更后会引发问题时，还需要有方法可以回滚；

我们在这里额外引入了一个名为“ScopeConfig”的配置变更生效范围的控制能力，即配置变更的灰度。什么是配置变更的灰度呢？

Istio 的官方实现，默认修改配置（Istio API 对应的各种 CRD）时新修改的配置会直接全量推动到所有生效的 Sidecar，即配置变更本身无法灰度。注意这里和平时说的灰度不同，比如最常见的场景，服务 A 调用服务 B，并假定服务 A 有 100 个实例，而服务 B 有 10 个 v1 版本的服务实例正在进行。此时需要更新服务 B 到新的 v2 版本。为了验证 v2 新版本，我们通常会选择先上线一个服务 B 的 v2 版本的新实例，通过 Istio 进行流量百分比拆分，比如切 1% 的流量到新的 v2 版本的，这被称为“灰度发布”。此时新的“切 1% 流量到 v2”的 CRD 被下发到服务 A 的 Sidecar，这 100 个 Sidecar 中的每个都会执行该灰度策略。如果 v2 版本有问题不能正常工作，则只影响到 1% 的流量，即此时 Istio 的灰度控制的是 CRD 配置生效之后 Sidecar 的流量控制行为。

但是，实际生产中，配置本身也是有风险的。假设在配置 Istio CRD 时出现低级错误，不小心将新旧版本的流量比例配反了，错误配置成了 99% 的流量去 v2 版本。则当新的 CRD 配置被下发到全部 100 个服务 A 的实例时并生效时， Sidecar 控制的流量就会发生非常大的变化，造成生产事故。

为了规避这个风险，就必须引入配置变更的范围控制，比如将新的 CRD 配置下发到少数 Sidecar，验证配置无误后再扩展到其他 Sidecar。

## 应用平滑迁移的终极方案

在 Service Mesh 落地的过程中，现有应用如何平滑迁移到 Service Mesh，是一个至关重要的话题。典型如基于传统微服务框架如 SpringCloud/Dubbo 的应用，如何逐个（或者分批）的迁移到 Service Mesh 上。

蚂蚁金服在去年进行落地实践时，就特别针对应用平滑迁移进行了深入研究和探索。这个问题是 Service Mesh 社区非常关注的核心落地问题，今天我们重点分享。

在今年 9 月份的云栖大会上，蚂蚁金服推出了双模微服务的概念，如下图所示：

![ppt-18.png](https://res.cloudinary.com/dogbaobao/image/upload/v1573178175/blog/servermesh%E6%B7%B1%E5%BA%A6%E5%AE%9E%E8%B7%B5/ba16ec5865cb61a6bb0ffc239ce46bbd_thrwkm.jpg)

“双模微服务”是指传统微服务和 Service Mesh 双剑合璧，即“基于 SDK 的传统微服务”可以和“基于 Sidecar 的 Service Mesh 微服务”实现下列目标：
- 互联互通：两个体系中的应用可以相互访问；
- 平滑迁移：应用可以在两个体系中迁移，对于调用该应用的其他应用，做到透明无感知；
- 灵活演进：在互联互通和平滑迁移实现之后，我们就可以根据实际情况进行灵活的应用改造和架构演进；

双模还包括对应用运行平台的要求，即两个体系下的应用，既可以运行在虚拟机之上，也可以运行在容器 /k8s 之上。

怎么实现这么一个美好的双模微服务目标呢？

我们先来分析一下传统微服务体系和 Service Mesh 体系在服务注册 / 服务发现 / 服务相关的配置下发上的不同。

首先看传统微服务体系，其核心是服务注册中心 / 配置中心，应用通过引用 SDK 的方式来实现对接各种注册中心 / 配置中心。通常不同的注册中心 / 配置中心都有各自的实现机制和接口协议，SDK 和注册中心 / 配置中心的交互方式属于内部实现机制，并不通用。

![ppt-19.png](https://res.cloudinary.com/dogbaobao/image/upload/v1573178174/blog/servermesh%E6%B7%B1%E5%BA%A6%E5%AE%9E%E8%B7%B5/b8aaf50dcc6005b54704c8b47969b133_s8mjmh.png)

优点是支持海量数据（十万级别甚至百万级别），具备极强的分发能力，而且经过十余年间的打磨，稳定可靠可谓久经考验。市面上有很多成熟的开源产品，各大公司也都有自己的稳定实现。如阿里集团的 Nacos，蚂蚁金服的 SOFARegistry。

> SOFARegistry：
https://github.com/sofastack/sofa-registry

缺点是注册中心 / 配置中心与 SDK 通常是透传数据，即注册中心 / 配置中心只进行数据的存储和分发。大量的控制逻辑需要在 SDK 中实现，而 SDK 是嵌入到应用中的。因此，任何变更都需要改动 SDK 并要求应用升级。

再来看看 Service Mesh 方案，以 Istio 为例：

![ppt-19-2.png](https://res.cloudinary.com/dogbaobao/image/upload/v1573178175/blog/servermesh%E6%B7%B1%E5%BA%A6%E5%AE%9E%E8%B7%B5/ba198ff9e1cdc7dff6d9ab7e62f9de94_a2j8lz.png)

Service Mesh 的优点是引入了控制平面（在 Istio 中具体指 Pilot 组件），通过控制平面来提供强大的控制逻辑。而控制平面的引入，MCP/xDS 等标准协议的制订，实现了数据源和下发数据的解耦。即存储于注册中心 / 配置中心（在 Istio 中体现为 k8s api server + Galley）的数据可以有多种灵活的表现形式，如 CRD 形式的 Istio API，通过运行于 Pilot 中的 Controller 来实现控制逻辑和格式转换，最后统一转换到 xDS/UDPA。这给 API 的设计提供了非常大的施展空间，极具灵活度，扩展性非常好。

缺点也很明显，和成熟的注册中心 / 配置中心相比，支持的容量有限，下发的性能和稳定性相比之下有很大差距。

控制平面和传统注册中心 / 配置中心可谓各有千秋，尤其他们的优缺点是互补的，如何结合他们的优势？

此外，**如何打通两个体系是 Service Mesh 社区的老大难问题。** 尤其是缺乏标准化的社区方案，只能自行其是，各自为战。

最近，在综合了过去一年多的思考和探索之后，蚂蚁金服和阿里集团的同事们共同提出了一套完整的解决方案，我们戏称为“终极方案”：希望可以通过这个方案打通传统微服务体系和 Service Mesh 体系，彻底终结这个困扰已久的问题。

这个方案的核心在于：**以 MCP 和 xDS/UDPA协议为基础，融合控制平面和传统注册中心 / 配置中心。** 

![ppt-20.png](https://res.cloudinary.com/dogbaobao/image/upload/v1573178175/blog/servermesh%E6%B7%B1%E5%BA%A6%E5%AE%9E%E8%B7%B5/985e2a35f4ffb7f20a0c0b58777a904e_r6021q.png)

如上图所示，如果我们将融合控制平面和传统注册中心 / 配置中心而来的新的产品形态视为一个整体，则这个新产品形态的能力主要有三块：
1. 传统注册中心的数据存储能力：支持海量数据；
2. Service Mesh 控制平面的能力：解耦之后 API 设计的弹性和灵活度；
3. 传统注册中心的分发能力：性能、速度、稳定性；

这个新的产品形态可以理解为“带控制平面的注册中心 / 配置中心”，或者“存储 / 分发能力加强版的控制平面”。名字不重要，重要的是各节点的通讯交互协议必须标准化：

- MCP 协议：MCP 协议是 Istio 中用 于 Pilot 和 Galley 之间同步数据的协议，源自 xDS 协议。我们设想通过 MCP 协议将不同源的注册中心集成起来，目标是聚合多注册中心的数据到 Pilot 中，实现打通异构注册中心（未来也会用于多区域聚合）。
- xDS/UDPA 协议：xDS 协议源自 Envoy，是目前数据平面的事实标准，UDPA 是正在进行中的基于 xDS 协议的标准化版本。Sidecar 基于 xDS/UDPA 协议接入控制平面，我们还有进一步的设想，希望加强 SDK 方案，向 Istio 的功能靠拢，具体表现为 SDK 支持 xDS 协议（初期版本先实现最小功能集）。目标是希望在对接控制平面的前提下，应用可以在 Service Mesh 和 SDK 方案之间自由选择和迁移。
基于这个思路，我们给出如下图所示的解决方案，希望最大限度的整合传统微服务框架和 Service Mesh。其基本指导原则是：**求同存异，保持兼容。**

![ppt-21.png](https://res.cloudinary.com/dogbaobao/image/upload/v1573179051/blog/servermesh%E6%B7%B1%E5%BA%A6%E5%AE%9E%E8%B7%B5/871a4373f591669219689180bd9054c7_dsgavo.png)

上图中，蓝色部分是通用的功能模块，我们希望可以和社区一起共建。红色部分是不兼容的功能模块，但是保持 API 兼容。

具体说，右边是各种注册中心（配置中心同理）：

- Galley 和底下的 k8s API Server 可以视为一个特殊的注册中心，这是 Istio 的官方方式；
- Nacos/SOFARegistry 是阿里集团和蚂蚁金服的注册中心，支持海量规模。我们计划添加 MCP 协议的支持，直接对接 Pilot；
- 其他的注册中心，也可以通过提供 MCP 协议支持的方式，接入到这个方案中；
- 对于不支持 MCP 的注册中心，可以通过开发一个 MCP Proxy 模块以适配器模式的方式间接接入。当然最理想的状态是出现成熟的通用开源方案来-统一解决，比如 Nacos Sync 有计划做类似的事情；

左边是数据平面：

- Service Mesh 体系下的 Sidecar（如 Envoy 和蚂蚁金服的 SOFAMosn）目前都已经支持 xDS/UDPA；
- 相对来说，这个方案中比较“脑洞”的是在 SDK 方案如 Spring Cloud/Dubbo/SOFARPC 中提供 xDS 的支持，以便对接到已经汇总了全局数据的控制平面。从这个角度说，支持 xDS 的 SDK 方案，也可以视为广义的数据平面。我们希望后面可以推动社区朝这个方向推进，短期可以先简单对接，实现 xDS 的最小功能集；长期希望 SDK 方案的功能能向 Istio 看齐，实现更多的 xDS 定义的特性；
- 这个方案对运行平台没有任何特别要求，只要网络能通，应用和各个组件可以灵活选择运行在容器（k8s）中或虚拟机中。

需要特别强调的是，这个方案最大的优点在于它是一个高度标准化的社区方案：通过 MCP 协议和 xDS 协议对具体实现进行了解耦和抽象，整个方案没有绑定到任何产品和供应商。因此，我们希望这个方案不仅仅可以用于阿里集团和蚂蚁金服，也可以用于整个 Istio 社区。阿里集团和蚂蚁金服目前正在和 Istio 社区联系，我们计划将这个方案贡献出来，并努力完善和加强 Pilot 的能力，使之能够满足我们上面提到的的美好愿景：融合控制平面和传统注册中心 / 配置中心的优点，打通传统微服务框架和 Service Mesh，让应用可以平滑迁移灵活演进。

希望社区认可这个方案的同学可以参与进来，和我们一起努力来建设和完善它。

# 是否采用 Service Mesh 的建议

在过去一年间，这个问题经常被人问起。借这个机会，结合过去一年中的实践，以及相比去年此时更多的心得和领悟，希望可以给出一些更具参考价值的建议。

<center> 建议一：有没有直接痛点 </center>

有没有短期急迫需求，通常取决于当前有没有迫切需要解决的痛点。
在 Service Mesh 的发展过程中，有两个特别清晰而直接的痛点，它们甚至对 Service Mesh 的诞生起了直接的推动作用：

![ppt-23.png](https://res.cloudinary.com/dogbaobao/image/upload/v1573180592/blog/servermesh%E6%B7%B1%E5%BA%A6%E5%AE%9E%E8%B7%B5/6d03b6eb52064931afaec57faec15708_fvfsli.png)

## 多语言支持

这是 SDK 方案的天然局限，也是 Service Mesh 的天然优势。需要支持的编程语言越多，为每个编程语言开发和维护一套 SDK 的成本就越高，就有越多的理由采用 Service Mesh。

### 类库升级困难

![ppt-23-2.png](https://res.cloudinary.com/dogbaobao/image/upload/v1573180592/blog/servermesh%E6%B7%B1%E5%BA%A6%E5%AE%9E%E8%B7%B5/24b236fa51314083a2c7527a1fecfc0b_fa3o9z.png)

同样，这也是 SDK 方案的天然局限，也是 Service Mesh 的天然优势（还记得前面那个不科学的 -7% 吗？）。SDK 方案中类库和业务应用打包在一起，升级类库就不得不更新整个业务应用，而且是需要更新所有业务团队的所有应用。在大部分公司，这通常是一个非常困难的事情，而且每次 SDK 升级都要重复一次这种痛苦。

而且，这两个痛点有可能会同时存在：有多个编程语言的类库需要升级版本......

所以，第一个建议是先检查是否存在这两个痛点。

<center> 建议二：老应用升级改造 </center>

Service Mesh 的无侵入性，在老应用升级改造，尤其是希望少改代码甚至完全不改代码的情况下，堪称神器。

![ppt-24.png](https://res.cloudinary.com/dogbaobao/image/upload/v1573180591/blog/servermesh%E6%B7%B1%E5%BA%A6%E5%AE%9E%E8%B7%B5/6332252c3e62adcf3f4ac99419a3b172_lerrvx.jpg)

所以，第二个建议是，如果有老应用无改动升级改造的需求，对流量控制、安全、可观测性有诉求，则可以考虑采用  Service Mesh。

<center> 建议三：维护统一的技术栈 </center>

这个建议仅仅适用于技术力量相对薄弱的企业，这些企业普遍存在一个问题：技术力量不足，或者主要精力投放在业务实现，导致无力维护统一的技术栈，系统呈现烟囱式架构。

![ppt-25.png](https://res.cloudinary.com/dogbaobao/image/upload/v1573180591/blog/servermesh%E6%B7%B1%E5%BA%A6%E5%AE%9E%E8%B7%B5/96ec90e49af171881832820ee1853434_fmhaif.jpg)

传统烟囱式架构的常见问题有：

- 重复建设，重复造轮子；
- 不同时期，不同厂商，用不同的轮子；
- 难以维护和演进，后续成本高昂；
- 掌控力不足，容易受制于人；

这种情况下，建议引入 Service Mesh 技术，通过 Service Mesh 将非业务逻辑从应用剥离并下沉的特性，来统一整个公司的技术栈。
特别需要强调的是，对于技术力量不足、严重依赖外包和采购的企业，尤其是银行 / 保险 / 证券类金融企业，引入 Service Mesh 会有一个额外的特殊功效，至关重要：

将乙方限制在业务逻辑的实现上

即企业自行建设和控制 Service Mesh，作为统一的技术栈，在其上再开发运行业务应用。由于这些业务应用运行在 Servcie Mesh 之上，因此只需要实现业务逻辑，非业务逻辑的功能由 Servcie Mesh 来提供。通过这种方式，可以避免乙方公司借项目机会引入各种技术栈而造成技术栈混乱，导致后期维护成本超高；尤其是要避免引入私有技术栈，因为私有技术栈会造成对甲方事实上的技术绑定（甚至技术绑架）。

<center> 建议四：云原生落地 </center>

最后一个建议，和云原生有关。在去年的 QCon 演讲中，我曾经提到我们在探索 Kubernetes / Service Mesh / Serverless 结合的思路。在过去一年，蚂蚁金服一直在云原生领域做深度探索，也有一些收获。其中，有一点我们是非常明确的：**Mesh 化是云原生落地的关键步骤。**

下图展示了蚂蚁金服在云原生落地方面的基本思路：

![ppt-27.png](https://res.cloudinary.com/dogbaobao/image/upload/v1573180592/blog/servermesh%E6%B7%B1%E5%BA%A6%E5%AE%9E%E8%B7%B5/c8e92254d26180aecb7e8786bf551058_ajnylq.png)

- 最下方是云，以 Kubernetes 为核心，关于这一点社区基本已经达成共识：Kubernetes 就是云原生下的操作系统；
- 在 Kubernetes 之上，是 Mesh 层。不仅仅有我们熟悉的 Service Mesh，还有诸如 Database Mesh 和 Message Mesh 等类似的其他 Mesh 产品形态，这些 Mesh 组成了一个标准化的通信层；
- 运行在各种 Mesh 的应用，不管是微服务形态，还是传统非微服务形态，都可以借助 Mesh 的帮助实现应用轻量化，非业务逻辑的各种功能被剥离到 Mesh 中后，应用得以“瘦身减负”；
- 瘦身之后的应用，其内容主要是业务逻辑实现。这样的工作负载形式，更适合 Serverless 的要求，为接下来转型 Serverless 做好准备；

所以，我的最后一个建议是，请结合你的长远发展方向考虑：如果云原生是你的诗和远方，那么 Service Mesh 就是必由之路。

![ppt-28.png](https://res.cloudinary.com/dogbaobao/image/upload/v1573180592/blog/servermesh%E6%B7%B1%E5%BA%A6%E5%AE%9E%E8%B7%B5/22521b91cec58acee5ef41d600c5bae8_yzjzsu.png)

Kubernetes / Service Mesh / Serverless 是当下云原生落地实践的三驾马车，相辅相成，相得益彰。

<center> Service Mesh 的核心价值 </center>

在最后，重申一下 Service Mesh 的核心价值：

实现业务逻辑和非业务逻辑的分离。

前面的关于要不要采用 Service Mesh 四个建议，归根到底，最终都是对这个核心价值的延展。只有在分离业务逻辑和非业务逻辑并以 Sidecar 形式独立部署之后，才有了这四个建议所依赖的特性：
Service Mesh 的多语言支持和应用无感知升级；

无侵入的为应用引入各种高级特性如流量控制，安全，可观测性；

形成统一的技术栈；

为非业务逻辑相关的功能下沉到基础设施提供可能，帮助应用轻量化，使之专注于业务，进而实现应用云原生化；

希望大家在理解 Service Mesh 的核心价值之后，再来权衡要不要采用 Service Mesh，也希望我上面给出的四个建议可以对大家的决策有所帮助。

# 总   结

在今天的内容中，首先介绍了蚂蚁金服 Service Mesh 的发展历程，给大家展示了双十一大规模落地的规模和性能指标，并解释了这些指标背后的原理。然后分享了蚂蚁金服在 Service Mesh 大规模落地中遇到的困难和挑战，以及我们为此做的工作，重点介绍了应用平滑迁移的所谓“终极方案”；最后结合蚂蚁金服在云原生和 Service Mesh 上的实践心得，对于是否应该采用 Service Mesh 给出了几点建议。

目前蚂蚁金服正在静待今年的双十一大考，这将是 Service Mesh 的历史时刻：全球最大规模的 Service Mesh 集群，Service Mesh 首次超大规模部署...... 一切都是如此的值得期待。

请对 Service Mesh 感兴趣的同学稍后继续关注，预期在双十一之后会有一系列的分享活动：

![ppt-31-1.png](https://res.cloudinary.com/dogbaobao/image/upload/v1573180591/blog/servermesh%E6%B7%B1%E5%BA%A6%E5%AE%9E%E8%B7%B5/30cf8b6b693807c8e1aae0a4aa04dad1_skkjmn.png)

- 经验分享：会有更多的技术分享，包括落地场景，经验教训，实施方案，架构设计…
- 开源贡献：蚂蚁金服会将落地实践中的技术实现和方案以不同的方式回馈社区，推动 Service Mesh 落地实践。目前这个工作正在实质性的进行中， 请留意我们稍后公布的消息；
- 商务合作：蚂蚁金服即将推出 Service Mesh 产品，提供商业产品和技术支持，提供金融级特性，欢迎联系；
- 社区交流：ServiceMesher 技术社区继续承担国内 Service Mesh 布道和交流的重任；欢迎参加我们今年正在持续举办的 Service Mesh Meetup 活动。

![ppt-32.png](https://res.cloudinary.com/dogbaobao/image/upload/v1573180592/blog/servermesh%E6%B7%B1%E5%BA%A6%E5%AE%9E%E8%B7%B5/9d7bc39127fd9682851c8d8b5ff3ba54_mjcxbz.jpg)

今年是我在 QCon 演讲的第三年，这三年中的三次演讲，可以说是从一个侧面反映了国内 Service Mesh 发展的不同阶段：

2017 年，国内 Service Mesh 一片蛮荒的时候，我做了 Service Mesh 的布道，介绍了 Service Mesh 的原理，喊出了“下一代微服务”的口号 ;

2018 年，以蚂蚁金服为代表的国内互联网企业，陆陆续续开始了 Service Mesh 的落地探索，所谓摸着石头过河不外如是。第二次演讲我分享了蚂蚁金服的探索性实践，介绍了蚂蚁金服的 Service Mesh 落地方式和思路。

今天，2019 年，第三次演讲，蚂蚁金服已经建立起了全球最大规模的 Service Mesh 集群并准备迎接双十一的严峻挑战，这次的标题也变成了深度实践。

从布道，到探索，再到深度实践，一路走来已是三年，国内的 Service Mesh 发展，也从籍籍无名，到炙手可热，再到理性回归。Service Mesh 的落地，依然还存在非常多的问题，距离普及还有非常远的路要走，然而 Service Mesh 的方向，已经被越来越多的人了解和认可。

高晓松说："生活不止眼前的苟且，还有诗和远方"。对于 Service Mesh 这样的新技术来说，也是如此。
鸣谢 InfoQ 和 Qcon 提供的机会，让我得以每年一次的为大家分享 Service Mesh 的内容。2020 年，蚂蚁金服将继续推进和扩大 Service Mesh 落地的规模，继续引领 Service Mesh 在金融行业的实践探索。希望明年，可以有更多更深入的内容带给大家！

## 作者介绍
敖小剑，蚂蚁金服高级技术专家，十七年软件开发经验，微服务专家，Service Mesh 布道师，ServiceMesher 社区联合创始人。专注于基础架构和中间件，Cloud Native 拥护者，敏捷实践者，坚守开发一线打磨匠艺的架构师。曾在亚信、爱立信、唯品会等任职，目前就职蚂蚁金服，在中间件团队从事 Service Mesh/ Serverless 等云原生产品开发。本文整理自 10 月 18 日在 QCon 上海 2019 上的演讲内容。  

## 个人理解
我在一家创业公司的时候有过多语言的经历，当时主要 Web 端有 Java 和 Go，其它还有 Python 和 Php的，每次改动需要更新各个语言的SDK，当时就在想我们难道不能有一个跨语言的一站式方案吗。

我在做偏老派的 Java 开发的时候特别烦的一件事情就是各种 SDK ，如果 SDK 有漏洞重新发布需要通知到各个服务 Owner 更改，人少的时候吼一声可能就解决了，等团队的人越来越多，即使在群里 @ 了大家，也不一定能普及到，你可能需要一个一个去确认，往往需要折腾个几天才改完，低效乏味。

> 经历过痛苦的人才会更加憧憬美好

这篇文章给了我非常大的感触，知道小剑也有一年多了，真心羡慕走在前沿的大佬。 我从15年认识了 Java，到17年认识了 docker 和 kubernetes， 给了我巨大的冲击， 18年认识了 服务网格，servless，听到云原生的赞歌，现在我后悔了，我只是一个 Java 开发让我不满，告诫自己保持一颗热爱的心，希望自己在2020年能和云原生牵上线。







