---
title: "Idempotent Kafka Consumers"
date: 2023-01-08
tags: ["Kafka", "Idempotency", "Software Architecture", "Event Driven Architecture", "Transactional Outbox"]
categories: Software Engineering
---

Duplicate messages in Kafka, or any other message broker, are inevitable. Ensuring your application is able to handle these is essential. As a Kafka Consumer there are different reasons why it is possible to consume a duplicate message: 

- There can be an actual duplicate message in the kafka topic you are consuming from. The consumer is reading 2 different messages that should be treated as duplicates.
- You consume the same message more than once due to various error scenarios that can happen either in your application in the communication with a Kafka broker.
  
If you are using Event-driven architecture to implement inter-service communication in your system it is likely that you will consume messages you don't control or own. Kafka topic is just your asynchronous API you are a consumer of. The topic and the producer can be owned by another team or a 3rd party. 

The most common approach is to handle duplicate messages on the consumer side.
