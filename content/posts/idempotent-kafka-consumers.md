---
title: "Idempotent Kafka Processing"
date: 2023-01-14
tags: ["Kafka", "Idempotency", "Software Architecture", "Event Driven Architecture", "Transactional Outbox"]
categories: Software Engineering
---

## Duplicate messages are inevitable

Duplicate messages in Kafka, or any other message broker, are inevitable. Ensuring your application is able to handle these is essential. As a Kafka Consumer, there are different reasons why it is possible to consume a duplicate message: 

- There can be an actual duplicate message in the kafka topic you are consuming from. The consumer is reading 2 different messages that should be treated as duplicates.
- You consume the same message more than once due to various error scenarios that can happen, either in your application, or in the communication with a Kafka broker.

## Kafka delivery guarantees

Kafka offers different message delivery guarantees between producers and consumers, namely _at-least-once_, _at-most-once_ and _exactly-once_. 

### What exactly happens exactly once in exactly-once guarantee?

Exactly-once would seem like an obvious choice to guard against duplicate messages, but it not that simple and the devil is in the details. Confluent has spent a lot of resources to deliver exactly-once delivery guarantee, and you can read [here](https://www.confluent.io/en-gb/blog/exactly-once-semantics-are-possible-heres-how-apache-kafka-does-it/) on how it works in detail. It requires you to enable specific Kafka features (i.e. Idempotent Producer and Kafka Transactions). 

First of all, it is only applicable in an application that consumes a Kafka message, does some processing, and writes a resulting message to a Kafka topic. 

Exactly-once messaging semantics ensures the **combined** outcome of multiple steps will happen exactly-once. Key word here is combined. A message will be consumed, processed, and resulting messages produced, exactly-once. 

Critical points to understand about exactly-once delivery are:

1) **All other actions occurring as part of the processing can still happen multiple times, if the original message is redelivered.**
    
    The guarantee only covers resulting messages from the processing to be written exactly once, so downstream transaction aware consumers will not have to handle duplicates. Hence, each individual action still needs to be processed in an idempotent fashion to ensure real end-to-end exactly once processing. Application may need to, for example, perform REST calls to other applications, write to the database etc.

2) **All participating consumers and producers need to be configured correctly.**

    Kafka exactly-once semantics is achieved by enabling Kafka Idempotent Producers and Kafka Transactions in **all** consumers and producers involved. That includes the upstream producer and downstream consumers from the perspective of you application.
   If you are using Event-driven architecture to implement inter-service communication in your system, it is likely that you will consume messages you don't control, or own. Kafka topic is just your asynchronous API you are a consumer of. The topic and the producer can be owned by another team or a 3rd party. Similarly, you may not control downstream consumers. To add to the first point, outbound messages can still be written to the topic multiple times before being successfully committed, it is the responsibility of any downstream consumers to only read committed messages (i.e. be transaction aware) in order to meet the exactly-once guarantee.

3) **it comes with a performance impact.**

    Exactly-once delivery comes with a performance overhead. There are simply more steps involved for a single kafka message to be processed (e.g. Kafka performs a two-phase commit to support transactions) and that results in lower throughput and increased latency.

In practice, it's often much simpler, and more common, to settle for at-least-once semantic and just de-duplicate messages on the consumer side. Especially in cases where application processing is more involved and consists of other actions (e.g. REST calls and DB writes). It's important to remember there is a transaction boundary gap between a DB transaction and a Kafka transaction.

## How to achieve Idempotent Kafka Processing

This will depend on what triggers the processing, what actions constitute the processing, and what is the shape of the output.

### Reading from Kafka (Idempotent Consumer Pattern)

An Idempotent Consumer pattern uses a Kafka consumer that can consume the same message any number of times, but only process it once. You can make use of Kafka's "transaction boundary" (not to be confused with Kafka transactions mentioned before) to retry message processing in case of failure. 

You need to make sure to configure your Kafka consumer to manually commit the offsets. Kafka message will need to include a unique identifier (i.e. idempotency key) to allow for deduplication logic in your application. Identifier can be included either within the payload, or as a Kafka message header.

To implement the Idempotent Consumer pattern you can store and track processed messages in a database. On consuming a new message you simply check for existence of the message id in your database and skip the message processing if it's a duplicate. 

If the message is not a duplicate, then you process a message, commit a message id to the database (within a database transaction), and then commit the new consumer offset to a Kafka broker. The order of DB transaction and Kafka offset commits is important.

### Responding to a REST call

If the trigger is a REST call you will need to rely on the client to retry the operation. The REST API will also need to include an idempotency key. Similarly, you can track received message IDs in the database to handle idempotency.

### Writing outputs to Kafka (With Transactional Outbox Pattern)

### Writing outputs to Kafka (Without Transactional Outbox Pattern)

Transactional Outbox Pattern comes with a complexity overhead, depending on what CDC solution your database supports. You can use [Debezium](https://debezium.io) and [Kafka Connect](https://docs.confluent.io/platform/current/connect/index.html) to integrate CDC with a Postgres DB for example. But that will still result in another component that needs to be managed and monitored, and another possible point of failure. In certain situations it is easier to avoid the Transactional Outbox Pattern and handle writes to Kafka within the application.

The solution is ony applicable for use cases where the trigger is a Kafka message, so your application will read from Kafka, process the message, and send the result to Kafka. 

If you implement the Idempotent Consumer Pattern explained above, and the end step of producing a message to Kafka....

- TODO explain more

### Idempotent Processing

Regardless of the pattern used, there is no exactly-once guarantee for application processing. All other actions occurring as part of the processing can still happen multiple times. For example, in case of REST calls to other services, calls themselves need to be idempotent, and the same idempotency key needs to be relayed over to those calls.