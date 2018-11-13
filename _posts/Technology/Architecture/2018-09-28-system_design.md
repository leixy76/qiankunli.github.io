---

layout: post
title: 系统设计的一些体会
category: 技术
tags: Architecture
keywords: 系统设计

---

## 简介

![](/public/upload/architecture/system_design.png)

1. 几种“架构”不可以混淆
1. 业务设计中，"数据匹配"可能只是一句话，而在工程实现中则是一个很复杂的项目。

## 笔者的历程

1. 觉得代码实现是难点
2. 觉得数据库设计是难点。因为数据库设计有了，代码也就定了。一个学生信息管理系统，数据库设计自然没啥。但对于一个权限管理系统、审批流系统，数据库设计就很有含量了
3. 觉得业务抽象是难点。比如一个配置中心系统、ABTest 系统有哪些基本抽象，如何入手。此时是不是用数据库存储都是一个问题，比如还可以存在zk上。
4. 实现一个有闭环的平台类系统，比如风控系统等。

	* 首先它的输入是各种各样，输出也是各种各样的。
	* 各种技术的结合。java 业务处理、spark数据处理等
	* 性能要求带来的复杂度

对于第4点，也可以换个角度看，笔者曾经实现一个推送系统，各阶段如下：

1. 可送达，将业务方的消息送达到客户端。这中间涉及到推送的基本概念等
2. 可调控，比如频控，限制某个业务方的消息，以防止推送过多对用户的打扰。此外，动态调度，保证推送服务器的负载相对均衡
3. 推送系统与业务方的关系问题。比如一个业务的推送点击率很低，那么如何提高？是推送系统做还是业务方来做？这里面就需要你自己形成一套方法论，这个方法论的形成需要你的实践与沟通。沟通可不简单啊，你找人家，人家也得有时间，或者过几天他想法又变了
4. 推送系统达到一个均衡。老板比较关心点击率，业务方却只想要更多的推送机会，你担心会对用户形成打扰。推送与业务方的接口，数据报告（用数据而不是感觉说话）。这个过程中，你的人力有限，先做什么后做什么。


本质上，springmvc等，也是各种信息 获取、存放和处理，存储机制可以是xml文件、内存等



## 业务设计

1. 定义项目解决的真正问题。很多时候，“有一个需求，我们做个系统吧”，然而系统做着做着，就会跑偏掉，要么自己想一堆伪需求，要么做出来的系统用处不大或只解决了部分问题。
1. 解决有什么抽象，设定几个概念术语来描述系统、向别人解释系统
2. 基本理念，比如推送系统，宁愿不推也不要打扰用户。基本理念很重要，要害就在于影响了很多决策

	* 假设你的算法 有一点错误率，那错误率 让哪些用户来扛
	* 有个新需求很紧急，你干不干，一个新feature 你接不接，风险点是否跟基本理念冲突
	* 有很多事可以你的项目做，也可以别人的项目做，到底是你做还是别人做
	
4. 画后台界面原型图（如果有的话），我发现这是一个很有含量的东西

	* 充分反应了你对系统的认识，如果你认识不到位，这个界面根本画不出来，画出来用户也不知道所以。
	* 逼着你思考：一共要做哪些事儿？用户入口是什么？哪些让用户看到（易变的部分），哪些不让（不变的、可以自动化的部分）？
	* 和用户最佳的沟通工具

5. 迭代优于一步到位


### 如何分析一个业务

1. 找到所有相关人，摸清需求，罗列123
2. 梳理需求，摸清楚需求背后的需求，汇总需求，将需求归类，总结为一两个核心点
3. 列出所有可能的方案，针对每一个方案，**先深度优先遍历**，即搞清楚其所有关键点。比如方案一 关键是 技术1和技术2，技术1关键的技术3和技术4。然后，根据需求、已有资源、未来规划等对方案进行“剪枝”，最终得到一个看起来可行的方案
4. 群策群力，将方案抛出来，接受各方的challenge。


api 设计原则

**高层API以操作意图为基础设计**。如何能够设计好API，跟如何能用面向对象的方法设计好应用系统有相通的地方，**高层设计一定是从业务出发，而不是过早的从技术实现出发**。低层API根据高层API的控制需要设计。设计实现低层API的目的，是为了被高层API使用，考虑减少冗余、提高重用性的目的，低层API的设计也要以需求为基础，要尽量抵抗受技术实现影响的诱惑。

1. 工作所需的组件会用，会基本的排查，最好读过源码
2. 实现过 相当复杂度的组件
3. 形成一种常识与判断。比如开车，你左右转向要提前观察左右后视镜、超车时要关注对向来车、前面的车什么行为什么意图。

