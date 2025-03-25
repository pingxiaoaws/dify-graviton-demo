# dify-helm
Deploy [langgenius/dify](https://github.com/langgenius/dify), an LLM based chat bot app on kubernetes with helm chart.

## Installation


## Supported Component 
### Components that could be deployed on kubernetes in current version
- [x] core (`api`, `worker`, `sandbox`)
- [x] ssrf_proxy
- [x] proxy (via built-in `nginx` or `ingress`)
- [x] redis
- [x] postgresql
- [x] persistent storage
- [ ] object storage
- [x] weaviate
- [ ] qdrant
- [ ] milvus
### External components that can be used by this app with proper configuration

- [x] Redis
- [x] PostgreSQL
- Object Storage:
  - [x] Amazon S3
  - [x] Microsoft Azure Blob Storage
  - [x] Alibaba Cloud OSS
  - [x] Google Cloud Storage
  - [x] Tencent Cloud COS
  - [x] Huawei Cloud OBS
- External Vector DB:
  - [x] Weaviate
  - [x] Qdrant
  - [x] Milvus
  - [x] PGVector
  - [x] Tencent Vector DB
  - [x] MyScaleDB

## Contributors
<a href="https://github.com/borispolonsky/dify-helm/graphs/contributors">
  <img src="https://contrib.rocks/image?repo=borispolonsky/dify-helm" />
</a>

