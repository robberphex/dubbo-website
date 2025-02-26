---
type: docs
title: "应用级服务发现地址迁移规则说明"
linkTitle: "应用级服务发现地址迁移规则"
weight: 42
description: "本文具体说明了地址迁移过程中使用的规则体信息，用户可以根据自己需求定制适合自己的迁移规则。"
---

{{% pageinfo %}} 此文档已经不再维护。您当前查看的是快照版本。如果想要查看最新版本的文档，请参阅[最新版本](/zh/docs3-v2/java-sdk/upgrades-and-compatibility/service-discovery/migration-service-discovery/)。
{{% /pageinfo %}}

## 状态模型


在 Dubbo 3 之前地址注册模型是以接口级粒度注册到注册中心的，而 Dubbo 3 全新的应用级注册模型注册到注册中心的粒度是应用级的。从注册中心的实现上来说是几乎不一样的，这导致了对于从接口级注册模型获取到的 invokers 是无法与从应用级注册模型获取到的 invokers 进行合并的。为了帮助用户从接口级往应用级迁移，Dubbo 3 设计了 Migration 机制，基于三个状态的切换实现实际调用中地址模型的切换。


![//imgs/v3/migration/migration-1.png](/imgs/v3/migration/migration-1.png)

当前共存在三种状态，FORCE_INTERFACE（强制接口级），APPLICATION_FIRST（应用级优先）、FORCE_APPLICATION（强制应用级）。


FORCE_INTERFACE：只启用兼容模式下接口级服务发现的注册中心逻辑，调用流量 100% 走原有流程
APPLICATION_FIRST：开启接口级、应用级双订阅，运行时根据阈值和灰度流量比例动态决定调用流量走向
FORCE_APPLICATION：只启用新模式下应用级服务发现的注册中心逻辑，调用流量 100% 走应用级订阅的地址


## 规则体说明


规则采用 yaml 格式配置，具体配置下参考如下：
```yaml
key: 消费者应用名（必填）
step: 状态名（必填）
threshold: 决策阈值（默认1.0）
proportion: 灰度比例（默认100）
delay: 延迟决策时间（默认0）
force: 强制切换（默认 false）
interfaces: 接口粒度配置（可选）
  - serviceKey: 接口名（接口 + : + 版本号）（必填）
    threshold: 决策阈值
    proportion: 灰度比例
    delay: 延迟决策时间
    force: 强制切换
    step: 状态名（必填）
  - serviceKey: 接口名（接口 + : + 版本号）
    step: 状态名
applications: 应用粒度配置（可选）
  - serviceKey: 应用名（消费的上游应用名）（必填）
    threshold: 决策阈值
    proportion: 灰度比例
    delay: 延迟决策时间
    force: 强制切换
    step: 状态名（必填）
```


- key: 消费者应用名
- step: 状态名（FORCE_INTERFACE、APPLICATION_FIRST、FORCE_APPLICATION）
- threshold: 决策阈值（浮点数，具体含义参考后文）
- proportion: 灰度比例（0～100，决定调用次数比例）
- delay: 延迟决策时间（延迟决策的时间，实际等待时间为 1～2 倍 delay 时间，取决于注册中心第一次通知的时间，对于目前 Dubbo 的注册中心实现次配置项保留 0 即可）
- force: 强制切换（对于 FORCE_INTERFACE、FORCE_APPLICATION 是否不考虑决策直接切换，可能导致无地址调用失败问题）
- interfaces: 接口粒度配置



参考配置示例如下：
```yaml
key: demo-consumer
step: APPLICATION_FIRST
threshold: 1.0
proportion: 60
delay: 0
force: false
interfaces:
  - serviceKey: DemoService:1.0.0
    threshold: 0.5
    proportion: 30
    delay: 0
    force: true
    step: APPLICATION_FIRST
  - serviceKey: GreetingService:1.0.0
    step: FORCE_APPLICATION
```


## 配置方式说明
### 1. 配置中心配置文件下发（推荐）


- Key:    消费者应用名 + ".migration"
- Group: DUBBO_SERVICEDISCOVERY_MIGRATION

 
配置项内容参考上一节


程序启动时会拉取此配置作为最高优先级启动项，当配置项为启动项时不执行检查操作，直接按状态信息达到终态。
程序运行过程中收到新配置项将执行迁移操作，过程中根据配置信息进行检查，如果检查失败将回滚为迁移前状态。迁移是按接口粒度执行的，也即是如果一个应用有 10 个接口，其中 8 个迁移成功，2 个失败，那终态 8 个迁移成功的接口将执行新的行为，2 个失败的仍为旧状态。如果需要重新触发迁移可以通过重新下发规则达到。


注：如果程序在迁移时由于检查失败回滚了，由于程序无回写配置项行为，所以如果此时程序重启了，那么程序会直接按照新的行为不检查直接初始化。


### 2. 启动参数配置


- 配置项名：dubbo.application.service-discovery.migration
- 允许值范围：FORCE_INTERFACE、APPLICATION_FIRST、FORCE_APPLICATION



此配置项可以通过环境变量或者配置中心传入，启动时优先级比配置文件低，也即是当配置中心的配置文件不存在时读取此配置项作为启动状态。


### 3. 本地文件配置



| 配置项名 | 默认值 | 说明 |
| --- | --- | --- |
| dubbo.migration.file | dubbo-migration.yaml | 本地配置文件路径 |
| dubbo.application.migration.delay | 60000 | 配置文件延迟生效时间（毫秒） |

配置文件中格式与前文提到的规则一致


本地文件配置方式本质上是一个延时配置通知的方式，本地文件不会影响默认启动方式，当达到延时时间后触发推送一条内容和本地文件一致的通知。这里的延时时间与规则体中的 delay 字段无关联。
本地文件配置方式可以保证启动以默认行为初始化，当达到延时时触发迁移操作，执行对应的检查，避免启动时就以终态方式启动。


## 决策说明
### 1. 阈值探测


阈值机制旨在进行流量切换前的地址数检查，如果应用级的可使用地址数与接口级的可用地址数对比后没达到阈值将检查失败。


核心代码如下：
```java
if (((float) newAddressSize / (float) oldAddressSize) >= threshold) {
    return true;
}
return false;
```


同时 MigrationAddressComparator 也是一个 SPI 拓展点，用户可以自行拓展，所有检查的结果取交集。


### 2. 灰度比例


灰度比例功能仅在应用级优先状态下生效。此功能可以让用户自行决定调用往新模式应用级注册中心地址的调用数比例。灰度生效的前提是满足了阈值探测，在应用级优先状态下，如果阈值探测通过，`currentAvailableInvoker` 将被切换为对应应用级地址的 invoker；如果探测失败 `currentAvailableInvoker` 仍为原有接口级地址的 invoker。


流程图如下：
探测阶段
![//imgs/v3/migration/migration-2.png](/imgs/v3/migration/migration-2.png)
调用阶段
![//imgs/v3/migration/migration-3.png](/imgs/v3/migration/migration-3.png)


核心代码如下：
```java
// currentAvailableInvoker is based on MigrationAddressComparator's result
if (currentAvailableInvoker != null) {
    if (step == APPLICATION_FIRST) {
        // call ratio calculation based on random value
        if (ThreadLocalRandom.current().nextDouble(100) > promotion) {
            return invoker.invoke(invocation);
        }
    }
    return currentAvailableInvoker.invoke(invocation);
}

```


## 切换过程说明


地址迁移过程中涉及到了三种状态的切换，为了保证平滑迁移，共有 6 条切换路径需要支持，可以总结为从强制接口级、强制应用级往应用级优先切换；应用级优先往强制接口级、强制应用级切换；还有强制接口级和强制应用级互相切换。
对于同一接口切换的过程总是同步的，如果前一个规则还未处理完就收到新规则则回进行等待。


### 1. 切换到应用级优先


从强制接口级、强制应用级往应用级优先切换本质上是从某一单订阅往双订阅切换，保留原有的订阅并创建另外一种订阅的过程。这个切换模式下规则体中配置的 delay 配置不会生效，也即是创建完订阅后马上进行阈值探测并决策选择某一组订阅进行实际优先调用。由于应用级优先模式是支持运行时动态进行阈值探测，所以对于部分注册中心无法启动时即获取全量地址的场景在全部地址通知完也会重新计算阈值并切换。
应用级优先模式下的动态切换是基于服务目录（Directory）的地址监听器实现的。
![//imgs/v3/migration/migration-4.png](/imgs/v3/migration/migration-4.png)


### 2. 应用级优先切换到强制


应用级优先往强制接口级、强制应用级切换的过程是对双订阅的地址进行检查，如果满足则对另外一份订阅进行销毁，如果不满足则回滚保留原来的应用级优先状态。
如果用户希望这个切换过程不经过检查直接切换可以通过配置 force 参数实现。
![//imgs/v3/migration/migration-5.png](/imgs/v3/migration/migration-5.png)
### 3. 强制接口级和强制应用级互相切换


强制接口级和强制应用级互相切换需要临时创建一份新的订阅，判断新的订阅（即阈值计算时使用新订阅的地址数去除旧订阅的地址数）是否达标，如果达标则进行切换，如果不达标会销毁这份新的订阅并且回滚到之前的状态。
![//imgs/v3/migration/migration-6.png](/imgs/v3/migration/migration-6.png)
