# 一、ES 本质是什么？

一句话：

> ElasticSearch = 基于 Lucene 的分布式全文检索引擎

架构分层：

```text
应用层
  ↓
ElasticSearch（分布式能力）
  ↓
Lucene（索引搜索能力）
  ↓
Segment 文件
  ↓
磁盘
```

## 能力划分

| 能力 | 提供者 |
|------|--------|
| 倒排索引 | Lucene |
| BM25 打分算法 | Lucene |
| Segment 管理 | Lucene |
| 分片（Shard） | ES |
| 副本（Replica） | ES |
| 集群路由 | ES |
| REST API | ES |

---

# 二、ES 核心理念：倒排索引（Inverted Index）

## 1. 正排索引

传统数据库：

```text
id → document
```

例如：

```text
1 -> iphone is good
2 -> huawei is good
```

查询：

```sql
where content like '%iphone%'
```

需要：

```text
全表扫描
```

时间复杂度：

```text
O(n)
```

---

## 2. 倒排索引

存储形式：

```text
term → document list
```

例如：

```text
iphone → [doc1]

huawei → [doc2]

good → [doc1, doc2]
```

查询：

```text
iphone
    ↓
posting list
    ↓
得到 doc id
```

时间复杂度：

```text
O(log n)
```

---

## 3. 倒排索引组成

### Term Dictionary（词典）

保存所有 term：

```text
iphone
ipad
macbook
```

底层：

```text
FST（Finite State Transducer）
```

特点：

- 压缩存储
- 节省内存

---

### Posting List（倒排链表）

记录：

```text
term 出现在哪些文档
```

例如：

```text
iphone → [1,8,10]
```

包含：

| 信息 | 作用 |
|------|------|
| doc id | 哪个文档 |
| term frequency | 出现次数 |
| position | 位置 |
| offset | 偏移量 |

position 用于：

```text
短语查询
```

例如：

```text
"hello world"
```

与：

```text
"world hello"
```

结果不同。

---

# 三、为什么 ES 写入快？

核心理念：

> Segment Immutable（Segment 不可变）

旧 Segment 永远不会修改。

写入方式：

```text
新增 Segment
```

即：

```text
Append Only
```

---

## 1. 优势

避免随机 IO：

传统 B+Tree：

```text
插入
↓
页分裂
↓
随机 IO
```

ES：

```text
追加写
↓
顺序 IO
```

磁盘效率更高。

---

## 2. 写入流程

```text
client
    ↓
coordinator
    ↓
primary shard
    ↓
memory buffer
    ↓
translog
    ↓
refresh
    ↓
segment
```

---

## 3. Translog

作用：

> 防止 refresh 之前机器宕机导致数据丢失

类似：

```text
MySQL redo log
```

属于：

```text
WAL（Write Ahead Log）
```

---

## 4. Refresh

作用：

```text
buffer
↓
segment
```

默认：

```text
1 秒
```

特点：

> Near Real Time（准实时）

因此 ES 并非实时数据库。

---

## 5. Flush

作用：

```text
segment 持久化
+
清空 translog
```

---

## Refresh 与 Flush

| 对比项 | Refresh | Flush |
|--------|---------|-------|
| 写 Segment | √ | √ |
| fsync 磁盘 | 不一定 | √ |
| 清空 translog | × | √ |
| 开销 | 小 | 大 |

---

# 四、为什么 ES 查询快？

查询性能来自四部分：

---

## 1. 倒排索引

term → doc list

避免全表扫描。

---

## 2. Segment Immutable

Segment 不可变：

```text
读操作无锁
```

支持高并发。

---

## 3. Page Cache

ES 极度依赖：

```text
Linux Page Cache
```

热数据大部分驻留内存。

因此：

> ES 快，本质上是内存命中快。

---

## 4. Doc Values

倒排索引适合：

```text
term → doc
```

而排序、聚合需要：

```text
doc → field
```

于是 ES 引入：

```text
Doc Values
```

本质：

```text
列式存储
```

用于：

- sort
- aggregation
- script

---

# 五、分布式核心

