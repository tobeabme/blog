
## 背景 
微服务极大地改变了软件的开发和交付模式，单体应用被拆分为多个微服务，单个服务的复杂度大幅降低，库之间的依赖也转变为服务之间的依赖。由此带来的问题是部署的粒度变得越来越细，众多服务给运维带来巨大压力，不过好在我们有 Kubernetes，可以解决大部分运维方面的难题。

随着服务数量的增多和内部调用链的复杂化，仅凭借日志和性能监控很难做到 “See the Whole Picture”，在进行问题排查或是性能分析的时候，无异于盲人摸象。分布式追踪能够帮助开发者直观分析请求链路，快速定位性能瓶颈，逐渐优化服务间依赖，也有助于开发者从更宏观的角度更好地理解整个分布式系统。

分布式追踪系统大体分为三个部分，数据采集、数据持久化、数据展示。数据采集是指在代码中埋点，设置请求中要上报的阶段，以及设置当前记录的阶段隶属于哪个上级阶段。数据持久化则是指将上报的数据落盘存储，例如 Jaeger 就支持多种存储后端，可选用 Cassandra 或者 Elasticsearch。数据展示则是前端根据 Trace ID 查询与之关联的请求阶段，并在界面上呈现。

## Opentracing

### 发展历史 

早在 2005 年，Google 就在内部部署了一套分布式追踪系统 Dapper，并发表了一篇论文《Dapper, a Large-Scale Distributed Systems Tracing Infrastructure》，阐述了该分布式追踪系统的设计和实现，可以视为分布式追踪领域的鼻祖。随后出现了受此启发的开源实现，如 Zipkin、SourceGraph 开源的 Appdash、Red Hat 的 Hawkular APM、Uber 开源的 Jaeger 等。但各家的分布式追踪方案是互不兼容的，这才诞生了 OpenTracing。OpenTracing 是一个 Library，定义了一套通用的数据上报接口，要求各个分布式追踪系统都来实现这套接口。这样一来，应用程序只需要对接 OpenTracing，而无需关心后端采用的到底什么分布式追踪系统，因此开发者可以无缝切换分布式追踪系统，也使得在通用代码库增加对分布式追踪的支持成为可能。

目前，主流的分布式追踪实现基本都已经支持 OpenTracing，包括 Jaeger、Zipkin、Appdash 等，具体可参考官方文档 《Supported Tracer Implementations》。


### 数据模型 

这部分在 OpenTracing 的规范中写的非常清楚，下面只大概翻译一下其中的关键部分，细节可参考原始文档 《The OpenTracing Semantic Specification》。

```
Causal relationships between Spans in a single Trace

        [Span A]  ←←←(the root span)
            |
     +------+------+
     |             |
 [Span B]      [Span C] ←←←(Span C is a `ChildOf` Span A)
     |             |
 [Span D]      +---+-------+
               |           |
           [Span E]    [Span F] >>> [Span G] >>> [Span H]
                                       ↑
                                       ↑
                                       ↑
                         (Span G `FollowsFrom` Span F)
```

Trace 是调用链，每个调用链由多个 Span 组成。Span 的单词含义是范围，可以理解为某个处理阶段。Span 和 Span 的关系称为 Reference。上图中，总共有标号为 A-H 的 8 个阶段。

```
Temporal relationships between Spans in a single Trace

––|–––––––|–––––––|–––––––|–––––––|–––––––|–––––––|–––––––|–> time

 [Span A···················································]
   [Span B··············································]
      [Span D··········································]
    [Span C········································]
         [Span E·······]        [Span F··] [Span G··] [Span H··]

```

上图是按照时间顺序呈现的调用链。

**每个阶段（Span）包含如下状态：** 

- 操作名称
- 起始时间
- 结束时间
- 一组 KV 值，作为阶段的标签（Span Tags）
- 阶段日志（Span Logs）
- 阶段上下文（SpanContext），其中包含 Trace ID 和 Span ID
- 引用关系（References）

