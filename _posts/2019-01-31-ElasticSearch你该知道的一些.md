---
    layout: post
    title: ElasticSearch你该知道的一些
---

## 1.什么是es
Elasticsearch is a distributed RESTful search engine built for the cloud. Features include:

- Distributed and Highly Available Search Engine.
    * Each index is fully sharded with a configurable number of shards.
    * Each shard can have one or more replicas.
    * Read / Search operations performed on any of the replica shards.
- Multi Tenant.
- Various set of APIs
- Document oriented
- Reliable, Asynchronous Write Behind for long term persistency.
- (Near) Real Time Search.
- Built on top of Lucene
    * Each shard is a fully functional Lucene index
    * All the power of Lucene easily exposed through simple configuration / plugins.
- Per operation consistency
