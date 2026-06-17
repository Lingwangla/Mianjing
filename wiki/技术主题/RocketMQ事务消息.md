## 一、事务消息的核心思想

RocketMQ 事务消息（Transactional Message）是一种实现 **分布式最终一致性** 的方案。

其核心思想：

> **通过「半消息（Half Message） + 本地事务 + 事务回查」实现最终一致性，而不是强一致性。**

整体流程：

```text
发送半消息
      ↓
执行本地事务
      ↓
提交/回滚事务消息
      ↓
消费者消费
      ↓
异常情况下 Broker 发起事务回查
```

---

## 二、事务消息整体流程

### Step1：发送 Half Message（半消息）

Producer 先发送一条 Prepare 消息：

```text
Producer
    ↓
Broker（存储 Half Message）
```

此时：

- 消息已经写入 Broker
- 消费者不可见
- 状态：

```text
TRANSACTION_NOT_TYPE
```

即：

> 消息暂存，但尚未决定是否生效。

---

### Step2：执行本地事务

例如电商下单：

```go
// 创建订单
createOrder()

// 扣减库存
deductStock()
```

这里执行的是：

- MySQL 本地事务
- RocketMQ 不参与数据库事务

---

### Step3：提交或回滚事务消息

根据本地事务执行结果：

#### 成功

发送：

```text
commitMessage
```

Broker：

```text
Half Message
      ↓
Commit
      ↓
消息可消费
```

---

#### 失败

发送：

```text
rollbackMessage
```

Broker：

```text
Half Message
      ↓
Rollback
      ↓
删除消息
```

---

### Step4：消费者消费消息

只有 Commit 之后：

```text
Broker
    ↓
Consumer
```

消费者才能收到消息。

---

### Step5：事务回查机制

若 Producer 在执行完本地事务后宕机：

例如：

```text
本地事务成功
    ↓
Producer挂掉
    ↓
commit消息没有发送成功
```

此时 Broker 无法判断：

- 应该提交？
- 还是回滚？

于是：

Broker 会主动发起事务回查：

```text
checkLocalTransaction()
```

Producer 通过查询数据库状态返回：

```go
func CheckLocalTransaction(msg Message) TransactionStatus {

    // 查询订单状态

    if success {
        return COMMIT
    }

    if fail {
        return ROLLBACK
    }

    return UNKNOWN
}
```

返回状态：

### COMMIT

```text
提交消息
```

### ROLLBACK

```text
删除消息
```

### UNKNOWN

继续等待，下次再回查。

---

# 三、事务消息时序图

```text
                 Producer                      Broker                     Consumer
                     │                           │                           │
                     │----- Half Message ------->│                           │
                     │                           │                           │
                     │---- 执行本地事务 ----------│                           │
                     │                           │                           │
                     │------ Commit ------------>│                           │
                     │                           │                           │
                     │                           │------ 投递消息 ---------->│
                     │                           │                           │
                     │                           │<------ 消费完成 ----------│
                     │                           │                           │
                     │<------ 回查（异常）--------│                           │
```

---

# 四、为什么不使用 2PC？

传统分布式事务：

## 两阶段提交（2PC）

问题：

- 协调者阻塞
- 性能差
- 可用性低

### RocketMQ 的方案

采用：

```text
半消息
+
本地事务
+
异步回查
```

实现：

> 最终一致性（Eventually Consistent）

属于一种：

> BASE 理论方案。

---

# 五、事务消息状态

RocketMQ 内部主要存在三种状态：

| 状态 | 含义 |
|------|-----|
| TRANSACTION_NOT_TYPE | 半消息 |
| COMMIT_MESSAGE | 提交成功 |
| ROLLBACK_MESSAGE | 回滚 |

状态流转：

```text
                Half Message
                     │
         ┌───────────┴───────────┐
         │                       │
     COMMIT                 ROLLBACK
         │                       │
    可被消费                  删除消息
```

---

# 六、事务回查机制

Broker 后台线程会周期性扫描：

```text
RMQ_SYS_TRANS_HALF_TOPIC
```

发现：

```text
长时间未提交的 Half Message
```

则发起：

```text
checkLocalTransaction()
```

查询 Producer 本地事务状态。

---

### 回查次数

默认：

```text
15 次
```

超过最大次数：

```text
自动 Rollback
```

防止大量 Half Message 堆积。

---

# 七、事务消息生命周期

