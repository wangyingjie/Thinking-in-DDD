# 幂等设计

## 1. 幂等的定义

幂等就是进行多次重复操作的结果和只进行一次操作的结果相同。

读请求不会导致数据的变更，因此读请求天然是幂等的。

写请求会导致数据状态的变更，因此需要考虑其幂等性：

写请求最后都可归纳到 SQL 的 insert/update/delete 这几个操作，在不做额外处理的情况下，它们的幂等性如下：

- insert：如果没有唯一性约束，每次 insert 时都使用数据库的自增主键，那么每次都新增新的记录，则不是幂等的

```sql
-- 非幂等，每执行一次会新增一条记录
insert into t(count) values (1);

-- id为唯一主键，则不管执行多次，最终都只会有一条id=1的记录，因而是幂等的
insert into t(id,count) values(1,2);

```

- update：不依赖历史状态的update操作是幂等的，如果需要依赖历史状态进行计算，则不是幂等的。

```sql

-- 幂等操作，直接设置结果，不管设置几次，count都是1
update t set count=1 where id=1;

-- 非幂等操作，最终count的值跟执行次数有关
update t set count=count+1 where id =1;

```

- delete：幂等操作，执行一次和执行多次的结果是一样的

```sql

delete from t where id =1;

```

之所以在领域事件之前先讲解幂等设计，是因为消费领域事件时，有可能因为多种原因需要进行重试，如果重试的逻辑不是幂等的，往往会导致错误的结果，比如多次扣款造成资损。

## 2.如何实现幂等

### 2.1 数据库级别

要求调用方在必须发起操作请求时必须携带RuequestId，服务端将每个请求落库保存到操作流水表中，并且在RuequestId对应的列上增加唯一索引。这样同一个RuequestId的操作请求由于数据库唯一键冲突无法落库保存，同时Java代码中会产生异常，捕获异常后直接返回重复执行提示或者丢弃消息，以此确保幂等性。

数据库级别的幂等控制虽然看起来比较重量级，但是可以作为最终的兜底方案，在唯一索引之前可以在代码中引入其他的幂等确认逻辑，减少写库。

对于重要的资金交易场景，一般都会有操作流水表作为兜底。

### 2.2 状态机

对于有限状态机类型的数据，如果状态机已经处于某一个状态，这时候收到上一个状态下的变更请求，我们就可以直接拒绝处理并返回提示。

例如账号的激活状态就是一个简单的状态机，可以简单的包括未激活、已激活、已失效这几种状态。

```java

public enum ActiveState {
    Awaiting_Activation(0, "待激活"),
    Activated(1, "已激活"),
    Expired(2, "已失效");

    private final Integer code;
    private final String name;

    ActiveState(Integer code, String name) {
        this.code = code;
        this.name = name;
    }
}

```

如果账号已经处于“已失效”状态，这个时候收到账号修改信息的请求，我们可以直接丢弃消息拒绝处理。

在数据库层面，也可以对 SQL 做一些限制，例如：

```sql
-- 只有active_state=1才会执行
update t set active_state=2 where id=1 and active_state=1;
```

### 2.3 token 机制

在客户端执行某个操作时，先向服务端申请一个 token，服务端生成 token 并缓存；

客户端在发起操作请求时，带上这个 token；

服务端判断该 token 是否在缓存中，如果不在则认定该 token 已使用，将拒绝执行业务逻辑并返回提示；

如果服务端判断缓存中存在该 token，则认为该 token 未使用过，先移除缓存中的 token 再继续执行业务逻辑。

<!--@include: ../footer.md-->
