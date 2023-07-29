---
layout: post
title: 策略模式
date: 2021-04-10
Author: bwt
categories: 设计模式
tags: [设计模式, 策略模式]
comments: true
toc: true
---

#### 一. 引入

在项目开发的过程中, 遇到过这样的需求: 设计一个"拆盲盒"的活动, 用户可以通过消耗抽奖次数进行抽奖, 奖品池中有不同的奖励, 例如: 头像框, 
账户零钱, 额外抽奖次数等, 抽到对应的奖励后自动执行入账.

不同的奖品入账需要调用不同的服务, 那么在入账的逻辑中, 可能会有这样的代码:

<!--break-->

```java
public void reward(Reward reward) {

    if ("AVATAR_DRESS".equals(reward.getRewardType())) {
        System.out.println("奖励头像框...");
    } else if ("MONEY".equals(reward.getRewardType())) {
        System.out.println("零钱入账...");
    } else if ("FREE_LOTTERY".equals(reward.getRewardType())) {
        System.out.println("额外抽奖次数...")
    } else {
        System.out.println("other...")
    }
    
}
```

这样做有以下问题:

1. 如果后面新增或者减少奖品, 就需要在代码中对应地新增或减少 if...else 语句, 这样违反了开闭原则(对扩展开启, 对修改关闭);
2. 代码中包含大量 if...else / switch...case, 逻辑冗长且不利于后期维护.

#### 二. 优化

**策略模式**是一种行为设计模式, 在此模式中, 一个类的行为可以在运行时确定或修改.

##### 1. 应用场景 

* 希望对象在运行过程中有不同的算法变体, 并实现在运行时切换, 可使用此模式;
* 将同一种行为的不同策略单独抽取到一个独立类层次结构中, 减少重复代码, 逻辑更加简洁;
* 如果算法在上下文逻辑中不是特别重要, 使用该模式可以将业务逻辑与算法隔离.

##### 2. 实现方式

其实现方式是: 将一个公共(或修改频率较高)的算法抽取到一个被称为策略的基类中, 此外, 还需要一个名为 *context(上下文)* 的对象, 上下文独立于策略, 它并不负责策略
的选取和具体的执行逻辑, 而是持有具体策略的引用, 并对外提供一个触发入口, 将任务委派给持有的策略对象去执行, 这样就可以在不修改上下文的
情况下增, 删或者修改已有的算法逻辑了.

以"拆盲盒"的入账场景为例, 使用策略模式优化:

(1) 我们将奖励入账的逻辑抽离出来, 定义在 RewardStrategy 基类中(接口)

```java
public interface RewardStrategy {

    /**
     * 所有的奖励入账都会走此方法
     * @param rewardDto 奖品入账的相关信息封装, 例如: 用户uid, 奖品id, 奖励个数, 有效时间等 
     */
    void reward(RewardDto rewardDto);
    
}
```

(2) 编写对应的子策略, 以入账零钱为例, 我们可以定义 RewardMoneyStrategy, 然后实现基类的 reward 方法(其他策略同理):

```java
public class RewardMoneyStrategy implements RewardStrategy {
    
    @Override
    public void reward(RewardDto rewardDto) {
        // 这里是零钱入账的具体逻辑...
    }
    
}
```

(3) 定义一个 context 上下文对象. 

```java
public class RewardContext {
    
    private RewardStrategy strategy;
    
    public RewardContext(RewardStrategy strategy) {
        this.strategy = strategy;
    }
    
    
    public void executeStrategy(Reward reward) {
        RewardDto rewardDto = new RewardDto();
        // convert Reward to RewardDto
        strategy.reward(rewardDto);
    }
    
}
```

(4) 修改最初的 if...else 代码

```java
public void reward(Reward reward) {
    RewardStrategy strategy = null;
    
    if ("AVATAR_DRESS".equals(reward.getRewardType())) {
        // 奖励头像框策略实现
        strategy = new RewardAvatarDressStrategy();
    } else if ("MONEY".equals(reward.getRewardType())) {
        // 零钱入账
        strategy = new RewardMoneyStrategy();
    } else if ("FREE_LOTTERY".equals(reward.getRewardType())) {
        // 奖励额外抽奖次数
        strategy = new RewardFreeLotteryStrategy();
    } else {
        // ...
    }
    
    RewardContext context = new RewardContext(strategy);
    context.executeStrategy(reward);
}
```

至此, 策略模式的封装已经完成, 类之间的关系如图:

