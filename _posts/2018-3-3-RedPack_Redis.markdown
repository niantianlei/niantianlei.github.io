---
layout:     post
title:      "Redis实现抢红包"
subtitle:   " \"Red Packet\""
date:   2018-3-3 22:20:18 +0800
author:     "Nian Tianlei"
header-img: "img/post-bg-2016.jpg"
header-mask: 0.4
tags:
    - 高并发
---

前文讨论了悲观锁、乐观锁实现抢红包，但并发性能远远不够，如果同时存在大量用户，则需要很长时间才能处理完全部请求，那对于后处理请求的用户早就上火了。  
使用Redis做缓存而非直接操作数据库可以大幅度提高性能，因为Redis直接操作内存，比数据库操作磁盘速度要快很多。  

有一个问题是，Redis存储数据并不是长久之计，它只是为了提供更为快速的缓存，而非永久存储的地方。所以当红包金额为0时，要将红包数据保存到数据库中，这样才能保证数据的安全性和持久性。  

首先要创建一个RedisTemplate对象，并注册到IoC容器中  
```
@Bean(name = "redisTemplate")
public RedisTemplate initRedisTemplate() {
	JedisPoolConfig poolConfig = new JedisPoolConfig();
	poolConfig.setMaxIdle(50);//最大空闲数
	poolConfig.setMaxTotal(100);//最大连接数
	poolConfig.setMaxWaitMillis(20000);//最大等待毫秒数
	//创建Jedis连接工厂
	JedisConnectionFactory connectionFactory = new JedisConnectionFactory(poolConfig);
	connectionFactory.setHostName("localhost");
	connectionFactory.setPort(6379);
	//调用后初始化方法，之前没写，抛出异常，网上查原因才发现必须写上
	connectionFactory.afterPropertiesSet();
	..
}
```
此时RedisTemplate就可以在Spring上下文中使用了。  
此外，使用Lua脚本保证原子性，保证数据的一致性访问。避免超发现象。   
第一次运行Lua脚本时，在Redis中编译并缓存，这样可以得到SHA字符串，之后通过这个字符串和参数就能调用Lua脚本了。   
在Redis运行可以使用`redis-cli --eval /路径 , 参数`类似的命令。  
Lua可以通过call()函数来执行Redis命令。  
因此可以将是否缓存抢红包信息的逻辑交给Lua执行，保证原子性。  
具体逻辑是：获取库存，红包库存大于0，库存减1，库存为0就需要将抢红包信息保存到数据库，给出一个状态标志即可。如果库存小于0直接返回。  
当全部红包抢完后，要开启一个新的线程将缓存数据保存到数据库中。否则如果用最后一个用户线程来执行这个任务，会造成该用户等待较长时间，影响体验。  
最后将缓存的两万条数据添加到数据库中，就完成了。  

当然首先要在Redis中设置缓存，然后启动程序，访问路径，发送请求。  
![a]({{ "/img/post/redPacket/Redis2.png" | prepend: site.baseurl }} )   
前两条为在缓存中写入数据，这里写入了stock剩余红包数量和unit_amount每个红包的面值。  
最后一条是我运行完程序，处理完请求后进行查询，得到当前库存为0，逻辑正确。  

又在数据库中进行查询，得到如下结果  
![b]({{ "/img/post/redPacket/Redis1.png" | prepend: site.baseurl }} )   
可以看到已经成功把缓存添加到了数据库中，数据库的红包数量也是0。  

再看一下性能  
![c]({{ "/img/post/redPacket/time.png" | prepend: site.baseurl }} )   
5秒多就完成了20000个抢红包操作，并且保证了数据一致性，没有超发现象。  
虽然1秒只完成几千个请求，不能实际应用，实际还有很多提高并发性的措施：主从服务器，消息队列等等。  
但比较下来性能还是远超悲观锁和乐观锁的（两者都用了一分半左右）。  

使用Redis实现高并发，通过Lua脚本规避了数据不一致的问题，当然也可以使用Redis事务锁的方式。最后另创建一个线程操作数据库，因此整个过程只有最后一次涉及到了数据库的操作，操作磁盘，性能得到提高。  
## 问题
最后提交数据的时候遇到了问题，如图  
![d]({{ "/img/post/redPacket/HotSpot.png" | prepend: site.baseurl }} )   
提示的错误信息是代码缓存空间不够用了，那我就改下虚拟机参数   
在Run->Debug configurations中进行设置，我设置到256m还是溢出，电脑比较差一共才4G内存，于是就改了下程序，不能一次性吧缓存数据全部添加到数据库，一部分一部分的添加。从Redis列表中每次取出以前个数据添加到数据库，直至完成，就成功解决了。  



