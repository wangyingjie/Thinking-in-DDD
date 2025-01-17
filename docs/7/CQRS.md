# CQRS

## 1. CQRS 的概念理解

CQRS（Command Query Responsibility Segregation）是一种架构模式，即将应用程序分为两个部分：命令（Command）和查询（Query）。

在 CQRS 架构中，应用程序被分为两个部分：命令端和查询端。命令端负责处理所有的写操作，包括创建、更新和删除数据。查询端负责处理所有的读操作，包括查询和展示数据。

在 CQRS
中，命令和查询是分离的，因此使用不同的模型来处理：命令端使用的模型通常是面向行为的模型，即使用领域模型执行业务操作，通过聚合根修改其内部状态以实现业务逻辑；查询端使用的模型通常是面向查询的模型，负责读取数据并将其呈现给用户，由于不关注聚合的一致性，因此可以通过数据模型进行实现。

## 2. CQRS 的优缺点

### 2.1 CQRS 的优点

- 更好的可扩展性

CQRS 将应用程序分为命令端和查询端，使得可以针对不同的需求进行分别扩展。

- 高并发处理能力

一方面由于 CQRS 具有更好的扩展性，我们可以通过资源扩容支持更高的并发能力；一方面由于读写分离了，我们可以分别针对读写操作进行技术架构、代码逻辑上的优化。

- 更好的业务逻辑分离

CQRS 可以更好地实现业务逻辑的分离：写操作关心聚合内的一致性，而读操作不需要关心一致性。

- 更好的性能

CQRS 将读写操作分离，我们可以分别针对读写进行资源扩展以更好地处理高并发情况。

写操作更多依赖事务型数据库，我们可以采用分库分表的方式提高其写库的性能；读操作可以引入搜索引擎、缓存等基础设施，使查询的性能得到极大提升。

### 2.2 CQRS 的缺点

- 复杂性高

CQRS 架构对命令和查询请求采用不同的模型，因此复杂性较高，需要对应用程序进行细致的设计和实现。

- 学习成本高

CQRS 架构需要开发人员具备领域驱动设计、数据驱动设计或者面向服务架构（SOA）的知识。

## 3. 实现 CQRS 的几个层次

### 3.1 第一个层次，方法级 CQRS

方法级的 CQRS 主要是指一个方法，要么执行操作（即命令 Command），要么进行查询(即 Query)
，职责要单一，不应该在命令方法中返回查询结果，也不应该在查询中修改对象的状态。

方法级的 CQRS 实际上与领域驱动设计无关，不管有没有在实践 DDD，我们都推荐将方法实现为 CQRS。

### 3.2 第二个层次，同数据源的 CQRS

只要命令和查询这两个操作使用的不是一套模型，其实就已经实现了 CQRS。

在实际落地过程中，可以先共用一套数据源（即共用同一套库表），只不过命令操作由领域模型完成，查询操作由数据模型完成。

同一套数据源承接命令和查询，实现起来非常简单，其实已经能满足大部分的业务场景了。

命令操作的伪代码：

```java
/**
 * 命令操作，主要用于修改聚合的状态
 */
public interface CommandApplicationService {
    void modify(ModifyCommand command);
}
```

```java
public class CommandApplicationServiceImpl implements CommandApplicationService {
    public void modify(ModifyCommand command) {
        //TODO 1. 加载聚合根

        //TODO 2. 聚合根执行业务操作

        //TODO 3. 保存聚合根
    }
}

```

可以看到，命令操作是典型的 DDD 执行过程：加载聚合跟、聚合根执行操作、保存聚合根。

查询操作的伪代码：

```java
/**
 * 查询操作
 */
public interface ArticleQueryService {
    public Page<ArticleView> pageList(PageArticleQuery query);
}

```

```java
/**
 * 查询操作
 */
public class ArticleQueryServiceImpl implements ArticleQueryService {
    @Resource
    private ArticleMapper mapper;

    public Page<ArticleView> pageList(PageArticleQuery query) {

        //TODO 1. 根据query内的参数，去数据库查询出数据模型
        Param param = this.toParam(query);
        Long total = mapper.queryList(param);
        List<Article> articleList = mapper.queryList(param);
        //TODO 2. 将数据模型转成DTO（即DataView对象）并返回
        return this.toPageListView(total, articleList);
    }
}
```

```java
/**
 * 数据模型的伪代码
 */
@Data
@Table("t_article")
public class Article {
    @Id
    private String id;
    @Column("title")
    private String title;
    @Column("content")
    private String content;
}
```

可以看到，Query 查询是直接在应用层调用基础设施层的 Mapper 去完成的，没有经过聚合根。

### 3.3 第三个层次，异构数据源的CQRS

第二个层次的 CQRS 基本满足大部分场景的需求，然而有时候我们需要用到数据中间件的一些特性，这个时候就可以用到异构数据源的CQRS，即查询与命令两种操作分别由不同的数据源承接。

例如，ElasticSearch具备很强的全文检索能力，可以说是业界全文检索的事实标准。我们在ElasticSearch中持久化一份数据的副本，然后由ElasticSearch承接部分查询的流量。当然，ElasticSearch中的副本不需要存储所有的字段，只需要持久化作为搜索条件的字段和唯一标识，我们通过ElasticSearch检索出唯一标识之后，再通过唯一标识加载完整的数据模型并返回。

异构数据源不一定就是两种不同的数据中间件，完全有可能两个数据源都是MySQL，但是数据源的表结构不一样。例如，我们可以把查询的数据源设计为宽表，通过冗余字段的方式减少查询的性能消耗。