![UML 类图](https://zonheng.net/tech/strategy_design_pattern.png-original)

我们发现, 在入账时还有一些 if...else 判断, 还能如何优化呢?

(5) 为了简化 if...else, 我们可以设计一个策略工厂(此处使用了工厂模式), 这个工厂根据奖励的不同类别, 返回不同的策略:

```java
public class RewardStrategyFactory {
    
    private static final HashMap<String, RewardStrategy> STRATEGY_MAP;
    
    static {
        STRATEGY_MAP = new HashMap<>();

        STRATEGY_MAP.put("AVATAR_DRESS", new RewardAvatarDressStrategy());
        STRATEGY_MAP.put("MONEY", new RewardMoneyStrategy());
        STRATEGY_MAP.put("FREE_LOTTERY", new RewardFreeLotteryStrategy());
    }
    
    public static RewardStrategy getRewardStrategy(String rewardType) {
        return STRATEGY_MAP.get(rewardType);
    }
    
}
```

(6) 继续优化 if...else

```java
public void reward(Reward reward) {
    RewardStrategy strategy = RewardStrategyFactory.getRewardStrategy(reward.grtRewardType());

    RewardContext context = new RewardContext(strategy);
    context.executeStrategy(new RewardDto());
}
```

优化完成, 今后如果有更改入账逻辑, 增删奖品等需求, 可以直接修改对应的策略, 以及策略工厂 RewardStrategyFactory, 调用方的逻辑无需更改.

##### 3. 优缺点分析

优点

* 符合开闭原则, 无需修改上下文即可修改或引入策略;
* 可以将算法的调用方和具体的算法执行隔离;
* 可以在运行时修改对象的行为

缺点

* 如果算法极少发生改变, 那么没有理由引入新的接口或类, 该模式可能让代码变得复杂;
* 客户端需要知晓并选取策略;
* 如果策略过多可能会有类膨胀的问题, 可以考虑混合模式.

#### 三. 其他

##### 1. 还有优化的空间吗?

我们上面在 `RewardStrategyFactory` 中, 将奖品的类别和对应的策略 `STRATEGY_MAP` 固定写死, 有啥办法可以再优雅一点吗? :dog:

如果你的项目集成了 spring, 那么可以这样做: 在基类 `RewardStrategy` 中新增一个 `getRewardType()` 方法, 代表该策略的类型:

```java
public interface RewardStrategy {

    /**
     * 所有的奖励入账都会走此方法
     * @param rewardDto 奖品入账的相关信息封装, 例如: 用户uid, 奖品id, 奖励个数, 有效时间等 
     */
    void reward(RewardDto rewardDto);

    /**
     * 返回该策略的类型
     * @return
     */
    String getRewardType();
    
}
```

然后在每个子策略中, 实现 `getRewardType()` 方法, 还是以零钱入账的子策略为例(其实就是把 `RewardStrategyFactory` 中的常量 `MONEY / 
AVATAR_DRESS / FREE_LOTTERY` 分散到各个子策略中去), 除此之外, 注意我们添加了 `@Service` 注解, 即: 将 bean 交给 spring 容器管理:

```java
@Service
public class RewardMoneyStrategy implements RewardStrategy {
    
    @Override
    public void reward(RewardDto rewardDto) {
        // 这里是零钱入账的具体逻辑...
    }
    
    @Override
    public String getRewardType() {
        return "MONEY";
    }
    
}
```

继续修改 `RewardStrategyFactory`, 思路是: 将 `RewardStrategyFactory` 交给 spring 容器托管(`@Component` 注解), 在 spring 
将 bean 初始化完成后, 获取 `RewardStrategy` 接口的所有实现类, 然后根据 `getRewardType()` 方法将策略分类并转换成 map.

```java
// 1. @Component: 将 RewardStrategyFactory 交给 spring 容器托管
@Component
public class RewardStrategyFactory implements ApplicationContextAware {

    private ApplicationContext applicationContext;

    private Map<String, RewardStrategy> STRATEGY_MAP;

    // 2. @PostConstruct: 当 spring 容器将 bean 初始化完成后, 执行此方法
    @PostConstruct
    public void init() {
        // 3. 获取 RewardStrategy 接口的所有实现类(RewardMoneyStrategy / RewardAvatarDressStrategy / RewardFreeLotteryStrategy)
        Map<String, RewardStrategy> beansOfTypesMap = applicationContext.getBeansOfType(RewardStrategy.class);
        // 4. 我们只需要 beansOfTypesMap 中的 values, 将其转化为我们需要的类型
        STRATEGY_MAP = beansOfTypesMap.values().stream().collect(Collectors.toMap(RewardStrategy::getRewardType, Function.identity()));
    }


    public RewardStrategy getRewardStrategy(String rewardType) {
        return STRATEGY_MAP.get(rewardType);
    }


    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }
    
}
```

最后, 对于 client 调用方, 做出如下更改:

```java
// 注入 RewardStrategyFactory
@Resource
private RewardStrategyFactory rewardStrategyFactory;

public void reward(Reward reward) {
    RewardStrategy strategy = rewardStrategyFactory.getRewardStrategy(reward.grtRewardType());

    RewardContext context = new RewardContext(strategy);
    context.executeStrategy(new RewardDto());
}
```


##### 2. context 在策略模式中起什么作用?

context 可以将调用方(client)与具体策略(strategy)之间解耦, 可以理解为充当黑盒的作用. 如果没有 context, 如果想要更改策略接口, 
那么对应地, 调用方 client 也要随之更改.

在大多数开发中，我们的需求往往会更加复杂, 实际较为常见的情况就是：上层的调用需要与接口之间有一定的交互(可能是一些属性，或是一些方法),
这样的交互往往会让接口变的难以调用. 于是上下文的引入就是势在必行, 将相关的属性或一些公共的方法封装到上下文中, 让上下文去和接口进行
复杂的交互, 而上层的调用只需要跟上下文打交道就可以.

#### 四. 参考

* [策略设计模式](https://refactoringguru.cn/design-patterns/strategy)
* [策略模式 - 菜鸟教程](https://www.runoob.com/design-pattern/strategy-pattern.html)
* [责任链模式与策略模式在售后系统里的实战](https://mp.weixin.qq.com/s/H2hylN0fBm_voUrNxkopFg)
* [Why we need Context class in strategy design pattern?](https://stackoverflow.com/questions/12545305/why-we-need-context-class-in-strategy-design-pattern)
* [策略模式, 上下文与内部类的思考](https://blog.csdn.net/kingsonyoung/article/details/50616072)