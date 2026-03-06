---
author : ['Hanlin']
title: "Distributed System Blog – Dynamo"
description: "Evaluation of Dynamo and Later Work"
date: 2026-03-05
lastmod: 2026-03-05
type: post
draft: false
translationKey: distributed_systems_blog3
coffee: 1
tags: ['distributed_systems']
categories: ['distributed_systems']
---

# Distributed System Blog – Dynamo

**Evaluation of Dynamo and Later Work**

## Part 1  Evaluation of Dynamo

Amazon evaluated Dynamo directly in production scenarios. They chose 99.9th percentile latency as the core indicator because they believed user experience (like adding a cart item) was affected by the slowest requests. Compared to the average latency (~20ms), 99.9th percentile latency reached ~200ms. In terms of partitioning, they compared three strategies. First they used virtual node partitioning, but the performance was not good: the sizes were not balanced and every time a new node came, partitioning boundaries changed. They finally used strategy three that pre-divided the ring into a fixed number of partitions and distributed them evenly to each node. Experimental data shows that this strategy achieved the best efficiency and was clear to maintain. Dynamo designed a vector clock mechanism for concurrent conflict detection. However, in a 24-hour evaluation, 99.94% of requests saw exactly one version, which means that it was rarely triggered in actual scenarios. Overall, in two years of running on Amazon services, Dynamo achieved 99.9995% availability and zero data loss. Dynamo has proven to be a high-performance and robust distributed system in large-scale production environments.

## Part 2  Later Work of Dynamo

Dynamo is a watershed in distributed systems. Before Dynamo, strongly consistent systems dominated the field, such as those based on Paxos and Raft. Dynamo chose not to ensure consistency, but rather availability, which opened up another choice under the CAP theorem. Dynamo also started the trend of NoSQL. I am interested in two subsequent systems: Cassandra and DynamoDB. Cassandra inherited Dynamo's decentralized architecture and integrated the data model of Bigtable. DynamoDB changed a lot from the original Dynamo: it added a centralized "AutoAdmin" back, offered an optional strong consistency mode, and enriched the data model. Currently, Cassandra and DynamoDB are widely used in the NoSQL database market.
