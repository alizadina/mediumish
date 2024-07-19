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

Managing Kafka clusters is all good as a platform team until you have a handful of teams and a handful of clusters. 

![managing-clusters-problem.png](../assets/blog-images/unified_access_layer/managing-clusters-problem.png)

Literally every platform team has ended up with a mish-mash of ungoverned clusters or topics that eventually needs some garden cleaning (for good reason). Let’s visit some common scenarios.

You need to move or migrate topics in the same cluster (surprise, Kafka can’t shrink partitions so you will have to create a new topic and replicate to mirror the old). You may need to move or migrate a topic (let’s say with hot, noisy neighbor partitions) to a different cluster altogether
You may want to move a whole cluster to better infrastructure (or more commonly because you have a disaster. Kaboom!). You need to change the authentication mechanism and (re)distribute credentials. Your devs are complaining and need help optimizing their configurations (and they have no clue what they’re doing).

If you’ve been there, done that and burnt your hands, you’re not alone. The part that makes this all pretty hard in the Kafka world is the nature of Kafka’s API, where clients do much of the song and dance. We have a proposal for a future proofing for all these concerns.

- The ability to route a client to a specific cluster (bootstrap.servers) dynamically, without having these hardcoded in the client
  - ProTip: Very useful for entire cluster migrations
- The ability to route a client to a specific topic, based on a higher order messaging or routing primitive, without being joined in the hip with a set of topics
  - ProTip: Very useful for topic migrations, but also routing multiple types of messages to the same topic
- Credential Mediation
  - Giving clients their auth instruments, dynamically. Potentially references to secret stores or serving the credentials straight up (but ability to rotate on the server-side as well as revoking them on-demand)
- Configuration profile
  - Ability to provide some of your well optimized for latency, throughput, durability, availability or otherwise just your distillation of best practices for producer, consumer, admin client configurations.

Having some abstractions to achieve these cross cutting concerns is a good measure and will yield sanity to platform operators who may need to defer great control to themselves. In general, for large scale use-cases, it is always good to have some insulation and indirection in between kafka clients and the server, ala proxy patterns very popular in the HTTP/REST world.

This post intends to propose common solutions (as a series) that would help overcome these problems and in the true spirit of a platform, provide an effective amount of self service.

# Approach 1: Discover topics through Language specific SDKs and a rule-based routing layer

This is a pattern where a language specific SDK handles the following concern, by talking to a metadata or a catalog service. 

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


# Approach 2: Use a Service Mesh (for Kubernetes Environment)

A service mesh (like Istio, Kuma) can be leveraged as a layer to access Kafka resources. It typically includes service discovery mechanisms that allow services to dynamically discover and communicate with each other, and this can be extended to Kafka by enabling services to discover Kafka brokers and other Kafka-related services (like schema registries or Kafka Connect connectors).

Downsides to using a service mesh: 1/ It is another piece of middleware that needs someone familiar with the internals (like Envoy proxy) to be able to operate it. 2/ Tenancy: the more tenants, the more valuable it is to operate. Careful planning is needed for policy, automation, tenancy and isolation. 3/ Being another piece of the services in the request path requires understanding on configuration, operation and integration within the organization. That with the governance between different teams.

# Approach 3: Virtualize the Kafka Cluster and Topic (through a Kafka Proxy)

In this approach, we provide a local kafka proxy embedded, ideally embedded inside a sidecar data-plane, like Envoy. In most cases, Envoy would come with a service mesh distribution which provides a control plane. 
 
This provides for robust L4-L7 routing capabilities as well as concerns like credential injection to the upstream. If mTLS is involved, life can be made even easier and certificates can be managed with SPIFFE or equivalent. Additional points for TCP observability. In Envoy land, you have the Kafka mesh and broker filters which can help accomplish this with kafka protocol support as well as additional metrics. Finally, this provides polyglot support out of the box because all this happens transparently to the kafka client at the network level.

The only thing is, this works well in maybe 50% of the use-cases. Mostly when Kubernetes is in the game and ideally both the Kafka cluster and the clients being run on containers, with k8s and ideally a service mesh. But larger, more complex use-cases usually require mesh federation, multi-cluster meshes and networking complexity which can quickly spin out of control. Some of the downsides listed in [Approach 2](#approach-2-use-a-service-mesh-for-kubernetes-environment) is also applicable here.

# Approach 4: Discover Topics using a Data Catalog + Self-Service Portal

You need a data catalog of some sort that supports streams. Streams must be discovered through a crawler or agent and registered into the catalog. Confluent has a streams catalog. GCP supports a catalog for Pub-Sub. Azure supports Azure Catalog. AWS Glue Catalog unfortunately has no support for streams. Collibra has some limited support for Kafka. Apache Atlas is the only Open Source solution. For everything else, there’s the Hive metastore. We can also look at OpenMetadata, Databricks unity catalog and Amundsen among others.

All of the above broadly captures business and technical metadata at best. You still need a discovery interface to query by these parameters. Ideally, you also need credential vending.

Kafka Streams catalogs offer benefits such as improved metadata management and easier integration with external systems, they also come with potential downsides related to complexity, performance, consistency, vendor lock-in, maintenance, and learning curve. Careful evaluation of these factors is essential to determine if the use of a catalog is suitable for your specific streaming application and organizational needs.

# Conclusion

Creating a system that helps a centralized Kafka infrastructure team to easily create, label and vend information reduces common problems and dependencies. Same benefits are passed on to the producers and consumers thus creating a scalable system/organization. 
