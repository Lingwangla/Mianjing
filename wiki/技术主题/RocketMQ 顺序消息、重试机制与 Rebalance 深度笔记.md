# 1 RocketMQ 核心组件关系

## 整体架构

```text
Producer
    │
    ▼
Topic
    │
    ▼
Broker
    │
 ┌──┴───────────────┐
 ▼                  ▼
CommitLog       ConsumeQueue
(消息存储)         (消费索引)
                     │
                     ▼
              ConsumerGroup
                     │
               Rebalance
                     │
               Consumer
```

---

# 2 核心组件关系

## Producer

生产者。

负责：

- 创建消息
    
- 选择 Topic
    
- 选择 Queue
    
- 发送消息
    

例如：

```go
orderId % QueueNum
```

决定消息进入哪个 Queue。

---

## Topic

逻辑主题。

例如：

```text
order-topic
pay-topic
logistics-topic
```

一个 Topic 会被拆分成多个 Queue：

```text
order-topic
    ├── Queue0
    ├── Queue1
    ├── Queue2
    └── Queue3
```

### Topic 是逻辑概念

### Queue 才是并行度单位

---

## Broker

消息服务器。

负责：

- 存储消息
    
- 管理 Queue
    
- 管理 Offset
    
- Rebalance
    
- 主从同步
    

---

## ConsumerGroup

消费组。

例如：

```text
order-consumer-group
```

一个 Group 内包含多个 Consumer：

```text
Consumer1
Consumer2
Consumer3
```

作用：

> 横向扩展消费能力

---

## Consumer

真正执行：

```go
handle(message)
```

业务逻辑。

---

# 3 CommitLog 与 ConsumeQueue

RocketMQ 最核心的数据结构。

---

## CommitLog

真正消息存储。

所有 Topic 共用一个 CommitLog：

```text
CommitLog

msg1
msg2
msg3
msg4
msg5
```

特点：

- 顺序写
    
- append only
    
- 高性能
    

### 不区分 Topic 和 Queue

例如：

```text
order msg1
pay msg1
order msg2
```

混合存储。

---

## ConsumeQueue

消费索引。

每个 Queue 对应一个 ConsumeQueue：

```text
TopicA Queue0

ConsumeQueue
```

内部保存：

```text
CommitLogOffset
MessageSize
TagHash
```

本质：

> CommitLog 的索引

---

## 存储结构

```text
               CommitLog

┌─────────────────────────┐
│ msg1 msg2 msg3 msg4 msg5│
└─────────────────────────┘
      ▲      ▲
      │      │
┌─────┴──────┴─────┐
│   ConsumeQueue   │
└──────────────────┘
```

---

# 4 Queue 的本质

Queue 不是消息。

Queue = 逻辑消费分片。

包含：

- ConsumeQueue
    
- QueueOffset
    

真正消息仍然在：

```text
CommitLog
```

---

# 5 一条消息的完整生命周期

## Step1 Producer发送消息

```text
Producer
    │
    ▼
Topic
    │
    ▼
Queue
```

例如：

```go
queueId = orderId % 8
```

---

## Step2 Broker写CommitLog

```text
Broker
    │
    ▼
CommitLog
```

顺序追加：

```text
msg1
msg2
msg3
```

---

## Step3 构建ConsumeQueue

异步建立索引：

```text
CommitLogOffset
MessageSize
TagHash
```

---

## Step4 Rebalance

ConsumerGroup：

```text
3 Consumer
8 Queue
```

分配：

```text
Q0 → C1
Q1 → C2
Q2 → C3
```

---

## Step5 Consumer消费

先读取 ConsumeQueue：

得到：

```text
CommitLogOffset
```

再去：

```text
CommitLog
```

读取真正消息。

---

## Step6 消费成功

提交：

```text
offset
```

表示：

```text
已经消费到哪里
```

---

## 生命周期图

```text
Producer
    │
    ▼
Topic
    │
    ▼
Queue
    │
    ▼
Broker
    │
┌───┴────────────┐
▼                ▼
CommitLog    ConsumeQueue
                    │
                    ▼
             ConsumerGroup
                    │
              Rebalance
                    │
                Consumer
                    │
              handle(msg)
                    │
           ┌────────┴────────┐
           ▼                 ▼
        Success            Failed
```