[技术攻关：从零到精通](http://zhangtielei.com/posts/blog-zero-to-professional.html)当为一个新系统编写代码的时候，代码应该从接口设计开始（我们平常做业务开发，在大部分情况下，都不用自己设计接口。比如做客户端开发，各种MVC, MVP模式已经把代码框架都定义好了，我们只用往接口实现里填东西。PS：但用不上不代表不重要）。先用代码定义出各层的接口（包括回调接口）(PS：第一次看到将回调接口 提升到接口设计的范畴)，没有实现，只是能够编译通过。有了这些接口，就可以拿它们与同事进行非常细节的讨论了。应该先把接口讨论得足够清楚，再进行下一步的具体实现。这也是一个比较痛苦的过程，我们需要反复抉择，而通常「选择」就意味着痛苦。

阅读技术文章来「循证」的做法。很多个人博主和团队博客会在网上发表他们自己系统的实现过程，以及系统前后版本的演进过程。如果我们恰好找到相关的类似这样的文章，那么它们就有很大的参考价值。我们从别人分享的技术方案中获得一个印证，确保自己的想法没有走向极端，或者漏掉了什么重要的东西。


## 《聊聊架构》 书评的笔记

[聊聊架构](http://www.infoq.com/cn/articles/talk-arch?utm_source=articles_about_talk-arch&utm_medium=link&utm_campaign=talk-arch)

### 多个范畴的生命周期

人一生的生命周期被各种不同的场景、任务、角色、身份切分成了各不相同的生命周期，其中有核心生命周期，有非核心生命周期，有些必须自己做，有些可以交给别人做。(读书、生活、谈恋爱)

在人类历史中，随着工作越来越复杂、工作任务越来越多，人类协作越来越精细，然后就产生了分工，**分工就是人类因为协作产生的生命周期的切分。**

明确了生命周期这个概念就会意识到，随着事物的发展，把它的一部分职能从其核心生命周期切分出去，构造出新的生命周期，能够帮助这一事物明确自身的核心生命周期、明确自己的职责和权力，**有更多时间用在自己擅长的事情上。**

### 代码、技术和业务

**要分得清楚访问代码、业务代码、存储代码、胶水代码各自应在哪些层级**，它们应该是什么角色，而不是所有代码散乱的混在一起，看起来似乎按照经典的MVC分层，实际上业务代码却同时出现在controller/service/DAO，这样其实并没有明确的划分。

正确的做法应该是controller完成访问逻辑；DAO完成存储逻辑；service完成胶水逻辑，承上启下，利用DTO转换访问参数、执行业务逻辑、调用DAO映射存储模型、再利用DTO把业务处理结果转换为响应结果，业务逻辑在业务模型中实现。如果把业务逻辑跟业务数据在一起实现就是充血模型，进一步深化就是DDD模式。

如果把业务逻辑跟业务数据在一起实现就是充血模型，进一步深化就是DDD模式。只有这样才完成了明确的软件层次划分，每层各司其职、权责对等，否则就是大泥球。
明白了这一点，自然就能分得清楚业务的事务跟关系数据库的事务不是一回事，也就不会考虑完成业务上的事务要依赖关系数据库事务确保数据完整性。

完全可以把二者分开，利用更符合业务规律的做法去实现，甚至业务本身已经有成熟的方案确保数据完整性，而不再需要依赖关系数据库事务。在业务上对关系数据库事务ACID特性的依赖既然不再是必须，对拥有ACID特性的数据库依赖自然也就不再是必须，完全可以根据业务需要选择合适的存储方案。

业务模型和具体实现不再依赖于某些具体方案的技术特性，**实现了业务与技术的解耦**，也就更容易实现横向扩展。现在又发现横向扩展也是自然界的一个基本特性。无数基本粒子构成原子、无数原子构成一个具体的宏观物体，一个人不够用就增加更多人。如果一个系统的规模在横向扩展上达到了瓶颈，不能再靠简单的增加数量获得提升的时候，一定是这个系统的组织架构存在某些不合理因素。

说到这里自然就要说到业务和技术的关系。前面说到软件是现实世界的映射抽象，由虚拟人代替自然人去完成一些工作。要做好软件自然就要理解业务，对业务的理解越深刻就越有可能做出优秀的软件。

但是现实世界太复杂了，随着业务发展，软件规模会越来越大，复杂性越来越高，一个人难以胜任全部架构工作，于是就产生了架构师团队，架构师也有了更细致的分工。架构师的生命周期也相应发生了拆分，也就产生了**业务架构、应用架构、系统架构。**

架构师为了能够实现自己的架构思想，自然需要与职能对等的权力。所以架构师其实不是一个纯粹的技术职位，而是拥有管理职能的职位，而不同角色的架构师对技术的要求也不尽相同。


小结一下

1. [](https://qiankunli.github.io/2018/07/14/nature_of_code.html)从编程语言/具体代码的角度 程序=逻辑+控制，逻辑与控制是分开的
1. 从代码整体结构设计的角度看，比如用controller/service/dao 但这不意味着业务逻辑也分散在 controller/service/dao，也就是说，如果有比较好的抽象（比如ddd），业务逻辑 与 代码设计 是分开的
2. 从其它技术角度看，事务的完整性既可以由业务保证，也可以由技术保证，