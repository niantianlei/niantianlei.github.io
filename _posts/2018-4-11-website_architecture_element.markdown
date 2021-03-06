---
layout:     post
title:      "网站核心架构要素"
subtitle:   " \"website architecture element\""
author:     "Nian Tianlei"
header-img: "img/post-bg-2016.jpg"
header-mask: 0.4
tags:
    - 架构设计
---


软件系统的组成部分以及它们之间的关系就是软件架构。  

软件架构除了关注功能需求外，还需要关注性能，可用性，伸缩性，扩展性，安全性 等因素。这些因素也是衡量一个网站质量的重要标准。  

**性能：**   
一个打开缓慢的网站会流失很多用户。  
性能是网站架构设计的重要方面。  
在浏览器端，可以缓存静态资源，压缩页面传输 等手段。  
可以使用CDN，将静态数据放在离用户最近的网络服务提供商。  
可以使用反向代理，缓存热点数据。  
服务器端，可以使用本地缓存和分布式缓存。  
也可以通过异步操作将用户请求发送到消息队列中，而生产者直接将请求返回给用户（不必等消费者处理完数据），以提高性能。（当然，不是所有请求都可以使用异步交互的方式）。  

有很多请求时，可以使用集群，来分担请求，提高性能。  
可以改善内存管理，比如设置JVM参数及设置合适的垃圾回收策略。  
数据库服务器端，建立索引，缓存，SQL优化。  
衡量网站性能的一系列指标：响应时间，TPS，系统性能计数器等。  
这些指标也是监控网站的重要标识。  
对于网站而言，性能符合预期只是最低要求。  
还需要考察系统在持续高并发的情况下，超负荷引起的性能问题。  


**可用性：**  
高可用性的目标是：服务器挂掉后，应用依然可用。  
高可用的主要手段是冗余。数据在多台服务器上相互备份。  
使得任何一台服务器挂掉后，都不会影响应用的整体可用性。服务器挂掉后，只需要把请求转发到其它（功能相同）服务器即可。  
但有一个条件：应用服务器不能保存会话信息，否则服务器宕机，会话丢失（两个服务器数据还是不一致）。  
衡量一个系统架构是否满足高可用的目标时，就是假设系统中任何一台或多台服务器宕机时，看系统是否能正常运转。  

**伸缩性：**  
不断向集群中加入服务器的手段，来缓解不断上升的用户并发压力和不断增加的数据量存储需求。  

衡量伸缩性 就是判断是否可以搭建集群，以及向集群中加入服务器是否容易，集群中服务器数量是否有限制。  

对于应用服务器集群，只要服务器上不保存数据（包括会话数据，数据库数据等），那么通过负载均衡就可以不断加入服务器。  

对于缓存服务器集群，新加入服务器可能会导致缓存数据失效，不过数据可以从数据库进行加载。  

缓存服务器集群，可以这样实现，将整个范围设置为一个环，将缓存服务器加入这个环中，将环分为很多个段，然后每个缓存服务器只需负责一段，当添加新的缓存服务器时，只是将某一段分为两份，切分点插入这个缓存服务器，这样只需要将一个缓存服务器的一部分数据移动到新的缓存服务器上，对缓存整体影响较小，当删除缓存服务器时，只需要将两个段连在一起，将待删除服务器上的数据，移动到一个服务器上即可，对整体影响较小。  
关系型数据库很难做到大规模集群的可伸缩性（主从热备机制也不适合大规模）。  
对于大部分NoSQL数据库产品，天生就是为海量数据而生。对伸缩性支持较好。  

**扩展性：**  

功能不断扩展，如何设计网站架构，使得其能够快速响应需求变化。  
衡量标准是 当有新功能添加时，看是否这个新功能对原来的代码有影响，不需要修改任何代码（或改动很少），就可以上线新产品。  
实现可扩展性的主要手段是事件驱动和分布式服务。  
事件驱动常常使用消息队列来实现。可以分离开消息生产者和消息消费者，使得可以透明地增加消息生产者任务和消息消费者任务。  
分布式服务将业务和可复用服务（服务层可以被上层业务层调用）分离开，新增的产品可以调用可复用服务，来实现自己的业务逻辑。  

**安全性：**  
保证网站不受恶意攻击，保护网站重要数据不被窃取。  