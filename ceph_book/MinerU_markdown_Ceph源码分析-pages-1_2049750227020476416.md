版权注意事项：1、书籍版权归著者和出版社所有；

2、本PDF仅用于个人获取知识,进行私底下知识交流；

3、PDF获得者不得在互联网以任何目的进行传播；如有需要，请尽量购买正版实体书！支持书籍作者！！

![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/40fa4abf-bf1f-487c-b4d7-1ee1c732e7cb/81051e3455831abd3579b042088d83f47499f14ea2e76113b11dd9f1523ef9bb.jpg)


The Source Code Analysis of Ceph 

# Ceph源码分析

常涛◎编著

# 作者简介

# 常涛

国内较早一批接触Ceph的先行者，具有多年分布式存储开发经验，曾在雅虎北京研发中心工作，后到京东、华为任职，目前在ZettaKit创业公司从事分布式存储、云计算相关技术研发。

The Source Code Analysis of Ceph 

# Ceph源码分析

常涛◎编著

![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/40fa4abf-bf1f-487c-b4d7-1ee1c732e7cb/ed518f04cd8359dd4202c9001701f2502406adb43e8d957215ba082198a5c8a2.jpg)


# 图书在版编目（CIP）数据

Ceph 源码分析 / 常涛编著. 一北京: 机械工业出版社, 2016.10 (大数据技术丛书)

ISBN 978-7-111-55207-9 

I.C… II.常… III.分布式文件系统 IV.TP316

中国版本图书馆CIP数据核字（2016）第257284号

# Ceph 源码分析

出版发行：机械工业出版社（北京市西城区百万庄大街22号邮政编码：100037）

责任编辑：吴怡

印刷：北京市荣盛彩色印刷有限公司

开 本： $186\mathrm{mm}\times 240\mathrm{mm}$ 1/16

书号：ISBN978-7-111-55207-9

责任校对：殷虹

版次：2016年11月第1版第1次印刷

印 张：16.75

定价：59.00元

凡购本书，如有缺页、倒页、脱页，由本社发行部调换

客服热线：（010）88379426 88361066

购书热线：（010）68326294 88379649 68995259

投稿热线：（010）88379604

读者信箱：hzit@hzbook.com

版权所有·侵权必究

封底无防伪标均为盗版

本书法律顾问：北京大成律师事务所 韩光/邹晓东

# 序言

自从2013年加入Ceph社区以来，我一直想写一本分析Ceph源码的书，但是两年多来提交了数万行的代码后，我渐渐放下了这个事情。Ceph每个月、每周都会发生巨大变化，我总是想让Ceph源码爱好者看到最新最棒的设计和实现，社区一线的模块维护和每周数十个代码提交集的阅读，让我很难有时间回顾和把握其他Ceph爱好者的疑问和需求点。

今天看到这本书让我非常意外，作者常涛把整个Ceph源码树枝解得恰到好处，如庖丁解牛般将Ceph的核心思想和实现展露出来。虽然目前Ceph分分钟都有新的变化，但无论是新的模块设计，还是重构已有逻辑，都是已有思想的翻新和延续，这些才是众多Ceph开发者能十年如一日改进的秘密！

我跟作者常涛虽然只有一面之缘，但是在开源社区中的交流已经足够成为彼此的相知。他对于分布式存储的设计和实现都有独到见解，其代码阅读和理解灵感更是超群。我在年前看到他一些对Ceph核心模块的创新性理解，相信这些都通过这本书展现出来了。

这本书是目前我所看到的从代码角度解读 Ceph 的最好作品，即使在全球范围内，都没有类似的书籍能够与之媲美。相信每个 Ceph 爱好者都能从这本书中找到自己心中某些疑问的解答途径。

作为Ceph社区的主要开发者，我也想在这里强调Ceph的魅力，希望每个读者都能充分感受到Ceph社区生机勃勃的态势。Ceph是开源世界中存储领域的一个里程碑！在过去很难想像，从IT巨无霸们组成的巨大存储壁垒中能够诞生一个真正被大量用户使用并投入生产环境的开源存储项目，而Ceph这个开源存储项目已经成为全球众多海量存储项目的主要选择。

众所周知，在过去十年里，IT技术领域中巨大的创新项目很多来自于开源世界，从垄断大数据的Hadoop、Spark，到风靡全球的Docker，都证明了开源力量推动了新技术的产生与发展。而再往以前看十年，从Unix到Linux，从Oracle到MySQL/PostgreSQL，从VMWare到KVM，开源世界从传统商业技术继承并给用户带来更多的选择。处于开源社区一线的我欣喜地看到，在IT基础设施领域，越来越多的创业公司从创立之初就以开源为基石，而越来越多的商业技术公司也受益于开源，大量的复杂商业软件基于开源分布式数据库、缓存存储、中间件构建。相信开源的Ceph也将成为IT创新的驱动力。正如Sage Weil在2016 Ceph Next会议上所说，Ceph将成为存储里的Linux！

王豪迈，XSKY公司CTO

2016年9月8日

# 前言

随着云计算技术的兴起和普及，云计算基石：分布式共享存储系统受到业界的重视。Ceph以其稳定、高可用、可扩展的特性，乘着开源云计算管理系统OpenStack的东风，迅速成为最热门的开源分布式存储系统。

Ceph作为一个开源的分布式存储系统，人人都可以免费获得其源代码，并能够安装部署，但是并不等于人人都能用起来，人人都能用好。用好一个开源分布式存储系统，首先要对其架构、功能原理等方面有比较好的了解，其次要有修复漏洞的能力。这些都是在采用开源分布式存储系统时所面临的挑战。

要用好 Ceph，就必须深入了解和掌握 Ceph 源代码。Ceph 源代码的实现被公认为比较复杂，阅读难度较大。阅读 Ceph 源代码，不但需要对 $\mathrm{C}++$ 语言以及 boost 库和 STL 库非常熟悉，还需要有分布式存储系统相关的基础知识以及对实现原理的深刻理解，最后还需要对 Ceph 框架和设计原理以及具体的实现细节有很好的把握。所以 Ceph 源代码的阅读是相当有挑战性的。

本着对 Ceph 源代码的浓厚兴趣以及实践工作的需要，需要对 Ceph 在源代码层级有比较深入的了解。当时笔者尽可能地搜索有关 Ceph 源代码的介绍，发现这方面的资料比较少，笔者只能自己对着 Ceph 源代码开始了比较艰辛的阅读之旅。在这个过程中，每一个小的进步都来之不易，理解一些实现细节，都需要对源代码进行反复地推敲和琢磨。自己在阅读的过程中，特别希望有人能够帮助理清整体代码的思路，能够解答一下关键的实现细节。本书就是秉承这样一个简单的目标，希望指引和帮助广大 Ceph 爱好者更好地理解和掌握 Ceph 源代码。

本书面向热爱 Ceph 的开发者，想深入了解 Ceph 原理的高级运维人员，想基于 Ceph 做优化和定制的开发人员，以及想对社区提交代码的研究人员。官网上有比较详细的介

绍 Ceph 安装部署以及操作相关的知识，希望阅读本书的人能够自己动手实践，对 Ceph 进一步了解。本书基于目前最新的 Ceph 10.2.1 版本进行分析。

本书着重介绍Ceph的整体框架和各个实现模块的实现原理，对核心源代码进行分析，包括一些关键的实现细节。存储系统的实现都是围绕数据以及对数据的操作来展开，只要理解核心的数据结构，以及数据结构的相关操作就可以大致了解核心的实现和功能。本书的写作思路是先介绍框架和原理，其次介绍相关的数据结构，最后基于数据结构，介绍相关的操作实现流程。

最后感谢一起工作过的同事们，同他们在Ceph技术上进行交流沟通并加以验证实践，使我受益匪浅。感谢机械工业出版社的编辑吴怡对本书出版所做的努力，以及不断提出的宝贵意见。感谢我的妻子孙盛南女士在我写作期间默默的付出，对本书的写作提供了坚强的后盾。

由于 Ceph 源代码比较多，也比较复杂，写作的时间比较紧，加上个人的水平有限，错误和疏漏在所难免，恳请读者批评指正。有任何的意见和建议都可发送到我的邮箱changtao381@163.com，欢迎读者与我交流 Ceph 相关的任何问题。

常涛

2016年6月于北京

# 目录

序言

前言

# 第1章 Ceph整体架构

1.1 Ceph的发展历程

1.2 Ceph的设计目标 2

1.3 Ceph基本架构图 2

1.4 Ceph 客户端接口 3

1.4.1 RBD· 4 

1.4.2 CephFS 4 

1.4.3 RadosGW 4 

1.5 RADOS 6 

1.5.1 Monitor 6 

1.5.2 对象存储

1.5.3 pool和PG的概念

1.5.4 对象寻址过程 8

1.5.5 数据读写过程 9

1.5.6 数据均衡 10

1.5.7 Peering 11 

1.5.8Recovery和Backfill 11

1.5.9 纠删码 11

1.5.10 快照和克隆 12

1.5.11 Cache Tier 12 

1.5.12 Scrub 13 

1.6 本章小结

# 第2章 Ceph通用模块 14

2.1 Object 14 

2.2 Buffer 16 

2.2.1 buffer::raw 16 

2.2.2 buffer::ptr 17 

2.2.3 buffer::list 17 

2.3 线程池 19

2.3.1 线程池的启动 20

2.3.2 工作队列 20

2.3.3 线程池的执行函数 21

2.3.4 超时检查 22

2.3.5 ShardedThreadPool 22 

2.4 Finisher 23 

2.5 Throttle 23 

2.6 SafeTimer 24 

2.7 本章小结 25

# 第3章 Ceph网络通信 26

3.1 Ceph网络通信框架 26

3.1.1 Message 27 

3.1.2 Connection 29 

3.1.3 Dispatcher 29 

3.1.4 Messenger 29 

3.1.5 网络连接的策略 30

3.1.6 网络模块的使用 30

3.2 Simple 实现 32

3.2.1 SimpleMessenger 33 

3.2.2 Acceptor 33 

3.2.3 DispatchQueue 33 

3.2.4 Pipe 34 

3.2.5 消息的发送 35

3.2.6 消息的接收 36

3.2.7 错误处理 37

3.3 本章小结

# 第4章CRUSH数据分布算法 39

4.1 数据分布算法的挑战 39

4.2 CRUSH算法的原理 40

4.2.1 层级化的ClusterMap· 40

4.2.2 Placement Rules 42 

4.2.3Bucket随机选择算法 46

4.3 代码实现分析

4.3.1 相关的数据结构 49

4.3.2 代码实现

4.4 对CRUSH算法的评价 52

4.5 本章小结

# 第5章 Ceph客户端 53

5.1 Librados 53 

5.1.1 RadosClient 54 

5.1.2 IoCtxImpl 56 

5.2 OSDC 56 

5.2.1 ObjectOperation 56 

5.2.2 op_target 57 

5.2.3 Op 57 

5.2.4 Striper 58 

5.2.5 ObjectCacher 59 

# 5.3 客户写操作分析 59

5.3.1 写操作消息封装 60

5.3.2 发送数据 op Submit 61

5.3.3 对象寻址 calc_target 61

# 5.4 CIs 62

5.4.1 模块以及方法的注册 62

5.4.2 模块的方法执行 63

5.4.3 举例说明 64

# 5.5 Librbd 65

5.5.1 RBD 的相关的对象 65

5.5.2 RBD元数据操作 66

5.5.3 RBD数据操作 67

5.5.4 RBD的快照和克隆 69

# 5.6 本章小结· 71

# 第6章 Ceph的数据读写

6.1 OSD模块静态类图 72

6.2 相关数据结构 73

6.2.1 Pool 74 

6.2.2 PG 75 

6.2.3 OSDMap 75 

6.2.4 OSDOp 77 

6.2.5 Object_info_t 77 

6.2.6 ObjectState 78 

6.2.7 SnapSetContext 79 

6.2.8 ObjectContext 79 

6.2.9 Session 80 

6.3 读写操作的序列图 81

6.4 读写流程代码分析 83

6.4.1 阶段1：接收请求 83

6.4.2 阶段2：OSD的op_wq处理 85

6.4.3 阶段3：PGBackend的处理 95

6.4.4 从副本的处理

6.4.5 主副本接收到从副本的应答 95

6.5 本章小结 96

# 第7章 本地对象存储

7.1 基本概念介绍 98

7.1.1 对象的元数据 98

7.1.2 事务和日志的基本概念 98

7.1.3 99 

7.2 ObjectStore对象存储接口 100

7.2.1 对外接口说明 101

7.2.2 ObjectStore代码示例 101

7.3 日志的实现 … 102

7.3.1 Jouanal 对外接口 … 102

7.3.2FileJournal 103 

7.4 FileStore的实现 109

7.4.1 日志的三种类型 110

7.4.2 JournalingObjectStore 111 

7.4.3 Filestore的更新操作 112

7.4.4 日志的应用 115

7.4.5 日志的同步 115

7.5omap的实现 116

7.5.1omap存储 117

7.5.2 omap的克隆 118

7.5.3 部分代码实现分析 119

7.6 CollectionIndex 120 

7.6.1 CollectIndex 接口 ..... 122

7.6.2 HashIndex 123 

7.6.3 LFNIndex 124 

7.7 本章小结 ………………………………………… 124

# 第8章 Ceph纠删码 125

8.1 EC的基本原理 125

8.2 EC的不同插件 126

8.2.1 RS编码 126

8.2.2 LRC编码 126

8.2.3 SHEC编码 128

8.2.4 EC和副本的比较 129

8.3 Ceph中EC的实现 129

8.3.1 Ceph 中 EC 的基本概念 … 129

8.3.2 EC支持的写操作 130

8.3.3 EC的回滚机制 131

8.4 EC的源代码分析 132

8.4.1 EC的写操作 132

8.4.2 EC的write_full 133

8.4.3 ECBackend 133 

8.5 本章小结 ………………………………………… 133

# 第9章 Ceph快照和克隆 134

9.1 基本概念 134

9.1.1 快照和克隆 134

9.1.2 RDB的快照和克隆比较 135

9.2 快照实现的核心数据结构 137

9.3 快照的工作原理 … 139

9.3.1 快照的创建 139

9.3.2 快照的写操作 139

9.3.3 快照的读操作 140

9.3.4 快照的回滚 141

9.3.5 快照的删除 141

9.4 快照读写操作源代码分析 141

9.4.1 快照的写操作 141

9.4.2 make_writeable 函数 ..... 142

9.4.3 快照的读操作 145

9.5 本章小结 ………………………………………… 146

# 第10章 Ceph Peering机制 147

10.1 statechart状态机 147

10.1.1 状态 147

10.1.2 事件

10.1.3 状态响应事件 148

10.1.4 状态机的定义 149

10.1.5 context函数 150

10.1.6 事件的特殊处理 ..... 150

10.2 PG状态机· 151

10.3 PG的创建过程 151

10.3.1 PG在主OSD上的创建 151

10.3.2 PG在从OSD上的创建 153

10.3.3 PG的加载 154

10.4 PG创建后状态机的状态转换 154

10.5 Ceph 的 Peering 过程分析 ..... 156

10.5.1 基本概念 156

10.5.2 PG日志 159

10.5.3 Peering的状态转换图 166

10.5.4 pg_info 数据结构 … 167

10.5.5 GetInfo 169 

10.5.6 GetLog 176 

10.5.7 GetMissing 181 

10.5.8 Active操作· 183

10.5.9 副本端的状态转移 187

10.5.10 状态机异常处理 188

10.6 本章小结 ………………………………………… 188

# 第11章 Ceph数据修复 189

11.1 资源预约 190

11.2 数据修复状态转换图 191

11.3Recovery过程· 193

11.3.1 触发修复 193

11.3.2 ReplicatedPG 195 

11.3.3 pgfrontend 199 

11.4 Backfill过程 205

11.4.1 相关数据结构 205

11.4.2 Backfill的具体实现 205

11.5 本章小结 210

# 第12章 Ceph一致性检查 211

12.1 端到端的数据校验 211

12.2 Scrub概念介绍 213

12.3 Scrub的调度 213

12.3.1 相关数据结构 214

12.3.2 Scrub的调度实现 214

12.4 Scrub的执行 217

12.4.1 相关数据结构 217

12.4.2 Scrub的控制流程 219

12.4.3 构建ScrubMap 221

12.4.4 从副本处理 ..... 224

12.4.5 副本对比 225

12.4.6 结束 Scrub 过程 ..... 228

12.5 本章小结 228

# 第13章 Ceph自动分层存储 230

13.1 自动分层存储技术 230

13.2 Ceph分层存储架构和原理 231

13.3 CacheTier的模式 231

13.4 CacheTier的源码分析 234

13.4.1 pool 中的 Cache Tier 数据结构 ..... 234

13.4.2 HitSet 236 

13.4.3 CacheTier的初始化 237

13.4.4 读写路径上的 CacheTier 处理 ..... 238

13.4.5 cache的flush和evict操作 245

13.5 本章小结 … 250

# 第1章

# Chapter 1

# Ceph 整体架构

本章从比较高的层次对 Ceph 的发展历史、Ceph 的设计目标、整体架构进行简要介绍。其次介绍 Ceph 的三种对外接口：块存储、对象存储、文件存储。还介绍 Ceph 的存储基石 RADOS 系统的一些基本概念、各个模块组成和功能。最后介绍了对象的寻址过程和数据读写的原理，以及 RADOS 实现的数据服务等。

# 1.1 Ceph的发展历程

