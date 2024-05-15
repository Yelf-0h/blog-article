# Sentinel 高可用流量防护框架

## 1、简介

Sentinel 是面向分布式服务架构的高可用流量防护组件，主要以流量为切入点，从限流、流量整形、熔断降级、系统负载保护、热点防护等多个维度来帮助开发者保障微服务的稳定性

- **丰富的应用场景**：Sentinel 承接了阿里巴巴近 10 年的双十一大促流量的核心场景，例如秒杀（即突发流量控制在系统容量可以承受的范围）、消息削峰填谷、集群流量控制、实时熔断下游不可用应用等。
- **完备的实时监控**：Sentinel 同时提供实时的监控功能。您可以在控制台中看到接入应用的单台机器秒级数据，甚至 500 台以下规模的集群的汇总运行情况。
- **广泛的开源生态**：Sentinel 提供开箱即用的与其它开源框架 / 库的整合模块，例如与 Spring Cloud、Dubbo、gRPC 的整合。您只需要引入相应的依赖并进行简单的配置即可快速地接入 Sentinel。
- **完善的 SPI 扩展点**：Sentinel 提供简单易用、完善的 SPI 扩展接口。您可以通过实现扩展接口来快速地定制逻辑。例如定制规则管理、适配动态数据源等

## 2、场景适配

当前我们系统集成了所有服务，会因为一个接口的突发流量而导致所有系统服务宕机

## 3、开始使用

### 3.1、下载安装

