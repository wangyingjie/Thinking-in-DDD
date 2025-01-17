# 聚合内事务实现

## 1. Repository 的事务控制

为了职责更明确以及保证系统性能和稳定性，我们把数据库事务放在 Repository 中。

Application 层示例代码如下：

```java
//示例方法，修改标题
public void modifyTitle(ArticleModifyTitleCmd cmd) {
    ArticleEntity entity = domainRepository.load(new ArticleId(cmd.getArticleId()));
    entity.modifyTitle(new ArticleTitle(cmd.getTitle()));
    domainRepository.save(entity);
}
```

Repository 层示例代码如下：

```java
@Repository
@Slf4j
public class ArticleDomainRepositoryImpl implements ArticleDomainRepository {
    //略去属性
    @Override
    @Transactional
    public void save(ArticleEntity entity) {
        //略
    }
}
```

## 2. 并发更新数据一致性

Application 层通过 domainRepository 从数据库加载 ArticleEntity（聚合根），然后 ArticleEntity 修改自己的 title 属性，之后 domainRepository 把修改后的结果保存到数据库中。

单个请求这样做是没有问题的，但是如果同时多个请求并发执行这个更新操作就会有数据一致性问题了。

我们举个例子，假设数据库有一张表，表内有个 count 字段，用来表示某种产品的数量，现在有两个请求 A 和 B，分别同时对这个 count 字段进行修改，A 请求对 count 加 1，B 请求对 count 加 2，预期的最终结果应该是 3。

假设原来 count 的值为 0，A 请求希望把 count 的值加 1，A 请求将聚合根加载到内存之后，由聚合根执行`count=count+1`的业务操作，执行成功之后 A 请求中 count=1，由于 GC 或者网络等原因 A 请求执行比较慢，还没来得及保存到数据库；

在 A 执行的同时来了一个 B，B 请求想把 count 的值加 2，由于 A 请求还没有写库成功，所以 B 请求读取的 count 字段的值还是 0，B 请求将聚合根加载到内存之后，由聚合根执行`count=count+2`的业务操作，此时 B 请求中 count=2，通过 domainRepository.save()保存到数据库，数据库中 count 字段的值为 2；

这时候，A 请求终于开始执行 domainRepository.save()这个事务操作了，由于 A 请求中 count 字段被设置为 1，所以 A 执行完之后，数据库 count 字段变成了 1，并不是我们预期的 count=3，说明我们执行出了问题。

执行示意图：

![缺乏乐观锁的并发更新问题](https://s1.ax1x.com/2023/04/22/p9VSVwd.png)

为了解决这个问题，我们引入数据库乐观锁，在数据库表中增加一个 version 字段，每次修改数据时必须对比聚合根的 version 与数据库行记录中的 version 是否相等：如果相等则执行修改操作，并将 version 都加 1；如果不等，抛出异常终止本次操作，或者在 Application 层加入重试逻辑，重新加载数据，经过一系列校验合法后进行计算。

我们可以在项目中引入 Spring Retry 组件然后在 Application 层进行重试。之所以在 Application 层使用重试，是因为 domainRepository.save（）执行失败时，我们需要重新加载聚合根以及 version 的值，重新再走一遍业务逻辑。

以下是使用注解式重试的示例：

```java
@Retryable(value = OptimisticLockingFailureException.class, maxAttempts = 2)
public void modifyTitle(ArticleModifyTitleCmd cmd) {
    ArticleEntity entity = domainRepository.load(new ArticleId(cmd.getArticleId()));
    entity.modifyTitle(new ArticleTitle(cmd.getTitle()));
    domainRepository.save(entity);
}
```

使用重试时，主要需要注意以下几点：

- 重试涉及到的相关服务要支持幂等操作。例如依赖的上游业务接口必须支持幂等，重试操作发起的重复接口调用不应被上游视为新的业务请求。

- 重试不适合频繁更新的热点数据。因为这样会造成其他请求的频繁重试，反而会降低性能。

- 重试的次数要提前规划好。一般根据响应时间的要求来确定重试的次数。例如一个接口如果有 RPC 调用，重试多次会造成响应时间过长，并且也会给 RPC 服务提供者造成流量压力，一般推荐重试一次即可。如果需要重试多次，说明待更新的数据锁竞争很激烈，建议考虑其他的实现方式。

- 重试的次数最好可配置。主要用于数据库压力过大时进行降级，可以调整重试次数，例如执行失败不再重试，以达到保护数据库的目的。要规划好什么异常才会重试，一般是数据库的乐观锁异常才会重试，其他的异常（如 RPC 调用异常）是否重试则要根据实际情况进行分析。

## 3. 数据库读写的性能问题

我们每次都从数据库加载数据，封装完整的聚合根执行业务逻辑，然后将状态更新到数据库，过程中如果涉及到读写大字段，这样做确实会造成性能问题。

针对读取可能存在的性能问题，我们可以采用懒加载（Lazy Load）的方式，只有真正使用的时候才会加载某个字段，这样可以避免加载聚合根时把大字段加载到内存；

针对写入可能存在的性能问题，我们可以采用脏跟踪（Dirty Tracing）的方式，只有真正被修改了的字段才更新到数据库。

但实际上，上面两种优化方案都并被没有大量使用，原因并不是实现起来有多难，懒加载有的 ORM 框架是支持的，脏跟踪也可以通过一些设计模式进行实现，没有大量使用这两种方案主要是因为这些性能问题可以通过其他方式得到规避或者缓解。

首先，大部分应用场景都是读多写少，针对读操作，我们在数据库层面做读写分离，应用层会加一层缓存去扛读流量，而针对复杂的查询操作还可能会引入专门的中间件来支持，例如全文检索可以通过 ElasticSearch 搜索引擎进行支持，这就是 CQRS 的思想。这一系列操作，切走了数据库主库的大部分的读流量，数据库主库的压力其实已经很小了，如果性能还有瓶颈，还可以引入分库分表，基本上没什么问题。

其次，由于复杂查询已经通过 CQRS 切走了，全文检索不再通过数据库，针对类似 cms 文章的正文富文本这样的字段，还可以采取先压缩再存储、存到HBase、存到对象存储等多种方式进行存储，没有必要保存到数据库，压力就更小了。

对于一个微服务，写请求日常能使数据库达到 1000+的 TPS，已经算是非常不错的业务了，我相信大部分公司都达不到。如果你们公司能达到，请问你们公司加班多不多？团队卷不卷？还招不招人？你看我合不合适？

<!--@include: ../footer.md-->