## 1. Index 与 Shard

Index：

```text
逻辑概念
```

Shard：

```text
物理存储单元
```

本质：

```text
一个 Shard = 一个 Lucene 实例
```

---

## 2. 为什么要分片？

单机资源有限：

- CPU
- 内存
- 磁盘

因此：

```text
水平扩容
```

---

## 3. Primary Shard

负责：

```text
写入
```

例如：

```text
3 Primary
```

---

## 4. Replica Shard

副本作用：

### 高可用

```text
Primary 挂掉
↓
Replica 晋升
```

### 查询扩展

查询可走：

```text
Replica
```

提升 QPS。

---

# 六、ES 查询流程

采用：

```text
Query Then Fetch
```

流程：

```text
client
    ↓
coordinator node
    ↓
scatter
    ↓
所有 shard 查询 topK
    ↓
gather
    ↓
merge sort
    ↓
fetch document
```

---

## 为什么只返回 TopK？

避免：

```text
大量网络传输
```

先返回：

```text
doc id + score
```

再统一排序。

---

# 七、Segment Merge

由于不断 refresh：

```text
产生大量 Segment
```

过多 Segment 会导致：

- 文件句柄增加
- 查询次数增加
- Merge 成本上升

因此后台线程会：

```text
小 Segment
↓
合并
↓
大 Segment
```

称为：

```text
Segment Merge
```

特点：

```text
耗费大量 IO
```

---

# 八、深分页为什么慢？

传统分页：

```json
{
    "from":100000,
    "size":10
}
```

实际上：

每个 shard：

```text
取前 100010 条
```

Coordinator：

```text
merge sort
```

最后：

```text
丢弃前 100000 条
```

问题：

- CPU 消耗大
- 内存消耗大
- 网络开销大

---

## 优化方案

### search_after

适用于：

```text
无限滚动
```

---

### Scroll

适用于：

```text
数据导出
```

---

### PIT（Point In Time）

适用于：

```text
一致性分页
```

---

# 九、为什么 ES 不适合事务系统？

## 1. 准实时

默认：

```text
1 秒 refresh
```

无法保证实时可见。

---

## 2. 无事务

没有：

- MVCC
- 行锁
- rollback

不支持：

```text
ACID
```

---

## 3. 更新代价高

由于：

```text
Segment Immutable
```

更新本质：

```text
delete old
+
insert new
```

导致：

- 写放大
- merge 压力增大

---

## 4. 聚合耗费内存

尤其：

- fielddata
- global ordinals

容易：

```text
OOM
```

---

# 十、text 与 keyword

## text

特点：

```text
分词
```

适合：

- 全文搜索

例如：

```json
"title": {
    "type":"text"
}
```

---

## keyword

特点：

```text
不分词
```

适合：

- 聚合
- 排序
- 精确匹配

例如：

- id
- 手机号
- email

```json
"user_id":{
    "type":"keyword"
}
```

---

## 对比

| 类型 | 分词 | 排序 | 聚合 | 搜索 |
|------|-----|-----|-----|-----|
| text | √ | × | × | √ |
| keyword | × | √ | √ | 精确匹配 |

---

# 十一、高频面试题

---

## Q1：为什么 ES 比 MySQL Like 快？

### 原因：

1. 倒排索引
2. Term Dictionary
3. Posting List
4. Skip List
5. Segment Immutable
6. Page Cache

---

## Q2：Refresh、Flush、Merge 区别？

### Refresh

```text
buffer → segment
```

特点：

- 默认 1 秒
- 数据可搜索

---

### Flush

```text
segment 持久化
+
清空 translog
```

---

### Merge

```text
多个小 Segment
↓
合并成大 Segment
```

减少：

- Segment 数量
- 查询开销

---

## Q3：为什么 Segment 太多会影响性能？

原因：

每个 Segment 都需要：

- 打开文件
- 执行查询
- merge result

因此：

```text
Segment 越多
查询越慢
```

---

## Q4：为什么 ES 不建议频繁 Update？

因为：

```text
update
=
delete
+
insert
```

会造成：