阶段（Span）可以有 ChildOf 和 FollowsFrom 两种引用关系。ChildOf 用于表示父子关系，即在某个阶段中发生了另一个阶段，是最常见的阶段关系，典型的场景如调用 RPC 接口、执行 SQL、写数据。FollowsFrom 表示跟随关系，意为在某个阶段之后发生了另一个阶段，用来描述顺序执行关系。

ChildOf relationship means that the rootSpan has a logical dependency on the child span before rootSpan can complete its operation. Another standard reference type in OpenTracing is FollowsFrom, which means the rootSpan is the ancestor in the DAG, but it does not depend on the completion of the child span, for example if the child represents a best-effort, fire-and-forget cache write.

### Concepts and Terminology, 概念与术语

#### Traces 
一个trace代表一个潜在的，分布式的，存在并行数据或并行执行轨迹（潜在的分布式、并行）的系统。一个trace可以认为是多个span的有向无环图（DAG）。

#### Spans 
一个span代表系统中具有开始时间和执行时长的逻辑运行单元。span之间通过嵌套或者顺序排列建立逻辑因果关系。

#### Operation Names 
每一个span都有一个操作名称，这个名称简单，并具有可读性高。（例如：一个RPC方法的名称，一个函数名，或者一个大型计算过程中的子任务或阶段）。span的操作名应该是一个抽象、通用的标识，能够明确的、具有统计意义的名称；更具体的子类型的描述，请使用Tags
例如，假设一个获取账户信息的span会有如下可能的名称：
| Operation Name | Guidance |
|:---------------|:--------|
| `get` | Too general |
| `get_account/792` | Too specific |
| `get_account` | Good, and `account_id=792` would make a nice **`Span` tag** |

#### References between Spans

A Span may reference zero or more other **SpanContexts** that are causally related. OpenTracing presently defines two types of references: `ChildOf` and `FollowsFrom`. **Both reference types specifically model direct causal relationships between a child Span and a parent Span.** In the future, OpenTracing may also support reference types for Spans with non-causal relationships (e.g., Spans that are batched together, Spans that are stuck in the same queue, etc).

**`ChildOf` references:** A Span may be the `ChildOf` a parent Span. In a `ChildOf` reference, the parent Span depends on the child Span in some capacity. All of the following would constitute `ChildOf` relationships:

- A Span representing the server side of an RPC may be the `ChildOf` a Span representing the client side of that RPC
- A Span representing a SQL insert may be the `ChildOf` a Span representing an ORM save method
- Many Spans doing concurrent (perhaps distributed) work may all individually be the `ChildOf` a single parent Span that merges the results for all children that return within a deadline

These could all be valid timing diagrams for children that are the `ChildOf` a parent.

~~~
    [-Parent Span---------]
         [-Child Span----]

    [-Parent Span--------------]
         [-Child Span A----]
          [-Child Span B----]
        [-Child Span C----]
         [-Child Span D---------------]
         [-Child Span E----]
~~~

**`FollowsFrom` references:** Some parent Spans do not depend in any way on the result of their child Spans. In these cases, we say merely that the child Span `FollowsFrom` the parent Span in a causal sense. There are many distinct `FollowsFrom` reference sub-categories, and in future versions of OpenTracing they may be distinguished more formally.

These can all be valid timing diagrams for children that "FollowFrom" a parent.

~~~
    [-Parent Span-]  [-Child Span-]


    [-Parent Span--]
     [-Child Span-]


    [-Parent Span-]
                [-Child Span-]
~~~


#### Logs
每个span可以进行多次Logs操作，每一次Logs操作，都需要一个带时间戳的时间名称，以及可选的任意大小的存储结构。
标准中定义了一些日志（logging）操作的一些常见用例和相关的log事件的键值，可参考Data Conventions Guidelines 数据约定指南。

