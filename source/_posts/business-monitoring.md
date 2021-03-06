layout: photo
title: DIGGER业务监控
tags:
- 原创
- 监控预警
- Elasticsearch
photos:
  - http://siye1982.github.io/img/blog/digger/digger.png
date: 2016-12-24 10:12:31
categories: 监控预警
---

# 系统定位
业务监控, 主要侧重对业务数据的实时监控, 以日志的方式进行数据收集, 对业务数据进行深入的统计分析, 帮助业务方发现问题, 定位问题根源. 这其中数据分为: 业务自身输出的业务日志(比如: 提单, 推单, 接单等状态数据), 业务异常, 报警事件等. 
<!--more-->
发现问题原因之后我们需要解决问题, 最终目的是可以基于我们分析的结果给运维动作做出决策, 以达到自动化运维的目的. 另外, 明确系统用户将有助于把控业务监控产品的设计方向, 业务监控系统的第一用户是RD, 不是老板, 我们是要帮助RD更快的发现问题, 预知问题, 提供标准化解决问题的建议.

# 系统设计
## 数据模型
![数据模型](http://siye1982.github.io/img/blog/digger/data_model.png)

* BusinessLog: 需要在业务中进行埋点, 业务主动以固定的格式通过日志输出的方式上报数据, 然后Digger会实时汇总分析, 通过各种曲线展示业务状态的走势.
* EventLog: 随着各个业务系统的成长, 每天会有各种事件产生. 当某些系统有问题时, 通常会产生各种报警事件. 另外, 系统大部分的故障都会和发布新版本,配置变更有关系, 记录这些事件并加以分析, 可以帮助我们快速定位问题的根源.
* ExceptionLog: 当业务代码运行时没有达到预期将会抛出运行时异常, 对这些异常进行收集并设置分类器, 将会帮助我们更细粒度的定位到问题的起因.


## 数据收集流向
![数据收集流向](http://siye1982.github.io/img/blog/digger/digger_data_flow.png)

1. 业务服务中产生的数据主要包含两大类:
	(1) 不需要业务服务埋点, 通用底层监控框架自动收集的数据. 这其中包括: 系统指标,性能指标,服务异常等.
	(2) 需要业务服务埋点,自己上报业务状态日志. 这部分数据会直接通过日志的方式流入日志收集通道.
2. 底层监控系统主要是类似zabbix的系统层级的监控系统, 我们主要使用的是Cat,Falcon等开源系统, 这些系统本身可以配置各种报警规则, 并可以对异常日志进行归类统计. 因为各个系统产生的事件发送方式繁多, 我们如果各个去适配,后续的维护成本会非常高.这时我们将考虑推动事件接收通道的统一.
3. 公司层面所有的事件都会通过IM的方式推动给各个团队, 这时我们申请开通了Digger系统自己的IM账号, 并推动各个事件产生方统一给Digger的IM账号发送事件信息. 
4. Digger系统会部署自己的数据收集器(Digger-collector), 用来收集IM通道中的各种事件,以及其他特殊原因产生的监控数据.
5. 不管是业务埋点发送的业务状态日志, 还是Digger-collector收集的事件等信息, 最终都会统一发送到统一的日志通道中.
6. 我们最终使用Elasticsearch让监控日志落地, 利用Elasticsearch做各种复杂的近实时的统计分析.
7. Elasticsearch中会存储聚合分析后的结果数据, 最终通过Digger-admin读取展示监控图表给用户.
 

## 系统结构
![系统结构](http://siye1982.github.io/img/blog/digger/digger_structure.png)

1. Digger-client: 业务监控系统首先需要有一个足够轻量的client用来帮助业务服务低成本的进行业务状态数据的埋点收集.
2. Digger-collector: 对已有的基础监控系统产生的数据进行收集, 这其中需要兼容各种数据收集接口,同时对于一些特殊的系统, 还需要暴露自己的http服务, 以便其他系统通过回调http接口的方式收集监控数据.
3. Digger-processor: 该组件主要是针对收集上来的各种类型的数据定时的进行统计分析, 并将结果数据以日志的方式回写到Elasticsearch中.
4. Digger-admin: 该组件主要是暴露给Digger业务监控的用户管理界面, 在这里面可以定制自己的监控图表,对自己关系的服务进行监控检查等. 关于核心业务监控产品都将在该组件中体现.

现有报警数据种类繁多, 每天产生的数量庞大, 这其中有相当一部分数据是因为阈值设定不合理而引起的误报而是会降低报警的实用性, 我们需要实时收集这些报警事件并做二次统计分析, 报警事件由多种方式发出, 所以需要有独立的collector组件负责收集这些事件, 后续其他类型业务数据的收集也可以通过collector完成.由于异常数据, 业务数据, 事件数据都有各自的结构, 为了简化日志格式化处理过程, 我们需要封装一个client来做这个事情, 尽量使日志格式化的动作对业务方透明.


# 核心产品功能

## 业务大盘
![业务大盘](http://siye1982.github.io/img/blog/digger/digger_business_panel.png)
在基础监控平台(比如:Cat, Falcon)中, 我们是可以看到各自服务的QPS, TP99, JVM, CPU等基础监控指标. 针对核心业务流程中的业务状态的监控, 每个业务服务的监控需求各不相同, 有的业务需要很多维度的监控, 并且需要快快速的帮助开发规范化问题排查的SOP.

1. 以订单核心业务流程为主, 针对提单-> 推单-> 接单-> 分配送等流程配置订单监控大盘.
2. 以分钟粒度可以查看近三日的业务曲线,给用户一个直观的趋势变化.
3. 当曲线中出现陡升或者陡降时, 通过点击出现菜单, 菜单中针对各个业务服务定制自己排查问题的SOP, 在出现问题时快速引导用户排查问题并可以在曲线中快速做好标注.
4. 针对每个业务曲线可以进行未来一天的预测, 并在每分钟对未来一分钟的预测数据进行实时修正, 已达降低根据预测曲线进行报警的错误率.
5. 每个业务流程点上可能会有多个数据状态产生, 

## 事件监控
![事件监控](http://siye1982.github.io/img/blog/digger/digger_event_panel.png)

## 健康检查
![健康检查](http://siye1982.github.io/img/blog/digger/digger_health_panel.png)

## 预测报警


## 监控收藏
![监控收藏](http://siye1982.github.io/img/blog/digger/digger_favorite.png)

## 降级限流管理

# 碰到的问题




<div style="margin-top: 15px; font-size: 11px;color: #cc0000;"><p align="center"><strong>（转载本站文章请注明作者和出处 <a href="http://siye1982.github.io">Panda</a>）</strong></p></div>