Ceph项目起源于其创始人Sage Weil在加州大学Santa Cruz分校攻读博士期间的研究课题。项目的起始时间为2004年，在2006年基于开源协议开源了Ceph的源代码。Sage Weil也相应成立了Inktank公司专注于Ceph的研发。在2014年5月，该公司被Red Hat收购。Ceph项目的发展历程如图1-1所示。

2012年，Ceph发布了第一个稳定版本。2014年10月，Ceph开发团队发布了Ceph的第七个稳定版本Giant。到目前为止，社区平均每三个月发布一个稳定版本，目前的最新版本为10.2.1。

![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/40fa4abf-bf1f-487c-b4d7-1ee1c732e7cb/d21b31e4ebeb666721db04732b43e0337a4a4aad139e54ab229aa21094723409.jpg)



图1-1 Ceph的发展历程


# 1.2 Ceph的设计目标

Ceph的设计目标是采用商用硬件（Commodity Hardware）来构建大规模的、具有高可用性、高可扩展性、高性能的分布式存储系统。

商用硬件一般指标准的 x86 服务器，相对于专用硬件，性能和可靠性较差，但由于价格相对低廉，可以通过集群优势来发挥高性能，通过软件的设计解决高可用性和可扩展性。标准化的硬件可以极大地方便管理，且集群的灵活性可以应对多种应用场景。

系统的高可用性指的是系统某个部件失效后，系统依然可以提供正常服务的能力。一般用设备部件和数据的冗余来提高可用性。Ceph通过数据多副本、纠错码来提供数据的冗余。

高可扩展性是指系统可以灵活地应对集群的伸缩。一般指两个方面，一方面指集群的容量可以伸缩，集群可以任意地添加和删除存储节点和存储设备；另一方面指系统的性能随集群的增加而线性增加。

大规模集群环境下，要求Ceph存储系统的规模可以扩展到成千上万个节点。当集群规模达到一定程度时，系统在数据恢复、数据迁移、节点监测等方面会产生一系列富有挑战性的问题。

# 1.3 Ceph基本架构图

Ceph 的整体架构由三个层次组成：最底层也是最核心的部分是 RADOS 对象存储系统。

第二层是 librados 库层；最上层对应着 Ceph 不同形式的存储接口实现，架构如图 1-2 所示。

![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/40fa4abf-bf1f-487c-b4d7-1ee1c732e7cb/3e364b6cf57966332100fb1e0ab1d985b7cb170b546cd98ca6d0b2d93a5f271b.jpg)



图1-2 Ceph基本架构图


Ceph的整体架构大致如下：

□最底层基于RADOS(reliable, autonomous,distributed object store)，它是一个可靠的、自组织的、可自动修复、自我管理的分布式对象存储系统。其内部包括ceph-osd后台服务进程和ceph-mon监控进程。

□中间层 librados 库用于本地或者远程通过网络访问 RADOS 对象存储系统。它支持多种语言，目前支持 C/C++ 语言、Java、Python、Ruby 和 PHP 语言的接口。

□最上层面向应用提供3种不同的存储接口：

- 块存储接口，通过 librbd 库提供了块存储访问接口。它可以为虚拟机提供虚拟磁盘，或者通过内核映射为物理主机提供磁盘空间。

- 对象存储接口，目前提供了两种类型的API，一种是和AWS的S3接口兼容的API，另一种是和OpenStack的Swift对象接口兼容的API。

- 文件系统接口，目前提供两种接口，一种是标准的posix接口，另一种通过libcephfs库提供文件系统访问接口。文件系统的元数据服务器MDS用于提供元数据访问。数据直接通过 librados库访问。

# 1.4 Ceph 客户端接口

Ceph的设计初衷是成为一个分布式文件系统，但随着云计算的大量应用，最终变成

支持三种形式的存储：块存储、对象存储、文件系统，下面介绍它们之间的区别。

# 1.4.1 RBD

RBD（rados block device）是通过librbd库对应用提供块存储，主要面向云平台的虚拟机提供虚拟磁盘。传统SAN就是块存储，通过SCSI或者FC接口给应用提供一个独立的LUN或者卷。RBD类似于传统的SAN存储，都提供数据块级别的访问。

目前RBD提供了两个接口，一种是直接在用户态实现，通过QEMU Driver供KVM虚拟机使用。另一种是在操作系统内核态实现了一个内核模块。通过该模块可以把块设备映射给物理主机，由物理主机直接访问。

块存储用作虚拟机的硬盘，其对I/O的要求和传统的物理硬盘类似。一个硬盘应该是能面向通用需求的，既能应付大文件读写，也能处理好小文件读写。也就是说，块存储既需要有较好的随机I/O，又要求有较好的顺序I/O，而且对延迟有比较严格的要求。

# 1.4.2 CephFS

CephFS 通过在 RADOS 基础之上增加了 MDS（Metadata Server）来提供文件存储。它提供了 libcephfs 库和标准的 POSIX 文件接口。CephFS 类似于传统的 NAS 存储，通过 NFS 或者 CIFS 协议提供文件系统或者文件目录服务。

Ceph最初的设计为分布式文件系统，其通过动态子树的算法实现了多元数据服务器，但是由于实现复杂，目前还远远不能使用。目前可用于生产环境的是最新Jewel版本的CephFS为主从模式（Master-Slave）的元数据服务器。

# 1.4.3 RadosGW

RadosGW 基于 librados 提供了和 Amazon S3 接口以及 OpenStack Swift 接口兼容的对象存储接口。可将其简单地理解为提供基本文件（或者对象）的上传和下载的需求，它有两个特点：

□提供RESTfulWebAPI接口。

□它采用扁平的数据组织形式。

# 1. RESTful 的存储接口

其接口值提供了简单的GET、PUT、DEL等其他接口，对应对象文件的上传、下载、删除、查询等操作。可以看出，对象存储的I/O接口相对比较简单，其I/O访问模型都是顺序I/O访问。

# 2.扁平的数据组织形式

NAS存储提供给应用的是一个文件系统或者是一个文件夹，其实质就是一个层级化的树状存储组织模式，其嵌套层级和规模在理论上都不受限制。

这种树状的目录结构为早期本地存储系统设计的信息组织形式，比较直观，容易理解。但是随着存储系统规模的不断扩大，特别是到了云存储时代，其难以大规模扩展的缺点就暴露了出来。相比于NAS存储，对象存储放弃了目录树结构，采用了扁平化组织形式（一般为三级组织结构），这有利于实现近乎无限的容量扩展。它使用业界标准互联网协议，更加符合面向云服务的存储、归档和备份需求。

![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/40fa4abf-bf1f-487c-b4d7-1ee1c732e7cb/92723d34889cce0a5a1082b079594023dbd6451155dc228fe8ec2039b27f7caa.jpg)



图1-3 AmazonS3的对象存储结构


由于Amazon在云存储领域的影响力，Amazon的S3接口已经成为事实上的对象存储的标准接口。如图1-3所示，其接口分三级存储：Account/Bucket/Object（账户/桶/对象）。一个Account可以看作一个用户（租户），其下可以包含若干个的Bucket，一个Bucket可以拥有若干对象，其数量在理论上都不受限制。

在云计算领域，OpenStack已经成为广泛采用的云计算管理系统，OpenStack的对象存储接口Swift也成为广泛采用的接口，如图1-4所示，其也采用分三级存储：Account/

Container/Object（账户/容器/对象），每层节点数均没有限制。可以看出，Swift接口和S3类似，Swift的Container对应S3的Bucket概念。Swift接口和S3接口没有太大的区别，但是一些管理接口会有一些差别。

![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/40fa4abf-bf1f-487c-b4d7-1ee1c732e7cb/597ab8505ef491cf860798b9291ecc66e7f6fbcea122532c6c6d7248310ba1bf.jpg)



图1-4OpenStackSwift对象存储结构


# 1.5 RADOS

RADOS是Ceph存储系统的基石，是一个可扩展的、稳定的、自我管理的、自我修复的对象存储系统，是Ceph存储系统的核心。它完成了一个存储系统的核心功能，包括：Monitor模块为整个存储集群提供全局的配置和系统信息；通过CRUSH算法实现对象的寻址过程；完成对象的读写以及其他数据功能；提供了数据均衡功能；通过Peering过程完成一个PG内存达成数据一致性的过程；提供数据自动恢复的功能；提供克隆和快照功能；实现了对象分层存储的功能；实现了数据一致性检查工具Scrub。下面分别对上述基本功能做简要的介绍。

# 1.5.1 Monitor

Monitor 是一个独立部署的 daemon 进程。通过组成 Monitor 集群来保证自己的高可用。Monitor 集群通过 Paxos 算法实现了自己数据的一致性。它提供了整个存储系统的节点信息等全局的配置信息。

ClusterMap保存了系统的全局信息，主要包括：

Monitor Map 

- 包括集群的 fsid

- 所有 Monitor 的地址和端口

- current epoch 

□OSDMap：所有OSD的列表，和OSD的状态等。

□MDSMap：所有的MDS的列表和状态。

# 1.5.2 对象存储

这里所说的对象是指RADOS对象，要和RadosGW的S3或者Swift接口的对象存储区分开来。对象是数据存储的基本单元，一般默认4MB大小。图1-5就是一个对象的示意图。

![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/40fa4abf-bf1f-487c-b4d7-1ee1c732e7cb/a2a644386cc1e3c0c4141a91171e01aee28ebd6b398b39b4bdf6b81025923260.jpg)



图1-5 对象示意图


一个对象由三个部分组成：

□对象标志（ID），唯一标识一个对象。

□对象的数据，其在本地文件系统中对应一个文件，对象的数据就保存在文件中。

□对象的元数据，以Key-Value（键值对）的形式，可以保存在文件对应的扩展属性中。由于本地文件系统的扩展属性能保存的数据量有限制，RADOS增加了另一种方式：以Leveldb等的本地KV存储系统来保存对象的元数据。

# 1.5.3 pool和PG的概念

pool 是一个抽象的存储池。它规定了数据冗余的类型以及对应的副本分布策略。目前实现了两种 pool 类型：replicated 类型和 Erasure Code 类型。一个 pool 由多个 PG 构成。

PG（placement group）从名字可理解为一个放置策略组，它是对象的集合，该集合里

的所有对象都具有相同的放置策略：对象的副本都分布在相同的OSD列表上。一个对象只能属于一个PG，一个PG对应于放置在其上的OSD列表。一个OSD上可以分布多个PG。

![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/40fa4abf-bf1f-487c-b4d7-1ee1c732e7cb/7eb6a29505a9233b9a0cba439a68b7bd363cc627486e9d91063d104ed1f5ab79.jpg)



图1-6 PG的概念示意图


PG的概念如图1-6所示，其中：

PG1和PG2都属于同一个pool，所以都是副本类型，并且都是两副本。

PG1和PG2里都包含许多对象，PG1上的所有对象，主从副本分布在OSD1和OSD2上，PG2上的所有对象的主从副本分布在OSD2和OSD3上。

一个对象只能属于一个PG，一个PG包含多个对象。

□一个PG的副本分布在对应的OSD列表中。在一个OSD上可以分布多个PG。示例中PG1和PG2的从副本都分布在OSD2上。

# 1.5.4 对象寻址过程

对象寻址过程指的是查找对象在集群中分布的位置信息，过程分为两步：

1）对象到PG的映射。这个过程是静态hash映射（加入pg split后实际变成了动态hash映射方式），通过对object_id，计算出hash值，用该pool的PG的总数量pg_num对hash值取模，就可以获得该对象所在的PG的id号：

```txt
pg_id = hash(object_id) % pg_num 
```

2）PG到OSD列表映射。这是指PG上对象的副本如何分布在OSD上。它使用Ceph自己创新的CRUSH算法来实现，本质上是一个伪随机分布算法。

![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/40fa4abf-bf1f-487c-b4d7-1ee1c732e7cb/950ca661ef4c884a5ea6bcdece32d96176144d30632d0025feeecd5c8e4e58f5.jpg)



图1-7 对象寻址过程


如图1-7所示的对象寻址过程：

1）通过hash取模后计算，前三个对象分布在PG1上，后两个对象分布在PG2上。

2）PG1通过CRUSH算法，计算出PG1分布在OSD1、OSD3上；PG2通过CRUSH算法分布在OSD2和OSD4上。

# 1.5.5 数据读写过程

Ceph的数据写操作如图1-8所示，其过程如下：

![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/40fa4abf-bf1f-487c-b4d7-1ee1c732e7cb/7792ae9a2225993b43cdf4cf3124d2541c4bc3cc00d39c40a982f1ccf13a5130.jpg)



图1-8 读写过程


1）Client向该PG所在的主OSD发送写请求。

2）主OSD接收到写请求后，同时向两个从OSD发送写副本的请求，并同时写入主OSD的本地存储中。

3）主OSD接收到两个从OSD发送写成功的ACK应答，同时确认自己写成功，就向客户端返回写成功的ACK应答。

在写操作的过程中，主OSD必须等待所有的从OSD返回正确应答，才能向客户端返回写操作成功的应答。由于读操作比较简单，这里就不介绍了。

# 1.5.6 数据均衡

当在集群中新添加一个OSD存储设备时，整个集群会发生数据的迁移，使得数据分布达到均衡。Ceph数据迁移的基本单位是PG，即数据迁移是将PG中的所有对象作为一个整体来迁移。

迁移触发的流程为：当新加入一个OSD时，会改变系统的CRUSHMap，从而引起对象寻址过程中的第二步，PG到OSD列表的映射发生了变化，从而引发数据的迁移。

举例来说明数据迁移过程，表1-1是数据迁移前的PG分布，表1-2是数据迁移后的PG分布。


表 1-1 数据迁移前的 PG 分布


<table><tr><td></td><td>OSD1</td><td>OSD2</td><td>OSD3</td></tr><tr><td>PGa</td><td>PGa1</td><td>PGa2</td><td>PGa3</td></tr><tr><td>PGb</td><td>PGb3</td><td>PGb1</td><td>PGb2</td></tr><tr><td>PGc</td><td>PGc2</td><td>PGc3</td><td>PGc1</td></tr><tr><td>PGd</td><td>PGd1</td><td>PGd2</td><td>PGd3</td></tr></table>


表 1-2 数据迁移后的 PG 分布


<table><tr><td></td><td>OSD1</td><td>OSD2</td><td>OSD3</td><td>OSD4</td></tr><tr><td>PGa</td><td>PGa†</td><td>PGa2</td><td>PGa3</td><td>PGa1</td></tr><tr><td>PGb</td><td>PGb3</td><td>PGb1</td><td>PGb2</td><td>PGb2</td></tr><tr><td>PGc</td><td>PGc2</td><td>PGc3</td><td>PGc1</td><td>PGc3</td></tr><tr><td>PGd</td><td>PGd1</td><td>PGd2</td><td>PGd3</td><td></td></tr></table>

当前系统有3个OSD，分布为OSD1、OSD2、OSD3；系统有4个PG，分布为PGa、

PGb、PGc、PGd；PG设置为三副本：PGa1、PGa2、PGa3分别为PGa的三个副本。PG的所有分布如表1-1所示，每一行代表PG分布的OSD列表。

当添加一个新的OSD4时，CRUSHMap变化导致通过CRUSH算法来计算PG到OSD的分布发生了变化。如表1-2所示：PGa的映射到了列表[OSD4，OSD2，OSD3]上，导致PGa1从OSD1上迁移到了OSD4上。同理，PGb2从OSD3上迁移到OSD4，PGc3从OSD2上迁移到OSD4上，最终数据达到了基本平衡。

# 1.5.7 Peering

当OSD启动，或者某个OSD失效时，该OSD上的主PG会发起一个Peering的过程。Ceph的Peering过程是指一个PG内的所有副本通过PG日志来达成数据一致的过程。当Peering完成之后，该PG就可以对外提供读写服务了。此时PG的某些对象可能处于数据不一致的状态，其被标记出来，需要恢复。在写操作的过程中，遇到处于不一致的数据对象需要恢复的话，则需要等待，系统优先恢复该对象后，才能继续完成写操作。

# 1.5.8 Recovery和Backfill

Ceph的Recovery过程是根据在Peering的过程中产生的、依据PG日志推算出的不一致对象列表来修复其他副本上的数据。

Recovery过程的依据是根据PG日志来推测出不一致的对象加以修复。当某个OSD长时间失效后重新加入集群，它已经无法根据PG日志来修复，就需要执行Backfill（回填）过程。Backfill过程是通过逐一对比两个PG的对象列表来修复。当新加入一个OSD产生了数据迁移，也需要通过Backfill过程来完成。

# 1.5.9 纠删码

纠错码（Erasure Code）的概念早在20世纪60年代就提出来了，最近几年被广泛应用在存储领域。它的原理比较简单：将写入的数据分成N份原始数据块，通过这N份原始数据块计算出M份效验数据块，N+M份数据块可以分别保存在不同的设备或者节点中。可以允许最多M个数据块失效，通过N+M份中的任意N份数据，就还原出其他数据块。

目前 Ceph 对纠删码（EC）的支持还比较有限。RBD 目前不能直接支持纠删码（EC）模式。其或者应用在对象存储 radosgw 中，或者作为 Cache Tier 的二层存储。其中的原因和具体实现都将在后面的章节详细介绍。

# 1.5.10 快照和克隆

快照（snapshot）就是一个存储设备在某一时刻的全部只读镜像。克隆（clone）是在某一时刻的全部可写镜像。快照和克隆的区别在于快照只能读，而克隆可写。

