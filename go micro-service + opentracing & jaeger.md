
## 背景
gin+grpc


## Opentracing

## jaeger 架构、部署
Jaeger can be deployed either as all-in-one binary, where all Jaeger backend components run in a single process, or as a scalable distributed system, discussed below. There two main deployment options:

Collectors are writing directly to storage.
Collectors are writing to Kafka as a preliminary buffer.

https://www.jaegertracing.io/img/architecture-v1.png
ArchitectureIllustration of direct-to-storage architecture

https://www.jaegertracing.io/img/architecture-v2.png
ArchitectureIllustration of architecture with Kafka as intermediate buffer

## 如何埋点



