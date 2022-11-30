---
title: Dubbo
date: 2022-11-30 19:18:10
tags:
---

# Dubbo学习

https://blog.csdn.net/qq_41157588/article/details/106737191

## 概述

### 分布式系统

分布式系统：建立在网络之上的软件系统

分布式系统是若干独立计算机（服务器）的集合，这些计算机对于用户来说就像单个系统

Dubbo就是用于治理这些分布式系统的系统

### 系统的发展

![image-20220603120448326](https://img-blog.csdnimg.cn/img_convert/7742ff161f087c849eb5023dd0168333.png)

#### 单一应用架构（1~10）

[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-yMwAjait-1657813726361)(https://s2.loli.net/2022/06/03/CwsRTEJD28MtKHm.png)]

一开始流量很小，整个应用可以设计为一个单节点应用，部署到一个服务器上，开发部署都很简单，如果流量变大了，可以增加服务器的个数来应对增大的并发量

缺点：

1. 难以扩展
2. 系统开发困难，多个人开发一个应用容易把应用改乱
3. 每次修改完都需要重新打包部署到所有服务器上，后面应用的jar包越来越大，单台服务器运行这个jar包都会比较吃力，靠增加结点也难有性能上的提升

### 垂直应用架构（10~1000）

![image-20220603120341997](https://img-blog.csdnimg.cn/img_convert/e6021eccf8f18d874f5ca87748326208.png)

把功能分解若干小服务，部署到不同的服务器上，每一个小服务都包含有界面，服务，数据访问，彼此之间相互独立

优点：

1. 协同开发变得容易，同事之间可以开发不同的小服务，互不干扰
2. 扩展容易，可以精确扩展我们想要的服务（为某个小服务添加服务器）

缺点：

1. 界面和业务逻辑需要分开（界面美化会比较频繁）
2. 应用和应用之间不可能完全独立，应用之间往往会有交互

### 分布式服务架构

![image-20220603120532659](https://img-blog.csdnimg.cn/img_convert/875c4c307e62fdbb9be2c8aeb9e75eb0.png)

将核心业务逻辑和界面分离出来（前后端分离），这样修改界面的时候，不需要重新部署业务逻辑相关的代码

调用服务的方式是RPC（远程过程调用）

分布式服务框架也还没有达到最完美的程度，当我们的服务越来越多，需要的服务器也就越来越多，服务器资源的浪费就十分严重，如果我们能动态监控各个服务的运行情况，根据服务的访问量来动态调度服务器资源，不就能解决资源问题，让服务器资源能得到充分利用了吗，正所谓好要用到刀刃上，问解决这个问题就有了流动计算架构

### 流动计算架构

![image-20220603121613225](https://img-blog.csdnimg.cn/img_convert/9303ea4e14355fa4bab4367a8442ed3b.png)

流动计算架构可以在上述基础上设置调度和治理中心，基于访问容量试试管理集群容量，提高集群利用率

### RPC

远程过程调用，进程通信的一种方式

![image-20220603122105603](https://img-blog.csdnimg.cn/img_convert/58ef0a02348a638808d633247f68ef36.png)

其实就是应用程序调用应用层协议（比如Http）和运输层协议（比如TCP）和其他使用相同协议的应用程序进行交互的过程

举个例子，我们进行RPC的时候，大概需要经过这些流程

[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-DnYK5ohC-1657813726364)(https://s2.loli.net/2022/06/03/Ku7cFkAEjeXBpxg.png)]

客户端发起调用命令，将参数进行序列化，传递给服务器，服务器将参数反序列化后，调用服务，将返回值序列化后返回给客户端

影响一个RPC框架性能的因素主要是连接建立的速度和序列化和反序列化的速度（XML<JOSN<二进制流）

## Dubbob基础

Dubbo是Java的高性能RPC框架

![image-20220603132516618](https://img-blog.csdnimg.cn/img_convert/8fd4f7a7f3e883cc1e9a66e0c4ba9e13.png)

- 面向接口代理，调用起来更加方便
- 负载均衡，一个服务在部署到多个服务器上时，让每个服务器的访问量更可能地均衡
- 服务注册和发现，监控服务的状态（运行还是宕机），统计服务部署到了哪些服务器上，完成这个工作的组件叫做注册中心
- 给第三方提供扩展的接口
- 运行流量调度，让不同的服务器承载不同 的流量。同时我们可以让开发后的新服务部署到一部分服务器上，另一些服务器依然使用老服务（新服务可能不稳定，可以测试新服务的表现），经过一段时间的测试，没有问题后再将新服务部署到所有的服务器上（灰度发布）
- 通过一个可视化的web界面来监控集群的运行状态

### Dobbo应用的架构

![image-20220603133928661](https://img-blog.csdnimg.cn/img_convert/b50cfb5974e30d0e9956bb06837c80ba.png)

服务提供者运行在Dubbo容器上提供服务，并将服务注册到注册中心里面，服务消费者从注册中心里拉取服务列表，并调用里面的服务，服务消费者和服务提供者定期给monitor发送自己的信息，告诉自己的存活状态

### 注册中心Zookeeper

Dubbo软件架构需要注册中心，官网推荐使用zookeeper

![image-20220603145517827](https://img-blog.csdnimg.cn/img_convert/04e3606cd58b1f4f35531617f697e0d6.png)

zookeeper是树形的目录结构

windows下直接启动或报错，报错原因是没有找到配置文件，所以我们需要重命名他的配置文件

![image-20220603150129725](https://img-blog.csdnimg.cn/img_convert/96ce22a7077e3b9b9dc1d2dc06004f0e.png)

然后通过bin目录下的zkServer.cmd启动服务，服务端口是2181，zkCli.cmd打开客户端

### 监控中心Monitor

监控中心可以为我们提供可视化的监控界面，这里是从github上下载的incubator-dubbo-ops-master用于监控dubbo，本质上是一个springboot项目，把它打成jar包dubbo-admin-0.0.1-SNAPSHOT.jar，然后运行这个jar包就能运行管理控制台，端口号是7001,账号密码都是root

### 入门项目

服务消费者调用订单服务，服务提供者提供订单服务

[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-W8IZ0XGo-1657813726367)(https://s2.loli.net/2022/06/03/K21GB3HEoXpzr6I.png)]

在分布式架构中，所以的模块会有一些公共的依赖文件，比如实体bean等，如果采用直接复制的方式，如果bean需要增加一些字段，所有的项目都需要更改，所以我们可以把这些公共的部分抽离出来，单独放在一个项目中供其他项目使用即可，这样既可以减少代码量，修改起来也会很方便

实体bean：

前面提到过，在传递参数的时候需要将对象进行序列化，所以对象必须实现Serializable接口

```java
public class UserAddress implements Serializable{
    private Integer id;
    private String userAddress; //用户地址
    private String userId; //用户id
    private String consignee; //收货人
    private String phoneNum; //电话号码
    private String isDefault; //是否为默认地址    Y-是     N-否
    //get     set 
    //有参构造  无参构造
  }
```

服务提供者：

服务接口

```java
//用户服务
public interface UserService {
	/**
	 * 按照用户id返回所有的收货地址
	 * @param userId
	 * @return
	 */
	public List<UserAddress> getUserAddressList(String userId);
}
```

接口实现：

```java
public class UserServiceImpl implements UserService {
	public List<UserAddress> getUserAddressList(String userId) {

		UserAddress address1 = new UserAddress(1, "河南省郑州巩义市宋陵大厦2F", "1", "安然", "150360313x", "Y");
		UserAddress address2 = new UserAddress(2, "北京市昌平区沙河镇沙阳路", "1", "情话", "1766666395x", "N");

		return Arrays.asList(address1,address2);
	}
}
```

服务消费者：

调用服务的接口

```java
public interface OrderService {
    /**
     * 初始化订单
     * @param userID
     */
    public void initOrder(String userID);
}

```

接口实现：

这里的UserService的实现并不在这个项目中，只是一个接口，所以需要我们使用RPC框架

```java
public class OrderServiceImpl implements OrderService {
    public UserService userService;
    public void initOrder(String userID) {
        //查询用户的收货地址
        List<UserAddress> userAddressList = userService.getUserAddressList(userID);
        System.out.println(userAddressList);
    }
}

```

我们可以看到这里的接口（UserService）和实体Bean（User）是所有项目通用的，我们把他们抽离出来放到一个新的项目里面，供其他项目引入

```java
		<dependency>
			<groupId>com.atguigu.gmall</groupId>
			<artifactId>gmall-interface</artifactId>
			<version>0.0.1-SNAPSHOT</version>
		</dependency>
```

`<groupId>com.atguigu.gmall</groupId>` 这个是项目名

`<artifactId>gmall-interface</artifactId>`  这个是工程目录

#### 服务提供者dubbo框架配置

引入dubbo和zookeeper

```xml
	 <!--dubbo-->
    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>dubbo</artifactId>
        <version>2.6.2</version>
    </dependency>
    <!--注册中心是 zookeeper，引入zookeeper客户端-->
    <dependency>
        <groupId>org.apache.curator</groupId>
        <artifactId>curator-framework</artifactId>
        <version>2.12.0</version>
    </dependency>

```

编写xml配置类，将当前服务注册进dubbo框架中

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
		http://dubbo.apache.org/schema/dubbo http://dubbo.apache.org/schema/dubbo/dubbo.xsd
		http://code.alibabatech.com/schema/dubbo http://code.alibabatech.com/schema/dubbo/dubbo.xsd">
    <!--1、指定当前服务/应用的名字(同样的服务名字相同，不要和别的服务同名)-->
   <dubbo:application name="user-service-provider"></dubbo:application>
    <!--2、指定注册中心的位置-->
    <!--<dubbo:registry address="zookeeper://127.0.0.1:2181"></dubbo:registry>-->
    <dubbo:registry protocol="zookeeper" address="127.0.0.1:2181"></dubbo:registry>
    <!--3、指定通信规则（通信协议? 服务端口）-->
    <dubbo:protocol name="dubbo" port="20880"></dubbo:protocol>
    <!--4、暴露服务 让别人调用 ref指向服务的真正实现对象-->
    <dubbo:service interface="com.lemon.gmail.service.UserService" ref="userServiceImpl"></dubbo:service>

    <!--服务的实现-->
    <bean id="userServiceImpl" class="com.lemon.gmail.service.impl.UserServiceImpl"></bean>
</beans>

```

启动spring容器：

```java
public class MailApplication {
    public static void main(String[] args) throws IOException {
        ClassPathXmlApplicationContext applicationContext= new ClassPathXmlApplicationContext("provider.xml");
        applicationContext.start();
        System.in.read();
    }
}
```

System.in.read();是为了阻塞住当前程序，防止程序退出，达到任意键退出的效果

#### 服务消费者dubbo框架配置

还是引入这两个中间件

```xml
  <!--dubbo-->
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>dubbo</artifactId>
            <version>2.6.2</version>
        </dependency>
        <!--注册中心是 zookeeper，引入zookeeper客户端-->
        <dependency>
            <groupId>org.apache.curator</groupId>
            <artifactId>curator-framework</artifactId>
            <version>2.12.0</version>
        </dependency>

```

编写xml配置文件，从注册中心拉取服务并将远程的组件放入当前项目的spring容器中

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://dubbo.apache.org/schema/dubbo"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
		http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-4.3.xsd
		http://dubbo.apache.org/schema/dubbo http://dubbo.apache.org/schema/dubbo/dubbo.xsd
		http://code.alibabatech.com/schema/dubbo http://code.alibabatech.com/schema/dubbo/dubbo.xsd">
   <!--包扫描-->
    <context:component-scan base-package="com.lemon.gmail.service.impl"/>

    <!--指定当前服务/应用的名字(同样的服务名字相同，不要和别的服务同名)-->
    <dubbo:application name="order-service-consumer"></dubbo:application>
    <!--指定注册中心的位置-->
    <dubbo:registry address="zookeeper://127.0.0.1:2181"></dubbo:registry>

    <!--调用远程暴露的服务，生成远程服务代理-->
    <dubbo:reference interface="com.lemon.gmail.service.UserService" id="userService"></dubbo:reference>
</beans>

```

使用的时候就可以直接用@Autowire注入，就好像这个组件就在本地的spring容器中

```java
@Service
public class OrderServiceImpl implements OrderService {
    @Autowired
    public UserService userService;
    public void initOrder(String userID) {
        //查询用户的收货地址
        List<UserAddress> userAddressList = userService.getUserAddressList(userID);
        
        //为了直观的看到得到的数据，以下内容也可不写
        System.out.println("当前接收到的userId=> "+userID);
        System.out.println("**********");
        System.out.println("查询到的所有地址为：");
        for (UserAddress userAddress : userAddressList) {
            //打印远程服务地址的信息
            System.out.println(userAddress.getUserAddress());
        }
        
    }
}
```

### 简易监控中心

将 dubbo-monitor-simple-2.0.0-assembly.tar.gz 压缩包解压至当前文件夹，解压后config文件查看properties的配置是否是本地的zookeeper，如果不是本地的zookeeper则需要修改ip和端口
打开解压后的 assembly.bin 文件，start.bat 启动dubbo-monitor-simple监控中心
在浏览器 localhost:8080 ，可以看到一个监控中心。
在服务提供者和消费者的xml中配置以下内容，再次启动服务提供和消费者启动类。

    <!--dubbo-monitor-simple监控中心发现的配置-->
    <dubbo:monitor protocol="registry"></dubbo:monitor>
    <!--<dubbo:monitor address="127.0.0.1:7070"></dubbo:monitor>-->

### Springboot整合Dubbo

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200612152443622.png)

导入pom依赖

```xml
	<parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.0.4.RELEASE</version>
        <relativePath />
    </parent>

    <dependencies>
        <dependency>
            <groupId>com.lemon.gmail</groupId>
            <artifactId>gmail-interface</artifactId>
            <version>1.0-SNAPSHOT</version>
        </dependency>
<!-- 这个starter帮我们引入了springboot,duboo,zookeeper相关的依赖-->
        <dependency>
            <groupId>com.alibaba.boot</groupId>
            <artifactId>dubbo-spring-boot-starter</artifactId>
            <version>0.2.0</version>
        </dependency>
    </dependencies>
```

<relativePath />用于获取父工程的版本信息<dependencyManagement>

#### 服务提供者

这个用来提供服务的实现类，一个一个在配置文件中配置端口显然过于麻烦，我们可以使用Dubbo家的@Service注解，而Spring的@Service可以使用@Component注解代替

```java
@Service//dubbo的服务暴露
@Component
public class UserServiceImpl implements UserService {
	public List<UserAddress> getUserAddressList(String userId) {

		UserAddress address1 = new UserAddress(1, "河南省郑州巩义市宋陵大厦2F", "1", "安然", "150360313x", "Y");
		UserAddress address2 = new UserAddress(2, "北京市昌平区沙河镇沙阳路", "1", "情话", "1766666395x", "N");

		return Arrays.asList(address1,address2);
	}
}
```

properties配置文件：

```properties
#端口需要和8080错开
server.port=8082
#服务名称
dubbo.application.name=boot-user-service-provider
#注册中心的地址
dubbo.registry.address=127.0.0.1:2181
#注册中心的类型
dubbo.registry.protocol=zookeeper
#协议名称
dubbo.protocol.name=dubbo
#对外暴露的端口号
dubbo.protocol.port=20880

#连接监控中心
dubbo.monitor.protocol=registry
```

启动springboot

```java
@EnableDubbo //开启基于注解的dubbo功能
@SpringBootApplication
public class BootProviderApplication {
    public static void main(String[] args) {
        SpringApplication.run(BootProviderApplication.class, args);
    }
}
```

#### 服务消费者

服务消费者所需的pom依赖和服务提供者相同

调用服务的方式也类似

```java
@Service
public class OrderServiceImpl implements OrderService {

    @Reference//引用远程提供者服务
    UserService userService;

    public List<UserAddress> initOrder(String userID) {
        //查询用户的收货地址
        List<UserAddress> userAddressList = userService.getUserAddressList(userID);

        System.out.println("当前接收到的userId=> "+userID);
        System.out.println("**********");
        System.out.println("查询到的所有地址为：");
        for (UserAddress userAddress : userAddressList) {
            //打印远程服务地址的信息
            System.out.println(userAddress.getUserAddress());
        }
        return userAddressList;
    }
}
```

只是这里不再使用@Autowired注解，而是使用Dubbo家的@Reference

用于测试的controller

```java
@Controller
public class OrderController {
    @Autowired
    OrderService orderService;

    @RequestMapping("/initOrder")
    @ResponseBody
    public List<UserAddress> initOrder(@RequestParam("uid")String userId) {
        return orderService.initOrder(userId);
    }
}
```

配置文件：

检测中心的端口号是8080，所以这里的端口需要和区分

```properties
server.port=8081
dubbo.application.name=boot-order-service-consumer
dubbo.registry.address=zookeeper://127.0.0.1:2181

#连接监控中心 注册中心协议
dubbo.monitor.protocol=registry
```

dubbo.registry.address=zookeeper://127.0.0.1:2181直接使用这个写法就不要设置protocol

编写启动类

也要加上@EnableDubbo注解，表示支持使用dubbo的注解

```java
@EnableDubbo //开启基于注解的dubbo功能
@SpringBootApplication
public class BootConsumerApplication {
    public static void main(String[] args){
        SpringApplication.run(BootConsumerApplication.class,args);
    }
}
```

服务消费者应当在服务提供者启动之后再启动，否则会启动报错（下面可以配置这个参数）

我们可以看到服务消费者在Controller中像是调用本地的service一样调用UserService（感觉甚至比Feign还要好用）

Feign是根据服务名称来进行RPC

Dubbo直接根据Service的Bean的类型来进行RPC

### 常用参数配置

[注解配置 | Apache Dubbo](https://dubbo.apache.org/zh/docs/v2.7/user/configuration/annotation/)

配置参数编写规则：

| Config Type     | 单数配置                                                     | 复数配置                                                     |
| --------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| application     | dubbo.application.xxx=xxx                                    | dubbo.applications.{id}.xxx=xxx dubbo.applications.{name}.xxx=xxx |
| protocol        | dubbo.protocol.xxx=xxx                                       | dubbo.protocols.{id}.xxx=xxx dubbo.protocols.{name}.xxx=xxx  |
| module          | dubbo.module.xxx=xxx                                         | dubbo.modules.{id}.xxx=xxx dubbo.modules.{name}.xxx=xxx      |
| registry        | dubbo.registry.xxx=xxx                                       | dubbo.registries.{id}.xxx=xxx                                |
| monitor         | dubbo.monitor.xxx=xxx                                        | dubbo.monitors.{id}.xxx=xxx                                  |
| config-center   | dubbo.config-center.xxx=xxx                                  | dubbo.config-centers.{id}.xxx=xxx                            |
| metadata-report | dubbo.metadata-report.xxx=xxx                                | dubbo.metadata-reports.{id}.xxx=xxx                          |
| ssl             | dubbo.ssl.xxx=xxx                                            | dubbo.ssls.{id}.xxx=xxx                                      |
| metrics         | dubbo.metrics.xxx=xxx                                        | dubbo.metricses.{id}.xxx=xxx                                 |
| provider        | dubbo.provider.xxx=xxx                                       | dubbo.providers.{id}.xxx=xxx                                 |
| consumer        | dubbo.consumer.xxx=xxx                                       | dubbo.consumers.{id}.xxx=xxx                                 |
| service         | dubbo.service.{interfaceName}.xxx=xxx                        | 无                                                           |
| reference       | dubbo.reference.{interfaceName}.xxx=xxx                      | 无                                                           |
| method          | dubbo.service.{interfaceName}.{methodName}.xxx=xxx dubbo.reference.{interfaceName}.{methodName}.xxx=xxx | 无                                                           |
| argument        | dubbo.service.{interfaceName}.{methodName}.{arg-index}.xxx=xxx | 无                                                           |

常用配置项

| 属性        | 对应URL参数         | 类型           | 缺省值                 | 描述                                                         | 兼容性                     |
| ----------- | ------------------- | -------------- | ---------------------- | ------------------------------------------------------------ | -------------------------- |
| timeout     | default.timeout     | int            | 1000                   | 远程服务调用超时时间(毫秒)                                   | 1.0.16以上版本             |
| retries     | default.retries     | int            | 2                      | 远程服务调用重试次数，不包括第一次调用，不需要重试请设为0,仅在cluster为failback/failover时有效 | 1.0.16以上版本             |
| loadbalance | default.loadbalance | string         | random                 | 负载均衡策略，可选值：random,roundrobin,leastactive，分别表示：随机，轮询，最少活跃调用 | 1.0.16以上版本             |
| async       | default.async       | boolean        | false                  | 是否缺省异步执行，不可靠异步，只是忽略返回值，不阻塞执行线程 | 2.0.0以上版本              |
| connections | default.connections | int            | 100                    | 每个服务对每个提供者的最大连接数，rmi、http、hessian等短连接协议支持此配置，dubbo协议长连接不支持此配置 | 1.0.16以上版本             |
| generic     | generic             | boolean        | false                  | 是否缺省泛化接口，如果为泛化接口，将返回GenericService       | 2.0.0以上版本              |
| check       | check               | boolean        | true                   | 启动时检查提供者是否存在，true报错，false忽略                | 1.0.16以上版本             |
| proxy       | proxy               | string         | javassist              | 生成动态代理方式，可选：jdk/javassist                        | 2.0.5以上版本              |
| owner       | owner               | string         |                        | 调用服务负责人，用于服务治理，请填写负责人公司邮箱前缀       | 2.0.5以上版本              |
| actives     | default.actives     | int            | 0                      | 每服务消费者每服务每方法最大并发调用数                       | 2.0.5以上版本              |
| cluster     | default.cluster     | string         | failover               | 集群方式，可选：failover/failfast/failsafe/failback/forking  | 2.0.5以上版本              |
| filter      | reference.filter    | string         |                        | 服务消费方远程调用过程拦截器名称，多个名称用逗号分隔         | 2.0.5以上版本              |
| listener    | invoker.listener    | string         |                        | 服务消费方引用服务监听器名称，多个名称用逗号分隔             | 2.0.5以上版本              |
| registry    |                     | string         | 缺省向所有registry注册 | 向指定注册中心注册，在多个注册中心时使用，值为<dubbo:registry>的id属性，多个注册中心ID用逗号分隔，如果不想将该服务注册到任何registry，可将值设为N/A | 2.0.5以上版本              |
| layer       | layer               | string         |                        | 服务调用者所在的分层。如：biz、dao、intl:web、china:acton。  | 2.0.7以上版本              |
| init        | init                | boolean        | false                  | 是否在afterPropertiesSet()时饥饿初始化引用，否则等到有人注入或引用该实例时再初始化。 | 2.0.10以上版本             |
| cache       | cache               | string/boolean |                        | 以调用参数为key，缓存返回结果，可选：lru, threadlocal, jcache等 | Dubbo2.1.0及其以上版本支持 |
| validation  | validation          | boolean        |                        | 是否启用JSR303标准注解验证，如果启用，将对方法参数上的注解进行校验 | Dubbo2.1.0及其以上版本支持 |

配置文件的优先级

虚拟机参数-D > application.properties > dubbo.properties

#### 启动时检查（check）

![image-20220603222025753](https://img-blog.csdnimg.cn/img_convert/fd04b53d22852ee0083b5349b5b94156.png)

dubbo.consumer对所有的接口进行统一配置

dubbo.consumer.check=false

在启动的时候会检查依赖的bean有没有注册要注册中心里面，如果没有则检查失败，启动报错。

如果出现了循环依赖需要将这个设置为false。这样在用到这个bean的时候才会检查是否存在

这种方式是配置所有的接口，我们也可以精确配置某一个接口，或者配置注册中心是否要检查

![image-20220603222954099](https://img-blog.csdnimg.cn/img_convert/ab96410ed357df554514611a90b06352.png)

#### 超时设置（timeout）

RPC的默认超时时间是1s，防止大量线程阻塞，导致资源耗尽

xml配置

全局配置：

```
 <dubbo:consumer check="false"></dubbo:consumer>
```

精确配置：

![image-20220603223908515](https://img-blog.csdnimg.cn/img_convert/e1db821cf9e25b0abd009bcf5db669ca.png)

精确优先，精确度相同时，消费者配置大于提供者配置

配置文件配置：

```
dubbo.consumer.timeout=5000
```

#### 重试次数（retries）

在服务调用失败的时候，重试的次数（第一次不算），配置方法和超时一样

幂等操作可以设置重试次数，非幂等操作不能设置重试次数

不重试设置为0即可

在上述的dubbo:consumer或者dubbo:reference设置参数即可

#### 多版本

放入注册中心的组件发送变化时，组件的版本也就发送了变化

可以用version设置bean的版本号，通过修改版本号来执行使用哪个版本的bean，设置为*时表示随机使用一个版本（这也表示依赖注入的时候，使用的是按照类型注入）

#### 本地存根对象（stub）

可以用sub设置本地存根对象，可以设置一个类，里面传入一个UserService类型的参数作为构造器,构造器必须是public

```java
public class UserServiceStub implements UserService{

    UserService userService;

    public UserServiceStub(UserService userService){
        super();
        this.userService=userService;
    }

    @Override
    public List<User> selectUsersById(Integer id) {
        System.out.println("进来");
        return userService.selectUsersById(id);
    }
}
```

相当于对dubbo容器中的远程对象进行动态代理，构造器中传入的userService对象是远程的代理对象，指定一个bean的sub为UserServiceSub后，或调用UserServiceSub中的方法，然后在UserServiceSub类的方法中调用远程对象的方法，进行RPC，相当于进行了动态代理

使用时指定stub类：

```java
@RestController
public class UserController {

    @Reference(stub = "com.lth.service.UserServiceStub")
    UserService userService;

    @GetMapping("/hello")
    public Object getUsersById(@RequestParam("id") Integer id){
        return userService.selectUsersById(id);
    }
}

```

### Dubbo和Springboot整合的配置

![image-20220603233246841](https://img-blog.csdnimg.cn/img_convert/8e1bfff67d8e682a9f8c1cde13d8f3db.png)

上面讲述的都是在xml中进行配置的方法，而在和springboot整合的时候也是完全适用的

消费者的全局配置参数在application.properties中配置

而精确配置在注解@Reference的参数中

服务提供者的全局配置也是在application.properties中配置

而精确配置在@Service的参数中

整合的三种方式：

1. 导入dubbo的starter，在application.properties中设置配置参数，用@Service暴露接口，用@Reference调用接口（并配置参数），并加上@EnableDubbo来扫描带有dubbo注解的类，如果在properties中设置包扫描路径则就不需要@EnableDubbo注解
2. 保留dubbo的xml配置文件，在spring的配置类中用@ImportResource("/provider.xml")来加载配置文件，也能达到相同的效果，这样就可以代理注解（显然注解更方便）
3. 每一个xml元素，properties配置项在spring容器中都会对应一个配置类，我们也可以将需要的配置类（@Configuration）注入到spring容器中，这样也可以完成我们想要的功能，用xxxConfig类的对象代替xml标签中的xxx

第三种方式我们可以举个例子

```java
@Configuration
public class MyDubboConfig {
	
	@Bean
	public ApplicationConfig applicationConfig() {
		ApplicationConfig applicationConfig = new ApplicationConfig();
		applicationConfig.setName("boot-user-service-provider");
		return applicationConfig;
	}
	
	//<dubbo:registry protocol="zookeeper" address="127.0.0.1:2181"></dubbo:registry>
	@Bean
	public RegistryConfig registryConfig() {
		RegistryConfig registryConfig = new RegistryConfig();
		registryConfig.setProtocol("zookeeper");
		registryConfig.setAddress("127.0.0.1:2181");
		return registryConfig;
	}
	
	//<dubbo:protocol name="dubbo" port="20882"></dubbo:protocol>
	@Bean
	public ProtocolConfig protocolConfig() {
		ProtocolConfig protocolConfig = new ProtocolConfig();
		protocolConfig.setName("dubbo");
		protocolConfig.setPort(20882);
		return protocolConfig;
	}
	
	/**
	 *<dubbo:service interface="com.atguigu.gmall.service.UserService" 
		ref="userServiceImpl01" timeout="1000" version="1.0.0">
		<dubbo:method name="getUserAddressList" timeout="1000"></dubbo:method>
	</dubbo:service>
	 */
	@Bean
	public ServiceConfig<UserService> userServiceConfig(UserService userService){
		ServiceConfig<UserService> serviceConfig = new ServiceConfig<>();
		serviceConfig.setInterface(UserService.class);
		serviceConfig.setRef(userService);
		serviceConfig.setVersion("1.0.0");
		
		//配置每一个method的信息
		MethodConfig methodConfig = new MethodConfig();
		methodConfig.setName("getUserAddressList");
		methodConfig.setTimeout(1000);
		
		//将method的设置关联到service配置中
		List<MethodConfig> methods = new ArrayList<>();
		methods.add(methodConfig);
		serviceConfig.setMethods(methods);
		
		//ProviderConfig
		//MonitorConfig
		
		return serviceConfig;
	}

}
```

## 高可用Dubbo

### Zookeeper宕机和Dubbo直连

![image-20220604120243072](https://img-blog.csdnimg.cn/img_convert/f07f308730ad586f1a64184da62b2c34.png)

注册中心起到的是服务注册和发现的功能，保存有各个服务器的Socket（ip和端口），服务器在通讯之前从注册中心中获取想要通讯的服务器的Socket，然后建立起Socket连接进行通讯，如果注册中心宕机了，服务消费者本地会换成原先调用过的服务提供者的Socket，因而不需要再从注册中心拉取服务列表，查询Socket，而可以直接通讯。我们也可以在@Reference注解的url参数直接指定要通讯的Socket（ip和端口），这样就可以绕开注册中心进行直连

### Dubbo负载均衡

#### 基于权重的随机策略

Random LoadBalance

![image-20220604122544120](https://img-blog.csdnimg.cn/img_convert/a57d57a0e13cb73efa588c4272ee8af1.png)

根据权重，设计各个服务的概率，然后随机访问

#### 基于权重的轮询策略

RoundRobin LoadBalance

[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-aGBl0iIr-1657813726371)(https://s2.loli.net/2022/06/04/nHWbga2SvKYuxIU.png)]

基于权重，设计每个周期中服务器的访问顺序

#### 最小活跃数

LeastActive LoadBalance

![image-20220604122851167](https://img-blog.csdnimg.cn/img_convert/ebed4e6829af25a0348af63d3c4cd0b8.png)

每次请求，总是访问上一次请求耗时最短的服务器

#### 一致性Hash

ConsistentHash LoadBalance

![image-20220604122929517](https://img-blog.csdnimg.cn/img_convert/9b45062986f034a01041590543642c46.png)

根据URL和参数计算hash值，根据hash值来决定访问哪个服务器，如果一直访问一个url携带相同的参数则会一直访问同一个服务器

使用负载均衡时需要去掉Dubbo直连，否则会绕过注册中心达不到负载均衡的效果

#### 实际使用

Dubbo默认采用随机策略，内部有这些类对应不同的策略

![image-20220604132555791](https://img-blog.csdnimg.cn/img_convert/ec8797a43571bf74526d4f4d332f4c27.png)

在@Reference注解的loadbalance参数中设置策略（在其余两种方式也有对应的方式比如在`<dubbo:service>`标签中设置这个参数）

可以在@Service标签中设置weight（权重），也可以在dubbo的控制台动态地调整

当一个服务器性能不佳时可以降低其权重来控制访问量

```
    @Reference(loadbalance = "random")
    UserService userService;
```

可以选择的值有random,roundrobin,leastactive

### 服务降级

![image-20220604134744824](https://img-blog.csdnimg.cn/img_convert/41bb9ab61c95cc66232dd6d77e2465d4.png)

在服务器压力很大的时候，我们可以牺牲一下不重要的业务来保证我们的核心业务正常运行

两种策略：

![image-20220604135010734](https://img-blog.csdnimg.cn/img_convert/b2db045aea8214e097eed8fa52a0b7c7.png)

第一种是直接不允许调用，用于不重要的服务不可用的时候

第二种是在调用失败的时候返回空值，用于服务能用但是不稳定的情况，比如超时后，服务就会终止，但如果这不是重要的服务，我们可以让他不抛出异常而是返回空值，这样请求的流程就可以继续进行，完成我们的核心业务

我们可以直接在web控制台来设置策略[Dubbo Admin](http://localhost:7001/)

[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-8UbxFDZf-1657813726374)(C:/Users/%E9%BB%8E%E6%98%8E%E7%BB%88%E7%82%B9x/AppData/Roaming/Typora/typora-user-images/image-20220604135749535.png)]

这样运维起来据很方便

屏蔽：调用时直接返回空

容错：调用失败返回空

静止：调用时会直接抛出异常

### 服务容错方式

可以在cluster属性设置容错策略

#### 自动切换（failover）

当出现请求失败时，重试其他服务器，默认的失败策略，可以用retries设置失败次数，通常用于读操作

![image-20220604152349998](https://img-blog.csdnimg.cn/img_convert/43c9a0b31535f7049ddc351bfff48df9.png)

#### 快速失败 failfast

只请求一次，失败后立即报错，通常用于写操作

#### 失败安全 failSafe

失败后直接忽略，通常用于写日志

#### 失败重试 fallback

失败后定时重试，通常用于消息通知（拉取消息）

#### 并行调度 forking

一次请求多个服务器，使用forks来设置并行数，有一个成功就算成功，适用于实时性要求很高的场景

#### 广播调度 broadcast

一次请求各个服务器，有一个失败本次请求就算失败报错，用于提醒所有的服务器更新缓存和日志之类的信息，进行一些同步操作

![image-20220604153523384](https://img-blog.csdnimg.cn/img_convert/f035bec86009c33287dcc6592d01f395.png)

这两种配置其实是等价的，都是设置注册中心中具体的服务

### 使用Hystrix进行服务容错

#### 依赖导入

引入SpringCloud的hystrix进行服务容错

```xml
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
        </dependency>
```

还要加上dependencyManagement用于版本管理

```xml
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>Hoxton.SR1</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
```

在启动类需要加上@EnableHystrix注解，导入Hystrix的组件（服务提供者和消费者都需要加上）

```java
@SpringBootApplication
@EnableDubbo
@EnableHystrix
public class ServiceProviderApplication {
    public static void main(String[] args) {
        SpringApplication.run(ServiceProviderApplication.class,args);
    }
}
```

#### 服务提供者

可以存在服务提供者进行容错

在需要被服务容错的方法上加上@HystrixCommand即可，加上注解后这个方法就会被hystrix代理

```java
@Component
@Service
public class UserServiceImpl implements UserService {

    @SneakyThrows
    @Override
    @HystrixCommand(fallbackMethod = "selectUsersByIdFallback")
    public List<User> selectUsersById(Integer id) {
        System.out.println("进来了………………");
        if(Math.random()<0.5) {
            throw new RuntimeException("???");
        }
        return Arrays.asList(
                new User("lth","2001-8-28","hubei",20),
                new User("zech","2002-8-8","hubei",20)
        );
    }
    public List<User> selectUsersByIdFallback(Integer id){
        return Arrays.asList(
                new User()
        );
    }
}
```

#### 服务消费者

也可以在服务消费者进行容错

在需要服务容错的方法上@HystrixCommand，表示被hystrix代理，同时可以设置fallbackMethod回调方法在方法出现时自动调用我们设置的方法（其实就相当于try-catch）

```java
@Component
public class UserServiceImpl implements UserService{

    @Reference
    UserService userService;

    @HystrixCommand(fallbackMethod = "hello")
    @Override
    public List<User> selectUsersById(Integer id) {
        System.out.println("正常执行");
        return userService.selectUsersById(id);
    }
    public List<User> hello(Integer id) {
        System.out.println("出错啦");
        return Arrays.asList(
                new User()
        );
    }
}
```

@HystrixCommand其实就算在服务错误时自动使用回调方法，和try-catch的作用类型，服务提供者和服务消费者并不都需要加上@HystrixCommand注解，在需要出错回调的时候加上这个注解即可，相当于进行了动态代理，并不需要区分服务提供者和服务消费者，在使用这个注解的时候需要在启动类开启@EnableHystrix，这样注解才会生效

并且设置的回调方法，除了方法名以外，要和原来的方法完全一样（参数，返回值），否则会启动报错

带有Spring注解的方法也能进行服务容错

```java
@RestController
public class UserController {

    @Autowired
    UserService userService;

    @GetMapping("/hello")
    @HystrixCommand(fallbackMethod = "getUsersByIdFallback")
    public Object getUsersById(@RequestParam("id") Integer id){
        if(new Random().nextInt(2)==0){
            throw new RuntimeException("小错误");
        }
        return userService.selectUsersById(id);
    }
    public Object getUsersByIdFallback(Integer id){
        System.out.println("出错啦");
        return "userService.selectUsersById(id)";
    }
}

```

在调用目标方法前，将方法的参数保存下来，在调用回调方法时，会根据方法名，反射找到对应的方法，然后将之前保存的参数，按照原来的顺序传递进去，然后执行。其实就是动态代理，用aop很容易实现类似的功能。

## 原理解析

### RPC原理

RPC框架的通用原理

![image-20220604185255544](https://img-blog.csdnimg.cn/img_convert/45aa41e6829fe06cbc413fc5d1294c7c.png)

![image-20220604185555662](https://img-blog.csdnimg.cn/img_convert/3a63b2fd3d3b3dd6445db5de464df198.png)

#### 通信原理：Netty

Netty框架极大简化了TCP和UDP网络编程的难度

BIO：阻塞IO，在IO完成之前，线程不会释放，等待IO完成后才能释放

NIO：非阻塞IO

![image-20220604190051812](https://img-blog.csdnimg.cn/img_convert/90dff32874e123b86a0b9670cf67af11.png)

监听各个IO事件，等某个IO事件完成后，再去通知服务器开启线程来处理

[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-ixrphDId-1657813726376)(C:/Users/%E9%BB%8E%E6%98%8E%E7%BB%88%E7%82%B9x/AppData/Roaming/Typora/typora-user-images/image-20220604190215009.png)]

Accept表示通道准备就绪

通道连接完成后需要进行一些准备，准备完成后就进入了Accept状态

这里的Channel在Linux中叫做文件句柄fd，在java中叫channel通道，都指的是的IO事件，包括本地磁盘IO和网络IO，都是以输入输出流进行交互

Netty原理

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200613171217369.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQxMTU3NTg4,size_16,color_FFFFFF,t_70)

Netty主线程组会监听我们指定的Socket（比如Dubbo服务提供者暴露给外部的端口）（每个主线线程监听一个Socket），当有请求发过来时，会先建立TCP连接，然后进入Connect状态，连接建立后需要框架进行一些处理，比如对IO流进行一些封装和初始化操作，这些操作都完成后向主线程发送一个Accept事件，主线程使用多路复用器来监听Accept事件（有请求过来，channel初始化完成后会产生accept事件），当有读事件和写事件出现时，主线程不会阻塞住，而是将其放入一个队列中，然后继续往下处理，直到这个被读的变量被使用才会阻塞住，等待IO事件完成，这也就是异步IO。完成IO事件的是工作线程，出现这些读写事件时，主线程会将channel（需要读写的IO流）和需要处理的事件放入对应Channel的等待队列中，这个队列由多路复用器来维护。当指定channel准备好后，完成IO事件，然后就可以关闭channel，主线程继续向下执行，遇到新的IO事件后继续由工作线程来处理。主线程用于完成请求，工作线程用于完成请求中的IO操作。

### Dubbo原理

#### 框架原理

![image-20220604205446611](https://img-blog.csdnimg.cn/img_convert/8565f8f4bc57a0f8d3f8058d9cee52a4.png)

##### 第一层 业务逻辑层

因为是面向接口编程，我们是需要编写公共的接口，然后编写他的实现类并暴露给Dubbo即可，框架给我们的只有这一层，剩下的都被框架封装了起来

##### 配置层

读取配置文件，设置相关属性

##### 代理层

生成代理对象

##### 注册中心层

为服务消费者订阅注册中心中需要的服务，同时负责服务的注册与发现，和服务调用的复杂均衡

##### 监控层

调用服务时会给监控层发送数据

##### 协议层

完成远程调用RPC

##### 交换层

服务端和客户端交换数据

##### 传输层

Netty框架工作

##### 序列化层

将参数和要调用的服务以及调用结果序列化后返回

每一层再Dubbo中都被写在一个包中，每一层都是单向依赖

### 标签解析

所有的标签（注解）解析器都会实现一个顶层接口：BeanDefinitionParser

Dubbo实现了这个接口得到的自己的解析器DubboBeanDefinitionParser

解析器可以解析xml标签和dubbo注解

不同的类型的标签由不同的解析规则

每一个标签由对应不同的配置类XxxProperties有两个特例

Service->ServiceBean

Reference->ReferenceBean

标签解析器的目的是将标签解析出来，保存在配置类中

### 服务暴露

![image-20220604224638440](https://img-blog.csdnimg.cn/img_convert/e44883fcec0ad3bb40992bf3010e49d2.png)

ServiceBean实现下面这些接口

```
ServiceBean<T> extends ServiceConfig<T> implements InitializingBean, DisposableBean, ApplicationContextAware, ApplicationListener<ContextRefreshedEvent>, BeanNameAware
```

Spring在调用构造器创建Bean示例后会进行一系列初始化方法

#### 保存配置

InitializingBean接口：里面有afterPropertiesSet方法，spring在创建属性赋值完后调用这个方法,回顾一下Bean的初始化流程

```
构造方法
postProcessBeforeInitialization:user
PostConstruct
xxxAware	此时以及完成了依赖注入
afterPropertiesSet	
methodInit
postProcessAfterInitialization:user
使用bean
preDestroy
destroy
methodDestroy
```

```
User被创建，调用了构造器
宠物被创建
拿到Bean的名字：user，此时bean的状态：User(name=null, pet=Pet(name=null, description=null))
调用afterPropertiesSet，此时bean的状态：User(name=null, pet=Pet(name=null, description=null))
postProcessBeforeInitialization：User(name=null, pet=Pet(name=null, description=null))
postProcessAfterInitialization：User(name=null, pet=Pet(name=null, description=null))
```

Dubbo在afterPropertiesSet方法里面将配置文件中的信息保存起来

#### 暴露服务

ApplicationListener<ContextRefreshedEvent>	每当一个事件发生时（Spring容器启动的一个阶段完成后），会调用这个方法，调用这个方法的时候，spring容器中的组件就全部创建完成了，开始用这些组件干一些特定场景的事情

```shell
ServletWebServerInitializedEvent	

ContextRefreshedEvent

ApplicationStartedEvent

AvailabilityChangeEvent

ApplicationReadyEvent

AvailabilityChangeEvent
```

Dubbo也实现了这个接口，在特定事件发生后完成自己想要的功能

Dubbo在这里，如果服务还没有被暴露，并且需要保留暴露服务给Dubbo（此时IOC容器以及初始化完成，组件都加载进了Spring容器）

暴露的方法是将IP，端口拼接成一个url：registery:127.0.0.1:2811然后添加上url参数表示服务的各种信息，比如服务名称，服务实现的接口，实现类等

我们当前使用的是dubbo，所以会进入dubbo协议保留服务，将要保留的执行者，注册中心的对象url传入暴露给注册中心的方法中注册服务，然后封装成一个exporter暴露器，传给交换层

然后交换层调用openServer，开启Netty服务器，监听这个url，这样暴露就完成了，最后会将暴露完成的url和对于执行器的映射关系保存在本地的一个map里面

这样服务消费者调用这个服务时，只需要想办法获取这个服务的url，然后向服务提供者发送请求（dubbo协议），服务提供者收到请求后从map里面找到对应的执行器，然后执行方法，获取返回值即可

### 服务引用

![image-20220605003341776](https://img-blog.csdnimg.cn/img_convert/2b5d7d1815afa07b018f5ecf0fd9f03a.png)

ReferenceBean继承了这些接口

```java
ReferenceBean<T> extends ReferenceConfig<T> implements FactoryBean, ApplicationContextAware, InitializingBean, DisposableBean
```

它是一个工厂Bean

我们回顾一下工厂bean的特点：

```java
public class UserBeanFactory implements FactoryBean<User> {

    @Override
    public User getObject() throws Exception {
        return new User("lth",20);
    }

    @Override
    public Class<?> getObjectType() {
        return User.class;
    }

    @Override
    public boolean isSingleton() {
        return true;
    }
}
```

当视图从Spring容器中获取这个工厂bean的时候，获取到的是它getObject()得到的bean，并且如果isSingleton返回true，第二次调用会直接从spring容器中获取第一次调用getObject()拿到的bean，想拿到工厂bean本身需要在bean的id前加上$

Dubbo也是使用的这个机制

在用@Autowire获取Bean的时候或调用这个工厂bean的getObject方法，方法里面如果没有这个对象则创建这个对象，这个对象本质上是一个代理对象，在设置完一些属性后，将配置的一些参数传入方法中创建代理对象，通过接口进行RPC，想要进行RPC就要使用dubbo协议，先根据url的地址拿到注册中心的信息，然后从注册中心获取查询参数拼接到url里面，然后在注册中心里订阅服务提供者提供的服务（当服务提供者上线对象的服务后，服务消费者就会得到对应的服务的Socket，这样就能通过url调用服务）

获取Dubbo客户端，拿到url地址，调用Netty进行RPC，创建连接，将连接封装成一个invoker返回，并放入一个map里面（本地缓存），代理对象里面直接调用这个invoker来完成功能

### 服务调用的流程

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200613171806762.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQxMTU3NTg4,size_16,color_FFFFFF,t_70)

先来到

```java
public class InvokerInvocationHandler implements InvocationHandler {

    private final Invoker<?> invoker;

    public InvokerInvocationHandler(Invoker<?> handler) {
        this.invoker = handler;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        String methodName = method.getName();
        Class<?>[] parameterTypes = method.getParameterTypes();
        if (method.getDeclaringClass() == Object.class) {
            return method.invoke(invoker, args);
        }
        if ("toString".equals(methodName) && parameterTypes.length == 0) {
            return invoker.toString();
        }
        if ("hashCode".equals(methodName) && parameterTypes.length == 0) {
            return invoker.hashCode();
        }
        if ("equals".equals(methodName) && parameterTypes.length == 1) {
            return invoker.equals(args[0]);
        }
        return invoker.invoke(new RpcInvocation(method, args)).recreate();
    }
}
```

invoker是我们层层封装的执行者，里面含有需要请求的URL和相关的配置信息

然后拿到代理对象（失败重试策略的对象），根据缓存的url连接客户端，将数据和配置参数也收集其他，调用Netty框架发送请求，得到返回结果

## 小结

Dubbo是一个简易高效的RPC框架，采用二进制传输的方式效率很高，可以帮助我们解决分布式系统中的一些问题，但是它的能力还是有限的，SpringCloud为分布式系统系统了一套完整的解决方案，Dubbo可以为学习SpringCoud打下基础。但其实我们个人感觉，使用起来Dubbo要比SpringCloud容易很多，服务调用起来也十分方便就像是一个公共的IOC容器。Dubbo和SpringCloud目前也可以进行融合（毕竟都是spring家的东西嘛），Dubbo最大的优点就算性能好，采用的是Netty框架下的TCP长连接而不是Http协议，这样的好处是RPC的效率很高，但是和其他的一些HTTP协议的框架兼容性可能不太好，而SpringCloud采用的是HTTP协议，并且是将功能作为一个服务，好处是可以和其他的一些基于HTTP框架兼容，并且对分布式系统的各种问题都有完整的解决方案，缺点就是性能欠佳，如果对性能要求高的话，使用Dubbo，如果系统比较复杂，需要强大的运维和错误处理能力用SpringCloud即可。