- 磁盘写放大
- Merge 压力
- IO 增大

---

## Q5：一个 Shard 的本质是什么？

答案：

> 一个 Shard 本质上就是一个独立的 Lucene 实例。

---

## Q6：Replica 有什么作用？

### 高可用

```text
Primary 挂掉
↓
Replica 晋升
```

### 查询扩展

```text
Search 可以走 Replica
```

提高 QPS。

---

# 十二、ES 核心设计哲学

## 写优化

```text
Append Only
+
Immutable Segment
+
Batch Merge
```

---

## 读优化

```text
倒排索引
+
Page Cache
+
Skip List
+
Doc Values
```

---

## 分布式扩展

```text
Shard
+
Replica
+
Scatter Gather
```

---

# 十三、面试高级总结（推荐背诵）

> ElasticSearch 本质上是基于 Lucene 的分布式全文检索引擎。

其核心设计思想包括：

1. 使用倒排索引解决全文搜索问题；
2. 使用 Immutable Segment 和 Append Only 提升写入性能；
3. 使用 Shard 实现水平扩展；
4. 使用 Replica 实现高可用和查询扩展；
5. 使用 Refresh 实现 Near Real Time 搜索；
6. 使用 Merge 控制 Segment 数量；
7. 使用 Doc Values 支持聚合和排序。

因此 ES 本质上是搜索引擎，而非事务数据库，更适合：

- 商品搜索
- 日志检索
- 推荐系统召回
- 数据分析

而不适合：

- 高事务系统
- 强一致业务
- 高频更新场景


# ElasticSearch 高频面试笔记

---

# 问题：search_after 和 scroll 实现原理、区别和使用场景

## 为什么不能使用 from + size 深分页？

例如：

```json
{
  "from":100000,
  "size":10
}
```

假设：

- 5 个 Shard

每个 Shard 实际执行：

```text
取前 100010 条数据
```

Coordinator：

```text
5 个 shard 返回 500050 条记录
        ↓
merge sort
        ↓
丢弃前 100000 条
        ↓
返回最后 10 条
```

问题：

- CPU 消耗巨大
- 内存占用高
- 网络传输大量无效数据

因此：

> from + size 不适合深分页。

---

# 1 Scroll 原理

## 核心思想

> 创建 Search Context（查询快照）

第一次查询：

```http
GET index/_search?scroll=5m
```

ES：

### 创建 Search Context

保存：

- 当前 Segment Snapshot
- Query 条件
- 当前游标位置

生成：

```text
scroll_id
```

返回：

```json
{
    "_scroll_id":"xxxx"
}
```

---

后续：

```http
GET _search/scroll
```

携带：

```json
{
    "scroll_id":"xxxx"
}
```

ES：

不重新执行 Query。

而是：

```text
从上次游标继续读取
```

类似：

```text
数据库 Cursor
```

---

## Scroll 底层结构

```text
SearchContext
        ↓
固定 Segment Snapshot
        ↓
scroll_id
        ↓
继续读取
```

属于：

### Stateful（有状态）

---

## Scroll 为什么快？

后续分页：

不需要：

- 重新 Query
- 重新排序
- 重新计算 TopK

只是：

```text
scan next batch
```

复杂度：

```text
O(size)
```

---

## Scroll 的缺点

### Search Context 占用内存

每个 Scroll 都维护：

```text
Segment Snapshot
+
Cursor
```

如果：

```text
10000 用户同时 Scroll
```

会产生大量 Context。

---

### 阻塞 Merge

因为：

Scroll 引用了旧 Segment。

导致：

```text
Merge 完成后
老 Segment 无法删除
```

造成：

- 磁盘空间膨胀
- Merge 压力增加

因此：

> ES 官方不推荐使用 Scroll 做在线分页。

---

## Scroll 生命周期

```text
第一次查询
      ↓
创建 SearchContext
      ↓
返回 scroll_id
      ↓
多次 scroll
      ↓
clear scroll
      ↓
释放资源
```

---

# 2 search_after 原理

ES 5.x 后推荐方案。

核心思想：