RADOS 对象存储系统本身支持 Copy-on-Write 方式的快照机制。基于这个机制，Ceph 可以实现两种类型的快照，一种是 pool 级别的快照，给整个 pool 中的所有对象统一做快照操作。另一种就是用户自己定义的快照实现，这需要客户端配合实现一些快照机制。RBD 的快照实现就属于后者。

RBD的克隆实现是在基于RBD的快照基础上，在客户端librbd上实现了Copy-on-Write（cow）克隆机制。

# 1.5.11 Cache Tier

RADOS 实现了以 pool 为基础的自动分层存储机制。它在第一层可以设置 cache pool，其为高速存储设备（例如 SSD 设备）。第二层为 data pool，使用大容量低速存储设备（如 HDD 设备）可以使用 EC 模式来降低存储空间。通过 Cache Tier，可以提高关键数据或者热点数据的性能，同时降低存储开销。

CacheTier的结构如图1-9所示，说明如下：

□ Ceph Client 对于 Cache 层是透明的。

□类Objecter负责请求是发给CacheTier层，还是发给StorageTier层。

- CacheTier层为高速I/O层，保存热点数据，或称为活跃的数据。

□ Storage Tier层为慢速层，保存非活跃的数据。

□在CacheTier层和StorageTier层之间，数据根据活跃度自动地迁移。

![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/40fa4abf-bf1f-487c-b4d7-1ee1c732e7cb/51a8fbf0a5b69a2fd88391e2b883f37b41a2fcd826754cb921214ff3cb4f7618.jpg)



图1-9 CacheTier结构图


# 1.5.12 Scrub

Scrub 机制用于系统检查数据的一致性。它通过在后台定期（默认每天一次）扫描，比较一个 PG 内的对象分别在其他 OSD 上的各个副本的元数据和数据来检查是否一致。根据扫描的内容分为两种，第一种是只比较对象各个副本的元数据，它代价比较小，扫描比较高效，对系统影响比较小。另一种扫描称为 deep scrub，它需要进一步比较副本的数据内容检查数据是否一致。

# 1.6 本章小结

本章介绍了Ceph的系统架构，通过本章，可以对Ceph的基本架构和各个模块的组件有了整体的了解，并对一些基本概念及读写的原理、各个数据功能模块有了大致了解。

本章介绍的 Ceph 客户端的基本概念，在第 5 章会详细介绍到。本章介绍的对象寻址的 CRUSH 算法将会在第 4 章详细介绍到。在本章介绍的本地对象存储将会在第 7 章详细介绍到。本章介绍的数据读写流程，将会在第 6 章详细介绍。本章介绍的纠删码将在第 8 章中详细介绍。本章介绍的快照和克隆将在第 9 章详细介绍。本章介绍的 Ceph Peering 的过程将会在第 10 章详细介绍。本章介绍的数据恢复和回填将会在第 11 章介绍到。本章介绍的 Ceph Srcub 机制将会在第 12 章详细介绍。本章介绍的 Cache Tier 将在第 13 章详细介绍。

# 第2章

# Ceph 通用模块

本章介绍 Ceph 源代码通用库中的一些比较关键而又比较复杂的数据结构。Object 和 Buffer 相关的数据结构是普遍使用的。线程池 ThreadPool 可以提高消息处理的并发能力。Finisher 提供了异步操作时来执行回调函数。Throttle 在系统的各个模块各个环节都可以看到，它用来限制系统的请求，避免瞬时大量突发请求对系统的冲击。SafteTimer 提供了定时器，为超时和定时任务等提供了相应的机制。理解这些数据结构，能够更好理解后面章节的相关内容。

# 2.1 Object

对象 Object 是默认为 4MB 大小的数据块。一个对象就对应本地文件系统中的一个文件。在代码实现中，有 object、sobject、hobject、ghobject 等不同的类。

结构 object_t 对应本地文件系统的一个文件，name 就是对象名：

```txt
struct object_t {
    string name;
    ...
} 
```

subject_t在object_t之上增加了snapshot信息，用于标识是否是快照对象。数据成员snap为快照对象的对应的快照序号。如果一个对象不是快照对象（也就是head对象），那么snap字段就被设置为CEPH_NOSNAP值。

```txt
struct object_t {
    object_t oid;
    snapid_t snap;
} 
```

hobject_t是名字应该是hash object的缩写。

```txt
struct hobject_t {
    object_t oid;
    snapid_t snap;
private:
    uint32_t hash;
    bool max;
    uint32_t nibblewise_key_cache;
    uint32_t hash_reverse_bits;
public:
    int64_t pool;
    string nspace;
private:
    string key;
} 
```

其在subject_t的基础上增加了一些字段：

□int64_tpool：所在的pool的id。

□ string nspace: nspace 一般为空，它用于标识特殊的对象。

□ string key: 对象的特殊标记。

□ string hash: hash 和 key 不能同时设置, hash 值一般设置为就是 pg 的 id 值。

ghobject_t在对象hobject_t的基础上，添加了generation字段和shard_id字段，这个用于ErasureCode模式下的PG：

□ shard_id用于标识对象所在的osd在EC类型的PG中的序号，对应EC来说，每个osd在PG中的序号在数据恢复时非常关键。如果是Replicate类型的PG，那么字

段就设置为NO_SHARD(-1)，该字段对于replicate是没用。

□ generation 用于记录对象的版本号。当 PG 为 EC 时，写操作需要区分写前后两个版本的 object，写操作保存对象的上一个版本（generation）的对象，当 EC 写失败时，可以 rollback 到上一个版本。

```c
struct ghobject_t {
    hobject_t hobj;
    gen_t generation;
    shard_id_t shard_id;
    bool max;
public:
    static const gen_t NO_GEN = UINT64_MAX;
} 
```

# 2.2 Buffer

Buffer就是一个命名空间，在这个命名空间下定义了Buffer相关的数据结构，这些数据结构在Ceph的源代码中广泛使用。下面介绍的buffer::raw类是基础类，其子类完成了Buffer数据空间的分配，buffer::ptr类实现了Buffer内部的一段数据，buffer::list封装了多个数据段。

# 2.2.1 buffer::raw

类buffer::raw是一个原始的数据Buffer，在其基础之上添加了长度、引用计数和额外的crc校验信息，结构如下：

```txt
class buffer::raw {
public:
    char *data; //数据指针
    unsigned len; //数据长度
    atomic_t nref; //引用计数
    mutable RwLock rclock; //读写锁，保护rcmap
    map<pair.size_t, size_t>, pair<uint32_t, uint32_t> > rcmap;
    //crc校验信息，第一个pair为数据段的起始和结束(from,to)，第二个pair是crc32校验码，pair的第一字段为basecrc32校验码，第二个字段为加上数据段后计算出的crc32校验码。
}
```

下列类都继承了buffer::raw，实现了data对应内存空间的申请：

□类 raw_malloc 实现了用 malloc 函数分配内存空间的功能。

□类classbuffer::raw_mmap_pages实现了通过mmap来把内存匿名映射到进程的地址空间。

□类classbuffer::raw_posix_aligned调用了函数posix_memalign来申请内存地址对齐的内存空间。

□类classbuffer::raw_hackAligned是在系统不支持内存对齐申请的情况下自己实现了内存地址的对齐。

□类classbuffer::raw_pipe实现了pipe做为Buffer的内存空间。

□类classbuffer::raw_char使用了 $\mathrm{C + + }$ 的new操作符来申请内存空间。

# 2.2.2 buffer::ptr

类buffer::ptr就是对于buffer::raw的一个部分数据段。结构如下：

```c
class CEPH_BUFFERER_API ptr {
    raw *_raw;
    unsigned _off, _len;
    ......
} 
```

ptr是raw里的一个任意的数据段，_off是在_raw里的偏移量，_len是ptr的长度。  
raw和ptr的示意图如图2-1所示。

![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/40fa4abf-bf1f-487c-b4d7-1ee1c732e7cb/0f7b85ee4a8cc0bdfe2cf33e6a72ebb52f5af1aaaa66953b6e723185337d505d.jpg)



图2-1 raw和ptr示意图


# 2.2.3 buffer::list

类buffer::list是一个使用广泛的类，它是多个buffer::ptr的列表，也就是多个内存数据段的列表。结构如下：

```cpp
class CEPH BUFFER_API list {
    std::list<ptr> _Buffers; // 所有的 ptr
    unsigned _len; // 所有的 ptr 的数据总长度
    unsigned __memcopy_count; // 当调用函数 rebuild 用来内存对齐时，需要内存拷贝的数据量
    ptr append_buffer; // 当有小的数据就添加到这个 buffer 里
    mutable iterator last_p; // 访问 list 的迭代器
}
```

buffer::list 的重要的操作如下所示。

□添加一个ptr到list的头部：

```txt
void push_front(ptr& bp) {
    if (bp.length() == 0)
        return;
    buffers.push_front(bp);
    len += bp.length();
} 
```

□添加一个raw到list头部中，先构造一个ptr，后添加list中：

```txt
void push_front (raw \*r) { ptr bp(r); push_front(bp); 
```

□判断内存是否以参数 align 对齐，每一个 ptr 都必须以 align 对齐：

