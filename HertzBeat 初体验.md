# HertzBeat 初体验

## 1、介绍

HertzBeat 赫兹跳动 是一个易用友好的开源实时监控告警系统，无需 Agent，高性能集群，兼容 Prometheus，提供强大的自定义监控和状态页构建能力。
![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/76f01af6733b5f0be2fb550b681116bf.png)

## 2、官方地址汇总

[官网地址](https://hertzbeat.com/zh-cn/) <===
[官方文档](https://hertzbeat.com/zh-cn/docs/) <===
[下载地址](https://gitee.com/dromara/hertzbeat/releases) <===

## 3、使用方式

可以选择本地服务或者docker服务，有时间的话以下都会一一尝试，其实都是很简单

## 4、方式1 本地服务

下载完后解压
![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/d4652eac2eb326ed93e457f59d3592a9.png)修改位于 hertzbeat/config/application.yml 的配置文件(可选)，您可以根据需求修改配置文件
（一般来说）此处使用mysql，如图
![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/75bd1c5dafe1f6870a85653c829109de.png)

配置完启动就可以了，运行startup.bat,然后输入 127.0.0.1:1157 访问Web界面

![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/c09c3a63b724e8b3d978b6cc7de16491.png)

然后登录，账号密码如果没有自定义的话 是 admin/hertzbeat，内部界面大致如图

![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/2cb201418e22ef060ca03974c0d87e0a.png)

它可以简单方便的配置许多监控，以下例举一些监控配置

### 4.1、Mysql监控

在数据库监控中选择想要的数据库，例举Mysql
![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/0e0d53f55c3907f884a84289cb44c545.png)
新建时输入数据库地址以及账号密码，监控的库可选可不选
![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/214e2e5a8e4d3ec1672a6007d8f16bc3.png)
![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/71016485e2ff6743186b64d7d5705476.png)

### 4.2、Redis监控

在缓存监控中选择redis数据库，配置对应的ip和相关信息
![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/d62148b2a59cc325a64a77f40cd214ad.png)
监控结果
![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/c934ba59e73db0dbd5034c30c8c974aa.png)
![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/07374bb9c8575a425d45b02fbfe09d55.png)
![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/86fc90b5b6bad913002a4541b1b8a22d.png)

### 4.3、Linux服务器监控

对服务器进行监控，对应配置参数就是ssh连接配置，默认22端口
![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/f69747503991b59ebf0d6575f6576073.png)
监控结果
![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/44c59deb5648281dc4ebaae18c6c327d.png)
![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/e3098261a11fcaec557db5e26c86322c.png)
![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/d282e4d2a84db881786475669530ca3a.png)

### 4.3、SSL证书监控

应用服务监控中找到SSL证书，配置证书所在的服务器域名或者ip地址
![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/dccc61213cd6ce9c272a0a024aeb6a6f.png)
![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/4aacf0032bfffb514878ed2c8b2e1c4e.png)
![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/9c2d26abc0e4565aea244efac9acb75e.png)
有许多只需要简单配置服务地址就能实行监控的类型，这种就不一一示范，相信只要一启动项目就会配置了，以下介绍一些，需要额外配置的监控类型

### 4.4、JVM监控

HertzBeat 使用 JMX 协议 对 JVM 虚拟机的通用性能指标(基础信息，内存池，类加载，线程信息等)进行采集监控。
所以要监控jvm需要程序运行时开启jmx，启动时加入对应参数
[开启教程](https://docs.oracle.com/javase/1.5.0/docs/guide/management/agent.html#remote)
[使用指南](https://hertzbeat.com/zh-cn/docs/help/jvm/)
以我本地项目为例，启动时加入以下参数

```powershell
-Djava.rmi.server.hostname=127.0.0.1  
-Dcom.sun.management.jmxremote.port=9999 
-Dcom.sun.management.jmxremote.ssl=false 
-Dcom.sun.management.jmxremote.authenticate=false 
```

![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/9e5f6d3c36bf4c9e7469e05e6194f58a.png)
![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/5f0d2771356eb5a80bd963bb952dfbc2.png)
监控结果
![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/294bbc1fbce1fe31310f39e7fc8486cc.png)
![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/bb75bb5cc657f057b3e2d5618f5be6a9.png)

### 4.5、SpringBoot2监控

![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/4186871a45b1933b814383108156f9e7.png)
如图，他是依靠 SpringBoot Actuator 暴露的通用性能指标(health、environment、threads、memory_used)进行采集监控，所以需要项目中就整合Actuator 
具体整合SpringBoot Actuator就不在这细讲了，粗略的说明如何开启，如果有安全框架记得排除Actuator相应的接口

```xml
// 依赖
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

```yaml
# 配置项
management:
  endpoints:
    web:
      exposure:
        include: '*'
    enabled-by-default: on
```

监控结果
![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/3d25be6704d65203000c1f59649183bc.png)

### 4.6、告警

可以在阈值规则中设置对应的监控以及阈值范围，再设置通知模板后，达到阈值就会发送相应的通知
![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/f3d5e107d2cd94abd71cd60b03c900da.png)
这边配置接收对象，然后测试消息
![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/3b4cbebf98adb7cd46e82c4909b86c0f.png)
![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/5008202c2f5d8962b5c08602409bb5e0.png)