> 利用排序值作为游标。

---

例如：

按照时间倒序：

```json
"sort":[
{
    "create_time":"desc"
}
]
```

第一页：

```json
{
    "size":3
}
```

结果：

```text
100
90
80
```

最后一条：

```text
80
```

返回：

```json
"sort":[80]
```

---

下一页：

```json
{
    "size":3,
    "search_after":[80]
}
```

ES：

直接定位：

```text
80 后面的记录
```

得到：

```text
70
60
50
```

本质：

类似：

```sql
where create_time < 80
limit 3
```

属于：

### Keyset Pagination（键集分页）

---

## search_after 工作流程

第一页：

```text
Query
 ↓
Sort
 ↓
返回数据
 ↓
记录最后一个 sort value
```

下一页：

```text
search_after(last_sort_value)
 ↓
重新执行 Query
 ↓
继续查
```

属于：

### Stateless（无状态）

---

## search_after 为什么资源消耗低？

因为：

不会保存：

- Search Context
- Segment Snapshot
- Cursor

每次：

```text
重新执行 Query
```

因此：

### 不阻塞 Merge

### 几乎不占内存

非常适合：

```text
高并发分页
```

---

## search_after 的问题

由于：

每次都会重新查询。

因此：

数据变化时：

可能出现：

### 数据丢失

第一页：

```text
100
90
80
```

此时插入：

```text
95
```

第二页：

```text
search_after(80)
```

得到：

```text
70
60
50
```

95 被漏掉。

---

### 数据重复

删除数据时：

可能导致重复。

因此：

> search_after 不保证一致性。

---

# 3 PIT + search_after

ES7 推荐方案。

流程：

创建：

```http
POST index/_pit
```

得到：

```text
pit_id
```

查询：

```json
{
  "pit":{
      "id":"pit_id"
  },
  "search_after":[80]
}
```

特点：

### Segment 固定

保证：

- 不重复
- 不丢失

同时：

### 不维护 Scroll Context

资源占用远小于 Scroll。

因此：

> 官方推荐：

```text
PIT + search_after
```

替代：

```text
Scroll
```

---

# Scroll 与 search_after 对比

| 对比项 | Scroll | search_after |
|--------|--------|--------|
| 是否保存上下文 | √ | × |
| 是否固定 Segment | √ | × |
| 是否重新执行 Query | × | √ |
| 是否阻塞 Merge | √ | × |
| 内存消耗 | 高 | 极低 |
| 一致性 | 强 | 弱 |
| 高并发能力 | 差 | 强 |
| 官方推荐程度 | 不推荐分页 | 推荐 |
| 本质 | Cursor | Keyset Pagination |

---

# 使用场景

## Scroll

适合：

### 数据导出

```text
导出千万订单
```

### Reindex

```text
索引迁移
```

### 离线批处理

```text
ES → Hive
ES → ClickHouse
```

特点：

> 一次性任务。

---

## search_after

适合：

### 无限下拉

- 商品列表
- 订单列表
- Feed 流

### 高并发分页

例如：

```text
朋友圈
推荐系统
时间线
```

---

## PIT + search_after

适合：

```text
百万订单后台分页
```

保证：

- 不重复
- 不漏数据
- 高并发

---

# 面试标准答案

> Scroll 会创建 SearchContext，并固定当前 Segment，通过 scroll_id 维护游标，因此属于有状态分页，可以保证一致性，但会占用内存并阻塞 Segment Merge，适合数据导出和离线任务。
>
> search_after 利用上一页最后一条记录的 sort value 作为游标，每次重新执行 Query，不保存上下文，因此属于无状态分页，资源占用极低，适合高并发无限下拉场景。
>
> ES7 之后官方推荐使用 PIT + search_after，兼顾一致性和资源利用率。

---

---

# 二、为什么 ES 比 MySQL Like 快？

---

# MySQL Like 为什么慢？

例如：

```sql
select *
from product
where title like '%iphone%'
```

由于：

```text
前导模糊匹配
```

导致：

### B+Tree 索引失效

只能：

```text
全表扫描
```

