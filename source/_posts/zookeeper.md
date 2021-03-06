

title: Zookeeper 扩容实战
date: 2015-06-16 15:52:25
categories: Zookeeper
tags: 
- 原创
- Zookeeper
- 分布式
---

## 场景描述:

1. zookeeper 版本 3.4.6

2. 现有zk集群是五台, myid分别为 0, 1, 2, 3, 4
<!--more-->
3. 三地机房  
  (1). 机房1, 现有集群在该机房, 主机房, 服务的主要流量在该机房. 目前zk的5台机器在该机房. 
  (2). 机房2, 热备机房, 有全量服务但是机器数量较机房1少, 分担少部分负载, 在机房1不可用时,将会对外提供所有服务.
  (3). 机房3(延时较大,在100ms).

4. 需要构建一个高可用zk环境, 服务主要部署在机房1, 机房2有全量服务但是机器数量较机房1少. 

5. 现在需要将机房1,2做成一个大的zk集群, 但是由于zk对双机房, 不能做到高可用, 所有加了一个机房3. 现在这三个机房的zk实例数为 5 + 5 + 1 .

6. 现有zk实例为5, 但是我们需要扩容到11台, 添加实例数比原有集群实例数大.

7. 在扩容过程中需要不影响使用现有zk集群的服务. 不可以全部停止, 进行升级.



## 需要注意的问题

1. 添加的机器数大于现有集群zk实例数.

2. 三地机房, 其中机房1为主机房, 资源最多, 尽量让leader落在该机房. 机房1和机房2的延时在容忍范围内, leader也可以落在该机房, 但是需要优先考虑机房1. 因为机房3延时较大, 尽量不可以让机房3的实例担任leader角色.

3. 历史遗留问题, 原有zk集群的myid是从0开始的, 这是个坑(稍后会说).


## 具体步骤

### 修改myid 

  为什么要先修改myid, 这是之前我们给自己挖的一个大坑, 这次一定要填上, 并且为以后的zk运维积累经验.因为, 我们需要leader尽量落在机房1的机器上, 鉴于zk集群进行leader中用到的快速选举算法, 集群中的机器会优先匹配zxid最大的实例(这样可以保证在数据同步时,这个实例上的数据是最新的), 如果所有实例中的zxid都一样, 那么所有实例会选举出myid最大的实例为leader. 基于这样的条件, 我们需要将机房1中的现有的5台的myid进行提升, 给机房3的zk实例腾出myid的位置(以确保在zxid一样时,它肯定不会是leader). 因为zk中myid的范围必须是大于等于0(没错,你没看错,我们使用了0, 即使官方sample配置中是从1开始, 但是我们还是使用了0), 所有我们需要先将myid=0的实例进行myid变更. 
  

1 . 修改myid=1的机器的myid为100, 依次对修改五个实例的zoo.cfg
   
  修改完之后的配置类似如下:
    ```	
	server.1=192.168.1.101:2555:3555
	server.2=192.168.1.102:2555:3555
	server.3=192.168.1.103:2555:3555
	server.4=192.168.1.104:2555:3555
	server.100=192.168.1.100:2555:3555
    ```

2 . 记录现在集群中哪台机器为leader, 该机器最后重启.

3 . 依次重启myid为1,2,3,4,100的实例(注意最后重启leader)

***ok, 这里我说另外一个坑, 我们重启服务的时候最好是依从myid从小到大依次重启, 因为这个里面又涉及到zookeeper另外一个设计.zookeeper是需要集群中所有集群两两建立连接, 其中配置中的3555端口是用来进行选举时机器直接建立通讯的端口, 为了避免重复创建tcp连接,如果对方myid比自己大，则关闭连接，这样导致的结果就是大id的server才会去连接小id的server，避免连接浪费.如果是最后重启myid最小的实例,该实例将不能加入到集群中,因为不能和其他集群建立连接, 这时你使用nc命令, 会有如下的提示: This ZooKeeper instance is not currently serving requests. 在zookeeper的启动日志里面你会发现这样的日志: Have smaller server identifier, so dropping the connection. 如果真的出现了这个问题, 也没关系, 但是需要先将报出该问题的实例起着,然后按照myid从小到大依次重启zk实例即可. 是的,我们确实碰到了这个问题, 因为我们稍后会将机房3的那个zk实例的myid变为0,并最后加入到11台实例的集群中,最后一直报这个问题.***


### 添加新机器进入集群

经过上面的步骤,现在来添加新机器进入集群. 因为新集群zk实例数量为11台, 那么如果能做到HA,需要保证集群中存活机器至少为6台. 鉴于这样的要求,我们并不能一次性将11台机器的配置修改为如下:
```
server.0=192.168.3.1:2555:355555
server.1=192.168.1.101:2555:3555
server.2=192.168.1.102:2555:3555
server.3=192.168.1.103:2555:3555
server.4=192.168.1.104:2555:3555
server.5=192.168.2.1:2555:3555
server.6=192.168.2.2:2555:3555
server.7=192.168.2.3:2555:3555
server.8=192.168.2.4:2555:3555
server.9=192.168.2.5:2555:3555
server.100=192.168.1.100:2555:3555 
```


	
我们只能先将原有的5台zk实例的集群先扩充到7台(为何不是8台?慢慢梳理一下就知道了), 然后再扩充到11台这样的步骤. 鉴于这样的思路,我们的步骤如下:

1 . 选出两台新的实例, 加上之前的5台, 将他们的配置文件修改为7台,依次重启原集群zk实例,然后启动两台新加入的实例, 注意最后重启leader.
    ```
	server.1=192.168.1.101:2555:3555
	server.2=192.168.1.102:2555:3555
	server.3=192.168.1.103:2555:3555
	server.4=192.168.1.104:2555:3555
	server.5=192.168.2.1:2555:3555
	server.6=192.168.2.2:2555:3555
	server.100=192.168.1.100:2555:3555 
	```

	




2 . 将zoo.cfg中的集群机器数量设为11台, 已经存在的7台zk实例集群进行重启,然后重启另外四台新zk实例. 这里你可能在启动myid=0的zk实例会出现上面描述的问题,没关系,按照上面说的步骤操作即可.



<div style="margin-top: 15px; font-size: 11px;color: #cc0000;"><p align="center"><strong>（转载本站文章请注明作者和出处 <a href="http://siye1982.github.io">Panda</a>）</strong></p></div>










