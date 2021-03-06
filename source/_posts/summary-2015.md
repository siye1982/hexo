layout: photo
title: 2015个人总结
tags:
  - 随笔
  - 个人总结
photos:
date: 2015-12-30 17:23:20
categories: 随笔
---


# 写在前面

2015年对自己来说, 是收获的一年, 快速成长的一年. 年关将至, 需要将这一年里面做的事情整理一下, 整理的过程也是自我反思的过程. 当然也不能免俗的展望一下2016年. 写这篇总结的时候正好看到了一篇 robbin 的一篇文章《怎样以创业者的心态打工》, 其中的观点非常好, 在此谨记: 你的"职业生涯"就是以你名字命名的公司. 即使你在打工, 其实你也是在创业. 只不过你创业的主体不是一家公司或者项目,而是'你自己',以你名字命名的职业生涯. 你应该像呵护一个创业公司那样呵护自己的职业生涯, 不断的提高自身的知识,经验,视野,能力, 以及在行业当中的影响力和人脉.
<!--more-->

# 工作列表
* IMAGO集中配置管理系统的设计,开发,推广. 公司初始所有开发项目的配置都是散落在各自项目中,每次发布生产,集中修改都是一件非常痛苦的事情, 基于这个核心痛点,进行设计开发. 当然, 当初设计的原则并没有放眼全公司都全部使用,只是考虑到商场Java项目的方便管理, 随着影响面越来越大, 居然做成了一个全公司跨多语言使用的项目. 这个项目也是经历了多次迭代开发, 需要做的足够轻量, 不强依赖任何组件, 需要兼容复杂的测试环境. 每一次迭代都是对自己技术和沟通的一次成长.

* LOOM服务治理框架的设计,开发,推广. 公司在年初时服务之间调用还处在一个混乱状态, 各个应用中需要写死调用服务的地址, 没有一个地方知道服务之间的依赖关系, 不能实时知道各个服务调用的健康状况, 运维成本极高, 高速的业务发展和原始的服务调用管理产生了激烈的矛盾. 本想直接使用阿里的Dubbo框架, 但是公司原有服务调用是采用的Twitter技术栈中的Finagle框架, 对原生Thrift协议做了改动, 并可以很方便的支持异步调用. 我们针对Dubbo服务治理的需求自己开发了基于Finagle-thrift协议的服务治理框架, 并自己开发了服务依赖关系, 监控状况监控等模块.LOOM一开始只是定位于Java的Thrift服务治理, 后来逐渐开发了Go, Nodejs, Ruby等语言的客户端, 形成为一个多语言服务治理框架. 这个过程是曲折而有趣的. 真的很荣幸有机会作为核心设计,开发人员参与进来. 

* 基于Finagle的Zipkin完成的RPC调用链跟踪. 因为公司的RPC调用主要依托Twitter的Finagle框架, 我们在使用Zipkin原始的调用链收集时, 有时会出现内存溢出, 调用链数据丢失,混乱等现象. 为了可以复用本身的Zipkin项目, 决定修正Zipkin自身的一些问题, 结合自己的EAGLEYE系统,将调用链收集集成到了服务治理框架中.

* EAGLEYE监控预警平台的设计,开发,推广. 这个项目比较有意思, 一开始进公司的第一个任务就跟踪业务流程, 并发现问题. 最开始想的是基于日志去做这个事情, 但是我们的一些基础设置, 中间件都不成熟, 就先去做集中配置管理和服务治理了, 但是自己还是最寄希望于这个项目. 最开始的设想是将EAGLEYE做成基于日志的监控预警平台. 全公司的日志都收集到EAGLEYE中, 不管业务日志,监控日志等. 然后EAGLEYE中做统一的预警规则, 统计图表, 诊断问题. 不过后来公司有另外一个团队去做了统一监控平台. 所以现在EAGLEYE中部分功能虽然已经完成, 但是没有全面推广. 现在主要是做全量日志收集, 并提供给所有开发人员快速查询日志. EAGLEYE采用的是ELK技术栈, 这期间我们积累了很多使用Elasticsearch的经验, 现在也在全公司推广Elasticsearch技术. 将来EAGLEYE会全面接入自动化部署中所有Docker容器中的抓包数据, 并做各种数据分析.

