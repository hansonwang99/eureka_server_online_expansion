
![Profile](https://upload-images.jianshu.io/upload_images/9824247-c5fa3fe2a839cb4f.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> 本文共 1329字，阅读大约需要 6分钟 ！

---

## 概述

业务微服务化以后，我们要求服务高可用，于是我们可以部署多个相同的服务实例，并引入负载均衡机制。而微服务注册中心作为微服务化系统的重要单元，其高可用也是非常必要的，因此在生产中我们可能需要多个微服务注册中心实例来保证服务注册中心的稳定性。本文就以 Eureka微服务注册中心为例，来实践一下如何 **在线扩充** Eureka Server实例来保证 微服务注册中心的高可用性！

>**注：** 本文首发于  [**My Personal Blog**](http://www.codesheep.cn)，欢迎光临 [**小站**](http://www.codesheep.cn)

本文内容脑图如下：

![本文内容脑图](https://upload-images.jianshu.io/upload_images/9824247-bd56f802aa4a7730.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


---

## 原理与实验流程介绍

我们欲模拟如下过程：

- 首先启动一个 eureka-server实例（eureka-server-1）

- 再启动一个 eureka-client实例（eureka-client-1）并注册到 eureka-server-1中

- 接下来我们启动第二个 eureka-server实例：eureka-server-2，并且让已启动的 eureka-server-1 和 eureka-client-1在不重启的情况下来感知到 eureka-server-2的加入

- 同理我们可以在启动第三个 eureka-server实例：eureka-server-3，并且让已启动的 eureka-server-1 、eureka-server-2 和 eureka-client-1 在不重启的情况下来感知到 eureka-server-3的加入

- 更多 eureka-server实例的加入原理完全相同，不再赘述

这样一来我们便实现了微服务注册中心的动态在线扩容！

>而如何才能让已启动的服务能够在不重启的情况下来感知到新的 eureka-server 的加入呢？

为此我们引入 Spring Cloud Config 配置中心 config-server，并将 eureka-server和 eureka-client服务的配置文件由 config-server进行统一管理，这样一来对配置文件的修改如果可以通过某种机制来通知已启动的服务，那么问题便迎刃而解了！

我们给出整个过程的原理图如下：

![实验原理图](https://upload-images.jianshu.io/upload_images/9824247-89cbe8477648438d.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

接下来我们来实践这整个过程！

---

## 基础工程搭建

- **创建一个 config-server工程**

pom中关键依赖如下：

```xml
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-config-server</artifactId>
        </dependency>
```

主类添加相应注解：

```
@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication {
  ...
}
```

bootstrap.yml配置文件如下：

```yml
spring:
  application:
    name: config-server
  cloud:
    config:
      server:
        git:
          uri: https://github.com/hansonwang99/xxxx
          username: xxxx
          password: xxxx
server:
  port: 1115
```

>该 yml配置很重要，其连接了预先准备好的 git仓库，后续 eureka-server 和 eureka-client 的配置文件都是存于该git仓库中的！

- **创建一个 eureka-server工程**

pom中关键依赖如下：

```
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-config</artifactId>
        </dependency>
```

主类添加相应注解：

```
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {
  ...
}

```

bootstrap.yml重要配置如下：

```
spring:
  application:
    name: eureka-server
  cloud:
    config:
      uri: http://localhost:1115
```

>注意该 yml中关于 spring cloud config 的配置同样非常重要，因为此 eureka-server需要从 config-server中拉取配置！

最后我们还需要添加一个 controller便于测试：

```java
@RestController
@RequestMapping("/test")
public class TestController {

    @Autowired
    EurekaClientConfigBean eurekaClientConfigBean;

    @GetMapping("/eureka-service-info")
    public Object getEurekaServerUrl(){
        return eurekaClientConfigBean.getServiceUrl();
    }
}
```
这个 Rest Controller 接口的意图很简单：打印出当前到底有多少个 eureka-server实例在提供服务（从而可以直观的判断出 eureka-server的扩容是否已经成功）

- **创建一个 eureka-client工程**

pom中关键依赖如下：

```xml
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-config</artifactId>
        </dependency>
```

主类添加相应注解：

```java
@SpringBootApplication
@EnableDiscoveryClient
public class EurekaClientApplication {

    public static void main(String[] args) {
        SpringApplication.run(EurekaClientApplication.class, args);
    }
}
```

bootstrap.yml 重要配置如下：

```
spring:
  application:
    name: eureka-client
  cloud:
    config:
      uri: http://localhost:1115
```

>这里基本同上面 eureka-server的配置，不再赘述

同样，我们也在 eureka-client中添加一个 controller便于测试：

```
@RestController
@RequestMapping("/test")
public class TestController {

    @Autowired
    EurekaClientConfigBean eurekaClientConfigBean;

    @GetMapping("/eureka-service-info")
    public Object getEurekaServerUrl(){
        return eurekaClientConfigBean.getServiceUrl();
    }
}
```

>三个工程的创建到此完毕！下来我们来依次启动各个工程

- 首先启动 config-server工程

- 然后用 peer1配置文件来启动 **第一个eureka server**，指令如下：

```
mvn spring-boot:run -Dspring.profiles.active=peer1
```
启动过程的开始打印出如下信息，一目了然：

![用 peer1配置文件来启动第一个eureka server](https://upload-images.jianshu.io/upload_images/9824247-d06fba924c45305f.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当然这个 peer1配置文件实际物理存在于git仓库之上：

![peer1配置文件的配置](https://upload-images.jianshu.io/upload_images/9824247-6c30e806fe6e6740.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


- 接下来我们启动 eureka_client工程，指令如下：

```
mvn spring-boot:run -Dspring.profiles.active=peer1
```
其依然需要通过 spring cloud config 去git仓库拉取 eureka-client的 peer1配置文件，这一点从 eureka-client的启动过程中也能看到：

![启动eureka-client](https://upload-images.jianshu.io/upload_images/9824247-5673258382d70796.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 接下来我们用浏览器访问仅存在的一个 eureka-server微服务注册中心，可以看到有服务已经注册上去了：

![微服务注册中心](https://upload-images.jianshu.io/upload_images/9824247-ac311a8f61480779.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

既然 config-server / eureka-server / eureka-client 三个服务已经启动了，那么接下来我们来测试一下：

浏览器访问 eureka-client的 Rest接口：`localhost:1114/test/eureka-service-info`：

![访问 eureka-client的 Rest接口](https://upload-images.jianshu.io/upload_images/9824247-bedcbd92ba9e3111.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

同理，浏览器访问 eureka-server的 Rest接口：`localhost:1111/test/eureka-service-info`：

![访问 eureka-server的 Rest接口](https://upload-images.jianshu.io/upload_images/9824247-eff3f6d540ae0f86.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

OK，一切正常，但目前微服务注册中心仅一个 eureka-server实例，下来我们来 **在线动态扩容微服务注册中心实例**。

---

## 扩充一次 Eureka Server

接下来我们再用 peer2配置文件来 **启动第二个 eureka server**，指令如下：

```
mvn spring-boot:run -Dspring.profiles.active=peer2
```

启动完毕后，浏览器访问微服务注册中心第一个实例 eureka-server-1：`localhost:1111/`
发现第二个微服务注册中心实例 eureka-server-2已经成功启动并加入

![第二个微服务注册中心实例 eureka-server-2已经成功启动并加入  ](https://upload-images.jianshu.io/upload_images/9824247-db63b9af89650916.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

为了让已启动的 eureka-client和 eureka-server-1能**在线感知**到 eureka-server-2的加入，此时我们需要修改两个地方： 

- 修改 Git上eureka-client的配置文件：

```yml
server:
  port: 1114

spring:
  application:
    name: eureka-client

eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:1111/eureka/,http://localhost:1112/eureka/  # 此处改为包含两个eureka-server
```

- 修改 Git上第一个eureka-server（eureka-server-1）的配置文件：

```yml
server:
  port: 1111

spring:
  application:
    name: eureka-server
eureka:
  instance:
    hostname: localhost
    preferIpAddress: true
  client:
    registerWithEureka: true
    fetchRegistry: true
    serviceUrl:
      defaultZone: http://localhost:1112/eureka/ # 此处改为第二个eureka-server地址
  server:
      waitTimeInMsWhenSyncEmpty: 0
      enableSelfPreservation: false
```

Git仓库里配置文件的修改完毕并不能触发已启动的 eureka-client和 eureka-server-1 在线更新，我们还需要向 eureka-client服务和 eureka-server-1服务来 POST两个请求来激活刚才所修改的配置：

```
POST localhost:1111/actuator/refresh  // 激活 eureka-client服务的配置
POST localhost:1114/actuator/refresh // 激活 eureka-server-1 服务的配置
```

POST请求一旦发出，我们从控制台里能直观看到更新配置文件的详情：

![POST触发更新配置文件](https://upload-images.jianshu.io/upload_images/9824247-e50489a8c4b5c397.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

配置文件更新完毕，浏览器再次访问 eureka-client的 Rest接口：`localhost:1114/test/eureka-service-info`：

![浏览器再次访问 eureka-client的 Rest接口](https://upload-images.jianshu.io/upload_images/9824247-6972fe23b7ec6d64.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

---

## 再扩充一次 Eureka Server

接下来还可以再用 peer3配置文件来启动第三个 eureka server加入高可用微服务注册中心集群，过程和上面类似，此处不再赘述！当然更多 eureka-server实例的扩充原理也相同。

---

## 后记

> 由于能力有限，若有错误或者不当之处，还请大家批评指正，一起学习交流！

- My Personal Blog：[CodeSheep  程序羊](http://www.codesheep.cn/)
- [我的半年技术博客之路](https://www.jianshu.com/p/28ba53821450)

---

---