流程：

```text
读第1行
↓
字符串比较

读第2行
↓
字符串比较

...

读1000万行
```

时间复杂度：

```text
O(N)
```

---

## CPU 开销大

每一行都需要：

```cpp
strstr(title,"iphone")
```

复杂度：

```text
O(N × 字符串长度)
```

---

## 磁盘 IO 大

Buffer Pool 不可能缓存全部数据。

因此：

大量：

```text
磁盘 IO
+
字符串比较
```

导致查询缓慢。

---

# ES 为什么快？

核心思想：

> 空间换时间

提前构建：

```text
term → doc list
```

避免扫描所有文档。

---

# 1 倒排索引

文档：

```text
doc1
iphone is good

doc2
huawei is good

doc3
iphone 16 pro
```

建立：

```text
iphone
↓
[1,3]

good
↓
[1,2]

huawei
↓
[2]
```

查询：

```text
iphone
↓
Posting List
↓
DocID
```

复杂度：

```text
O(logN)
```

---

# 2 FST（Term Dictionary）

词典：

```text
iphone
iphone16
iphone16pro
ipad
ipadpro
```

共享公共前缀：

构成：

```text
FST
```

特点：

### 压缩率高

### 内存占用小

### 查询接近 O(1)

---

# 3 Posting List

保存：

```text
DocID
TF
Position
Offset
```

支持：

### Phrase Query

```text
"iphone 16"
```

### Highlight

高亮显示。

---

# 4 Skip List

多关键词查询：

```text
iphone AND good
```

通过：

```text
Skip List
```

快速跳跃。

复杂度：

接近：

```text
O(logN)
```

而不是：

```text
O(N)
```

---

# 5 Segment Immutable

Lucene：

采用：

```text
Immutable Segment
```

特点：

### 查询无锁

支持：

```text
高并发读
```

避免：

B+Tree 页分裂导致的锁竞争。

---

# 6 Page Cache

Segment：

通过：

```text
mmap
```

映射到：

```text
Linux Page Cache
```

热数据：

基本驻留内存。

访问延迟：

```text
100ns
```

SSD：

```text
100us
```

速度差：

```text
1000 倍
```

所以：

> ES 快，本质上是内存快。

---

# 7 Doc Values

用于：

### Sort

```json
sort:
{
    "price":"asc"
}
```

### Aggregation

例如：

```json
terms aggregation
```

本质：

```text
列式存储
```

提升：

- CPU Cache 命中率
- 聚合效率

---

# 8 BM25

MySQL：

```text
匹配 / 不匹配
```

没有相关性排序。

ES：

采用：

### BM25

考虑：

- TF
- IDF
- Field Length

得到：

```text
score
```

返回：

最相关结果。

---

# 9 分布式能力

Shard：

```text
水平扩展
```

查询：

```text
Scatter
 ↓
多个节点并行执行
 ↓
Gather
```

理论：

QPS：

接近：

```text
N 倍扩展
```

---

# 本质区别

## MySQL

优化目标：

```text
事务
ACID
高并发写
```

数据结构：

```text
B+Tree
```

擅长：

- 精确查询
- 范围查询

---

## ES

优化目标：

```text
全文检索
```

数据结构：

```text
倒排索引
```

擅长：

- 模糊查询
- 关键词搜索
- 聚合分析
- 相关性排序

---

# 面试标准答案

> MySQL 使用 B+Tree，适合精确查询和范围查询，对于 like '%xxx%' 无法利用索引，只能全表扫描，时间复杂度为 O(N)，同时伴随大量磁盘 IO 和字符串比较。
>
> ES 基于 Lucene 倒排索引，通过 FST 构建 term dictionary，通过 Posting List 和 SkipList 快速定位文档，并利用 Segment Immutable、Page Cache、Doc Values 和 BM25 提升查询性能，再结合 Shard 的分布式能力，使其在全文检索场景下远优于 MySQL Like。
>
> 因此在实际业务中通常采用：

```text
MySQL
负责事务

ES
负责搜索
```

形成：

```text
MySQL + ES
```

双写架构。