* Kafka集群监控预警. 公司现在对Kafka集群监控处在缺失状态, 也一直想做这块的监控, 但是人员不够,所以暂时搁置.

* Zookeeper集群监控. 借鉴了阿里的Taokeeper的一些收集数据的思路, 自己完成了对多个Zookeeper集群的监控, 历史曲线展示, 实时预警功能, 并集成在了EAGLEYE系统中.

* FLO集中调度系统. 这个项目只在脑袋中, 笔尖上设计过, 并没有进行实际的开发, 也是因为公司有其他团队在做这个事情了.

* MODIS是一个针对Redis单实例和集群监控的系统. 借助这个系统深入了解了一下Redis集群的设计思路,对国内Codis项目做了深入的测试. 认为公司现在使用Redis的痛点是没有一个系统很好的让运维人员管理现有杂乱的Redis实例, 而不是去推广Redis cluster, 并迁移到Redis cluster中, 现在迁移的需求还不是很强烈.

* 组织了技术团队15次Opentalk. 大家分享了各个小组做过的业务, 用到的技术. 给整个团队做Opentalk对自身成长非常有帮助, 这一点深有体会. Opentalk本身也是非常好的团建形式.以后争取每周做一次, 这个机会太难得.

* 年底参与到了基于Docker的私有云环境的搭建中, 这是一个很酷的过程, 这个过程有可能会颠覆之前做过的很多工作. 这就是一个已经圈起很长时间的浪头, 你不去适应它, 就会被狠狠的甩在后面.

* 多个大型项目的推动, 会涉及到大量的人员交互, 自己性格的缺陷少不了碰壁, 这就需要借助上司的威信排除困难, 自己性格缺陷也在逐渐认知并修复.

# 工作之余参加的活动
因为有了孩子, 所以周末的时候基本不去参加一些技术沙龙了, 不过还是回忆一下这一年都参加了哪些活动, 以后还是想抽出时间多参加一些技术活动, 对自身成长还是很有帮助的.

* 参加了两期阿里的技术沙龙
* 参加了8月份的蘑菇街技术分享
* 参加了两期光环组织的项目管理分享,并担当摄影志愿者(主要是想攒PDU积分), 又一个三年, PMP证书在11月份进行了换审.
* 10月份参加了Elasticsearch在北京组织的开发者会议, 并在会议中担当摄影师, 最后的大合影是我照的.

# 今年读过的书单
* 《JavaScript权威指南》 阅读20%
* 《Linux命令速查手册》 阅读100%
* 《Redis设计与实现》 阅读63% 主要是在做公司的redis集群监控时参考查看.
* 《ELK stack 权威指南》 重点看了Elasticsearch部分, 三斗大神的书不错.
* 《Scala程序设计》 阅读60%
* 《TCP/IP入门经典》 阅读100%
* 《把时间当做朋友》 阅读100%
* 《白帽子讲Web安全》 阅读21%
* 《白夜行》 阅读100%
* 《创业维艰》 阅读100%
* 《简约至上》 阅读100%
* 《跑步该怎么跑?》 阅读100%
* 《如何控制自己的情绪》 阅读9%
* 《三体》 阅读45%
* 《上帝掷筛子吗?》 阅读100%
* 《硬派健身》 阅读100%
* 《增长黑客》 阅读60%
* 《中国通史》 阅读20%
* 《走进搜索引擎》 阅读25%
* 《Flask Web开发》 阅读32%
* 《Java8函数式编程》 阅读100%
* 《程序是怎样跑起来的》 阅读20%
* 《黑客与画家》 阅读100%
* 《MacTalk人生元编程》 阅读100%
* 《痛风看这本就够了》 阅读100%
* 《滚蛋吧!肿瘤君》 阅读100%
* 《极简欧洲史》 阅读100%

