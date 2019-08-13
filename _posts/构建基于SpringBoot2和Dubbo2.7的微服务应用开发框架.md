---
title: 构建基于SpringBoot2和Dubbo2.7的微服务应用开发框架
date: 2019-08-12 22:16:00
tags:
- dubbo
- springboot
categories:
- 微服务
- 分布式
- 架构
---

![](/images/SpringBoot2和Dubbo2.7.jpg)

# 介绍

## 什么是分布式应用？

简单点说就是一个完整的应用被**拆分**成了多个应用，每个拆出来的应用都可以单独部署，每个应用负责原来那个完整应用的一部分功能，只有当这些应用全部提供服务时才是完整的。这其实很好理解，举个例子，比如某商城系统被划分成了营销子系统、订单子系统和会员子系统，在这里面每个子系统有其固有职责，对于整个商城系统来说是不可缺少的一部分，通常这种划分也被称为**垂直拆分**。

## 什么是微服务应用？

其实所谓微服务应用算是分布式应用的一种特殊形式，它们之间区别不大，相同点很多。唯一比较明显的区分是前者对系统拆分的更加**细**，更加注重拆分后的子系统（也称为服务）功能复用，职责非常单一，小团队可维护。另外一点是注重DevOps的工作流，让服务的开发、迭代、测试和部署更加自动化，极大提高项目交付效率。

即使是在同一个系统里面，因为依据不同的拆分原则，最终得到的微服务可能不一致，这和站的角度、业务特点以及架构思路有关。实际工作中有些人把握不好原则，容易把服务拆分的过细导致维护非常麻烦，也有拆分的过大而不能复用已有的模块。虽然业界也没有完全统一的结论，不过也有一些原则被总结出来，有兴趣的小伙伴可以在网络上看看相关的文章。

<!-- more -->
## 什么是Dubbo？

