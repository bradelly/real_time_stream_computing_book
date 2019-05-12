## Ad-Hoc查询

Ad-Hoc查询也称为即席查询，是用户为了某个查询目的，选择查询条件并提交数据库执行，最终生成相应的查询结果。
比如，一种常见的Ad-Hoc查询，就是用数据库命令行客户端连接到数据库后，编写SQL语句并执行，然后得到感兴趣的查询结果。

在实时流计算系统中，有时候会需要将计算结果最终呈现给最终用户看。
这里的最终用户，可能是运营人员，也可能是做决策的领导。
对于最终用户而言，直接写SQL并不方便，而且也不现实，因为他们并不知道数据的具体细节。
因此，这个时候通常会有一个用户友好的图形化界面，来引导用户设置查询条件，然后点击按钮后即可以呈现出各种图形报表。

从上面的定义中，我们可以知道Ad-Hoc查询是一类通用的查询方式。
但是这里我们只讨论实时流计算系统中的报表系统。

支持报表系统的查询技术通常是联机分析处理（OLAP）。联机分析处理能够支持复杂的分析操作，提供直观易懂的查询结果，以支持用户做出决策。
联机分析处理在各种维度或组合维度对数据进行统计和分析，比如排序、过滤、分组和聚合等。

联机分析处理的一个典型特点就是，分析的内容丰富多样。可能是某个维度的排序，也可能是某几个组合维度的聚合，
还可能是关键字的匹配过滤。特别是在UI上，可能还会提供给用户选择查询条件的功能，比如用户可以选择时间条件，
可以要求按设备排序，可以要求按欺诈得分排序等等。总之，这是个需求千变万化的场景。
作为一个开发人员，可能现在你的脑海中已经浮现出产品经理那轻松的一句："这不就是加一个查询条件的事嘛！"。

除了需求灵活多变、查询千变万化以外，这类查询还有个特点，它们是需要近实时返回结果的。
也就是说，当触发查询后，必须在数秒以内将结果报表呈现在UI上，否则用户会等得不耐烦，造成不好的用户体验。
同时，这类查询的频次却不会太高，只有在用户需要查看报表时才需要进行查询操作。
所以相比数据上报接口使用的频率而言，报表查询的频次会低很多。

针对报表这种查询灵活、需要近实时响应结果的场景而言，比较好的选择是使用搜索引擎一类的数据存储和查询方案，
比如ElasticSearch就是一个不错的选择。这是因为，搜索引擎通常采用倒排索引的方式来管理查询数据。

诸如MySQL和MongoDB这样的非倒排索引数据库，如果要针对指定的查询条件加快查询效率，必须预先建好索引。
可是，在报表系统这种场景下，一方面查询需要灵活多变、随意组合，另一方面随着产品演进，需求可能不断调整和增加。
这就需要我们在数据库中建立大量的单键索引和多键索引。而在产品新版本上线时，还可能需要更新索引和新增新索引。
这一切都增加了开发和运维的复杂性。比如MongoDB，当数据量已经很多时，在上面新增一个索引可能需要耗用数小时的时间，
这会严重影响线上数据库的可用性，甚至有可能直接导致线上数据库不可用。

相比较而言，采用倒排索引的数据库自动为数据建立字典和倒排索引表，
不需要我们再为查哪些字段、建哪些索引的问题而耗费过多精力。
下面我们就先看看什么是倒排索引。

### 倒排索引
倒排索引（Inverted index）是一种新颖的索引方法，常被用于搜索引擎，它也是文档检索中最常用的数据结构。
下面我们用搜索同时包含"我"、"爱"、"你"三个字文档的例子来讲解倒排索引的原理。

```
文档1：我喜欢你
文档2：我爱你
文档3：我很爱你
```

为了实现倒排索引，首先需要对每片文档进行"分词"处理。所谓"分词"，就是将文档切分成一个个单独的词。
为了简单起见，我们就将把每个字作为为一个词。经过分词处理后，结果为：

```
文档1：我、喜、欢、你
文档2：我、爱、你
文档3：我、很、爱、你
```

这些文档中，所有出现的词，构成了一个字典：

```
{我, 喜, 欢, 你, 爱, 很}
```

以这个字典为基础，构建倒排索引，也就是统计字典中的每个字，出现在哪些文档中：

```
"我": {文档1， 文档2， 文档3}
"喜": {文档1}
"欢": {文档1}
"你": {文档1， 文档2， 文档3}
"爱": {文档2，文档3}
"很": {文档3}
```

