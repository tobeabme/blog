
## 背景
gin+grpc
![MicroserviceFramework-3.png](https://github.com/tobeabme/blog/blob/master/images/MicroserviceFramework-3.png)
![Microservice-3.png](https://github.com/tobeabme/blog/blob/master/images/Microservice-3.png)


## Opentracing

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

## 如何埋点



