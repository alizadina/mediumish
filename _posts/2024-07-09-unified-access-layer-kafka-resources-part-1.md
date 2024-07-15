---
layout: post
title: "Unified Access Layer for Kafka Resources: Part 1"
authors: p6,raghav
categories: [Platform Engineering, Infrastructure, Kafka, Unified Access Layer]
image: assets/blog-images/unified_access_layer/unified_access_layer_1.png
featured: false
hidden: false
teaser: Unified access layer for Kafka resources, Part 1
toc: true
---

# Introduction

One of the most frequently occurring themes we’ve seen with enterprise customers running a large Kafka footprint is that of a unified access layer to the Kafka resources. In the world of HTTP/REST, this is usually solved simply through a proxy providing a unified ingress and routing layer, but it gets pretty tricky with Kafka.

To work with an example: Let’s say you’re a company that runs multiple Kafka clusters. These could be on different cloud providers or on-premise. It could even be other flavors of streaming that are also Kafka compatible. It is a proverbial PITA for a platform team to provide the following concerns without some kind of manual onboarding:

- Topic discovery
  - Domain based routing
  - Client based routing
- Credential vending and distribution
  - Mediated through a secret store
- Governance and audit
- Change controls: or what happens if you move a resource into another cluster.
- Migration

The basic point being, it is eminently useful to insulate teams that build Kafka producers, consumers (or connectors for that matter) as part of their products and services from the internal details of the platform with a unified access layer.

We’ve seen almost everyone that does this resorting to home-grown solutions of various sorts, with varying levels of sophistication and utility.

This post intends to propose common solutions (as a series) that would help overcome these problems and in the true spirit of a platform, provide an effective amount of self service.

# Approach 1: Discover topics through a rule-based routing layer

This is a pattern where a language specific SDK handles the following concern, by talking to a metadata or a kafka catalog service. 

Typically, the client here would, before instantiating a kafka client, locate the target kafka service using

- A HTTP/REST call to the catalog service (using some form of security: Such as basic auth or OAuth)
  - Express an intent (such as to produce or consume) on a certain topic or domain of topics (such as prefix pattern) OR by the virtue of it’s client ID, be routed to the appropriate cluster
- Receive the kafka bootstrap location
  - Optionally, mediated credentials for both kafka and typically, schema registry
- The above, being cached with a TTL
- Return a Kafka client instance

This approach has a wider-domain of applicability, beyond containerized environments, but it does require

- A secured, metadata or catalog service  (run by you)
  - Which ideally has automation for inventory discovery across multiple clusters
  - Combined with routing rules
- Language specific, idiomatic service locator implementation (“SDK”)
  - And therefore code changes to existing applications including exception handling

We get into the description and implementation details here: [Unified Access Layer: Part 2: Rule Based Routing](https://platformatory.io/blog/unified-access-layer-part-2-rule-based-routing)


# Approach 2: Use a service mesh

A service mesh (like Istio, Kuma) can be leveraged as a layer to access Kafka resources. It typically includes service discovery mechanisms that allow services to dynamically discover and communicate with each other, and this can be extended to Kafka by enabling services to discover Kafka brokers and other Kafka-related services (like schema registries or Kafka Connect connectors).

Downsides to using a service mesh: 1/ It is another piece of middleware that needs someone familiar with the internals (like Envoy proxy) to be able to operate it. 2/ Tenancy: the more tenants, the more valuable it is to operate. Careful planning is needed for policy, automation, tenancy and isolation. 3/ Being another piece of the services in the request path requires understanding on configuration, operation and integration within the organization. That with the governance between different teams.

# Approach 3: Virtualize the Kafka cluster and Topic (through a proxy)

In this approach, we provide a local kafka proxy embedded, ideally embedded inside a sidecar data-plane, like Envoy. In most cases, Envoy would come with a service mesh distribution which provides a control plane. 
 
This provides for robust L4-L7 routing capabilities as well as concerns like credential injection to the upstream. If mTLS is involved, life can be made even easier and certificates can be managed with SPIFFE or equivalent. Additional points for TCP observability. In Envoy land, you have the Kafka mesh and broker filters which can help accomplish this with kafka protocol support as well as additional metrics. Finally, this provides polyglot support out of the box because all this happens transparently to the kafka client at the network level.

The only thing is, this works well in maybe 50% of the use-cases. Mostly when Kubernetes is in the game and ideally both the Kafka cluster and the clients being run on containers, with k8s and ideally a service mesh. But larger, more complex use-cases usually require mesh federation, multi-cluster meshes and networking complexity which can quickly spin out of control.

# Approach 4: Discover Topics using a Data Catalog

You need a data catalog of some sort that supports streams. Streams must be discovered through a crawler or agent and registered into the catalog. Confluent has a stream catalog as well. GCP supports a catalog for Pub-Sub. Azure supports Azure Catalog. AWS Glue Catalog unfortunately has no support for streams. Collibra has some limited support for Kafka. Apache Atlas is the only OSS one. For everything else, there’s the Hive metastore. We can also look at OpenMetadata, Databricks unity catalog and Amundsen among others.

All of the above broadly captures business and technical metadata at best. You still need a discovery interface to query by these parameters.

Ideally, you also need credential vending.

# Conclusion

Creating a system that helps a centralized Kafka infrastructure team to easily create, label and vend information reduces common problems and dependencies. Same benefits are passed on to the producers and consumers thus creating a scalable system/organization. 
