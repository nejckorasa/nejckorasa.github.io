---
title: "Idempotent Kafka Processing"
date: 2023-01-14
tags: ["Kafka", "Idempotency", "Software Architecture", "Event Driven Architecture", "Transactional Outbox"]
categories: Software Engineering
---

## Duplicate messages are inevitable

Duplicate messages are an inherent aspect of message-based systems and can occur for various reasons. In the context of Kafka, it is essential to ensure that your application is able to handle these duplicates effectively. As a Kafka consumer, there are several scenarios that can lead to the consumption of duplicate messages. These include:

- There can be an actual duplicate message in the kafka topic you are consuming from. The consumer is reading 2 different messages that should be treated as duplicates.
- You consume the same message more than once due to various error scenarios that can happen, either in your application, or in the communication with a Kafka broker.
  
To ensure the idempotent processing and handle these scenarios, it's important to have a proper strategy to detect and handle duplicate messages.

## Kafka delivery guarantees

Kafka offers different message delivery guarantees between producers and consumers, namely _at-least-once_, _at-most-once_ and _exactly-once_. 

### Understanding the intricacies of exactly-once semantics in Kafka

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

In practice, it's often much simpler, and more common, to settle for at-least-once semantic and just de-duplicate messages on the consumer side. Especially in cases where application processing is more involved and consists of other actions (e.g. REST calls and DB writes). It's important to remember there is a transaction boundary gap between a DB transaction and a Kafka transaction, more on that later.

## Achieving Idempotent Processing in Kafka

This will depend on what triggers the processing, what actions constitute the processing, and on the shape of the output. To enable idempotent processing, the trigger for the processing - whether it be a Kafka message or an HTTP request - must carry a unique identifier (i.e. an idempotency key).

### Consuming from Kafka (Idempotent Consumer Pattern)

An [Idempotent Consumer Pattern](https://microservices.io/patterns/communication-style/idempotent-consumer.html) ensures that a Kafka consumer can handle duplicate messages correctly. You can make a consumer idempotent by recording in the database the IDs of the messages that it has processed successfully. When processing a message, a consumer can detect and discard duplicates by querying the database.

#### Ordering of Messages

Choosing an appropriate topic key can help to ensure ordering guarantees within the same Kafka partition. For example, if messages are being processed in the context of a customer, using a customer ID as the topic key will ensure that messages for any individual customer will always be processed in the correct order.

#### Retry handling

Kafka's offset commits can be used to create a "transaction boundary" (not to be confused with Kafka transactions mentioned before) for retrying message processing in case of failure. The same message can then be consumed again until the consumer offset is committed. Retry handling is a complex topic and various strategies can be employed depending on the specific requirements of the application. Confluent has written about [Kafka Error Handling Patterns](https://www.confluent.io/en-gb/blog/error-handling-patterns-in-kafka/) that can be used to handle retries in a Kafka-based application.

### Responding to a REST call

When the trigger for the processing is a REST call, the client must be responsible for retrying the operation. The REST API must also include an idempotency key. Similarly to the Idempotent Consumer Pattern, received message IDs can be tracked in a database to handle idempotency. However, there is no ordering guarantee when responding to HTTP calls, and additional care must be taken to avoid certain race conditions during processing.

### Writing Outputs to Kafka and maintaining Data Consistency

When it comes to publishing messages back to Kafka after processing is complete, the complexity increases. In a Microservices architecture, services along with updating their own local data store they often need to notify other services within the organization of changes that have occurred. This is where event-driven architecture shines, allowing individual services to publish changes as events to a Kafka topic that can be consumed by other services. But how can this be achieved in a way that ensures data consistency and enables idempotent processing?

#### Handling REST calls

When processing a REST call, the retry strategy is out of the control of the application, making it more susceptible to failure scenarios and inconsistent states. To address this, the recommended approach is to use the Transactional Outbox Pattern and atomically update the database and publish a message to Kafka.

#### Consuming from Kafka

On the other hand, consuming from Kafka has a built-in retry mechanism. If the processing is naturally idempotent, deterministic, and does not interact with other services (i.e. all its state resides in Kafka), then the solution can be relatively simple:

1) Consume the message from a Kafka topic.
2) Process the message.
3) Publish the resulting message to a Kafka topic.
4) Commit the consumer offset.