所以，如果搜索包含"我"、"爱"、"你"这三个字的文档，结果就是这三个字各自所在文档集合的交集，也就是：
{文档1， 文档2， 文档3} 交 {文档2，文档3} 交 {文档1， 文档2， 文档3} ＝ {文档2， 文档3}

我们再直观地检查下，文档1和文档3正好包含"我"、"爱"、"你"这三个字，而文档2则不是，这正是我们要达到的目的。

上面的过程就是构建倒排索引的过程。

从上面的过程可以看出，倒排索引实际上为文档集合中的每一个分词都构建了它包含在哪些文档中的索引。
当我们需要按多个条件（也就是多个分词）来查询文档时，只需要将它们各自出现的文档集合求交集就好了。
在具体工程实现中，可以通过位图（Bitmap）来记录索引，这样极大地节省了存储空间，
并且通过布尔运算就能实现集合间的交集、并集等操作，极大地提高了查询的效率。

倒排索引的这种特点，使得它非常适合运用于搜索引擎领域。

而在我们的报表系统里，倒排索引构建所有分词的索引，并且查询迅速的的特点，也完全能够满足OLAP查询的灵活性和准实时性要求。

### ElasticSearch
ElasticSearch是一个基于Lucene开发的全文搜索引擎，它可以在TB级别的数据量规模下，做到准实时搜索。
ElasticSearch支持的查询类型丰富多样，运行稳定可靠，集群部署、扩展和运维都非常方便。

ElasticSearch的一些列特点，使得它非常适合做OLAP分析。
其一，ElasticSearch支持丰富的过滤、分组、聚合、排序等查询，可以充分、灵活地从一个或多个维度分析数据，这正式报表系统OLAP查询的核心所在。
其二，ElasticSearch在OLAP场景性能优异。在TB级别的数据规模下，做各种OLAP查询，能够做到准实时，也就是数秒级别，是不是很赞。
其三，ElasticSearch集群搭建和扩展都非常容易，并且稳定可靠。
可以说，ElasticSearch是笔者使用过的所有分布式系统中，最方便、最可靠、最省心的分布式系统了。
其四，数据存入ElasticSearch，不需要专门预先针对OLAP查询设计各种聚合任务。这点在产品在不断演进时非常重要，
因为可能一开始的时候，产品经理自己也不知道以后会展示哪些报表，所以留下数据和查询的灵活性非常重要，可以减少太多以后的麻烦。

### 分索引存储
虽然ElasticSearch有很多优点，但是在使用过程中，还是需要注意些问题。

ElasticSearch有两种方式组织大量数据。一种是用一个大索引，索引内部分成很多shard。
另一种是用多个索引，每个索引内部用较少的shard。
由于ElasticSearch的查询有个非常好的特性，即同一个查询是可以跨多个索引的。
所以这两种方案对于查询范围相同的查询请求而言，是没有太大区别的。

但是，维护一个较小的索引，对于不停往系统里追加新文档的场景来说，是更加高效的。
因此，对于数据不停追加，数据量与日俱增的场景来说，最好还是将大索引分成多个小索引。
以笔者经验来看，对于需要频繁追加数据的应用场景而言，在单台4核8G的云主机上，单个索引控制在20～30G左右比较合适。

ElasticSearch分索引的方式，可以分为三种：按时间分片、按数据量分片以及同时按时间和数据量分片

#### 按时间分片
按时间分片是指根据时间周期，在每个新的时间周期使用一个新的索引。比如按天分片、按小时分片等。
按时间分片的好处在于实现简单、数据时间范围清晰明确，容易做TTL。
但是如果时间周期不好选择，或者数据流量在每个周期间的变化比较大的话，
会造成每个索引内数据量的分布不均匀，索引数过多或者过少，这给运维带来麻烦。

#### 按数据量分片
按数据量分片是指根据索引内记录的条数，来决定是否使用新的索引。
比如如果平均每条记录是1K字节，每个索引存放两千万条记录。
那么，当索引中的数量超过两千万条时，就创建一个新的索引来存放新的数据。

按数据量分片的好处在于每个索引数据量比较均匀。
如果非要说缺点的话，就是不能通过索引名直接确定里面数据的时间范围。

#### 同时按时间和数据量分片
同时按时间和数据量分片既可以保留每个索引内数据比较均匀的优点，
还可以通过索引名直接确定里面数据的时间范围，是个不错的选择，
只是在代码实现时相对更复杂些。