[Apache Dubbo](http://dubbo.apache.org/zh-cn/index.html) |ˈdʌbəʊ| 是一款高性能、轻量级的开源Java RPC框架，它提供了三大核心能力：面向接口的远程方法调用，智能容错和负载均衡，以及服务自动注册和发现。

![](/images/dubbo_architecture.png)

dubbo最初由阿里内部开源，后来因各种原因停止更新了很长时间，现在终于开始重新维护了，并把这个项目捐赠给了Apache基金会，目前处于孵化阶段。

## 为什么需要Dubbo？

为什么会诞生dubbo肯定是有原因的，看看下面这张图：

![](/images/dubbo-architecture-roadmap.jpg)

随着互联网的不断发展，各种系统应用的规模不断扩大，常规的单体架构以及垂直应用架构已无法应对，慢慢开始往分布式服务架构以及流动计算架构势演进，这个时候就需要一个治理系统确保架构有条不紊的演进，基于此dubbo就应运而生了。

dubbo的[官方文档](http://dubbo.apache.org/zh-cn/docs/user/preface/background.html)对这块有详细介绍，可以链接进去看看。

## Dubbo的应用场景

从它诞生的背景就可以知道，单体架构等小型系统肯定不适用，过度的架构设计、引入新技术只会把项目做死，因为项目没有到一定规模是没有能力去承担它的副作用的，它带来的是系统结构复杂，对开发人员水平以及硬件性能有高要求。

dubbo是天生为分布式应用提供支持的，不管是所谓SOA的应用，还是现在流行的微服务应用。它是应用框架的基础组件，对你用在什么业务上是不敏感的，也就说基本上没有业务限制。

# 应用示例

下面通过一个小的电商系统演示如何使用dubbo，为了保证演示效果，项目中的代码是失真的。

## 需求

1. 用户在访问某在线商城，将某件商品添加到购物车，并在购物车中进行结算。
2. 商城根据结算请求生成订单，并把结果反馈给用户。

## 方案设计

### 服务设计

为了演示简单，我们创建三个服务，每个服务可以单独维护和部署，可以根据访问压力动态扩容（指每个服务可以多节点部署）。

![](/images/dubbo演示项目服务设计.jpg)

### 业务设计

**调用关系：**购物车服务调用会员服务获取用户信息，调用订单服务生成订单。

为了演示简单专注使用dubbo，业务上的东西本次设计的简单些，因为这个并不是重点。

### 接口设计

根据上面的业务要求我们设计了两个远程接口，分别需要会员和订单服务提供支持。

| 序号 | 接口名称        | 接口描述         | 请求参数    | 响应参数           |
| ---- | --------------- | ------------- | ----------- | ------------------ |
| 1    | genOrder        | 生成订单     | OrderReq    | OrderGenResultResp |
| 2    | getUserInfoById | 根据用户编号获取对应信息 | UserInfoReq | UserInfoResp       |

## 构建项目

本次演示所用框架：

| 软件名称    | 版本                                                      |
| ----------- | --------------------------------------------------------- |
| 构建工具    | [gradle5](https://gradle.org/releases/)                   |
| Dubbo       | [2.7.2](http://dubbo.apache.org/zh-cn/blog/download.html) |
| Spring boot | 2.1.6.RELEASE                                             |

项目结构和代码比较简单，下面基本上会贴出所有代码。

### 构建接口项目

为项目维护所有服务开放接口，有调用需求的服务必须引入本项目。

**`build.gradle`**

```groovy
plugins {
    id 'java'
}

group 'com.tianmingxing.example'
version '1.0-SNAPSHOT'

dependencies {
    testCompile group: 'junit', name: 'junit', version: '4.12'
}
```

**`OrderService.java`**

```java
package com.tianmingxing.dubbo.example.api.order;

/**
 * 订单服务开放接口
 *
 * @author tianmingxing <mx.tian@qq.com>
 * @date 2019-07-31
 */
public interface OrderService {

    /**
     * 生成订单
     *
     * @param req 订单生成所需要的数据
     * @return 订单生成结果，用于展示在页面上。
     */
    public OrderGenResultResp genOrder(OrderReq req);
}
```

**`OrderReq.java`**

```java
package com.tianmingxing.dubbo.example.api.order;

import com.tianmingxing.dubbo.example.api.user.UserInfoResp;

import java.io.Serializable;

/**
 * 生成订单所需要的参数
 *
 * @author tianmingxing <mx.tian@qq.com>
 * @date 2019-07-31
 */
public class OrderReq implements Serializable {

    /**
     * 用户信息
     */
    private UserInfoResp userInfoResp;
    /**
     * 商品编号
     */
    private String goodsId;
    /**
     * 商品数量
     */
    private Integer qty;

    //省略getter/setter
}
```

**`OrderGenResultResp.java`**

```java
package com.tianmingxing.dubbo.example.api.order;

import java.io.Serializable;

/**
 * 订单生成之后的确认信息
 *
 * @author tianmingxing <mx.tian@qq.com>
 * @date 2019-07-31
 */
public class OrderGenResultResp implements Serializable {

    /**
     * 订单号
     */
    private String orderId;
    /**
     * 生单成功与否
     */
    private boolean isSuccess;

    //省略getter/setter
}
```

**`UserInfoService.java`**

```java
package com.tianmingxing.dubbo.example.api.user;

/**
 * 会员服务开放接口
 *
 * @author tianmingxing <mx.tian@qq.com>
 * @date 2019-07-31
 */
public interface UserInfoService {

    /**
     * 根据用户编号获取对应信息
     *
     * @param req 查询用户的参数模型
     * @return 用户信息
     */
    public UserInfoResp getUserInfoById(UserInfoReq req);
}
```

**`UserInfoReq.java`**

```java
package com.tianmingxing.dubbo.example.api.user;

import java.io.Serializable;

/**
 * 调用会员服务接口时请求数据封装模型
 *
 * @author tianmingxing <mx.tian@qq.com>
 * @date 2019-07-31
 */
public class UserInfoReq implements Serializable {

    /**
     * 用户编号
     */
    private Integer id;

    public UserInfoReq(Integer id) {
        this.id = id;
    }

    //省略getter/setter
}
```

**`UserInfoResp.java`**

```java
package com.tianmingxing.dubbo.example.api.user;

import java.io.Serializable;

/**
 * 调用会员服务接口时响应数据封装模型
 *
 * @author tianmingxing <mx.tian@qq.com>
 * @date 2019-07-31
 */
public class UserInfoResp implements Serializable {

    /**
     * 用户编号
     */
    private Integer id;
    /**
     * 昵称
     */
    private String nickName;
    /**
     * 年龄
     */
    private Integer age;
    /**
     * 邮箱地址
     */
    private String email;

    //省略getter/setter
}
```

### 构建会员服务

**`build.gradle`**

```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '2.1.6.RELEASE'
}

group 'com.tianmingxing.example'
version '1.0-SNAPSHOT'


dependencies {
    testCompile(
            'org.springframework.boot:spring-boot-starter-test:2.1.6.RELEASE',
    )

    compile(
            project(':api'),
            'org.springframework.boot:spring-boot-starter-web:2.1.6.RELEASE',
            //连接zk
            'org.apache.curator:curator-framework:4.2.0',
            //dubbo
            'org.apache.dubbo:dubbo-spring-boot-starter:2.7.1',
            'org.apache.dubbo:dubbo-registry-zookeeper:2.7.3',
            'org.apache.dubbo:dubbo-configcenter-zookeeper:2.7.3',
            'org.apache.dubbo:dubbo-config-spring:2.7.3',
            'org.apache.dubbo:dubbo-rpc-dubbo:2.7.3',
            'org.apache.dubbo:dubbo-remoting-netty4:2.7.3',
            'org.apache.dubbo:dubbo-serialization-hessian2:2.7.3',
            'org.apache.dubbo:dubbo-metadata-report-zookeeper:2.7.3',
    )

    configurations {
        all {
            exclude group: 'org.springframework.boot', module: 'spring-boot-starter-tomcat'
        }
    }
}
```

**`application.yml`**

```yaml
spring:
  application:
    name: example-user
dubbo:
  application:
    name: ${spring.application.name}
  protocol:
    name: dubbo
    port: 20881
    server: netty
    host: 127.0.0.1
  registry:
    client: curator
    address: zookeeper://127.0.0.1:2181
  metadata-report:
    address: zookeeper://127.0.0.1:2181
```

**`Application.java`**

```java
package com.tianmingxing.dubbo.example.user;

import org.apache.dubbo.config.spring.context.annotation.EnableDubbo;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

/**
 * @author tianmingxing <mx.tian@qq.com>
 * @date 2019-08-02
 */
@SpringBootApplication(scanBasePackages = "com.tianmingxing.dubbo.example.user")
@EnableDubbo(scanBasePackages = "com.tianmingxing.dubbo.example.user")
public class Application {

    /**
     * 初始化spring系列框架
     *
     * @param args 启动参数
     */
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

**`UserInfoServiceImpl.java`**

```java
package com.tianmingxing.dubbo.example.user.api;

import com.tianmingxing.dubbo.example.api.user.UserInfoReq;
import com.tianmingxing.dubbo.example.api.user.UserInfoResp;
import com.tianmingxing.dubbo.example.api.user.UserInfoService;
import org.apache.dubbo.config.annotation.Service;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

/**
 * 服务提供者，实现用户信息接口
 *
 * @author tianmingxing <mx.tian@qq.com>
 * @date 2019-08-02
 */
@Service(version = "1.0.0")
public class UserInfoServiceImpl implements UserInfoService {

    private final static Logger log = LoggerFactory.getLogger(UserInfoServiceImpl.class);

    @Override
    public UserInfoResp getUserInfoById(UserInfoReq req) {
        log.info("开始执行获取用户信息的业务。。。");
        UserInfoResp userInfoResp = new UserInfoResp();
        userInfoResp.setId(1);
        userInfoResp.setNickName("张三 zhan shan");
        userInfoResp.setAge(32);
        userInfoResp.setEmail("mx.tian@qq.com");
        return userInfoResp;
    }
}
```

### 构建订单服务

**`build.gradle`**

```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '2.1.6.RELEASE'
}

group 'com.tianmingxing.example'
version '1.0-SNAPSHOT'

dependencies {
    testCompile(
            'org.springframework.boot:spring-boot-starter-test:2.1.6.RELEASE',
    )

    compile(
            project(':api'),
            'org.springframework.boot:spring-boot-starter-web:2.1.6.RELEASE',
            //连接zk
            'org.apache.curator:curator-framework:4.2.0',
            //dubbo
            'org.apache.dubbo:dubbo-spring-boot-starter:2.7.1',
            'org.apache.dubbo:dubbo-registry-zookeeper:2.7.3',
            'org.apache.dubbo:dubbo-configcenter-zookeeper:2.7.3',
            'org.apache.dubbo:dubbo-config-spring:2.7.3',
            'org.apache.dubbo:dubbo-rpc-dubbo:2.7.3',
            'org.apache.dubbo:dubbo-remoting-netty4:2.7.3',
            'org.apache.dubbo:dubbo-serialization-hessian2:2.7.3',
            'org.apache.dubbo:dubbo-metadata-report-zookeeper:2.7.3',
    )

    configurations {
        all {
            exclude group: 'org.springframework.boot', module: 'spring-boot-starter-tomcat'
        }
    }
}
```

**`application.yml`**

```yaml
spring:
  application:
    name: example-order
dubbo:
  application:
    name: ${spring.application.name}
  protocol:
    name: dubbo
    port: 20882
    server: netty
    host: 127.0.0.1
  registry:
    client: curator
    address: zookeeper://127.0.0.1:2181
  metadata-report:
    address: zookeeper://127.0.0.1:2181
```

**`Application.java`**

```java
package com.tianmingxing.dubbo.example.order;

import org.apache.dubbo.config.spring.context.annotation.EnableDubbo;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

/**
 * @author tianmingxing <mx.tian@qq.com>
 * @date 2019-08-02
 */
@SpringBootApplication(scanBasePackages = "com.tianmingxing.dubbo.example.order")
@EnableDubbo(scanBasePackages = "com.tianmingxing.dubbo.example.order")
public class Application {

    /**
     * 初始化spring系列框架
     *
     * @param args 启动参数
     */
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

**`OrderServiceImpl.java`**

```java
package com.tianmingxing.dubbo.example.order.api;

import com.tianmingxing.dubbo.example.api.order.OrderGenResultResp;
import com.tianmingxing.dubbo.example.api.order.OrderReq;
import com.tianmingxing.dubbo.example.api.order.OrderService;
import org.apache.dubbo.config.annotation.Service;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

/**
 * 服务提供者，具体实现生成订单业务
 *
 * @author tianmingxing <mx.tian@qq.com>
 * @date 2019-08-02
 */
@Service(version = "1.0.0")
public class OrderServiceImpl implements OrderService {

    private final static Logger log = LoggerFactory.getLogger(OrderServiceImpl.class);

    @Override
    public OrderGenResultResp genOrder(OrderReq req) {
        log.info("开始执行生成订单的业务。。。");
        OrderGenResultResp orderGenResultResp = new OrderGenResultResp();
        orderGenResultResp.setOrderId("SN2143253");
        orderGenResultResp.setSuccess(true);
        return orderGenResultResp;
    }
}
```

### 构建购物车服务

**`build.gradle`**

```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '2.1.6.RELEASE'
}

group 'com.tianmingxing.example'
version '1.0-SNAPSHOT'

dependencies {
    testCompile(
            'org.springframework.boot:spring-boot-starter-test:2.1.6.RELEASE',
    )

    compile(
            project(':api'),
            'org.springframework.boot:spring-boot-starter-web:2.1.6.RELEASE',
            //连接zk
            'org.apache.curator:curator-framework:4.2.0',
            //dubbo
            'org.apache.dubbo:dubbo-spring-boot-starter:2.7.1',
            'org.apache.dubbo:dubbo-registry-zookeeper:2.7.3',
            'org.apache.dubbo:dubbo-configcenter-zookeeper:2.7.3',
            'org.apache.dubbo:dubbo-config-spring:2.7.3',
            'org.apache.dubbo:dubbo-rpc-dubbo:2.7.3',
            'org.apache.dubbo:dubbo-remoting-netty4:2.7.3',
            'org.apache.dubbo:dubbo-serialization-hessian2:2.7.3',
            'org.apache.dubbo:dubbo-metadata-report-zookeeper:2.7.3',
            //模型字段校验
            'org.hibernate.validator:hibernate-validator:6.0.17.Final',
    )
}
```

**`application.yml`**

```yaml
erver:
  port: 8082
spring:
  application:
    name: example-cart
dubbo:
  scan:
    base-packages: com.tianmingxing.example.dubbo.cart.provider
  application:
    name: ${spring.application.name}
  protocol:
    name: dubbo
    port: 20880
    server: netty
    host: 127.0.0.1
    transporter: netty
  registry:
    client: curator
    address: zookeeper://127.0.0.1:2181
  metadata-report:
    address: zookeeper://127.0.0.1:2181

```

**`Application.java`**

```java
package com.tianmingxing.dubbo.example.cart;

import org.apache.dubbo.config.spring.context.annotation.EnableDubbo;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

/**
 * @author tianmingxing <mx.tian@qq.com>
 * @date 2019-08-02
 */
@SpringBootApplication(scanBasePackages = "com.tianmingxing.dubbo.example.cart")
@EnableDubbo(scanBasePackages = "com.tianmingxing.dubbo.example.cart")
public class Application {

    /**
     * 初始化spring系列框架并打开内置tomcat服务。
     *
     * @param args 启动参数
     */
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

**`CartController.java`**

```java
package com.tianmingxing.dubbo.example.cart.controller;

import com.tianmingxing.dubbo.example.api.order.OrderGenResultResp;
import com.tianmingxing.dubbo.example.cart.model.CommonResp;
import com.tianmingxing.dubbo.example.cart.model.SettlementReq;
import com.tianmingxing.dubbo.example.cart.service.SettlementService;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import javax.annotation.Resource;
import javax.validation.Valid;

/**
 * 购物车系列请求处理控制器
 *
 * @author tianmingxing <mx.tian@qq.com>
 * @date 2019-07-31
 */
@RestController
@RequestMapping("cart")
public class CartController {

    private final static Logger log = LoggerFactory.getLogger(CartController.class);

    @Resource
    private SettlementService settlementService;

    /**
     * 当请求地址为“POST /cart/settlement”时进入本方法，处理用户在购物车内结算业务。
     *
     * @return 处理结果
     */
    @PostMapping("settlement")
    public CommonResp<OrderGenResultResp> settlement(@Valid SettlementReq req) {
        log.info("开始处理请求：/cart/settlement");
        return settlementService.settlement(req);
    }
}
```

**`SettlementReq.java`**

```java
package com.tianmingxing.dubbo.example.cart.model;

import javax.validation.constraints.NotNull;

/**
 * 前端页面请求结算的参数
 *
 * @author tianmingxing <mx.tian@qq.com>
 * @date 2019-07-31
 */
public class SettlementReq {

    /**
     * 用户编号
     */
    @NotNull
    private Integer uid;
    /**
     * 商品编号
     */
    private String goodsId;
    /**
     * 商品数量
     */
    private Integer qty;

    //省略getter/setter
}
```

**`CommonResp.java`**

```java
package com.tianmingxing.dubbo.example.cart.model;

import java.io.Serializable;
import java.time.LocalDateTime;
import java.time.ZoneId;

/**
 * 统一接口返回数据结构
 *
 * @author tianmingxing <mx.tian@qq.com>
 * @date 2019-07-31
 */
public class CommonResp<T> implements Serializable {

    /**
     * 响应返回去的时间，格式：2018-12-04T19:55:39.814，采用亚洲/上海时区
     */
    private String timestamp;
    /**
     * 错误代码
     */
    private Integer code;
    /**
     * 消息内容
     */
    private String message;
    /**
     * 响应数据
     */
    private T data;

    /**
     * 适用于业务处理正常时的响应，默认code=0表示请求响应成功
     *
     * @param data 正常响应回去的数据
     */
    public CommonResp(T data) {
        this(0, null, data);
    }

    /**
     * 适用于业务处理正常时的响应
     *
     * @param code    错误代码
     * @param message 返回信息
     */
    public CommonResp(Integer code, String message) {
        this(code, message, null);
    }

    /**
     * 通常适用于业务处理失败时的响应
     *
     * @param code    错误代码
     * @param message 错误信息
     * @param data    正常响应回去的数据
     */
    public CommonResp(Integer code, String message, T data) {
        this.timestamp = LocalDateTime.now(ZoneId.of("Asia/Shanghai")).toString();
        this.code = code;
        this.message = message;
        this.data = data;
    }

    //省略getter/setter
}
```

**`SettlementService.java`**

```java
package com.tianmingxing.dubbo.example.cart.service;

import com.tianmingxing.dubbo.example.api.order.OrderGenResultResp;
import com.tianmingxing.dubbo.example.api.order.OrderReq;
import com.tianmingxing.dubbo.example.api.order.OrderService;
import com.tianmingxing.dubbo.example.api.user.UserInfoReq;
import com.tianmingxing.dubbo.example.api.user.UserInfoResp;
import com.tianmingxing.dubbo.example.api.user.UserInfoService;
import com.tianmingxing.dubbo.example.cart.controller.CartController;
import com.tianmingxing.dubbo.example.cart.model.CommonResp;
import com.tianmingxing.dubbo.example.cart.model.SettlementReq;
import org.apache.dubbo.config.annotation.Reference;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;

/**
 * 结算业务处理逻辑
 *
 * @author tianmingxing <mx.tian@qq.com>
 * @date 2019-07-31
 */
@Service
public class SettlementService {

    private final static Logger log = LoggerFactory.getLogger(SettlementService.class);

    @Reference(version = "1.0.0")
    private UserInfoService userInfoService;
    @Reference(version = "1.0.0")
    private OrderService orderService;

    /**
     * 具体实现结算功能，需要通过RPC调用其它微服务接口。
     * 以下只是业务示意，并非真实业务。
     *
     * @param req
     * @return
     */
    public CommonResp<OrderGenResultResp> settlement(SettlementReq req) {
        //调用远程接口获取用户信息
        log.info("开始远程调用UserInfoService");
        UserInfoResp userInfoResp = userInfoService.getUserInfoById(new UserInfoReq(req.getUid()));

        //调用远程生单接口
        OrderReq orderReq = new OrderReq();
        orderReq.setUserInfoResp(userInfoResp);
        orderReq.setGoodsId(req.getGoodsId());
        orderReq.setQty(req.getQty());
        log.info("开始远程调用OrderService");
        OrderGenResultResp orderGenResultResp = orderService.genOrder(orderReq);

        //判断生单是否完成，组装响应到前端的数据对象
        log.info("cart服务处理结算完成，准备返回。");
        return new CommonResp<>(0, "成功", orderGenResultResp);
    }
}
```

## 测试

分别启动三个服务，使用postman或其它模拟http请求工具，请求购物车链接 `http://localhost:8081/cart/settlement`，注意观察响应结果，同时查看控制台日志输出。

1. 购物车服务有开启2个端口，分别是http端口以及dubbo的端口。
2. 其它两个服务只开启了dubbo端口，在实际项目中根据需求来关闭http端口。

# Dubbo Web控制台

## 介绍

由于服务接口会全部注册到ZK中，所以对接口的管理工具也是基于ZK来操作的，但手动操作是比较复杂和麻烦的，官方提供管理这些接口的WEB控制台，可以让我们非常方便的设置接口权重、负载和路由等功能。

![](/images/dubbo-search.jpg)

## 安装

```bash
git clone https://github.com/apache/dubbo-admin.git
cd dubbo-admin
mvn clean package
cd dubbo-admin-distribution/target
java -jar dubbo-admin-0.1.jar
```
## 使用

具体使用可以见[官方文档](http://dubbo.apache.org/zh-cn/docs/admin/serviceSearch.html)。