写到这, 看了一下, 读过的书还不少, 可能有很多内容已经忘记了,不过没关系, 觉得重要, 我再看一遍. 其中相当一部分技术书在阅读的时候做了笔记. 这都是因为有Kindle的威力, 很多人都在犹豫要不要买Kindle, 这根本不需要犹豫, Kindle会给你一个安静的阅读环境, 没有外界干扰, 绝对值得拥有, 不管你有没有大屏手机,ipad, 你都值得拥有一个Kindle. 

# 长期关注并订阅的网络内容
为什么会有这么一块, 是因为, 其中很多时间是花在了手机阅读上, 占了相当的比重.

* 微信公众号
	* 一小时爸爸
	* 高可用架构
	* InfoQ
	* 年糕妈妈
	* 学习学习再学习
	* 技术创业空间
	* 大玩家张磊
	* caoz的梦呓
	* 丁香医生
	* 丁香妈妈
	* 痛风医生
	* 小道消息
	* 肉饼铺子
	* TimYang
	* 老鹰说
	* 协童旅行社区

* 微博
	* 北京人不知道的北京事儿

* 网站
	* elastic.co
	* dockone.io
	* duokan.com
	* github.com
	* stackoverflow.com
	* infoq.com
	* coolshell.com
	* songshuhui.net
	* google.com
	* 廖雪峰的python教程


# 生活列表
之前的工作时生活和工作混合在一起, 感觉效率低下. 现在将工作和生活分开, 保证高效的工作效率. 非常反感漫无目的的加班, 不做任何精确到天或小时的规划的加班行为应该静下心来好好思考一下.今年四月份和朋友喝了场大酒, 结果晚上第一次痛风发作, 很是痛苦, 接下来三个月走路都不能很顺畅.这以后就比较关注自己的血尿酸指标,并进行锻炼,控制体重. 这期间看了大量和痛风相关的科技文章, 也看了一些和健身相关的书籍. 我将自己主要的运动方式确定为快走和慢跑.这里主要是用咕咚应用记录自己的运动轨迹. 下面是一些数据列表

* 运动数据
	* 从7月份开始累计跑步261.27公里
	* 从4月份开始累计步行500公里
	* 平均每次慢跑在5公里
	* 最长一次跑步为15公里, 距离半马还差6公里.
	* 体重削减14斤.

* 今年六月份开始将技术积累迁移到自己的Github博客, 目前写了十四篇技术文章,其中两篇未完.
* Evernote笔记累计到512篇, 其中今年笔记75篇,这些会逐渐迁移到Github的page上.
* 将家庭网络和办公网络都设置了Shdowsocks, 没有Google的日子是没法工作的.
* 每天坚持用挖财记录财务支出明细.
* 升级Nikon相机从D300到D810
* 自己给孩子拍了白天和满月照
* 从11月份看了李笑来得《把时间当做朋友》, 现在每天记录自己的行动日志, 希望可以一直坚持下去.
* 带着去年的全家福完成今年的全家福
* 在回家吃饭定了80多天外卖

# 明年计划

* 书单
	这里想记录一下今年没有来得及看的好书. 还有一些是看过了,但是有必要重读的.

	* 《必然》
	* 《Python cookbook》
	* 《文明之光》
	* 《如丧》
	* 《把时间当做朋友》 重读
	* 《光荣与梦想》

* 戒烟
* 对Python进行深入的学习
* 深入学习产品设计(中间件产品, 非技术用户产品)
* 带家人进行两次旅行





<div style="margin-top: 15px; font-size: 11px;color: #cc0000;"><p align="center"><strong>（转载本站文章请注明作者和出处 <a href="http://siye1982.github.io">Panda</a>）</strong></p></div>