```text
发送 Half Message
          ↓
Broker 持久化
          ↓
执行本地事务
          ↓
Commit/Rollback
          ↓
CommitQueue
          ↓
ConsumeQueue
          ↓
Consumer 消费
          ↓
消费成功
```

异常路径：

```text
Half Message
      ↓
Producer宕机
      ↓
Broker回查
      ↓
查询数据库状态
      ↓
Commit / Rollback
```

---

# 八、典型应用场景

## 场景1：订单创建 + 发消息

业务流程：

```text
创建订单
      ↓
发送订单创建成功事件
      ↓
积分系统
优惠券系统
营销系统
```

保证：

> 订单成功，消息一定发送。

避免：

```text
订单失败
消息却发送成功
```

---

## 场景2：支付成功 + 发券

流程：

```text
支付成功
      ↓
发送事务消息
      ↓
优惠券服务发券
```

保证：

```text
支付成功
⇔
发券消息成功
```

---

## 场景3：支付成功 + 增加积分

```text
支付服务
    ↓
事务消息
    ↓
积分中心
```

实现：

最终一致性。

---

## 场景4：订单完成 + 库存释放

```text
订单取消
    ↓
事务消息
    ↓
库存服务恢复库存
```

---

# 九、事务消息 VS 本地消息表

## 本地消息表方案

流程：

```text
业务表
+
消息表
```

事务提交：

```text
订单成功
消息记录成功
```

后台线程：

```text
扫描消息表
↓
发送 MQ
```

### 优点

- 可控性强
- 与 MQ 无关

### 缺点

- 侵入业务
- 多维护一张消息表

---

## RocketMQ事务消息

由 MQ 自身实现：

```text
Half Message
+
事务回查
```

### 优点

- 无需消息表
- 接入简单
- 中间件保证一致性

### 缺点

- 依赖 RocketMQ
- Producer 必须实现回查接口

---

### 对比

|方案|优点|缺点|
|---|---|---|
|2PC|强一致|性能差、阻塞|
|本地消息表|稳定可靠|业务侵入大|
|RocketMQ事务消息|实现简单|依赖 MQ|
|最大努力通知|成本低|可靠性一般|

---

# 十、事务消息会重复消费吗？

会。

RocketMQ 保证：

```text
At Least Once
```

即：

> 至少消费一次。

因此消费者必须实现：

### 幂等性

常见方案：

#### 唯一业务ID

例如：

```text
order_id
```

#### Redis 去重

```text
SETNX(order_id)
```

#### 数据库唯一索引

```sql
UNIQUE(order_id)
```

#### 状态机控制

```text
INIT
 ↓
PROCESSING
 ↓
SUCCESS
```

---

# 十一、事务回查失败怎么办？

Broker 会：

```text
第一次回查
第二次回查
...
第15次回查
```

如果仍然：

```text
UNKNOWN
```

则：

```text
自动 Rollback
```

因此：

业务系统必须保证：

- 可重试
- 可补偿
- 幂等

---

# 十二、RocketMQ事务消息本质

其设计思想：

```text
发送半消息
        +
执行本地事务
        +
Commit确认
        +
Broker回查补偿
```

实现：

```text
最终一致性
（Eventually Consistent）
```

而不是：

```text
强一致性
（Strong Consistency）
```

---

# 十三、一句话总结（面试标准答案）

> RocketMQ 事务消息通过 **Half Message + 本地事务 + Broker 回查机制**，实现分布式系统中的最终一致性。消息先以半消息形式写入 Broker，本地事务执行成功后再 Commit，若 Producer 异常导致状态未知，Broker 会主动回查本地事务状态，最终决定 Commit 或 Rollback，从而保证业务数据和消息发送的一致性。

---

# 十四、面试高频追问

### Q1：为什么 RocketMQ 不采用 2PC？

答：

- 阻塞严重
- 可用性差
- 性能开销大

RocketMQ 采用：

```text
异步补偿 + 最终一致性
```

---

### Q2：事务消息一定可靠么？

不是。

RocketMQ 保证：

```text
At Least Once
```

属于：

```text
最终一致性
```

而非：

```text
强一致性
```

---

### Q3：消费者为什么必须幂等？

因为：

```text
消息可能重复投递
```

因此：

```text
业务处理必须支持重复执行。
```

---

### Q4：事务消息适合哪些业务？

适合：

- 创建订单 + 发消息
- 支付成功 + 发券
- 支付成功 + 积分
- 订单取消 + 恢复库存
- 用户注册 + 发欢迎礼包

本质：

> 一个本地事务成功之后，需要可靠地通知其它系统。