---

# 6 RocketMQ 顺序消息原理

RocketMQ 只能保证：

> Queue级有序

不能保证：

> 全局有序

---

## Producer

相同 Key 路由到同一个 Queue：

```go
queue = orderId % QueueNum
```

例如：

```text
order123

create
pay
delivery
refund
```

全部进入：

```text
Queue7
```

---

## Broker

CommitLog 顺序写。

Queue 内天然有序。

---

## Consumer

顺序消费：

```java
MessageListenerOrderly
```

机制：

```text
一个 Queue
=
一个线程
```

```text
Queue7

msg1
msg2
msg3
```

串行执行。

---

# 7 Queue Lock机制

顺序消费时：

Consumer 需要向 Broker 申请：

```text
Queue Lock
```

只有获得锁：

才能消费。

保证：

```text
同一时刻

一个 Queue
只有一个 Consumer
```

---

# 8 顺序消费失败机制

返回：

```java
SUSPEND_CURRENT_QUEUE_A_MOMENT
```

作用：

暂停当前 Queue。

```text
msg1 ✓

msg2 ✗

msg3
不能消费
```

稍后重试：

```text
msg2
↓
成功
↓
msg3
```

特点：

> 顺序消费失败不会进入 Retry Topic，而是阻塞 Queue。

---

# 9 并发消费失败机制

返回：

```java
RECONSUME_LATER
```

Broker 接管失败消息：

```text
消费失败
    │
    ▼
%RETRY%ConsumerGroup
    │
    ▼
SCHEDULE_TOPIC_XXXX
    │
    ▼
重新投递
    │
    ▼
Consumer
```

---

## Retry Topic

自动创建：

```text
%RETRY%ConsumerGroup
```

例如：

```text
%RETRY%order-consumer-group
```

Retry Topic 是按 ConsumerGroup 隔离，而不是按 Topic 隔离。

---

## 最大重试次数

默认：

```text
16
```

超过后：

```text
%DLQ%ConsumerGroup
```

进入死信队列。

---

# 10 Rebalance导致乱序

假设：

```text
A正在消费 msg2
```

此时：

```text
ConsumerB上线
```

触发：

```text
Rebalance
```

Queue7：

```text
A → B
```

---

## 时间线

```text
t1

A消费msg2

t2

Rebalance

t3

Queue7分配给B

t4

B从旧offset重新消费msg2

t5

A完成msg2
```

结果：

```text
msg2重复消费
```

甚至：

```text
msg3先完成
```

产生短暂乱序窗口。

---

## 根因

Queue Lock释放：

与

Offset提交：

不是原子操作。

---

## RocketMQ设计哲学

```text
At Least Once
```

宁可：

```text
重复消费
```

也不允许：

```text
消息丢失
```

---

# 11 大厂保证强顺序方案

真正顺序保证：

不是 MQ。

而是：

```text
MQ顺序
+
状态机
+
CAS
+
幂等
```

共同完成。

---

## 状态机

```text
WAIT_PAY
↓
PAID
↓
DELIVERED
↓
FINISHED
```

非法流转：

```text
WAIT_PAY
↓
DELIVERED
```

直接拒绝。

---

## CAS更新

```sql
UPDATE orders
SET status='DELIVERED'
WHERE order_id=?
AND status='PAID'
```

只有：

```text
PAID
```

才能更新。

---

## 幂等

防止：

```text
重复消费
```

例如：

- Redis
    
- 唯一索引
    
- 去重表
    

---

# 12 工业级总结

RocketMQ 本质：

> 基于 CommitLog 的分布式日志系统。

其中：

```text
CommitLog
负责存储

ConsumeQueue
负责索引

Queue
负责分片

ConsumerGroup
负责消费扩展
```

消息生命周期：

```text
Producer
↓
CommitLog
↓
ConsumeQueue
↓
ConsumerGroup
↓
Consumer
↓
Success / Retry / DLQ
```

RocketMQ只能保证：

```text
Queue级有序
```

不能保证：

```text
业务强顺序
```

真正的大厂方案：

```text
RocketMQ顺序消息
        +
状态机
        +
CAS
        +
幂等
        +
补偿系统
```

最终实现：

> 局部强顺序 + 全局高可用 + 最终一致性。