#### Tags
每个span可以有多个键值对（key:value）形式的Tags，Tags是没有时间戳的，支持简单的对span进行注解和补充。
和使用Logs的场景一样，对于应用程序特定场景已知的键值对Tags，tracer可以对他们特别关注一下。更多信息，可参考Data Conventions Guidelines 数据约定指南。

#### SpanContext 
每个span必须提供方法访问SpanContext。SpanContext代表跨越进程边界，传递到下级span的状态。(例如，包含<trace_id, span_id, sampled>元组)，并用于封装Baggage (关于Baggage的解释，请参考下文)。SpanContext在跨越进程边界，和在追踪图中创建边界的时候会使用。(ChildOf关系或者其他关系，参考Span间关系 )。

#### Baggage 
Baggage是存储在SpanContext中的一个键值对(SpanContext)集合。它会在一条追踪链路上的所有span内全局传输，包含这些span对应的SpanContexts。在这种情况下，"Baggage"会随着trace一同传播，他因此得名（Baggage可理解为随着trace运行过程传送的行李）。鉴于全栈OpenTracing集成的需要，Baggage通过透明化的传输任意应用程序的数据，实现强大的功能。例如：可以在最终用户的手机端添加一个Baggage元素，并通过分布式追踪系统传递到存储层，然后再通过反向构建调用栈，定位过程中消耗很大的SQL查询语句。

Baggage拥有强大功能，也会有很大的消耗。由于Baggage的全局传输，如果包含的数量量太大，或者元素太多，它将降低系统的吞吐量或增加RPC的延迟。

#### Baggage vs. Span Tags 
- Baggage在全局范围内，（伴随业务系统的调用）跨进程传输数据。Span的tag不会进行传输，因为他们不会被子级的span继承。
- span的tag可以用来记录业务相关的数据，并存储于追踪系统中。实现OpenTracing时，可以选择是否存储Baggage中的非业务数据，OpenTracing标准不强制要求实现此特性。

#### Inject and Extract 
SpanContexts可以通过Injected操作向Carrier增加，或者通过Extracted从Carrier中获取，跨进程通讯数据（例如：HTTP头）。通过这种方式，SpanContexts可以跨越进程边界，并提供足够的信息来建立跨进程的span间关系（因此可以实现跨进程连续追踪）。

#### Global and No-op Tracers 
每一个平台的OpenTracing API库(opentracing-go, opentracing-java等)，都必须实现一个空的Tracer，No-op Tracer的实现必须不会出错，并且不会有任何副作用。这样在业务方没有指定collector服务、storage、和初始化全局tracer时，但是rpc组件，orm组件或者其他组件加入了探针。这样全局默认是No-op Tracer实例，则对业务不会有任何影响。



## jaeger 架构、部署
Jaeger can be deployed either as all-in-one binary, where all Jaeger backend components run in a single process, or as a scalable distributed system, discussed below. There two main deployment options:

Collectors are writing directly to storage.
Collectors are writing to Kafka as a preliminary buffer.

