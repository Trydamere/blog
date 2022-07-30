---
type: docs
title: "路由规则"
linkTitle: "路由规则"
weight: 33
description: "通过 Dubbo 中的路由规则做服务治理"

---

此任务将展示通过 Dubbo 中的路由规则做服务治理

路由规则在发起一次RPC调用前起到过滤目标服务器地址的作用，过滤后的地址列表，将作为消费端最终发起RPC调用的备选地址。

- 条件路由。支持以服务或 Consumer 应用为粒度配置路由规则。
- 标签路由。以 Provider 应用为粒度配置路由规则。

后续我们计划在 2.6.x 版本的基础上继续增强脚本路由功能。



## 开始之前

- 安装idea

- 克隆[dubbo-samples](https://github.com/apache/dubbo-samples.git)到本地，idea打开

- 运行dubbo-admin镜像并运行镜像


## 概览



dubbo-samples-governance示例包含dubbo-samples-configconditionrouter和dubbo-samples-tagrouter子示例，这2个子示例分别应用**条件路由**和**标签路由**的路由规则做服务治理。



此任务的最初目标是应用路由规则做服务治理。稍后，您将应用**条件路由**和**标签路由**的路由规则做服务治理。



## 条件路由规则

dubbo-samples-configconditionrouter示例中，BasicConsumer类和BascicProvider类为消费者类和提供者类，现在运行BasicProvider。成功后在Dubbo-Admin查询服务，可以看到应用governance-conditionroute-provider，如图所示：

![BasicProvider1_check](https://img.yilonghuang.com/BasicProvider1_check.png)



现在再运行一个提供者示例，运行在20881端口，首先修改dubbo-demo-provider.xml，将< dubbo:protocol />标签中端口修改修改为20881，运行BasicProvider，验证提供者示例运行在20880端口和20881端口，点击详情，如图所示：

![BasicProvider2_check](https://img.yilonghuang.com/BasicProvider2_check.png)



可以看到提供者示例分别运行在20880端口和20881端口。



现在完成环境搭建，开始条件路由的任务。



### 任务1：对Consumer应用配置条件路由规则



在Dubbo-Admin点击服务治理/条件路由，点击创建，规则如下：

```
# DemoService的sayHello方法只能消费所有端口为20880的服务实例
# DemoService2的sayHello方法只能消费所有端口为20881的服务实例
---
scope: application
force: true
runtime: true
enabled: true
priority: 2
key: governance-conditionrouter-consumer
conditions:
- interface=org.apache.dubbo.samples.governance.api.DemoService&method=sayHello=>address=*:20880
- interface=org.apache.dubbo.samples.governance.api.DemoService2&method=sayHello=>address=*:20881
```



> **提示**
>
> 如果创建路由规则失败，请先运行一个消费者示例，然后在创建路由规则



运行BasicConsumer，DemoService的sayHello方法只能消费所有端口为20880的服务实例， DemoService2的sayHello方法只能消费所有端口为20881的服务实例，说明路由规则生效，运行结果截图：

![condition-rule-result1](https://img.yilonghuang.com/condition-rule-result1.png)



### 任务2：对服务配置条件路由规则



在Dubbo-Admin点击服务治理/条件路由，点击创建，规则如下：

```
# DemoService的sayHello方法只能消费所有端口为20880的服务实例
# DemoService的sayHi方法只能消费所有端口为20881的服务实例
---
scope: service
force: true
runtime: true
enabled: true
key: org.apache.dubbo.samples.governance.api.DemoService
conditions:
  - method=sayHello => address=*:20880
  - method=sayHi => address=*:20881
...
```



> **提示**
>
> 如果创建路由规则失败，请在DemoService创建sayHi方法，DemoServiceImpl创建sayHi方法实现，重新运行BasicProvider示例



运行BasicConsumer，DemoService的sayHello方法只能消费所有端口为20880的服务实例， DemoService的sayHi方法只能消费所有端口为20881的服务实例，说明路由规则生效，运行结果截图：

![condition-rule-result2](https://img.yilonghuang.com/condition-rule-result2.png)



### 规则详解

#### 各字段含义

- `scope`表示路由规则的作用粒度，scope的取值会决定key的取值。**必填**。
  - service 服务粒度
  - application 应用粒度
- `Key`明确规则体作用在哪个服务或应用。**必填**。
  - scope=service时，key取值为[{group}:]{service}[:{version}]的组合
  - scope=application时，key取值为application名称
- `enabled=true` 当前路由规则是否生效，可不填，缺省生效。
- `force=false` 当路由结果为空时，是否强制执行，如果不强制执行，路由结果为空的路由规则将自动失效，可不填，缺省为 `false`。
- `runtime=false` 是否在每次调用时执行路由规则，否则只在提供者地址列表变更时预先执行并缓存结果，调用时直接从缓存中获取路由结果。如果用了参数路由，必须设为 `true`，需要注意设置会影响调用的性能，可不填，缺省为 `false`。
- `priority=1` 路由规则的优先级，用于排序，优先级越大越靠前执行，可不填，缺省为 `0`。
- `conditions` 定义具体的路由规则内容。**必填**。

#### Conditions规则体

    `conditions`部分是规则的主体，由1到任意多条规则组成，下面我们就每个规则的配置语法做详细说明：

1. **格式**

- `=>` 之前的为消费者匹配条件，所有参数和消费者的 URL 进行对比，当消费者满足匹配条件时，对该消费者执行后面的过滤规则。
- `=>` 之后为提供者地址列表的过滤条件，所有参数和提供者的 URL 进行对比，消费者最终只拿到过滤后的地址列表。
- 如果匹配条件为空，表示对所有消费方应用，如：`=> host != 10.20.153.11`
- 如果过滤条件为空，表示禁止访问，如：`host = 10.20.153.10 =>`

2. **表达式**

参数支持：

- 服务调用信息，如：method, argument 等，暂不支持参数路由
- URL 本身的字段，如：protocol, host, port 等
- 以及 URL 上的所有参数，如：application, organization 等

条件支持：

- 等号 `=` 表示"匹配"，如：`host = 10.20.153.10`
- 不等号 `!=` 表示"不匹配"，如：`host != 10.20.153.10`

值支持：

- 以逗号 `,` 分隔多个值，如：`host != 10.20.153.10,10.20.153.11`
- 以星号 `*` 结尾，表示通配，如：`host != 10.20.*`
- 以美元符 `$` 开头，表示引用消费者参数，如：`host = $host`

3. **Condition示例**

- 排除预发布机：

```
=> host != 172.22.3.91
```

- 白名单：

```
register.ip != 10.20.153.10,10.20.153.11 =>
```

> **注意**
>
> 一个服务只能有一条白名单规则，否则两条规则交叉，就都被筛选掉了

- 黑名单：

```
register.ip = 10.20.153.10,10.20.153.11 =>
```

- 服务寄宿在应用上，只暴露一部分的机器，防止整个集群挂掉：

```
=> host = 172.22.3.1*,172.22.3.2*
```

- 为重要应用提供额外的机器：

```
application != kylin => host != 172.22.3.95,172.22.3.96
```

- 读写分离：

```
method = find*,list*,get*,is* => host = 172.22.3.94,172.22.3.95,172.22.3.96
method != find*,list*,get*,is* => host = 172.22.3.97,172.22.3.98
```

- 前后台分离：

```
application = bops => host = 172.22.3.91,172.22.3.92,172.22.3.93
application != bops => host = 172.22.3.94,172.22.3.95,172.22.3.96
```

- 隔离不同机房网段：

```
host != 172.22.3.* => host != 172.22.3.*
```

- 提供者与消费者部署在同集群内，本机只访问本机的服务：

```
=> host = $host
```



## 标签路由规则



### 简介

标签路由通过将某一个或多个服务的提供者划分到同一个分组，约束流量只在指定分组中流转，从而实现流量隔离的目的，可以作为蓝绿发布、灰度发布等场景的能力基础。



dubbo-samples-tagrouter示例中，BasicConsumer类和BascicProvider类为消费者类和提供者类，现在运行BasicProvider。成功后在Dubbo-Admin查询服务，可以看到应用 governance-tagrouter-provider，如图所示：

![BasicProvider3_check](https://img.yilonghuang.com/BasicProvider3_check.png)



现在再运行一个提供者示例，运行在20881端口，首先修改dubbo-demo-provider.xml，将< dubbo:protocol />标签中端口修改修改为20881，运行BasicProvider，验证提供者示例运行在20880端口和20881端口，点击详情，如图所示：

![BasicProvider2_check](https://img.yilonghuang.com/BasicProvider2_check.png)



可以看到提供者示例分别运行在20880端口和20881端口。



现在完成环境搭建，开始标签路由的任务。



标签主要是指对Provider端应用实例的分组，目前有两种方式可以完成实例分组，分别是`动态规则打标`和`静态规则打标`，其中动态规则相较于静态规则优先级更高，而当两种规则同时存在且出现冲突时，将以动态规则为准。



### 任务1：对Provider应用动态规则打标

在Dubbo-Admin点击服务治理/标签路由，点击创建，规则如下：

```
# governance-tagrouter-provider应用增加了两个标签分组tag1和tag2
# tag1包含一个实例 127.0.0.1:20880
# tag2包含一个实例 127.0.0.1:20881
---
  force: false
  runtime: true
  enabled: true
  key: governance-tagrouter-provider
  tags:
    - name: tag1
      addresses: ["127.0.0.1:20880"]
    - name: tag2
      addresses: ["127.0.0.1:20881"]
 ...
```



运行BasicConsumer，DemoService的sayHello方法只能消费所有端口为20880的服务实例， DemoService2的sayHello方法只能消费所有端口为20881的服务实例，说明路由规则生效，运行结果截图：

![tag-rule-result](https://img.yilonghuang.com/tag-rule-result.png)



### 任务2：对Provider应用静态规则打标



在dubbo-demo-provider.xml中修改如下配置：



```
<dubbo:service tag="tag1" interface="org.apache.dubbo.samples.governance.api.DemoService" ref="demoService"/>
<dubbo:service tag="tag2" interface="org.apache.dubbo.samples.governance.api.DemoService2" ref="demoService2"/>
```



运行BasicConsumer，DemoService的sayHello方法只能消费所有端口为20880的服务实例， DemoService2的sayHello方法只能消费所有端口为20881的服务实例，说明路由规则生效，运行结果截图：

![tag-rule-result](https://img.yilonghuang.com/tag-rule-result.png)





### 规则详解

#### 格式

- `Key`明确规则体作用到哪个应用。**必填**。
- `enabled=true` 当前路由规则是否生效，可不填，缺省生效。
- `force=false` 当路由结果为空时，是否强制执行，如果不强制执行，路由结果为空的路由规则将自动失效，可不填，缺省为 `false`。
- `runtime=false` 是否在每次调用时执行路由规则，否则只在提供者地址列表变更时预先执行并缓存结果，调用时直接从缓存中获取路由结果。如果用了参数路由，必须设为 `true`，需要注意设置会影响调用的性能，可不填，缺省为 `false`。
- `priority=1` 路由规则的优先级，用于排序，优先级越大越靠前执行，可不填，缺省为 `0`。
- `tags` 定义具体的标签分组内容，可定义任意n（n>=1）个标签并为每个标签指定实例列表。**必填**。
  - name， 标签名称
- addresses， 当前标签包含的实例列表



#### 降级约定

1. `dubbo.tag=tag1` 时优先选择 标记了`tag=tag1` 的 provider。若集群中不存在与请求标记对应的服务，默认将降级请求 tag为空的provider；如果要改变这种默认行为，即找不到匹配tag1的provider返回异常，需设置`dubbo.force.tag=true`。

2. `dubbo.tag`未设置时，只会匹配tag为空的provider。即使集群中存在可用的服务，若 tag 不匹配也就无法调用，这与约定1不同，携带标签的请求可以降级访问到无标签的服务，但不携带标签/携带其他种类标签的请求永远无法访问到其他标签的服务。

> **提示**
>
> `2.6.x` 版本以及更早的版本请使用[老版本路由规则](../routing-rule-deprecated)
> 自定义路由参考[路由扩展](../../../dev/impls/router)