GitHub地址：[https://github.com/alibaba/Sentinel](https://github.com/alibaba/Sentinel)
![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/d9999b4f76ec3d11dd42ad6a8e1b2049.png)
这边演示环境 Windows10 16g 

下载jar包然后直接运行

```powershell
//默认如果不指定为8080端口 账号密码为sentinel
java -Dserver.port=9000 -jar sentinel-dashboard-1.8.7.jar
```

访问界面 [http://localhost:8080/](http://localhost:8080/)
![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/17ab7e8d7a34600f151c5ed94380465a.png)
到此为止如果登录进去后应该是没有数据的，因为我们自己的服务还没有启动，并指定sentinel服务端口，而且需要启动后访问一次接口才会被记录在sentinel中

### 3.2、服务端配置

配置一般两种方式

- 一种是直接引入cloud集成的Sentinel依赖-较为方便
- 一种是自己引入多个依赖，并且自己捕获处理-侵入性较强

**依赖**

- Cloud

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
    <version>2.2.8.RELEASE</version>
</dependency>
```

- 单独依赖

```xml
<!-- 版本与控制台保持一致即可 -->
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-core</artifactId>
    <version>1.8.7</version>
</dependency>
<!-- 版本与控制台保持一致即可 -->

<!-- Sentinel本地应用接入控制台，版本与控制台保持一致即可 -->
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-transport-simple-http</artifactId>
    <version>1.8.7</version>
</dependency>
<!-- Sentinel本地应用接入控制台，版本与控制台保持一致即可 -->

```

**配置文件**

```yaml
spring:
  cloud:
    sentinel:
      enabled: true
      transport:
        dashboard: localhost:8080 #sentinel服务的地址
        port: 8719

```

**测试类**

- 单独依赖版本

```java
package com.aurora.controller;

import com.alibaba.csp.sentinel.Entry;
import com.alibaba.csp.sentinel.SphU;
import com.alibaba.csp.sentinel.slots.block.BlockException;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/test")
public class TestController {

    @GetMapping("/sendMsg")
    public String sendMsg(){
        try {
            // 设置一个资源名称为 SendMsg
            Entry ignored = SphU.entry("SendMsg");
            System.out.println("发送一条消息");
            return "发送一条消息";
        } catch (BlockException e) {
            System.out.println("系统繁忙，请稍后");
            e.printStackTrace();
            return "系统繁忙，请稍后";
        }
    }

    /**
     * 使用代码编写流控规则，项目中不推荐使用，这是硬编码方式
     *
     * 注解 @PostConstruct 的含义是：本类构造方法执行结束后执行
     */
    @PostConstruct
    public void initFlowRule() {
        /* 1.创建存放限流规则的集合 */
        List<FlowRule> rules = new ArrayList<>();
        /* 2.创建限流规则 */
        FlowRule rule = new FlowRule();
        /* 定义资源，表示 Sentinel 会对哪个资源生效 */
        rule.setResource("SendMsg");
        /* 定义限流的类型(此处使用 QPS 作为限流类型) */
        rule.setGrade(RuleConstant.FLOW_GRADE_QPS);
        /* 定义 QPS 每秒通过的请求数 */
        rule.setCount(2);
        /* 3.将限流规则存放到集合中 */
        rules.add(rule);
        /* 4.加载限流规则 */
        FlowRuleManager.loadRules(rules);
    }

    @GetMapping("/clear")
    public String clear(){
        return "清空消息";
    }
}

```

- cloud集成版本

```java
package com.aurora.controller;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/test")
public class TestController {

    @GetMapping("/sendMsg")
    public String sendMsg(){
            System.out.println("发送一条消息");
            return "发送一条消息";
    }

    @GetMapping("/clear")
    public String clear(){
        return "清空消息";
    }
}

```

- 我们访问接口地址 [http://localhost:9080/test/sendMsg](http://localhost:9080/test/sendMsg)

![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/acfc010900e7f28defb7bfc9679cd438.png)

- 这时候查看sentinel服务界面，我们能看到请求时间，请求状态，通过的qps拒绝的qps

![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/63a447ec4a635433a462647df7ea4250.png)

- 点击簇点链路，我们能看到实时的分钟通过数等信息，我在来查看前连点接口10下

![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/b66e0a62677077e8caa33eb2b86c109f.png)

#### 3.2.1、流控规则

- 接下来我们对这个接口添加流控规则

![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/ab930197678748ff1f5cac0bae5e0eba.png)

- 对QPS的单机阈值设置为2，也就是说每秒只能有2个正常通过的流量，多余流量会被拒绝，设置后点击流控规则就能查看已经设置的规则信息了

![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/172e66b64436805acadbbca827815882.png)

- 然后我们对接口进行访问测试，分别是 1秒1次、1秒2次、1秒3次

![screenshots.gif](http://kodo.yelingfa.top/xyblog/aurora/articles/screenshots202404261839.gif)
![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/4f31371529ad49e1b5b77d7d8930acbd.png)

- 至于报错出现拒绝流量时，我们也可以定义拒绝时的处理方法

```java

    @GetMapping("/sendMsg")
    @SentinelResource(value = "sendMsg", blockHandler = "exceptionHandler")
    public String sendMsg(){
        System.out.println("发送一条消息");
        return "发送一条消息";
    }
    /**
     * 异常处理程序
     *
     * @param ex ex
     * @return {@link String}
     */
    public String exceptionHandler(BlockException ex) {
        return "系统繁忙，请稍后再试，自定义限流处理";
    }

```

![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/a36387b20d6b581613cc12e3590df227.png)

- 可对流控规则设置关联，如图，我对sendMsg设置流控，流控模式选择关联，关联资源我们选择另一个接口。效果就是关联的接口达到阈值时，再访问sendMsg就会失败！

![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/7a5f17aca0e6cd2d2a24ac2c54afcc0e.png)

![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/screenshots202404262102.gif)

#### 3.2.2、熔断规则

```java

@SentinelResource(value = "clear",fallback = "fallbackHandler")
    @GetMapping("/clear")
    public String clear(Integer id){
        System.out.println("清空消息");
        if (id == 2){
            System.out.println("异常");
            throw new BizException("清空消息异常");
        }
        return "清空消息";

    }

    /**
     * 异常处理程序
     *
     * @param ex ex
     * @return {@link String}
     */
    public String fallbackHandler(Integer id,Throwable ex) {
        return "系统繁忙，请稍后再试，自定义降级处理";
    }

```

- 点击熔断规则，对资源设置熔断规则，我选择的是最容易出现的规则就是异常数，当一个接口在 1 秒内，最少有 1 个请求，且异常数量出现大于 2 次后，也就是 3 次，触发熔断，熔断时间为 10 秒

![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/41cbc7528dbc5d37b644ba868816a89e.png)

- 以下进行演示

![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/screenshots202404262225.gif)