![architecture-v1.png](https://www.jaegertracing.io/img/architecture-v1.png)

Illustration of direct-to-storage architecture

![architecture-v1.png](https://www.jaegertracing.io/img/architecture-v2.png)

Illustration of architecture with Kafka as intermediate buffer

An instrumented service creates spans when receiving new requests and attaches context information (trace id, span id, and baggage) to outgoing requests. Only ids and baggage are propagated with requests; all other information that compose a span like operation name, logs, etc. are not propagated. Instead sampled spans are transmitted out of process asynchronously, in the background, to Jaeger Agents.

The instrumentation has very little overhead, and is designed to be always enabled in production.

Note that while all traces are generated, only a few are sampled. Sampling a trace marks the trace for further processing and storage. By default, Jaeger client samples 0.1% of traces (1 in 1000), and has the ability to retrieve sampling strategies from the agent.

**Agent** 

The Jaeger agent is a network daemon that listens for spans sent over UDP, which it batches and sends to the collector. It is designed to be deployed to all hosts as an infrastructure component. The agent abstracts the routing and discovery of the collectors away from the client.

**Collector** 

The Jaeger collector receives traces from Jaeger agents and runs them through a processing pipeline. Currently our pipeline validates traces, indexes them, performs any transformations, and finally stores them.

Jaeger’s storage is a pluggable component which currently supports Cassandra, Elasticsearch and Kafka.

**Query** 

Query is a service that retrieves traces from storage and hosts a UI to display them.

**Ingester** 

Ingester is a service that reads from Kafka topic and writes to another storage backend (Cassandra, Elasticsearch).

## 微服务框架接入opentracing流程

一个微服务框架包括两个部分，http(gin) & grpc两部分，对外提供rest，对内提供grpc服务。

下面是微服务通讯架构图

![Microservice-3.png](https://github.com/tobeabme/blog/blob/master/images/Microservice-3.png)

下面是微服务软件框架图

![MicroserviceFramework-3.png](https://github.com/tobeabme/blog/blob/master/images/MicroserviceFramework-3.png)


下面是微服务框架接入opentracing的大概流程。

**为每个http请求创建一个tracer** 

```
tracer, closer := tracing.Init("hello-world")
defer closer.Close()
opentracing.SetGlobalTracer(tracer)
```

**创建Span，如果http header中有trace和span信息，则从头部获取，否则创建新的。** 
```
spanCtx, _ := tracer.Extract(opentracing.HTTPHeaders, opentracing.HTTPHeadersCarrier(r.Header))
span := tracer.StartSpan("format", ext.RPCServerOption(spanCtx))

defer span.Finish()

// 把span写入context，函数见的内部调用需要传递ctx，或者说span之间需要传递ctx。
ctx := opentracing.ContextWithSpan(context.Background(), span)
```


**Http/GRPC 服务函数的进程内部** 
```
span, _ := opentracing.StartSpanFromContext(ctx, "formatString")
defer span.Finish()


// 跨进程调用，如调用一个rest api，则需要把span信息注入http header中。
// tracing.InjectToHeaders(ctx, "GET", url, req.Header)
func InjectToHeaders(ctx context.Context, method string, url string, header http.Header) {
	span := opentracing.SpanFromContext(ctx)
	if span != nil {
		ext.SpanKindRPCClient.Set(span)
		ext.HTTPUrl.Set(span, url)
		ext.HTTPMethod.Set(span, "GET")
		span.Tracer().Inject(
			span.Context(),
			opentracing.HTTPHeaders,
			opentracing.HTTPHeadersCarrier(header),
		)
	}
}

span.LogFields(
        log.String("event", "string-format"),
        log.String("value", helloStr),
)
```


## 如何埋点

### Gin 框架 
**router 埋点** 

在每个需要追踪请求的http路由方法上，添加「tracing.NewSpan」函数。

```
import ".../go_common/tracing"

...

authorized := r.Group("/v1")
authorized.Use(handlers.TokenCheck, handlers.MustLogin())
{
    authorized.GET("/user/:id", handlers.GetUserInfo)
    authorized.GET("/user", handlers.GetUserInfoByToken)
    authorized.PUT("/user/:id", tracing.NewSpan("put /user/:id", "handlers.Setting", false), handlers.Setting)
}
```

参数说明

NewSpan(service string, operationName string, abortOnErrors bool, opts ...opentracing.StartSpanOption) 

service generally fill with the endpoint of api.
operationName can be filled with HandleFunc's name.

**Handler 函数埋点**

```
func Setting(c *gin.Context) {
    ...
    // 从gin context中获取span；必须埋点！
    span, found := tracing.GetSpan(c)

    //添加tag和log
    if found == true && span != nil {
        span.SetTag("req", req)
        span.LogFields(
            log.Object("uid", uid),
        )
    }

    // opentracing.ContextWithSpan，将span和context绑定；在handler函数中，这个地方也是必须埋点的。
    ctx, cancel := context.WithTimeout(opentracing.ContextWithSpan(context.Background(), span), time.Second*3)
    defer cancel()

    // call by grpc；这块不需要特殊处理
    auth := passportpb.Authentication{
        LoginToken: c.GetHeader("Qsc-Peduli-Token"),
    }
    cli, _ := passportpb.Dial(ctx, grpc.WithPerRPCCredentials(&auth))
    reply, err := cli.Setting(ctx, req)

    // directly call by local rpc method；这块不需要特殊处理
    ctx = metadata.AppendToOutgoingContext(ctx, "logintoken", c.GetHeader("Qsc-Peduli-Token"))
    reply, err := rpc.Srv.Setting(ctx, req)

    ...
}
```

### GRPC

**grpc 客户端SDK**

```
import "github.com/grpc-ecosystem/go-grpc-middleware/tracing/opentracing"

...

// Dial grpc server
func (c *Client) Dial(serviceName string, opts ...grpc.DialOption) (*grpc.ClientConn, error) {
	
	...
	
	unaryInterceptor := grpc_middleware.ChainUnaryClient(
		grpc_opentracing.UnaryClientInterceptor(),
	)

	c.Dialopts = append(c.Dialopts, grpc.WithUnaryInterceptor(unaryInterceptor))

	conn, err := grpc.Dial(serviceName, c.Dialopts...)
	if err != nil {
		return nil, fmt.Errorf("Failed to dial %s: %v", serviceName, err)
	}
	return conn, nil
}
```

**grpc 服务端SDK**

```
import "github.com/grpc-ecosystem/go-grpc-middleware/tracing/opentracing"

...

func NewServer(serviceName, addr string) *Server {
	var opts []grpc.ServerOption
	opts = append(opts, grpc_middleware.WithUnaryServerChain(
		grpc_opentracing.UnaryServerInterceptor(),
	))

	srv := grpc.NewServer(opts...)
	return &Server{
		serviceName: serviceName,
		addr:        addr,
		grpcServer:  srv,
	}
}
```


**RPC 函数埋点** 

```
func (s *Service) Setting(ctx context.Context, req *passportpb.UserSettingRequest) (*passportpb.UserSettingReply, error) {
    // 如果不是grpc调用，即本地rpc函数调用方式，则从上下文中提取span。
    if !s.meta.IsGrpcRequest(ctx) {
	span, _ := opentracing.StartSpanFromContext(ctx, "rpc.srv.Setting")
	defer span.Finish()
    }

    // 如果在rpc函数中，存在请求其它grpc函数，则正常调用即可，因为在grpc的请求上下文中已经有了trace和span信息，直接繁殖就行，无需额外操作。
    reqVerify := new(passportpb.VerifyRequest)
    reqVerify.UID = req.UserID
    cli, _ := passportpb.Dial(ctx)
    replyV, _ := cli.Verify(ctx, reqVerify)
}    
```
  
**Jaeger UI 最终效果** 

![opentracing-gin-grpc.jpg](https://github.com/tobeabme/blog/blob/master/images/opentracing-gin-grpc.jpg)


## 参考  

[jaegertracing](https://www.jaegertracing.io/docs/1.12/deployment/)  

[opentracing-tutorial](https://github.com/yurishkuro/opentracing-tutorial)   

[grpc-opentracing](https://github.com/grpc-ecosystem/go-grpc-middleware/tree/master/tracing/opentracing) 

[gin-opentracing](https://github.com/gin-contrib/opengintracing/tree/c082d5d9c71fbc49b820f8fd22347d36b89d7d9a) 

[go-opentracing-guides](https://opentracing.io/guides/golang/) 

[opentracing 文档中文版](https://wu-sheng.gitbooks.io/opentracing-io/content)