bool buffer::list::isalicenced(unsigned align) const   
{ for(std::listptr>::const_iteratorit $=$ _bufferse.begin(); it $! =$ _bufferse.end(); ++it) if(!it->isalicenced(algn)) return false; return true; 

□添加一个字符到list中，先查看append_buffer是否有足够的空间，如果没有，就新申请一个4KB大小的空间：

```rust
void buffer:::list:::append(char c)  
{ // 检查当前的 append_buffer 是否有足够的空间  
unsigned gap = append_buffer.unused.tail_length();
```

if(!gap）{ //如果没有空间，就申请一个append_buffer！ append_buffer $=$ createAligned(CEPH BUFFERED_APPEND_SIZE, CEPH BUFFERED_APPEND_SIZE); append_buffer.set_length(0); //到目前为止，没有用到 } append(append_buffer，append_buffer.append(c)-1，1); //把该数据段添加到append_buffer中

□内存对齐：有些情况下，需要内存地址对齐，例如当以directIO方式写入数据至磁盘时，需要内存地址按内存页面大小（page）对齐，也即buffer::list的内存地址都需按page对齐。函数rebuild用来完成对齐的功能。其实现的方法也比较简单，检查没有对齐的ptr，申请一块新对齐的内存，把数据拷贝过去，释放内存空间就可以了。

□ buffer::list还集成了其他额外的一些功能：

- 把数据写入文件或从文件读取数据的功能。

- 计算数据的CRC32校验。

# 2.3 线程池

线程池（ThreadPool）在分布式存储系统的实现中是必不可少的，在Ceph的代码中广泛用到。Ceph中线程池的实现也比较复杂，结构如下：

class ThreadPool : public md_config_obs_t { CephContext *cct; string name; //线程池的名字 string lockname; //锁的名字 Mutex_lock; //线程互斥的锁，也是工作队列访问互斥的锁 Cond_cond; //锁对应的条件变量 bool_stop; //线程池是否停止的标志 int Pause; //暂时中止线程池的标志 int_draining; Cond_wait_cond; int ioprio_class, ioprio_priority; vector<WorkQueue $\text{串} >$ work_queue; //工作队列 int last_work_queue; //最后访问的工作队列 set<WorkThread\*> threads; //线程池中的工作线程

```rust
list<WorkThread*> _old Threads; //等待进joined操作的线程
int processing;
```

类 ThreadPool 里包函一些比较重要的数据成员：

□工作线程集合 Threads。

□等待Join操作的旧线程集合_old Threads。

□工作队列集合，保存所有要处理的任务。一般情况下，一个工作队列对应一个类型的处理任务，一个线程池对应一个工作队列，专门用于处理该类型的任务。如果是后台任务，又不紧急，就可以将多个工作队列放置到一个线程池里，该线程池可以处理不同类型的任务。

线程池的实现主要包括：线程池的启动过程，线程池对应的工作队列的管理，线程池对应的执行函数如何执行任务。下面分别介绍这些实现，然后介绍一些Ceph线程池实现的超时检查功能，最后介绍ShardedThreadPool的实现原理。

# 2.3.1 线程池的启动

函数 ThreadPool::start() 用来启动线程池，其在加锁的情况下，调用函数 start Threads，该函数检查当前线程数，如果小于配置的线程池，就创建新的工作线程。

# 2.3.2 工作队列

工作队列（WorkQueue）定义了线程池要处理的任务。任务类型在模板参数中指定。在构造函数里，就把自己加入到线程池的工作队列集合中：

```cpp
template<class T> class WorkQueue : public WorkQueue {
    ThreadPool *pool;
    WorkQueue(string n, time_t ti, time_t sti,ThreadPool* p) : WorkQueue_(n, ti, sti), pool(p) {
        pool->add_work_queue(this);
    }
} 
```

WorkQueue 实现了一部分功能：进队列和出队列，以及加锁，并用通过条件变量通知


相应的处理线程：


bool queue(T \*item){ pool->lock.Lock(); bool r $=$ _enqueue(item); pool->_cond SignalsOne(); pool->_lock.Unlock(); return r; } void dequeue(T \*item){ pool->_lock.Lock(); _queue(item); pool->_lock.Unlock(); } void clear(){ pool->_lock.Lock(); _clear(); pool->_lock.Unlock();   
1 

还有一部分功能，需要使用者自己定义。需要自己定义实现保存任务的容器，添加和删除的方法，以及如何处理任务的方法：

virtual bool _enqueue(T \*）=0; //从提交的任务中去除一个项  
virtual void _dequeue(T \*）=0; //去除一个项并返回原始指针  
virtual T\*_dequeue() $= 0$ ·  
virtual void _process(T \*t){ assert(0);}  
virtual void _process(T \*t，TPHandle&）{ process(t);

# 2.3.3 线程池的执行函数

函数 worker 为线程池的执行函数：

```cpp
void ThreadPool::worker(WorkThread *wt) 
```

其处理过程如下：

1）首先检查_stop标志，确保线程池没有关闭。

2）调用函数join_old Threads把旧的工作线程释放掉。检查如果线程数量大于配置的数量_num Threads，就把当前线程从线程集合中删除，并加入_old Threads队列中，并退出循环。

3）如果线程池没有暂时中止，并且 work_queue 不为空，就从 last_work_queue 开始，遍历每一个工作队列，如果工作队列不为空，就取出一个 item，调用工作队列的处理函数做处理。

# 2.3.4 超时检查

TPHandle 是一个有意思的事情。每次线程函数执行时，都会设置一个 grace 超时时间，当线程执行超过该时间，就认为是 unhealthy 的状态。当执行时间超过 suicide_grace 时，OSD 就会产生断言而导致自杀，代码如下：

```c
struct heartbeat_handle_d {
    const std::string name;
    atomic_t timeout, suicide_timeout;
    time_t grace, suicide_grace;
    std::list<heartbeat_handle_d*>::iterator list_item;
} 
```

结构 heartbeat_handle_d 记录了相关信息，并把该结构添加到 HeartbeatMap 的系统链表中保存。OSD 会有一个定时器，定时检查是否超时。

# 2.3.5 ShardedThreadPool

这里简单介绍一个 ShardedThreadPool。在之前的介绍中，ThreadPool 实现的线程池，其每个线程都有机会处理工作队列的任意一个任务。这就会导致一个问题，如果任务之间有互斥性，那么正在处理该任务的两个线程有一个必须等待另一个处理完成后才能处理，从而导致线程的阻塞，性能下降。

例2-1如表2-1所示，线程Thread1和Thread2分别正在处理Job1和Job2。

由于Job1和Job2的关联性，二者不能并发执行，只能顺序执行，二者之间用一个互斥锁来控制。如果Thread1先获得互斥锁就先执行，Thread2必须等待，直到Thread1执行

完Job1后释放了该互斥锁，Thread2获得该互斥锁后才能执行Job2。显然，这种任务的调度方式应对这种不能完全并行的任务是有缺陷的。实际上Thread2可以去执行其他任务，比如Job5。Job1和Job2既然是顺序的，就都可以交给Thread1执行。


表 2-1 ThreadPool 的处理模型示列


<table><tr><td>线程\任务</td><td>Job1</td><td>Job2</td><td>Job3</td><td>Job4</td><td>Job5</td></tr><tr><td>Thread1</td><td>*</td><td></td><td></td><td></td><td></td></tr><tr><td>Thread2</td><td></td><td>*</td><td></td><td></td><td></td></tr><tr><td>Thread3</td><td></td><td></td><td></td><td>*</td><td></td></tr><tr><td>Thread4</td><td></td><td></td><td>*</td><td></td><td></td></tr></table>

因此，引入了 ShardedThreadPool 进行管理。ShardedThreadPool 对上述的任务调度方式做了改进，其在线程的执行函数里，添加了表示线程的 thread_index：

```txt
void shardedthreadpoolworker uint32_t thread_index); 
```

具体如何实现Shard方式，还需要使用者自己去实现。其基本的思想就是：每个线程对应一个任务队列，所有需要顺序执行的任务都放在同一个线程的任务队列里，全部由该线程执行。

# 2.4 Finisher

类 Finisher 用来完成回调函数 Context 的执行，其内部有一个 FinisherThread 线程来用于执行 Context 回调函数：

class Finisher{ vector<Context $\text{串}$ > finisher_queue; //需要执行的Context，成功返回值为0 list<pair<Context\*，int> $>$ finisher_queue_rval; //需要执行的Context，返回值为int类型的有效值

# 2.5 Throttle

类Throttle用来限制消费的资源数量（也常称为槽位“slot”），当请求的slot数量达

到max值时，请求就会被阻塞，直到有新的槽位释放出来，代码如下：

```cpp
class Throttle{ CephContext \*cct; const std::string name; PerfCounters \*logger; ceph::atomic_t count,max; //count：当前占用的slot的数量 //max:sloct数量的最大值 Mutex lock; //等待的锁 list<Cond\*> cond; //等待的条件变量
```

函数 get 用于获取数量为 c 个 slot，参数 c 默认为 1，参数 m 默认为 0，如果 m 不为默认的 0 值，就用 m 值重新设置 slot 的 max 值。如果成功获取数量为 c 个 slot，就返回 true，否则就阻塞等待。例如：

```cpp
bool Throttle::get(int64_t c, int64_t m) 
```

函数get_or_fail当获取不到数量为c个slot时，就直接返回false，不阻塞等待：

```txt
bool Throttle::get_or_fail(int64_t c) 
```

函数 put 用于释放数量为 c 个 slot 资源：

```cpp
int64_t Throttle::put(int64_t c) 
```

# 2.6 SafeTimer

类 SafeTimer 实现了定时器的功能，代码如下：

```cpp
class SafeTimer   
{ CephContext \*cct; Mutex& lock; Cond cond; bool safe_callbackst; //是否是safe_callbackst SafeTimerThread \*thread; //定时器执行线程 std::multimap<utime_t,Context\*> schedule; //目标时间和定时任务执行函数Context
```

std::map<Context\*，std::multimap<utime_t，Context\*>::iterator>events; //定时任务 $\text{一} - \text{一} >$ 定时任务在shedule中的位置映射 bool stopping; //是否停止

添加定时任务的命令如下：

```cpp
void SafeTimer::add_event_at(utime_t when, Context *callback) 
```

取消定时任务的命令如下：

```txt
bool cancel_event(Context *callback); 
```

定时任务的执行如下：

```txt
void SafeTimer::timer_thread() 
```

本函数一次检查schedule中的任务是否到期，其循环检查任务是否到期执行。任务在schedule中是按照时间升序排列的。首先检查，如果第一任务没有到时间，后面的任务就不用检查了，直接终止循环。如果第一任务到了定时时间，就调用callback 函数执行，如果是 safe_callback，就必须在获取 lock 的情况下执行 Callback 任务。

# 2.7 本章小结

本章介绍了src/common目录下的一些公共库中比较常见的类的实现。BufferList在数据读写、序列化中使用比较多，它的各种不同成员函数的使用方法需要读者自己进一步了解。对于ShardedThreadPool，本章只介绍了实现的原理，具体实现在不同的场景会有不同，需要读者面对具体的代码自己去分析。

# 第3章

# Ceph 网络通信

本章介绍Ceph网络通信模块，这是客户端和服务器通信的底层模块，用来在客户端和服务器之间接收和发送请求。其实现功能比较清晰，是一个相对较独立的模块，理解起来比较容易，所以首先介绍它。

# 3.1 Ceph网络通信框架

一个分布式存储系统需要一个稳定的底层网络通信模块，用于各节点之间的互联互通。对于一个网络通信系统，要求如下：

□高性能。性能评价的两个指标：带宽和延迟

□ 稳定可靠。数据不丢包，在网络中断时，实现重连等异常处理。

网络通信模块的实现在源代码 src/msg 的目录下，其首先定义了一个网络通信的框架，三个子目录里分别对应：Simple、Async、XIO 三种不同的实现方式。

Simple是比较简单，目前比较稳定的实现，系统默认的用于生产环境的方式。它最大的特点是：每一个网络链接，都会创建两个的线程，一个专门用于接收，一个专门用于发送。这种模式实现比较简单，但是对于大规模的集群部署，大量的链接会产生大量的线

程，会消耗CPU资源，影响性能。

Async模式使用了基于事件的I/O多路复用模式。这是目前网络通信中广泛采用的方式，但是在Ceph中，官方宣称这种方式还处于试验阶段，不够稳定，还不能用于生产环境。

XIO方式使用了开源的网络通信库accelio来实现。这种方式需要依赖第三方的库accelio稳定性，需要对accelio的使用方式以及代码实现都比较熟悉。目前也处于试验阶段。特别注意的是，前两种方式只支持TCP/IP协议，XIO可以支持Infiniband网络。

在msg目录下定义了网络通信的抽象框架，它完成了通信接口和具体实现的分离。在其下分别有msg/simple子目录、msg/Async子目录、msg/xio子目录，分别对应三种不同的实现。

# 3.1.1 Message

类 Message 是所有消息的基类，任何要发送的消息，都要继承该类，格式如图 3-1 所示。

<table><tr><td>header</td><td>user_data</td><td>footer</td></tr></table>

图3-1 消息发送格式

消息的结构如下：header是消息头，类似一个消息的信封（envelope），user_data是用于要发送的实际数据，footer是一个消息的结束标记，如下所示：

```txt
class Message : public RefCountedObject {
    ceph msg header; // 消息头
    ceph msg footer; // 消息尾
    // 用户数据
    bufferlist payload; // "front" unaligned blob
    bufferlist middle; // "middle" unaligned blob
    bufferlist data; // data payload (page-alignment will be preserved where possible)
    // 消息相关的时间戳
    utime_t recv_stamp; // 开始接收数据的时间戳
    utime_t dispatch_stamp; // dispatch 的时间戳
    utime_t throttle_stamp; // 获取 throttle 的 slot 的时间戳
    utime_t recvcomplete_stamp; // 接收完成的时间戳
```

```cpp
ConnectionRef connection; //网络连接类  
uint32_t magic; //消息的魔术字  
bi::list_member-hook<> dispatch_q; //boost::intrusive需要的字段
```

下面分别介绍其中的重要参数。

ceph msg header 为消息头，它定义了消息传输相关的元数据：

```c
struct ceph msg_header {
    __le64 seq; //当前session内消息的唯一序号
    __le64 tid; //消息的全局唯一的id
    __le16 type; //消息类型
    __le16 priority; //优先级
    __le16 version; //消息编码的版本
    __le32 front_len; //payload的长度
    __le32 middle_len; //middle的长度
    __le32 data_len; //data的长度
    __le16 data_off; //对象的数据偏移量
    struct ceph_structure_name src; //消息源
    //一些旧的代码，用于兼容，如果为零就忽略
    __le16 compat_version;
    __le16 reserved;
    __le32CRC; //消息头的crc32c校验信息
} __attribute_ ((packed));
```

ceph msg footer 为消息的尾部，附加了一些 rc 校验数据和消息结束标志：

```txt
struct ceph msg footer {
    __le32 frontCRC, middleCRC, dataCRC;
    // 分别对应CRC效验码
    __le64 sig; // 消息的64位signature
    __u8 flags; // 结束标志
} __attribute__(packed));
```

消息带的数据分别保存在 payload、middle、data 这三个 bufferlist 中。payload 一般保存 Ceph 操作相关的元数据，middle 目前没有使用到，data 一般为读写的数据。

在源代码 src/messages 下定义了系统需要的相关消息，其都是 Message 类的子类。

# 3.1.2 Connection

类Connection对应端（port）对端的socket链接的封装。其最重要的接口是可以发送消息：

```c
struct Connection : public RefCountedObject {
    mutable Mutex lock; //锁保护Connection的所有字段
    Messenger *msgr;
    RefCountedObject *priv; //链接的私有数据
    int peer_type; //链接的peer类型
    entity_addr_t peer_addr; //peer的地址
    utime_t last_keepalive, last_keepalive_ACK;
    //最后一次发送keepalive的时间和最后一次接收keepalive的ACK的时间
    private:
        uint64_t features; //一些feature的标志位
    public:
        bool failed; //当值为true时，该链接为lossy链接已经失效了
        int rxBuffersers_version; //接收缓存区的版本
        map<ceph_tid_t,pair<bufferlist,int> > rxBuffersers; //接收缓冲区消息的标识ceph_tid -->(buffer,rxBuffersers_version)的映射
```

其最重要的功能就是发送消息的接口：

```txt
virtual int send_message(Message *m) = 0; 
```

# 3.1.3 Dispatcher

类 Dispatcher 是消息分发的接口，其分发消息的接口为：

virtual bool ms_dispatch(Message \*m) $= 0$ virtual void ms_fastdispatch(Message \*m); 

Server端注册该Dispatcher类用于把接收到的Message请求分发给具体处理的应用层。Client端需要实现一个Dispatcher函数，用于处理收到的ACK应对消息。

# 3.1.4 Messenger

Messenger是整个网络抽象模块，定义了网络模块的基本API接口。网络模块对外提

供的基本功能，就是能在节点之间发送和接受消息。

向一个节点发送消息的命令如下：

```txt
virtual int send_message (Message *m, const entity_inst_t& dest) = 0; 
```

注册一个 Dispatcher 用来分发消息的命令如下：

```txt
void add dispatcher_head(Dispatcher *d) 
```

# 3.1.5 网络连接的策略

Policy 定义了 Messenger 处理 Connection 的一些策略：

```c
struct Policy {
    bool lossy; //如果为 true，该当该连接出现错误时就删除
    bool server; //如果为 true，为服务端，都是被动连接
    bool standby; //如果为 true，该连接处于等待状态
    bool resetcheck; //如果为 true，该连接出错后重连
    //该connection相关的流控操作
    Throttle *throttler_bytes;
    Throttle *throttler/messages;
    //本地端的一些feature标志
    uint64_t featuressupported;
    //远程端需要的一些feature标志
    uint64_t features_required;
}
```

# 3.1.6 网络模块的使用

通过下面最基本的服务器和客户端的实例程序，了解如何调用网络通信模块提供的接口来完成收发请求消息的功能。

# 1. Server 程序分析

Server 程序源代码在 test/simple_server.cc 里，这里只展示有关网络部分的核心流程。

1）调用 Messenger 的函数 create 创建一个 Messenger 的实例，配置选项 g_conf->ms_type 为配置的实现类型，目前有三种方式：simple、async、xio:

messenger $=$ Messenger::create(g_ceph_context, g_conf->ms_type, entity_name_t::MON(-1), "simple_server", 0 /* nonce */); 

2）设置 Messenger 的属性：

```txt
messenger->set_magic(MSG_MAGIC_TRACE_CTRL);   
messenger->set_default_policy( Messenger::Policy::stateless_server(CEPH FEATURES_ALL,0)); 
```

3）对于Server，需要bind服务端地址：

```txt
r = messenger->bind/bind_addr);  
if (r < 0)  
    goto out;  
common_init_finish(g_ceph_context); 
```

4）创建一个 Dispatcher，并添加到 Messenger：

```txt
dispatcher = new SimpleDispatcher(messenger); messenger->add Dispatcher_head(decaper); 
```

5）启动 Messenger：

```txt
messenger->start();  
messenger->wait(); //本函数必须等start完成才能调用
```

SimpleDispatcher函数里实现了ms_dispatch，用于把接收到的各种请求消息分发给相关的处理函数。

# 2. Client 程序分析

源代码在test/simple_client.cc里，这里只展示有关网络部分的核心流程。

1）调用 Messenger 的函数 create 创建一个 Messenger 的实例：

messenger $=$ Messenger::create(g_ceph_context，g_conf->ms_type,entity_name_t::MON(-1),"client",getpid(); 

2）设置相关的策略：

```txt
messenger->set_magic(MSG_magic_TRACE_CTRL); 
```

```txt
messenger->set_default_policy(Messenger::Policy::lossy_client(0, 0)); 
```

3）创建Dispatcher类并添加，用于接收消息：

```txt
dispatcher = new SimpleDispatcher(messenger); messenger->add dispatcher_head(dispatcher); dispatcher->set.active(); 
```

4）启动消息：

$\mathbf{r} =$ messenger->start(); if $(\mathrm{r} <   0)$ goto out; 

5）下面开始发送请求，先获取目标Server的链接：

```txt
conn = messenger->get_connection(dest_server); 
```

6）通过Connection来发送请求消息。这里的消息发送方式都是异步发送，接收到请求消息的ACK应答消息后，将在Dispatcher的ms_dispatch或者ms_fast Dispatch处理函数里做相关的处理：

Message \*m;   
for (msg_iX $= 0$ ;msg_iX $<$ n msgs; $+ +$ msg_iX）{ //如果需要，这里要添加实际的数据 if(!n_dsize){ m $=$ new MPing(); }else{ m $=$ newsimplePing_with_data("simple_client",n_dsize); } conn->send_message(m);

综上所述，通过Ceph的网络框架发送消息比较简单。在Server端，只需要创建一个Messenger实例，设置相应的策略并绑定服务端口，然后就设置一个Dispatcher来处理接收到的请求。在Client端，只需要创建一个Messenger实例，设置相关的策略和Dispatcher用于处理返回的应答消息。通过获取对应Server的connection来发送消息即可。

# 3.2 Simple 实现

Simple在Ceph里实现比较早，目前也比较稳定，是在生产环境中使用的网络通信模块。如其名字所示，实现相对比较简单。下面具体分析一下，Simple如何实现Ceph网络

通信框架的各个模块。

# 3.2.1 SimpleMessenger

类SimpleMessenger实现了Messenger接口。

classSimpleMessenger:publicSimplePolicyMessenger{ Accepter accepter; //用于接受客户端的链接请求 DispatchQueue dispatch_queue; //接收到的请求的消息分发队列 booldid_bind; //是否绑定 u32global_seq://生成全局的消息seq ceph.spinlock_t global_seq_lock;//用于保护global_seq //地址 $\rightarrow$ pipe映射 ceph::unordered_map<entity_addr_t，Pipe\*>rank_pipe; //正在处理的pipes set<Pipe\*>acceptingpipes; //所有的pipes set<Pipe\*> pipes; //准备释放的pipes list<Pipe\*> pipe_reap_queue; //内部集群的协议版本 int cluster_protocol;

# 3.2.2 Acceptor

类 Accepter 用来在 Server 端监听端口，接收链接，它继承了 Thread 类，本身是一个线程，来不断地监听 Server 的端口：

```cpp
class Accepter : public Thread {
    SimpleMessenger *msgr;
    bool done;
    int listen_sd; //监听的端口
    uint64_t nonce;
}
```

# 3.2.3 DispatchQueue

DispatchQueue 类用于把接收到的请求保存在内部，通过其内部的线程，调用

SimpleMessenger 类注册的 Dispatch 类的处理函数来处理相应的消息：

class DispatchQueue{   
......   
mutableMutexlock;   
Condcond;   
class QueueItem{ int type; ConnectionRef con; MessageRef m; 1;   
PrioritizedQueue<QueueItem，uint64_t>mqueue; //接收消息的优先队列 set<pair<double,Message $\text{串} ^ { \text{串} }$ >marrival; //接收到的消息集合pair为（recv_time，message） map<Message\*，set<pair<double,Message>>>>>iterator>marrival_map; //消息 $\rightarrow$ 所在集合位置的映射

其内部的mqueue为优先级队列，用来保存消息，marrival保存了接收到的消息。marrival_map保存消息在集合中的位置。

函数DispatchQueue::enqueue用来把接收到的消息添加到消息队列中，函数DispatchQueue::entry为线程的处理函数，用于处理消息。

# 3.2.4 Pipe

类 Pipe 实现了 PipeConnection 的接口，它实现了两个端口之间的类似管道的功能。

对于每一个pipe，内部都有一个Reader和一个Writer线程，分别用来处理这个Pipe有关的消息接收和请求的发送。线程DelayedDelivery用于故障注入测试：

```txt
class Pipe : public RefCountedObject {
    class Reader : public Thread {
        reader_thread;
    //接收线程，用于接收数据
    class Writer : public Thread {
```

```c
} writer_thread; //发送线程，用于发送数据 SimpleMessenger \*msgr; // msgr的指针 uint64_t conn_id; //分配给Pipe自己唯一的id char \*recv_buf; //接收缓存区 int recv_max(prefetch; //接收缓冲区一次预取的最大值 int recv_ofs; //接收的偏移量 int recv_len; //接收的长度 int sd; //pipe对应的socked fd struct iovec msgvec[IOV_MAX]; //发送消息的iovec结构 int port; //链接端口 int peer_type; //链接对方的类型 entity_addr_t peer_addr; //对方地址 Messenger::Policy policy; //策略   
Mutex pipe_lock; int state; //当前链接的状态 atomic_t state Closed; //如果非0，那么状态为STATE_CLOSDED PipeConnectionRef connection_state; //PipeConnection的引用   
utime_t backoff; //backoff的时间 map<int, list<Message>>>out_q; //准备发送的消息优先队列 DispatchQueue \*in_q; //接收消息的DispatchQueue list<Message>>sent; //要发送的消息 Cond cond; bool send_keepalive; bool send_keepalive_ACK; utime_t keepalive_ACK_stamp; boolhaltdelivery; //如果Pipe队列消毁，停止增加 _u32 connect_seq,peer_global_seq; uint64_t out_seq; //发送消息的序列号 uint64_t in_seq,in_seq_aced; //接收到消息序号和ACK的序号
```

# 3.2.5 消息的发送

1）当发送一个消息时，首先要通过 Messenger 类，获取对应的 Connection：

```txt
conn = messenger->get_Connection(dest_server); 
```

具体到SimpleMessenger的实现如下所示：

a）首先比较，如果dest(addr是myinst(addr，就直接返回localconnection。

b）调用函数 lookup_pipe 在已经存在的 Pipe 中查找。如果找到，就直接返回 pipeConnectionRef；否则调用函数 connect_rank 新创建一个 Pipe，并加入到 msgr 的 register_pipe 里。

2）当获得一个Connection之后，就可以调用Connection的发送函数来发送消息。

```txt
conn->send_message(m); 
```

其最终调用了SimpleMessenger::submit_message函数：

a）如果Pipe不为空，并且状态不是Pipe::STATE_CLOSED状态，调用函数pipe $\rightarrow$ send把发送的消息添加到outq发送队列里，触发发送线程。

b）如果Pipe为空，就调用connect_rank创建Pipe，并把消息添加到out_q发送队列中。

3）发送线程 writer 把消息发送出去。通过步骤 2，要发送的消息已经保存在相应 Pipe 的 out_q 队列里，并触发了发送线程。每个 Pipe 的 Writer 线程负责发送 out_q 的消息，其线程入口函数为 Pipe::writer，实现功能：

a）调用函数 get next outgoing 从 out q 中获取消息。

b）调用函数write_message(header,footer,blist)把消息的header、footer、数据blast发送出去。

# 3.2.6 消息的接收

1）每个Pipe对应的线程Reader用于接收消息。入口函数为Pipe::reader，其功能如下：

a）判断当前的state，如果为STATE_ACCEPTING，就调用函数Pipe::accept来接收连接，如果不是STATE_CLOSED，并且不是STATE connecting状态，就接收消息。

b）先调用函数tcpread来接收一个tag。

c）根据tag，来接收不同类型的消息如下所示：

- CEPH_MSGTAG_KEEPALIVE消息。

- CEPH_MSGR_TAG_KEEPALIVE2，在CEPH_MSGR_TAG_KEEPALIVE的基础上，添加了时间。

- CEPH_MSGTAG_KEEPALIVE2_ACK。 

- CEPH MSGR TAG ACK 

- CEPH MSGR TAG MSG，这里才是接收的消息。

- CEPH MSGR TAG CLOSE 

d）调用函数 read_message 来接收消息，当本函数返回后，就完成了接收消息。

2）调用函数in_q->fast_preprocess(m)预处理消息。

3）调用函数 in_q->can_fastishly(m)，如果可以进行 fastishly，就 in_q->fastishly处理。fastishly 并不把消息加入到 mqueue 里，而是直接调用 msgr->msfastishly 函数，并最终调用注册的 fastishly 函数处理。

4）如果不能 fast_dispatch，就调用函数 in_q->enqueue(m, m->getpriority(), conn_id) 把接收到的消息加入到 DispatchQueue 的 mqueue 队列里，由 DispatchQueue 的分发线程调用 ms dispatch 处理。

ms_fastishly 和 ms_dispach 两种处理的区别在于：ms_dispach 是由 DispatchQueue 的线程处理的，它是一个单线程；ms_fast_dispach 函数是由 Pipe 的接收线程直接调用处理的，因此性能比前者要好。

# 3.2.7 错误处理

网络模块复杂的功能是如何处理网络错误。无论是接收还是发送，会出现各种异常错误，包括返回异常错误码，接收数据的 magic 验证不一致，接收的数据的效验验证不一致，等等。错误的原因主要是由于网络本身的错误（物理链路等），或者字节跳变引起的。

目前错误处理的方法比较简单，处理流程如下：

1）关闭当前socket的连接。

2）重新建立一个socket连接。

3）重新发送没有接受到ACK应对的消息。

函数Pipe::fault用来处理错误：

1）调用shutdown_SOCKET关闭pipe的socket。

2）调用函数 requeue_sent 把没有收到 ACK 的消息重新加入发送队列，当发送队列有请求时，发送线程会不断地尝试重新连接。

# 3.3 本章小结

本章介绍了Ceph的网络通信模块的框架，及目前生产环境中使用的Simple实现。它对每个链接都会有一个发送线程和接收线程用来处理发送和接收。实现的难点还在于网络链接出现错误时的各种错误处理。

![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/40fa4abf-bf1f-487c-b4d7-1ee1c732e7cb/8f4d816f8332e3412f8e6b1144b4733fed7e915b3caa06297eb66c7557aa7cbf.jpg)



第4章


Chapter 9 

# CRUSH 数据分布算法

本章介绍Ceph的数据分布算法CRUSH，它是一个相对比较独立的模块，和其他模块的耦合性比较少，功能比较清晰，比较容量理解。在客户端和服务器都有CRUSH的计算，了解它可以更好地理解后面的章节。

CRUSH算法解决了PG的副本如何分布在集群的OSD上的问题。本章首先介绍CRUSH算法的原理，并给出相应的示列，然后进一步分析其实现的一些核心代码。

# 4.1 数据分布算法的挑战

存储系统的数据分布算法要解决数据如何分布到集群中的各个节点和磁盘上，其面临如下的挑战：

□数据分布和负载的均衡。首先是数据分布均衡，使数据能均匀地分别在各个节点和磁盘上。其次是负载均衡，使数据访问（读写等操作）的负载在各个节点和磁盘上的负载均衡。

□ 灵活应对集群伸缩。系统可以方便地增加或者删除存储设备（包括节点和设备失效的处理）。当增加或者删除存储设备后，能自动实现数据的均衡，并且迁移的数据

尽可能地少。

□支持大规模集群。为了支持大规模的存储集群，就要求数据分布算法维护的元数据相对较小，并且计算量不能太大。随着集群规模的增加，数据分布算法的开销比较小。

在分布式存储系统中，数据分布算法对于分布式存储系统至关重要。目前有两种基本实现方法，一种是基于集中式的元数据查询的方式，如HDFS的实现：文件的分布信息（layout信息）是通过访问集中式元数据服务器获得；另一种是基于分布式算法以计算获得。例如一致性哈希算法（DHT）等。Ceph的数据分布算法CRUSH就属于后者。

# 4.2 CRUSH算法的原理

CRUSH算法的全称为：Controlled、Scalable、Decentralized Placement of Replicated Data，可控的、可扩展的、分布式的副本数据放置算法。

由第1章中介绍过的RADOS对象寻址过程可知，CRUSH算法解决PG如何映射到OSD列表中。其过程可以看成函数：

$$
\operatorname {C R H U S H} (X) \rightarrow (\mathrm {O S D i}, \mathrm {O S D j}, \mathrm {O S D k})
$$

输入参数：

□X为要计算的PG的pg_id。

□ Hierarchical Cluster Map 为 Ceph 集群的拓扑结构。

□ Placement Rules 为选择策略。

输出一组可用的OSD列表。

下面将分别详细介绍 Hierarchical Cluster Map 的定义和组织方式。Placement rules 定义了副本选择的规则。最后介绍 Bucket 随机选择算法的实现。

# 4.2.1 层级化的ClusterMap

层级化的 Cluster Map 定义了 OSD 集群具有层级关系的静态拓扑结构。OSD 的层级使得 CRUSH 算法在选择 OSD 时实现了机架感知（rack awareness）的能力，也就是通过规

则定义，使得副本可以分布在不同的机架、不同的机房中，提供数据的安全性。

层级化的ClusterMap的一些基本概念如下：

□Device：最基本的存储设备，也就是OSD，一个OSD对应一个磁盘存储设备。

□ bucket：设备的容器，可以递归的包含多个设备或者子类型的 bucket。bucket 的类型：bucket 可以有很多的类型，例如 host 就代表了一个节点，可以包含多个 device。Rack 就是机架，包含多个 host 等。在 Ceph 里默认的有 root、datacenter、room、row、rack、host 六个等级。用户也可以自己定义新的类型。每个 device 都设置了自己的权重，和自己的存储空间相关。bucket 的权重就是子 bucket（或者设备）的权重之和。

下列举例说明 bucket 的用法。

# 例4-1 ClusterMap的定义

```txt
host test1{ //类型host，名字为test1id-2 //bucket的id，一般为负值#weight3.000 //权重，默认为子item的权重之和alg straw //bucket随机选择的算法hash0 //bucket随机选择的算法使用的hash函数，这里0代表使用hash函数jenkins1item osd.l weight 1.000 //iteml:osd.l和权重值item osd.2 weight 1.000  
item osd.3 weight 1.000  
}  
host test2{id-3#weight3.000alg straw  
hash0  
item osd.3 weight 1.000  
item osd.4 weight 1.000  
item osd.5 weight 1.000  
}  
root default{ // root类型的bucket，名字为defaultid-1 // id号#weight6.000alg straw //随机选择的算法hash0//rjenkins1item test1 weight 3.000item test2 weight 3.000
```

根据上面 ClusterMap 的语法定义，图 4-1 给出了比较直观的层级化的树型结构。

![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/40fa4abf-bf1f-487c-b4d7-1ee1c732e7cb/5db4d10ff17274a127b041fd2fae95fd7740708fa79b986d1fac77e89ec7b524.jpg)



图4-1 ClusterMap示例图


在上面的 ClusterMap 定义中：

□有一个root类型的bucket，名字为default。

□ root 下面有两个 host 类型的 bucket，名字分别为 test1 和 test2，其下分别有三个 osd 设备，每个 device 的权重都为 1.000，说明它们的容量大小都相同。host 的权重为子设备之和为 3.000，它是自动计算的，不需要设置。

- Hash 设置了使用的 hash 函数，值 0 代表使用 rjenkins1 函数。

□alg代表在该bucket里选择子item的算法。

# 4.2.2 Placement Rules

ClusterMap反映了存储系统层级的物理拓扑结构。PlacementRules决定了一个PG的对象副本如何选择的规则，通过这些可以自己设定规则，用户可以设定副本在集群中的分布。其定义格式如下：

tack(a)   
choose choose firstn {num} type {bucket-type} chooseleaf firstn {num} type {bucket-type}. If {num} $\equiv$ 0,choose pool-num-replicas buckets (all available). If {num} $>0\& \& <$ pool-num-replicas,choose that many buckets. If {num} $<  0$ ,it means pool-num-replicas - {num}. Emit 

Placement Rules 的执行流程如下：

1）take操作选择一个bucket，一般是root类型的bucket。

2）choose操作有不同的选择方式，其输入都是上一步的输出：

a）choose firstn 深度优先选择出 num 个类型为 bucket-type 个的子 bucket。

b）chooseleaf先选择出num个类型为bucket-type个子bucket，然后递归到页节点，选择一个OSD设备：

- 如果 num 为 0, num 就为 pool 设置的副本数。

- 如果 num 大于 0，小于 pool 的副本数，那么就选择出 num 个。

- 如果 num 小于 0，就选择出 pool 的副本数减去 num 的绝对值。

3）emit输出结果。

操作chooseleaffirstn{num}type{bucket-type}可以等同于两个操作：

a) choose firstn {num} type {bucket-type} 

b) choose firstn 1 type osd 

例4-2 PlacementRules：三个副本分布在三个Cabinet中。

如图4-2所示的ClusterMap：顶层是一个rootbucket，每个root下有四个row类型bucket。每个row下面有4个cabinet，每个cabinet下有若干个OSD设备（图中有4个host，每个host有若干个OSD设备，但是在本crushmap中并没有设置host这一级别的bucket，而是直接把4个host上的所有OSD设备定义为一个cabinet）：

```txt
rule replicated_ruleset {
    ruleset 0 // ruleset 的编号 id
    type replicated // 类型 repliated 或者 erasure code
    min_size 1 // 副本数最小值
    max_size 10 // 副本数最大值
    step take_root // 选择一个 root bucket，做下一步的输入
    step choose_first 1 type row // 选择一个 row，同一排
    step choose_first 3 type cabinet // 选择三个 cabinet，三副本分别在不同的 cabinet
    step choose_first 1 type osd // 在上一步输出的三个 cabinet 中，分别选择一个 osd
    step emit
```

![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/40fa4abf-bf1f-487c-b4d7-1ee1c732e7cb/59d1cd87295220827146ba93b35b9cdede4b3274a882c8702d85b7048ff6eaad.jpg)



图4-2 例4-2 ClusterMap


根据上面的定义和图4-2的ClusterMap所示，选择算法的执行过程如下：

1）选中 root bucket 作为下一个步骤的输入。

2）从 root 类型的 bucket 中选择一个 row 类的子 bucket，其选择的算法在 root 的定义中设置，一般设置为 straw 算法。

3）从上一步的输出row中，选择三个cabinet，其选择的算法在row中定义。

4）从上一步输出的三个cabinet中，分别选择一个OSD，并输出。

根据本 rule sets，选择出三个 OSD 设备分布在一个 row 上的三个 cabinet 中。

例4-3 PlacementRules：主副本分布在SSD上，其他副本分布在HDD上。

如图4-3所示的ClusterMap：定义了两个root类型的bucket，一个是名为SSD的root类型的bucket，其OSD存储介质都是SSD盘。它包函两个host，每个host上的设备都是SSD磁盘；另一个是名为HDD的root类型的bucket，其OSD的存储介质都是HDD磁盘，它有两个host，每个host上的设备都是HDD磁盘。

```txt
rule sdd-primary {
    ruleset 5
    type replicated
    min_size 5
    max_size 10
    step take ssd 
```

// 选择 SSD 这个 root bucket 为输入

```txt
step chooseleaf firstn 1 type host //选择一个host，并递归选择叶子节点osd  
step emit //输出结果  
step take hdd //选择hdd这个root bucket为输入  
step chooseleaf firstn -1 type host  
//选择总副本数减一个host，并分别递归选择一个叶子节点osd  
step emit //输出结果
```

![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/40fa4abf-bf1f-487c-b4d7-1ee1c732e7cb/b9b6c99efea4a172beffbc14daa6bc7a2f2df20a5b0ab23aaf5ceaf700e92b96.jpg)



图4-3 例4-3 ClusterMap


根据图4-3所示的ClusterMap，代码中的rulesets的执行过程如下：

1）首先take操作选择ssd为root类型的bucket。

2）在ssd的root中先选择一个host，然后以该host为输入，递归至叶子节点，选择一个osd设备。

3）输出选择的设备，也就是ssd设备。

4）选择hdd作为root的输入。

5）选择2个host（副本数减一，默认3副本），并分别递归选择一个OSD设备，最终选择出两个hdd设备。

6）输出最终的结果。

最终输出3个设备，一个是SSD类型的磁盘，另外两个是HDD磁盘。通过上述规则，就可以把PG的主副本分布在SSD类型的OSD上，其他副本分布在HDD类型的磁盘上。

# 4.2.3 Bucket随机选择算法

Bucket随机选择算法解决了如何从Bucket中选择出需要的子item问题。它定义了四种不同的Bucket选择算法，每种Bucket的选择算法基于不同的数据结构，采用不同伪随机选择函数。

在本节涉及的Hash函数，其参数分别为；

hash(x, r, i) 

□ $\mathbf{x}$ 为要计算的PG的id。

□r为选择的副本序号。

□i为bucket的id号。

下面具体介绍Bucket的四种随机选择算法的过程，并介绍当选择算法出现冲突、失效或过载等特殊情况的处理。

# 1. Uniform Bucket

Uniform 类型适用于每个 item，具有相同权重，且 item 很少添加和删除，也就是 item 的数量比较固定。它用了伪随机排列算法。

# 2. List Bucket

List 类型的 Bucket 中，其子 item 在内存中使用数据结构中的链表来保存，其所包含的 item 可以具有任意的权重。具体查找方法如下：

1）从ListBucket的表头item开始查找，它先得到表头item的权重Wh，剩余链表中所有item的权重之和为Ws。

（2）根据本节提到的 $\mathrm{hash}(\mathbf{x},\mathbf{r},\mathbf{i})$ 函数得到一个[0~1]的值 $\mathbf{v}$ ，假如这个值 $\mathbf{v}$ 在[0~Wh]

Ws) 之中，则选择表头 item，并返回表头 item 的 id 值。

3）否者继续遍历剩余的链表，继续递归选择。

通过上述介绍可知，List类型的Bucekt查找复杂度是 $\mathrm{O(n)}$ 

# 3. Tree Bucket

Tree 类型的 Bucket 其 item 的组织成树结构：每个 item 组成决策树的叶子节点。根节点和中间节点是虚节点，其权重等于左右子树的权重之和。由于 item 在叶子节点，所以每次选择只能走到叶子节点才能选择一个 item 出来。其具体查找方法如下：

1）从该Tree bucket的root item（虚节点）开始遍历。

2）它先得到节点的左子树的权重Wl，得到节点的权重Wn，然后根据哈希函数 $\mathrm{hash}(\mathbf{x},\mathbf{r},\mathbf{i})$ 得到一个[0~1]的值v：

a）如果值 $\mathbf{v}$ 在[0~W1/Wn)之间，那么左子树中继续选择item。

b）否者在右子树中继续选择 item。

c）继续遍历子树，直到到达叶子节点，叶子节点 item 为最终选出的一个结果。

由上述过程可知，TreeBucket每次选择一个item都要遍历到子节点。其查找复杂度是 $O(\log n)$ 

# 4. Straw Bucket

Straw 类的 Bucket 为默认的选择算法。该 Bucket 中的 item 选中概率是相同的，其实现如下：

1）函数f(Wi)为和item的权重Wi相关的函数，决定了每个item被选中的概率。

2）给每个item计算出一个长度，其计算公式为：

$$
\text {l e n g t h} = f (\mathrm {W i}) * \text {h a s h} (\mathrm {x}, \mathrm {r}, \mathrm {i})
$$

length 值最大的 item 就是被选中的 item。

# 5. Bucket 选择算法的对比

如表4-1所示。


表 4-1 Bucket 选择算法对比


<table><tr><td>Bucket 选择算法</td><td>选择的速度</td><td>item 添加难易程度</td><td>item 删除难易程度</td></tr><tr><td>uniform</td><td>O(1)</td><td>poor</td><td>poor</td></tr><tr><td>list</td><td>O(n)</td><td>optimal</td><td>poor</td></tr><tr><td>tree</td><td>O(log n)</td><td>Good</td><td>Good</td></tr><tr><td>straw</td><td>O(n)</td><td>Better</td><td>Better</td></tr><tr><td>straw2</td><td>O(n)</td><td>optimal</td><td>optimal</td></tr></table>

算法 straw 比较容易应对 item 的添加和删除，为默认的 Bucket 选择算法。算法 straw2 是对算法 straw 的一些改进，可以减少数据的迁移数量。

# 6. 冲突、失效或者过载

当通过上述Bucket选择算法选出一个OSD后，有可能出现冲突（重复选择），该OSD已经失效了，或者过载（负载过重）的情况，就需要重新选择一次。

根据上述算法分析，选择时都依赖哈希函数：

```txt
hash(x, r, i) 
```

其中， $\mathbf{x}$ 为PG的id，r为选择的副本数，i为Bucket的id号。当选择出现上述情况需要重新选择。上述各种Bucket选择算法都依赖hash函数。当重新选择时，把参数r顺序增加即可通过上述hash函数重新计算一个新的hash值。

# 例4-4 冲突选择过程

过程见表4-2。


表 4-2 冲突选择过程示例


<table><tr><td></td><td>r=0</td><td>r=1</td><td>r=2</td><td>r=3</td></tr><tr><td>pg 1.0</td><td>osd1</td><td>osd2</td><td>osd3</td><td></td></tr><tr><td>pg 1.1</td><td>osd2</td><td>osd2</td><td>osd3</td><td>osd4</td></tr></table>

说明如下：

1）pg 1.0 根据副本 r 分别等于 0、1、2 来计算 hash(x, r, i)，处理 OSD 列表 {osd1, osd2, osd3}。

2）pg1.1根据同样的方法：在r等于0时选择出了osd2，在r等于1时又选择了osd2，产生了冲突。这时就用r分别等于2，3来继续选择剩余的副本。最终pg1.1选择出的OSD列表为{osd2，osd3，osd4}.

# 4.3 代码实现分析

在介绍了CRUSH算法的原理之后，下面就分析CRUSH算法实现的关键数据结构，并对算法具体实现函数进行分析。

# 4.3.1 相关的数据结构

CRUSH算法相关的数据结构有crush_map结构、crush_bucket结构和crush_rule结构，下面将详细介绍。

# 1. crush_map

结构 crush_map 定义了静态的所有 Cluster Map 的 bucket。bucket 为动态申请的二维数组，保存了所有的 bucket 结构。rules 定义了所有的 crush_rule 结构：

```c
struct crush_map{ struct crush_bucket \*\*buckets; struct crush_rule \*\*rules; 1 
```

# 2. crush_bucket

结构 crush_bucket 用于保存 Bucket 相关的信息：

```javascript
struct crush_bucket{s32id; //bucket的id，一般为负值u16type; //类型，如果是0，就是OSD设备u8alg; //bucket的选择算法u8hash; //bucket的hash函数u32weight; //bucket的权重u32size; //bucket下的item的数量s32\*items; //子bucket在crush_bucket结构buckets数组的下标，这里特别要注意的是，其子item的cursh_bucket结构体都统一保存在crush map结
```

构中的 buckets 数组中，这里只保存其在数组中的下标

```c
//以下是随机排序选择算法的一些Cache的参数  
_u32perm_x; //要选择的x  
_u32perm_n; //排列的总的元素  
_u32\*perm; //排列组合的结果  
};
```

# 3. crush_rule

# 结构 crush_rule

```c
struct crush_rule {
    __u32 len; //steps的数组的长度
    struct crush_rule_mask mask; //ruleset相关的配置参数
    struct crush_rule_step steps[0]; //操作步
}; 
struct crush_rule_mask {
    __u8 ruleset; //ruleset的编号
    __u8 type; //类型
    __u8 min_size; //最新size
    __u8 max_size; //最大size
}; 
struct crush_rule_step {
    __u32 op; //step操作步的操作码
    __s32 arg1; //如果是take，参数就是选择的bucket的id号
    __s32 arg2; //如果是select，就是选择的数量
};
```

# 4.3.2 代码实现

代码 builder.c 和 builder.h 文件里主要实现了如何构造 crush_map 数据结构。Crush.c 和 Crush.h 文件里定义了 crush_map 相关的数据结构和 destroy 方法。文件 CrushCompiler.h 和 CrrushCompiler.cc 为 crush map 的词法和语法分析相关处理。类 CrushWarpper 是对 CRUSH 的所有核心实现进行的封装。CRUSH 算法的核心实现在 mapper.c 文件里。

# 1. crush_do_rule

函数 crush_do_rule 里完成了 CRUSH 算法的选择过程：

```c
int crush_do_rule(const struct crush_map *map, //crush map结构 int rulerno, //ruleset的号
```

```txt
int x, //输入，一般是pg的id  
int *result, //输出osd列表  
int result_max, //输出osd列表的数量  
const __u32 *weight, //所有osd的权重，通过它来判断osd是否out  
int weight_max, //所有osd的数量  
int *scratch)
```

函数cursh_do_rule根据step的数量，循环调用相关的函数选择bucket。如果是深度优先，就调用函数crush_choose_first，如果是广度优先，就调用函数crush_choose_indep来选择。

# 2. crush_choose_first

函数调用 crush_bucket_select 选择需要的副本数，并对选择出来的 OSD 做了相关的冲突检查，如果冲突或者失效或者过载，继续选择新的 OSD。

# 3. bucket 算法

函数 crush_bucket_choose 根据不同的类型 bucket，选择不同的算法来实现从 bucket 中选出 item，这里介绍最常用的 straw 算法：

```txt
static int bucket_straw_choose(struct crush_bucket_straw *bucket, int x, int r) 
```

函数bucket_straw_choose用于straw类型的bucket的选择，输入参数x为pgid，r为副本数，其具体实现如下：

1）对每个 item，计算 hash 值：

```javascript
draw = crush_hash32_3(bucket->h memcmp, x, bucket->h.items[i], r); 
```

2）获取低16位，并乘以权重相关的修正值：

draw $\& = 0$ xffff;  
draw $\star =$ bucket->straws[i]; 

3）选取 draw 值最大的 item 为选中的 item。

由上可知，这种算法类似抽签，是一种伪随机选择算法。

# 4.4 对CRUSH算法的评价

通过以上分析，可以了解到CRUSH算法实质是一种可分层确定性伪随机选择算法，它是Ceph分布式文件系统的一个亮点和创新。

优点如下：

□输入元数据（clustermap、placementrule）较少，可以应对大规模集群。

□可以应对集群的扩容和缩容。

□采用以概率为基础的统计上的均衡，在大规模集群中可以实现数据均衡。

目前存在的缺点如下：

□在小规模集群中，会有一定的数据不均衡现象。

□增加新设备时，导致旧设备之间也有数据的迁移。

# 4.5 本章小结

本章介绍了RADOS的数据分布算法CRUSH的原理。Hierarchical Cluster Map实质定义了存储集群的静态拓扑结构。Placement Rules开放了数据副本的选择规则，可以由用户自己定义和编辑。Bucket算法定义了从bucket选择一个item的算法。通过本章可以了解CRUSH算法的具体实现，它实质是一个可分层的伪随机分布选择算法，是Ceph的一个创新，但它并不完美，需要许多改进。

# 第5章

# Chapter 3

# Ceph 客户端

本章介绍 Ceph 的客户端实现。客户端是系统对外提供的功能接口，上层应用通过它来访问 Ceph 存储系统。本章首先介绍 Librados 和 Osdc 两个模块，通过它们可直接访问 RADOS 对象存储系统。其次介绍 Cls 扩展模块使用它们可方便地扩展现有的接口。最后介绍 Librbd 模块。由于 Librados 和 Librbd 的多数实现流程都比较类似，本章在介绍相关的数据结构后，只选取一些典型的操作流程介绍。

# 5.1 Librados

Librados是RADOS对象存储系统访问的接口库，它提供了pool的创建、删除、对象的创建、删除、读写等基本操作接口。架构如图5-1所示。

在最上层是类 RadosClient，它是 Librados 的核心管理类，处理整个 RADOS 系统层面以及 pool 层面的管理。类 IoctxImpl 实现单个 pool 层的对象读写等操作。OSDC 模块实现了请求的封装和通过网络模块发送请求的逻辑，其核心类 Objecter 完成对象的地址计算、消息的发送等工作。

![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/40fa4abf-bf1f-487c-b4d7-1ee1c732e7cb/6c3e722f6f69f2b7b1c66be5dd7e1973d27283a082730f8913bd82f5bb7f588c.jpg)



图5-1 Librados架构图


# 5.1.1 RadosClient

代码如下：

```cpp
class librados::RadosClient:public Dispatcher  
{public: md_config_t \*conf; //配置文件private:enum {DISCONNECTED,CONNECTING,CONNECTED，} state; //和Monitor的网络连接状态MonClient monclient; //Monitor客户端Messenger \*messenger; //网络消息接口uint64_tinstance_id; //rados客户端实例的idObjecter \*objecter; //objecter对象指针Mutex lock;Cond cond;SafeTimer timer; //定时器int refcnt; //引用计算Finisher finisher; //用于执行回调函数的finisher类
```

通过RadosClient的成员函数，可以了解RadosClient的实现功能如下：

1）网络连接。

Connect函数是RadosClient的初始化函数，完成了许多的初始化工作：

a）调用函数monclient.build_initial_monmap，从配置文件里检查是否有初始的Monitor的地址信息。

b）创建网络通信模块 messenger，并设置相关的 Policy 信息。

c) 创建 objecter 对象并初始化。

d）调用monclient.init()函数初始化monclient。

e）Timer定时器初始化，Finisher对象初始化

2）pool的同步和异步创建。

a）函数pool_create同步创建pool。其实现过程为调用Objecter::create_pool函数，构造PoolOp操作，通过Monitor的客户端monc发送请求给Monitor创建一个pool，并同步等待请求的返回。

b）函数pool_create_async异步创建。与同步方式的区别在于注册了回调函数，当创建成功后，执行回调函数通知完成。

3）pool的同步和异步删除。

函数 delete_pool 完成删除，函数 delete_pool_async 异步删除。其过程和 pool 的创建过程相同，向 Monitor 发送删除的请求。

4）查找pool和列举pool。

函数 lookup_pool 用于查找 pool，函数 pool_list 用于列出所有的 pool。pool 的相关的信息都保存在 OsdMap 信息当中。

5）获取pool和系统的信息。

函数 get_pool/stats 用于获取 pool 的统计信息，函数 get.fs.stats 用于获取系统的统计信息。

# 6）命令处理。

函数mon_command处理Monitor相关的命令，它调用函数monclient.start_mon_command把命令发送给Monitor处理。函数osd_command处理OSD相关的命令，它调用函数objecter->osd_command把命令发送给对应OSD处理。函数pg_command处理PG相关的命令，它调用函数objecter->pg_command把命令发送给该PG的主OSD来处理。

# 7）创建IoCtxImpl对象。

函数 create_ioctx 创建一个 pool 相关的上下文信息 IoCtxImpl 对象。

# 5.1.2 IoCtxImpl

类 IoCtxImpl 是 pool 相关的上下文信息，一个 pool 对应一个 IoCtxImpl 对象，可以在该 pool 里创建，删除对象，完成对象数据读写等各种操作，包括同步和异步的实现。其处理过程都比较简单，而且过程类似：

1）把请求封装成ObjectOperation类（该类定义在osdc/Objecter.h中）。

2）然后再添加pool的地址信息，封装成Object::Op对象。

3）调用函数 objecter->op Submit 发送给相应的 OSD，如果是同步操作，就等待操作完成。如果是异步操作，就不用等待，直接返回。当操作完成后，调用相应的回调函数通知。

# 5.2 OSDC

OSDC是客户端比较底层的模块，其核心在于封装操作数据，计算对象的地址，发送请求和处理超时。

# 5.2.1 ObjectOperation

类 ObjectOperation 用于操作相关的参数统一封装在该类里，该类可以一次封装多个对象的操作：

struct ObjectOperation { 

```txt
vector<OSDOp> ops; //多个操作  
int flags; //操作的标志  
int priority; //优先级  
vector<bufferlist*> out_bl; //每个操作对应的输出缓存区队列  
vector<Context*> outhandler; //每个操作对应的回调函数队列  
vector<int*> out_rval; //每个操作对应的操作结果队列
```

类OSDOp封装对象的一个操作。结构体ceph_osd_op封装一个操作的操作码和相关的输入和输出参数：

```c
struct OSDOp {
    ceph_osd_op op; //各种操作码和操作参数
    subject_t soid; //操作对象
    bufferlist indata, outdata; //输入和输出 bufferlist
    int32_t rval; //操作结果
```

# 5.2.2 op_target

结构 op_target 封装了对象所在的 PG，以及 PG 对应的 OSD 列表等地址信息：

```txt
struct op_target_t {
    int flags; //标志
    object_t base_oid; //读取的对象
    object locator_t base_oloc; //对象的pool信息
    object_t target_oid; //最终读取的目标对象
    object locator_t target_oloc; //最终目标对象的pool信息
    在这里由于Cache-tier的存在，导致产生最终读取的目标和pool的不同。
```

# 5.2.3 Op

结构Op封装了完成一个操作的相关上下文信息，包括target地址信息、链接信息等：

```rust
struct Op : public RefCountedObject {
    OSDSession *session; //OSD相关的Session信息
    int incarnation; //引用次数
    op_target_t target; //地址信息
    vector<OSDOp> ops; //对应多个操作的封装
    snapid_t snapid; //快照的id
```

SnapContext snapc; //pool层级的快照信息bufferlist \*outbl; //输出的bufferlistvector<bufferlist $\text{串}$ out_bl; //每个操作对应的bufferlistvector<Context $\text{串}$ outhandler; //每个操作对应的回调函数vector<int $\text{串}$ out_rval; //每个操作对应的输出结果

# 5.2.4 Striper

对象有分片（stripe）时，类 Striper 用于完成对象分片数据的计算。数据结构 ceph_file.layout 用来保存文件或者 image 的分片信息：

```c
struct ceph_file.layout {
    uint32_t fl_stripe_unit; //文件 -> 对象的映射 *
    uint32_t fl_stripe_count; // stripe 的单位，必须是 page_size 的倍数
    uint32_t fl_object_size; //对象的大小
    uint32_t fl_cas_hash; //哈希值，当为 0 没有设置；当为 1 = sha256 的哈希值
    /* pg -> disk layout 的映射 */
    uint32_t fl_object_stripe_unit; //没有用到
    /*对象 -> pg layout 映射 */
    uint32_t fl_pg preferringed; //PG 优先选择的主 OSD
    uint32_t fl_pg_pool; //pool 的 id
} __attribute__(packed));
```

对象 ObjectExtent 用来记录对象内的分片信息：

```cpp
class ObjectExtent{ public: object_t oid; //对象的id uint64_t objectno; //分片序号 uint64_t offset; //对象内的偏移 uint64_t length; //长度 uint64_t truncate_size; //对象truncate的操作的size object locator_t oloc; //对象位置信息，例如在哪个pool中，等等 vector<pair<uint64_t,uint64_t> >buffer_extents; //Extents在buffer中的偏移和长度，有可能多个extents } void Striper::file_to_extents( CephContext*cct, const char *object_format, const ceph_file.layout *layout, //分片信息 uint64_t offset, uint64_t len, //文件的偏移，长度 uint64_t trunc_size, map<object_t,vector<objectExtent>]&object_extents，//分布到每个对象的数据段
```

```txt
uint64_t buffer_offset) //在buffer中的偏移量
```

函数 file_to_extents 完成了 file 到对象 stripe 后的映射。只有了解清楚了每个概念，计算方法都比较简单。下面举例说明。

# 例5-1 file_to_extents示例

![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/40fa4abf-bf1f-487c-b4d7-1ee1c732e7cb/015fea505ff2b6bbb961b4fe8f30b2cdcc64a52a676a219b74b31097ddb8e9c1.jpg)



图5-2 file_to_extents示例


如图5-2所示，要计算的文件的offset为2KB，length为16KB。文件的分片信息，stripe为4KB，stripe_count为3，object_size为8KB。对象obj1对应的ObjectExtent为：

```txt
object_extents[obj1] = { oid = "obj1", objectno = 0, offset = 2k, length = 6k, buffer_extents = { [0,2k], [6k,4k]} } 
```

其中，oid就是映射对象的id，objectno为stirpe对象的序号，offset为映射的数据段在对象内的起始偏移，length为对象内的长度。buffer_extents为映射的数据在buffer内的偏移和长度。

# 5.2.5 ObjectCacher

类 ObjectCacher 提供了客户端的基于 LRU 算法对象数据缓存功能，其实比较简单，这里就不深入分析了。

# 5.3 客户写操作分析

以下代码是通过Librados库的接口写入数据到对象中的典型例程，对象的其他操作过程如下：

# 程都类似：

```c
rados_t cluster;  
rados_ioctl_x_t ioctx;  
rados_create(&cluster, NULL);  
rados_conf_read_file(cluster, NULL);  
rados_connect(cluster);  
rados_ioctl_x_create(cluster, pool_name.c_str(), &ioctx);  
rados_write(ioctx, "foo", buf, sizeof(buf), 0) 
```

上述代码是 C 语言接口完成的，其流程如下：

1）首先调用 rados_create 函数创建一个 RadosClient 对象，输出为类型 rados_t，它是一个 void 类型的指针，通过 librados::RadosClient 对象的强制转换产生。第二个参数 id 为一个标识符，一般传入为 NULL。

2）调用函数rados_conf_read来读取配置文件。第二个参数为配置文件的路径，如果是NULL，就搜索默认的配置文件。

3）调用rados_connect函数，它调用了RadosClient的connect函数，做相关的初始化工作。

4）调用函数 rados_ioctl_x_create，它调用 RadosClient 的 create_ioctl_x 函数，创建 pool 相关的 IoCtxImpl 类，其输出为类型 rados_ioctl_x_t，它也是 void 类型的指针，由 IoCtxImpl 对象转换而来。

5）调用函数 rados_write 函数，向该 pool 的名为“foo”的对象写入数据。其调用 IoCtxImpl 类的 wrie 操作。

# 5.3.1 写操作消息封装

本函数完成具体的写操作：代码如下：

```cpp
librados::IoCtxImpl::write(const object_t& oid, bufferlist& bl, size_t len, uint64_t off) 
```

其实现过程如下：

1）创建ObjectOperation对象，封装写操作的相关参数。

2）调用函数 operate 完成处理。

a）调用函数 objecter->prepare_mutate_op 把 ObjectOperation 类型的封装成 Op 类型，添加了 object locator_t 相关的 pool 信息。

b）调用 objecter $\rightarrow$ op submit 把消息发送出去。

c）等到操作完成。

# 5.3.2 发送数据op_submit

函数 op_submit 用来把封装好的操作 Op 通过网络发送出去。

函数_op Submit_with-budget用来处理Throttle相关的流量限制。如果osd_timeout大于0，就是设置定时器，当操作超时，就调用定时器回调函数op_cancel取消操作。

函数_op_submit完成了关键的地址寻址和发送工作，其处理过程如下：

1）调用函数 calc_target 来计算对象的目标 OSD。

2）调用函数_get_session 获取目标OSD的链接，如果返回值为-EAGAIN，就升级为写锁，重新获取。

3）检查当前的状态标志，如果当前是CEPH_OSDMAP_PAUSEWR或者OSD空间满，就暂时不发送请求，否则调用函数prepare_osd_op准备请求的消息，调用函数send_op发送出去。

# 5.3.3 对象寻址_calc_target

函数 calc_target 用于完成对象到 osd 的寻址过程：

```cpp
int Objeter::_calc_target(op_target_t *t, epoch_t *last_force_resend, bool any_change) 
```

其处理过程如下：

1）首先根据 $t->base\_ocp$ 的 pool 信息，获取 pg_pool_t 对象。

2）检查如果强制重发，force resend 设置为 true。

3）检查cache tier，如果是读操作，并且有读缓存，就设置target_oloc.pool为该pool的read tier值；如果是写操作，并且有写缓存，就设置target_oloc.pool为该pool的write_tier值。

4）调用函数osdmap->object locator_to_pg获取目标对象所在的PG。

5）调用函数osdmap->pg_to_up Acting_osds，通过CRUSH算分，获取该PG对应的OSD列表。

6）如果是写操作，target的OSD就设置为主OSD；如果是读操作，如果设置了CEPH_OSD_FLAG_BALANCE_READS标志，就随机选择一个副本读取。如果设置了CEPH_OSD_FLAG_LOCALIZE_READS标志，就尽可能选择本地副本读取。

# 5.4 Cls

Cls是Ceph的一个模块扩展，它允许用户自定义对象的操作接口和实现方法，为用户提供了一种比较方便的接口扩展方式。目前rbd和lock等模块都使用了这种机制。

# 5.4.1 模块以及方法的注册

类 ClassHandler 用来管理所有的扩展模块。函数 register_class 用来注册模块：

class ClassHandler{ CephContext \*cct; Mutex mutex; map<string,ClassData> classes; //所有注册的模块：模块名 $\rightarrow$ 模块元数据信息 .......

类 ClassData 描述了一个模块的相关的元数据信息。它描述一个扩展模块的相关信息，包括模块名、模块相关的操作方法以及依赖的模块：

```txt
struct ClassData {
    enum Status {
        CLASS_UNKNOWN, //初始未知状态
        CLASS MISSING, //缺失状态（动态链接库找不着）
        CLASS MISSING_DEPS, //依赖的模块缺失
        CLASS_INITIALIZING, //正在初始化
        CLASS_OPEN, //已经初始化（动态链接库以及加载成功）
    } status; //当前模块的加载状态
    string name; //模块的名字
    ClassHandler *handler; //管理模块的指针
    void *handle;
```

map<string,ClassMethod> methods_map; //模块下所有注册的方法 map<string,ClassFilter> filters_map; //模块下所有注册过滤方法 set<classData $\text{串}$ >dependencies; //本模块依赖的模块 set<classData $\text{串}$ > missingDependencies; //缺失的依赖模块 }

ClassMethod定义一个模块具体的方法名，以及函数类型：

```cpp
struct ClassMethod {
    struct ClassHandler::ClassData *cls; // 所属模块的 ClassData 的指针
    string name; // 方法名
    int flags; // 方法相关的标志
    cls_method_call_t func; // C 类型函数指针
    cls_method_cxx_call_t cxx_func; // C++ 类型函数指针
```

在src/objectclass/class_api.c里定义了一些辅助函数用来注册模块以及方法：

注册一个模块如下：

```txt
intcls register(const char \*name，cls_handle_t \*handle); 
```

注册一个模块的方法如下：

```cpp
intcls_register_method(cls_handle_t hclass，const char \*method, int flags，cls_method_call_t class_call，cls_method_handle_t \*handle) 
```

# 5.4.2 模块的方法执行

模块方法的执行在类ReplicatedPG的函数do_osd ops里实现。执行方法对应的操作码为CEPH_OSD_OP_CALL值：

```cpp
int ReplicatedPG::do_osdOps(OpContext *ctx, vector<OSDOp>& ops) {  
    .......  
    case CEPH OSD_OP_CALL:  
        // 加载相关的模块  
        ClassHandler::ClassData *cls;  
        result = osd->classhandler->open_class(cname, &cls);  
        assert(result == 0); // 函数 init_op_flags() 已经对结果做了验证  
        // 根据方法名获取方法  
        ClassHandler::ClassMethod *method = cls->get_method(mname.c_str());  
        // 执行方法  
        result = method->exec((cls_method_context_t) & ctx, indata, outdata);
```

```txt
1 
```

# 5.4.3 举例说明

以 CIs 下的 rbd 扩展接口来说明模块的定义、注册和调用过程：

1）rbd的定义和注册。在cls_rbd.cc的函数里，注册了rbd模块，以及自定义的方法：

```txt
cls_register("rbd", &h_class);  
cls_register_cxx_method(h_class, "create", CLS_METHOD_RD | CLS_METHOD_WR, create, &h_create); 
```

2）cls_rbd_client.h和cls_rbd_client.cc里定义了客户端访问rbd函数，其调用ioctx $\rightarrow$ exec函数，封装成CEPH_OSD_OP_CALL类型的ObjectOperation操作，发送给相应的osd处理。

3）cls_rbd.cc里定义了相应的函数在服务端的实现，其把输入参数从bufferlist中解析处理，调用相应的实现，把输出结果封装到输出bufferlist中。

```txt
int create(cls_method_context_t hctx, bufferlist *in, bufferlist *out)  
{ string object_prefix; uint64_t features, size; uint8_t order; try { bufferlist::iterator iter = in->begin(); ::decode(size, iter); ::decode(order, iter); ::decode/features, iter); ::decode(object_prefix, iter); } catch (const buffer::error &err) { return -EINVALID; ....} 
```

通过以上介绍，可以了解Cls的扩展模块：如何定义、注册一个新的模块，以及调用机制等。

# 5.5 Librbd

Librbd模块实现了RBD（rados block device）接口，其基于Librados实现了对RBD的基本操作。Librbd的架构如图5-3所示。

![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/40fa4abf-bf1f-487c-b4d7-1ee1c732e7cb/5ae62307a3e67132b381ae5130d7a483d10b652366d9e609dad69692f65c63c4.jpg)



图5-3 Librbd的架构图


在最上层是Librbd层，模块cls_rbd是一个Cls扩展模块，实现了RBD的元数据相关的操作。RBD的数据访问直接通过该Librados来访问。在最底层是OSDC层完成数据的发送。

# 5.5.1 RBD 的相关的对象

RBD的相关对象如下所示。

□rbd_directory对象：该对象在每个pool里都存在，用来保存该pool下所有的RBD的目录信息。当创建RBD块设备时，会检查如果rbd_directory对象不存在，就会创建该对象。该对象的omap属性里保存所有的RBD设备的名字和id信息。如表5-1所示。


表 5-1 rbd Directory 的 omap 的 kv 属性


<table><tr><td>Key 值</td><td>Value 值</td></tr><tr><td>&quot;name_ &quot; + rbd_name1</td><td>&quot;id_ &quot; + rbd_id1</td></tr><tr><td>&quot;name_ &quot; + rbd_name2</td><td>&quot;id_ &quot; + rbd_id2</td></tr><tr><td>......</td><td>......</td></tr></table>

omap 的属性里，保存所有 RBD 的设备名和 id 信息，key 值为“name”和设备名的拼接，value 为“id_”和 RBD 的 id 的拼接。

□对于每一个RBD设备，其创建如下对应的对象（最新的版本v2）：

- rbd_id.<name>：称为RBD的id_obj对象，其名字保存了RBD的name信息，其对象的内容保存了该RBD的id信息。

- rbd_header.<id>：称为RBD的head_obj对象，其omap保存了RBD相关的元数据信息。

- rbd_object_map.<id>：保存了其对象和父 image 对象映射信息。

- 数据对象（多个）。

```ruby
rbd_data.<id>.00000000  
rbd_data.<id>.0000001 
```

# 5.5.2 RBD元数据操作

结构 ImageCtx 用来处理一个 Image 的上下文相关信息。在 Internal.cc 和 Internal.h 定义了 RBD 的元数据相关的操作函数，其调用 Cls 下的 RBD 模块来完成 RBD 设备的创建、删除、快照等元数据操作。

下面通过分析RBD的创建过程来分析RBD的元数据操作过程，创建代码如下：

```txt
extern "C" int rbd_create4(rados_ioctl_t p, const char *name, uint64_t size, rbd_image_options_t opts) 
```

RBD的创建根据传入参数的不同，有4个函数入口，下面主要研究rbd_create4函数，其实现过程如下：

1）调用函数librd::create设置RBD相应的参数：

a）设置RBD的format格式。

b）设置RBD的feature信息。

c）设置RBD的分片信息stripe_unit和stripe_count参数。

d）设置RBD的order值，其决定了RBD对象size大小。

e）设置bid为rados的instance_id，根据它来生成RBD的id值。

2）如果是版本v2，就调用函数create_v2来继续创建：

a）获取id_obj的名字：rbd_id.<name>，调用函数ioctx.create创建该对象。

b）和bid随机产生一个id,做为新的image的id值。

c）调用函数cls_client::set_id设置id，就是在对象id obj的内容中写入id。

d）调用函数cls_client::dir_add_image把新创建的RBD加入rbd_directory目录中，也就是在对象rbd_directory的omap属性中加入该RBD的名字和id的键值对。

e）获取对象head_obj的名字，也就是rbd_header.<id>的格式。

f）调用函数cls_client::create_image创建image，其在head_obj对象中的omap属性中设置size、order、feature等RBD相关的元数据信息。

g）如果对象有stripe，就调用函数cls_client::set stripe_unit_count设置stripe相关的参数。

h）如果有 ObjectMap 属性，就设置 ObjectMap，如果 RBD 有 mirror，就完成 rbd journal 相关的设置。

通过RBD的创建过程，可以了解其元数据相关的操作，就是通过Cls的RBD模块设置相关的元数据信息。其他的元数据操作过程都类似。

# 5.5.3 RBD数据操作

每个 ImageCtx 中都有一个对象 AioImageRequestWQ，它是一个工作队列，基于 ThreadPool::PointerWQ 来实现了异步请求的发送工作。

如图5-4所示：类AioObjectRequest是单个对象的异步请求。AioObjectRead和AbstractAioObjectWrite分别为对象的异步读和异步写操作。对象的异步操作truncate、write、trim、zero等操作都继承AbstractAioObjectWrite类。

如图5-5所示：类AioImageRequest是所有Image操作的基类，AioImageRead实现了image的异步读操作的逻辑。AbstractAioImageWrite为异步写操作的抽象类。AioImageWrite，AioImageDiscard，AioImageFlush继承了AbstractAioImageWrite类，分别完成了异步写，异步丢弃某一段数据（Discard），异步数据Flush等操作。

![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/40fa4abf-bf1f-487c-b4d7-1ee1c732e7cb/0d3bc3073b4960426905bbb24ccc1e6ce951372206ba78acc50eed8aa3f9707f.jpg)



图5-4 对象请求类图


![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/40fa4abf-bf1f-487c-b4d7-1ee1c732e7cb/3587a89013295454afe184a9a546216a4fd51de23253d0794f053fd19eef21bd.jpg)



图5-5 Image请求类图


当一个AioImageRequest跨越多个对象时，每个对象就会产生一个AioObjectRequest请求，每个AioObjectRequest请求分别处理。

下面以rbd_aio_write为例，来介绍RBD的数据读写的流程：

```c
extern "C" int rbd_aio_write(rbd_image_t image, uint64_t off, size_t len, const char *buf, rbd Completion_t c) 
```

其参数分别为：

□ image：对象。

□ off: image 内的偏移。

□len：写操作的长度。

□buf：写操作的数据buf。

□ 异步操作的回调函数。

# 处理流程如下：

1）调用ictx->aio_work_queue->aio_write函数来处理。

a）如果是非阻塞IO，或者有rbd mirror，或者有阻塞的IO请求，就调用函数AioImageWrite对象，加入到AioImageRequestWQ的工作队列里。

b）否则调用AioImageRequest::aio_write，构建AioImageWrite对象，并调用req.send()发送。

2）加入AioImageRequestWQ队列的AioImageWrite请求，在线程池的处理函数AioImageRequestWQ::process里，同样调用req.send()函数来处理。

3）AioImageWrite类的send函数继承自AioImageRequest的send函数，其直接调用send_request函数，AioImageWrite的send_request函数继承来自AbstractAioImageWrite的send_request函数。

4）在AbstractAioImageWrite的send_request函数里，调用了Striper::file_to_extents函数，它计算该image的要写入的数据段映射到对象上的数据段。

5）在函数 send_object_request 里，针对每一个对象，构建 AioObjectWrite 请求。如果有 rbd journal，就把请求加入到 aio_object_request 里；否则就调用 AioObjectWrite 的 send 函数处理。

6）在AbstractAioObjectWrite::send函数里，调用AbstractAioObjectWrite::send_pre函数。该函数处理object_map相关的操作后；如果是写操，就调用AbstractAioObjectWrite::send_write函数。

7）在函数 AbstractAioObjectWrite:::send_write 里，区分了两种操作：如果已经确认该对象不存在，并且有父 image，就 handle_write_guard 函数处理 clone 相关的 copyup 操作。否则就直接调用函数 AbstractAioObjectWrite:::send_write_op 处理。

8）在函数 AbstractAioObjectWrite::send_write_op 里，调用 m_ictx->datactx.ao operate 函数，通过该 rados 层的操作函数，完成对象数据的写入。

# 5.5.4 RBD的快照和克隆

RBD的快照基于rados的snapshot的机制实现。RBD的快照在客户端的实现比较简单，其核心流程就是申请新的snap_seq，把snap_name和snap_id记录在RBD的元数据

中。写操作的copy-on-write机制是依赖rados在OSD服务端实现的。

RBD 的 clone 操作在客户端实现了 RBD 的 copy-on-write 机制。当对 RBD 的 image 发起读操作或者写操作时，如果对象不存在，就需要检查：如果有有父 image，则读取父 image 所对应的对象数据。

# 1. RBD 的 snapshot 的创建

RBD 创建 snapshot 的核心逻辑在 SnapshotCreateRequest 里处理。其流程如下：

1）调用函数SnapshotCreateRequest<I>::send_suspend_aio来阻塞RBD的写操作。

2）调用函数 SnapshotCreateRequest<I>::send_allocate_snapshot_id 向 Monitor 申请一个新的 snap_seq 序号。

3）在函数 SnapshotCreateRequest<I>::send_create_snapshot() 里，调用cls_client::snapshot_add 把 snap_name 和 snap_id 添加到 RBD 的 head_obj 的元数据中。

4）调用函数 ObjectMap::snapshot_add，构建 object_map::SnapshotCreateRequest，做 ObjectMap 相关的处理。

# 2.RBD的CopyUp操作

当RBD在读写一个对象时，如果该对象不存在，并且有父image，就需要CopyUp操作，读取父image所对应的对象数据。

其对应的实现在函数void AbstractAioObjectWrite::send_write()里，如果对象不存在，并且有parent，就调用函数handle_write_guard处理：

if(!m_object_exists && has_parent()) { m_state $=$ LIBRBD_AIO_WRITE_guard; handle_write_guard(); }else{ send_write_op(true);   
1 

在函数void AbstractAioObjectWrite::handle_write_guard里，如果有parent，就调用函数send_copyup函数：

```cpp
if (has_parent) {  
    send_copyup();  
} else {  
    // parent may have disappeared -- send original write again  
    ldout(m_ictx->cct, 20) << "shouldcomplete(" << this << ")": parent overlap now 0" << endl;  
    send_write();  
} 
```

在函数 AbstractAioObjectWrite::send_copyup 里，构建 CopyupRequest 请求，来处理读取 parent image 对应的对象。

# 3.ObjectMap

在RBD里，新添加了一个特性，就是ObjectMap，它添加了RBD的一个属性，用Bit vector的形式来记录一个对象是否存在，这样就可以极大地提供clone卷的读写性能。

# 5.6 本章小结

本章大致介绍了客户端的各个模块的功能以及核心典型操作的处理流程。由于Librados和Librbd的代码比较庞大，承载了所有功能的接口，故一些功能本章没有介绍到或者介绍得比较简略，但是客户端的代码有很大的类似性，通过了解典型流程，便不难理解其他代码。

# 第6章

# Ceph 的数据读写

本章介绍Ceph的服务端OSD（书中简称OSD模块或者OSD）的实现。其对应的源代码在src/osd目录下。OSD模块是Ceph服务进程的核心实现，它实现了服务端的核心功能。本章先介绍OSD模块静态类图相关数据结构，再着重介绍服务端数据的写入和读取流程。

# 6.1 OSD模块静态类图

OSD模块的静态类图如图6-1所示

OSD模块的核心类及其之间的静态类图说明如下：

□类OSD和类OSDService是核心类，处理一个osd节点层面的工作。在早期的版本中，OSD和OSDService是一个类。由于OSD的类承载了太多的功能，后面的版本中引入OSDService类，分担一部原OSD类的功能。

□类PG处理PG相关的状态维护以及实现PG层面的基本功能。其核心功能是用boost库的statechart状态机来实现的PG状态转换。

□类ReplicatedPG继承了类PG，在其基础上实现了PG内的数据读写以及数据恢复

相关的操作。

□类PGBackend的主要功能是把数据以事务的形式同步到一个PG其他从OSD节点上。

- PGBackend 的内部类 PGTransaction 就是同步的事务接口，其两个类型的实现分别对应 RPGTransaction 和 ECTransaction 两个子类。

- PGBackend 两个子类 ReplicatedBackend 和 ECBackend 分别对应 PG 的两种类型的实现。

□类SnapMapper额外保存对象和对象的快照信息，在对象的属性里保存了相关的快照信息。这里保存的快照信息为冗余信息，用于数据效验。

![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/40fa4abf-bf1f-487c-b4d7-1ee1c732e7cb/06fa52f9779f527a7710c8df30e2826da993f993c3fdc6e644bb8d1ee2323b7d.jpg)



图6-1 OSD模块的静态类图


# 6.2 相关数据结构

下面将介绍OSD模块相关的一些核心的数据结构。从最高的逻辑层次为pool的概念，然后是PG的概念。其次是OSDMap记录了集群的所有的配置信息。数据结构OSDOp是一个操作上下文的封装。结构object_info_t保存了一个对象的元数据信息和访问信息。对象ObjectState是在object_info_t基础上添加了一些内存的状态信息。SnapSetContext和ObjectContext分别保存了快照和对象的上下文相关的信息。Session保

存了一个端到端的链接相关的上下文信息。

# 6.2.1 Pool

Pool是整个集群层面定义的一个逻辑的存储池。对一个Pool可以设置相应的数据冗余类型，目前有副本和纠删码两种实现。数据结构pg_pool_t用于保存Pool的相关信息。Pool的数据结构如下：

```c
struct pg_pool_t {
    enum {
        TYPE_REPLICATED = 1, //副本
        //TYPE RAID4 = 2, //从来没有实现的 raid4
        TYPE_ERASURE = 3, //纠删码
    };
uint64_t flags; //pool的相关的标志
_u8_type; //类型
_u8_size, min_size; //pool的size和min_size，也就是副本数和至少保证的副本数
_u8_crush_rule set; //rule set的编号
_u8_object_hash; //对象映射的hash函数
_u32_pg_num, pgp_num; //PG的数量，pgp的值（用于rule set的设置）
string erasure_code_profile; //EC的配置信息
```

□ type: 定义了 Pool 的类型，目前有 replication 和 ErasureCode 两种类型。

size和min_size定义了Pool的冗余模式：

- 如果是replication模式,size定义了副本数目,min_size为副本的最小数目。例如：如果size设置为3，副本数为3，min_size设置为1就只允许两副本损坏。

- 如果是 Erasure Code（M+N），size 是总的分片数 M+N；min_size 是实际数据的分片数 M。

□ crush_rule: Pool 对应的 crush 规则号。

□ erasure_code_profile: EC 的配置方式。

- object_hash：通过对象名映射到 PG 的 hash 函数。

□ pg_num: Pool 里 PG 的数量。

通过上面的介绍可以了解到，Pool根据类型不同，定义了两种模式，分别保存了两种模式相关的参数。此外，在结构pg_pool_t里还定义了Pool级别的快照相关的数据结构、

CacheTier相关的数据结构，以及其他一些统计信息。在介绍快照（参见第9章）和CacheTier（参见第13章）时再详细介绍相关的字段。

# 6.2.2 PG

PG可以认为是一组对象的集合，该集合里的对象有共同特征：副本都分布在相同的OSD列表中。PG的数据结构如下：

```c
struct pg_t {
    uint64_t m_pool; //pg所在的pool
    uint32_t m_seed; //pg的序号
    int32_t m Preferences; //pg优先选择的主osd
};
```

结构体 pg_t 只是一个 PG 的静态描述信息。类 PG 及其子类 ReplicatedPG 都是和 PG 相关的处理。

```c
struct spg_t {
    pg_t pgid;
    shard_id_t shard;
}; 
```

数据结构spg_t在pg_t的基础上，加了一个shard_id字段，代表了该PG所在的OSD在对应的OSD列表中的序号。

在 Erasure Code 模式下，该字段保存了每个分片的序号。该序号在 EC 的数据 encode 和 decode 过程中很关键；对于副本模式，该字段没有意义，都设置为 shard_id_t::NO_SHARD 值。

PG的分裂 当一个pool里的PG数量不够时，系统允许通过命令增加PG的数量，就会产生PG的分裂，使得一个PG分裂为2的幂次方个PG。PG的分裂后，新的PG和其父PG的OSD列表是一致的，其数据的移动也是本地数据的启动，开销比较小。

# 6.2.3 OSDMap

类OSDMap定义了Ceph整个集群的全局信息。它由Monitor实现管理，并以全量或者增量的方式向整个集群扩散。每一个epoch对应的OSDMap都需要持久化保存在meta下对应对象的omap属性中。

下面介绍OSDMap核心成员，内部类Incremental以增量的形式保存了OSDMap新增的信息，其内部成员和OSDMap类似，这里就不介绍了。

```txt
class OSDMap{ 
```

//系统相关的信息

```c
uuid_d fsid; //当前集群的fsid值  
epoch_t epoch; //当前集群的epoch值  
utime_t created, modified; //创建修改的时间戳  
int32_t pool_max; //最大的pool数量  
uint32_t flags; //一些标志信息
```

//OSD 相关的信息

```txt
int num_osd; //OSD的总数量  
int num_up_osd; //处于up状态的OSD的数量  
int num_in_osd; //处于in状态的OSD的数量  
int32_t max_osd; //OSD的最大数目  
vector<uint8_t> osd_state; //OSD的状态  
ceph::shared_ptr<addresses_s> osd_addresses; //OSD的地址  
vector<_u32> osd_weight; //OSD的权重  
vector<osd_info_t> osd_info; //OSD的基本信息  
ceph::shared_ptr< vector<uuid_d> > osduuid; //OSD对应的uuid  
vector<osd_xinfo_t> osd_xinfo; //OSD的一些扩展信息
```

//PG相关的信息

ceph::shared_ptr< map<pg_t,vector<int32_t> $\rightharpoondown$ pg_temp; // temp pg mapping (e.g. while we rebuild)   
ceph::shared_ptr< map<pg_t,int32_t $>$ primary_temp; // temp primary mapping (e.g. while we rebuild)   
ceph::shared_ptr< vector<_u32> osd_primary-affinity;   
//< 16.16 fixed point,0x10000 = baseline   
//pool 相关的信息   
map<int64_t,pg_pool_t> pools; //pool 的 id 到类 pg_pool_t 的映射   
map<int64_t,strong> pool_name; //pool 的 id 到 pool 的名字的映射   
map<string,map<string,strong> erasure_codeProfiles; //pool 的 EC 相关的信息   
map<string,int64_t> name_pool; //pool的名字到pool的id的映射   
//Crush 相关的信息   
ceph::shared_ptr<CrushWrapper> crush; //CRUSH 算法

通过OSDMap数据成员的了解，可以看到，OSDMap包含了四类信息：首先是集群的信息，其次是pool相关的信息，然后是临时PG相关的信息，最后就是所有OSD的状态信息。

# 6.2.4 OSDOp

类MOSDOp封装了一些基本操作相关的数据。

```cpp
class MOSDOp : public Message {
    object_t oid; // 操作的对象
    object locator_t oloc; // 对象的位置信息
    pg_t pgid; // 对象所在的 PG 的 id
    vector<OSDOp> ops; // 针对 oid 的多个操作集合 private:
        // 快照相关
        snapid_t snapid; // snapid, 如果是 CEPH_NOSNAP, 就是 head 对象; 否则就是等于 snap_seq;
        snapid_t snap_seq;
        // 如果是 head 对象, 就是最新的快照序号;
        // 如果是 snap 对象, 就是 snap 对应的 seq
        vector<snapid_t> snaps; // 所有的 snap 列表
        uint64_t features; // 一些 feature 的标志
        osd_reqid_t reqid; // 请求的唯一 id 标识
```

MOSDOp在其成员ops向量里分装了多个类型为OSDOp操作数据。MOSDOp封装的操作都是关于对象oid相关的操作，一个MOSDOp只封装针对同一个对象oid的操作。但是对于rados Clone_range这样的操作，需要有一个目标对象oid，还有一个源对象oid，那么源对象的oid就保存在结构OSDOp里。

数据结构OSDOp封装了一个OSD操作需要的数据和元数据：

```solidity
struct OSDOp {
    ceph_osd_op op; //具体操作数据的封装
    subject_t soid;
    //src oid，并不是op操作的对象，而是源操作对象
    //例如radosClone_range需要目标obj和源obj
    bufferlist indata, outdata; //操作的输入输出的data
    int32_t rval; //操作返回值
    OSDOp(): rval(0) {
        memset(&op, 0, sizeof(ceph_osd_op));
    }
}
```

# 6.2.5 Object_info_t

结构 object_info_t 保存了一个对象的元数据信息和访问信息。其做为对象的一个属性，持久化保存在对象 xattr 中，对应的 key 为 OI_ATTR (" _ ")，value 就是 oject_info_t 的 encode 后的数据。

```c
struct object_info_t {
    hobject_t sold; //对应的对象
    eversion_t version, prior_version; //对象的当前版本，前一个版本
    version_t user_version; //用户操作的版本
    osd_reqid_t last_reqid; //最后请求的请求id
    uint64_t size; //对象的大小
    utime_t mtime; //修改时间
    utime_t local_mtime; //修改的本地时间
typedef enum {
    FLAG_LOST = 1 << 0,
    FLAG_WHITEOUT = 1 << 1, //object logically does not exist
    FLAG_DIRTY = 1 << 2,
    //object has been modified since last flushed or undirtied
    FLAG OMAP = 1 << 3, //has (or may have) some/any OMAP data
    FLAG_DATA_DIGEST = 1 << 4, //has data crc
    FLAG OMAP_DIGEST = 1 << 5, //has OMAP crc
    FLAG_CACHE_PIN = 1 << 6, //pin the object in cache tier
    // ...
    FLAG Uses TMAP = 1 << 8, //deprecated; no longer used.
} flag_t;
flag_t flags; //对象的一些标记
......
vector<snapid_t> snaps; //clone对象的快照信息
uint64_t truncate_seq, truncate_size;
//truncate操作的序号和size
map<pair<uint64_t, entity_name_t>, watch_info_t> watchers;
//watchers记录了客户端监控信息，一旦对象的状态发送变化，需要通知客户端
_u32 data_digest; //< data crc32c>
_u32omap_digest; //<omapcrc32c
//数据或者omap信息的crc32校验信息，可能有，也可能没有
```

# 6.2.6 ObjectState

对象 ObjectState 是在 object_info_t 基础上添加了一个字段 exists，用来标记对象是否存在。

```c
struct ObjectState {
    object_info_t oi;
    bool exists;
    //the stored object exists (i.e., we will remember the object_info_t) 
```

```txt
ObjectState(): exists(false) {} ObjectState(const object_info_t &oi_, bool exists_) : oi(oi_, exists(exists_ {}) }; 
```

为什么要加一个额外的 bool 变量来标记呢？因为 object_info_t 可能是从缓存的 attrs[OI_ATTR] 中获取的，并不能确定对象是否存在。

# 6.2.7 SnapSetContext

SnapSetContext 保存了快照的相关信息，即 SnapSet 的上下文信息。关于 SnapSet 的内容，可以参考快照相关的介绍：

```txt
struct CapsSetContext {
    hobject_t oid; //对象
    int ref; //本结构的引用计数
    bool registered; //是否在 CapsSet Cache 中记录
    CapsSet snapshot; //SnapSet 对象快照相关的记录
    bool exists; //snapset 是否存在
    CapsSetContext(const hobject_t& o):
        oid(o), ref(o), registered(false), exists(true) {}};
```

# 6.2.8 ObjectContext

ObjectContext可以说是对象在内存中的一个管理类，保存了一个对象的上下文信息。

```txt
struct ObjectContext {
    ObjectState obs; //主要是object_info_t，描述了对象的状态信息
    SnapSetContext *ssc; //快照上下文信息，如果没有快照就为空
    Context *destructor_callback; //析构函数的
    private:
        Mutex lock;
    public:
        Cond cond;
    int unstableWrites, readers, writers Waiting, readers Waiting;
    //正在写操作的数目，正在读操作的数目
    //等待写操作的数目，等待读操作的数目
```

//如果该对象的写操作被阻塞去恢复另一个对象，设置这个属性 ObjectContextRef blocked_by; //本对象被某个对象阻塞 set<objectContextRef>blocking; //本对象阻塞的对象集合

//任何在obs.oi.watchers中的watchers在watchers队列中或者在unconnected-watchers中map<pair<uint64_t,entity_name_t>,WatchRef>watchers;

```txt
//属性的缓存  
map<string, bufferlist> attr_cache;  
list<OpRequestRef> waiters; //等待状态变化的 waiters  
int count; //读或写的数目  
struct RwState{enum State{RWNONE,RWREAD,RWRITE,RWEXCL，}；State state:4;//读写的状态//如果设置，获得锁后，重新执行 backfill 操作bool recovery_readmarker:1;//如果设置，获得锁后重新加入 snaptrim 队列中bool snaptrimmer_writemarker:1;  
}
```

下面两个字段比较难理解，进行一些补充说明：

□ blocked_by 记录了当前对象被其他对象阻塞，blocking 记录了本对象阻塞其他对象的情况。当一个对象的写操作依赖其他对象时，就会出现这些情况。这一般对应一个操作涉及多个对象，比如 copy 操作。把对象 obj1 上的部分数据拷贝到对象 obj2，如果源对象 obj1 处于 missing 状态，需要恢复，那么 obj2 对象就 block 了 obj1 对象。

□ 内部类RWState通过定义了4种状态，实现了对对象的读写加锁。

# 6.2.9 Session

类 Session 是和 Connection 相关的一个类，用于保存 Connection 相关的上下文相关的信息。

```txt
struct Session : public RefCountedObject {
    EntityName entity_name; //peer实例的名字
    OSDCap caps;
```

int64_t auid;   
ConnectionRef con; //相关的Connection   
WatchConState wstate;   
Mutex session_dispatch_lock;   
list<OpRequestRef>waiting_on_map;   
所有的OpRequest请求都先添加在这个队列里   
OSDMapRef osdmap; //Map as of which waiting_for_pg is current map<spg_t，list<OpRequestRef $>$ waiting_for_pg; //当前需要更新Osdmap的pg和对应的请求   
Spinlock sent_epoch_lock; //通过消息向外通知的 epoch epoch_t last_sent_epoch; //发送对端的epoch   
Spinlock received_map_lock;   
epoch_t received_map_epoch; //最新的MOSDMap消息接收到的received_map_epoch   
Session(CephContext \*cct）： RefCountedObject(cct), auid(-1)，con(0), session_dispatch_lock("Session::sessiondispatch_lock"), last_sent_epoch(0)，received_map_epoch(0)   
}；

函数update_waiting_for_pg用于检查是否有最新的osdmap：

1）如果该PG有分裂的PG，就把分裂出的新的PG以及对应的OpRequest加入到session的waiting_for_pg队列里。

2）如果该PG不分裂，就并把PG和opRequest加入到waiting_for_pg队列里。

# 6.3 读写操作的序列图

写操作序列图如图6-2所示。

写操作分为三个阶段：

□阶段一从函数ms_fast_dispatch到函数op_wq.queue函数为止，其处理过程都在网络模块的回调函中处理，主要检查当前OSD的状态，以及epoch是否一致。

□阶段二这个阶段在工作队列op_wq中的线程池里处理，在类ReplicatedPG里，

其完成对PG的状态、对象的状态的检查，并把请求封装成事务。

□阶段三本阶段也是在工作队列op_wq中的线程池里处理，主要功能都在类ReplicatedBackend中实现。核心工作就是把封装好的事务通过网络分发到从副本上，最后调用本地FileStore的函数完成本地对象的数据写入。

![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/40fa4abf-bf1f-487c-b4d7-1ee1c732e7cb/efe1f8c9693ca612ebb82884efa82f29546dd3c8974154dede158a2314d10295.jpg)



图6-2OSD处理写操作的序列图