This approach ensures data consistency and enables idempotent processing. It guarantees that a published message is produced for every consumed message. To ensure at least-once delivery of published messages, it's also necessary to ensure that the message is actually sent to the Kafka broker and that the Kafka producer has flushed its outgoing message queue.

#### Transactional Outbox Pattern

Another approach is to utilize [Transactional Outbox Pattern](https://microservices.io/patterns/data/transactional-outbox.html) which fills the gap between the database and Kafka transaction boundary by atomically updating both. The reason being that it is not possible to have a single transaction that spans the application’s database as well as Kafka.

One possible implementation of this pattern is to have an “Outbox” table and instead of publishing API-called events directly to Kafka, the messages are published to the Outbox table in an AVRO-compatible format.

However, this pattern comes with additional complexity. The message must not only be written to the database but also published to Kafka. This can be implemented by a separate message relay service that continuously polls the database for new outbox messages, publishes them to Kafka, and marks them as processed. However, this approach has several drawbacks:

- Increased load on the database: Frequently polling the database can cause a high level of read traffic, which can lead to increased load on the database and potentially slow down other processes that are trying to access it.
- Latency: Depending on the interval at which the database is polled, there may be a significant delay between when a message is added to the outbox and when it is published to Kafka.
- Scalability: If the number of messages to be published to Kafka increases, the rate of polling will need to be increased, which can further increase the load on the database and make the system less scalable.
- Missed messages: There is a chance that a message is not picked up by the poller and not published to Kafka.
- Lack of real-time: The messages are not published to kafka in real-time as it depends on the polling interval.

A better approach is to utilize [CDC (change data capture)](https://learn.microsoft.com/en-us/sql/relational-databases/track-changes/about-change-data-capture-sql-server?view=sql-server-ver16) if your database supports it. You can use [Debezium](https://debezium.io) and [Kafka Connect](https://docs.confluent.io/platform/current/connect/index.html) to integrate CDC with a Postgres DB for example. That way, the database and Kafka stay in sync, and you don't have to deal with the drawbacks of database polling.

Even with the use of CDC, that will still result in another component that needs to be managed and monitored, and another possible point of failure. In certain situations it is easier to avoid the Transactional Outbox Pattern and handle writes to Kafka within the application, as explained with the first simple solution above.

### Idempotent Processing

Regardless of the pattern used - whether the application publishes to Kafka directly, or with the help of Transactional Outbox Pattern - there is no exactly-once guarantee for application processing. All other actions occurring as part of the processing can still happen multiple times. For example, in case of REST calls to other services, calls themselves need to be idempotent, and the same idempotency key needs to be relayed over to those calls. Similarly, all database writes need to be idempotent as well.

## Final Thoughts

Kafka is an ideal platform for implementing idempotent processing in your application, and it offers several key advantages over traditional synchronous processing methods such as REST APIs. Its built-in retry mechanism and ordering guarantees are essential for ensuring idempotence in the presence of failures.

When it comes to message delivery guarantees, the exactly-once semantics offered by Kafka can be a powerful tool to guard against duplicate messages. However, it's important to understand the intricacies of this feature and the requirements for its implementation. Additionally, the performance impact and complexity of exactly-once semantics should be taken into consideration.

Achieving idempotent processing requires a thorough understanding of the triggers, actions, and outputs of the processing. Different approaches such as [Idempotent Consumer Pattern](https://microservices.io/patterns/communication-style/idempotent-consumer.html) and [Transactional Outbox Pattern](https://microservices.io/patterns/data/transactional-outbox.html) can be used to ensure that messages are processed correctly and that data consistency is maintained. It's important to weigh the complexity and potential drawbacks of each approach before deciding on the best solution for your application. The outbox pattern is not always necessary.