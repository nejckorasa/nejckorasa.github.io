---
title: "Idempotent Processing with Kafka"
date: 2023-02-04
tags: ["Kafka", "Idempotency", "Software Architecture", "Event Driven Architecture", "Asynchronous Processing", "Transactional Outbox", "Distributed Systems"]
categories: Software Engineering
---

- [Duplicate Messages are Inevitable](#duplicate-messages-are-inevitable)
- [Understanding the Intricacies of exactly-once semantics in Kafka](#understanding-the-intricacies-of-exactly-once-semantics-in-kafka)
- [Achieving Idempotent Processing with Kafka](#achieving-idempotent-processing-with-kafka)
    - [Idempotent Consumer Pattern](#idempotent-consumer-pattern)
    - [Ordering of Messages](#ordering-of-messages)
    - [Retry Handling](#retry-handling)
    - [Idempotent Processing and External Side Effects](#idempotent-processing-and-external-side-effects)
- [Publishing Output Messages to Kafka and Maintaining Data Consistency](#publishing-output-messages-to-kafka-and-maintaining-data-consistency)
    - [The Simplest Solution](#the-simplest-solution)
    - [Transactional Outbox Pattern](#transactional-outbox-pattern)
    - [Without Transactional Outbox](#without-transactional-outbox)
- [How it compares to Synchronous REST APIs](#how-it-compares-to-synchronous-rest-apis)
- [Final Thoughts](#final-thoughts)

## Duplicate Messages are Inevitable

Duplicate messages are an inherent aspect of message-based systems and can occur for various reasons. In the context of Kafka, it is essential to ensure that your application is able to handle these duplicates effectively. As a Kafka consumer, there are several scenarios that can lead to the consumption of duplicate messages:

- There can be an actual duplicate message in the kafka topic you are consuming from. The consumer is reading 2 different messages that should be treated as duplicates.
- You consume the same message more than once due to various error scenarios that can happen, either in your application, or in the communication with a Kafka broker.
  
To ensure the idempotent processing and handle these scenarios, it's important to have a proper strategy to detect and handle duplicate messages.

## Understanding the Intricacies of exactly-once semantics in Kafka

Kafka offers different message delivery guarantees, or [delivery semantics](https://kafka.apache.org/documentation/#semantics), between producers and consumers, namely _at-least-once_, _at-most-once_ and _exactly-once_.

Exactly-once would seem like an obvious choice to guard against duplicate messages, but it not that simple and the devil is in the details. Confluent has spent a lot of resources to deliver exactly-once delivery guarantee, and you can read [here](https://www.confluent.io/en-gb/blog/exactly-once-semantics-are-possible-heres-how-apache-kafka-does-it/) on how it works in detail. It requires enabling specific Kafka features (i.e. Idempotent Producer and Kafka Transactions). 

First of all, it is only applicable in an application that consumes a Kafka message, does some processing, and writes a resulting message to a Kafka topic. Exactly-once messaging semantics ensures the **combined** outcome of multiple steps will happen exactly-once. Key word here is combined. A message will be consumed, processed, and resulting messages produced, exactly-once. 

Critical points to understand about exactly-once delivery are:

- **All other actions occurring as part of the processing can still happen multiple times, if the original message is re-consumed.**
    
    The guarantee only covers resulting messages from the processing to be written exactly once, so downstream transaction aware consumers will not have to handle duplicates. Hence, each individual action (internal or external) still needs to be processed in an idempotent fashion to ensure real end-to-end exactly once processing. Application may need to, for example, perform REST calls to other applications, write to the database etc.

- **All participating consumers and producers need to be configured correctly.**

    Kafka exactly-once semantics is achieved by enabling [Kafka Idempotent Producers](https://www.conduktor.io/kafka/idempotent-kafka-producer) and [Kafka Transactions](https://www.confluent.io/en-gb/blog/transactions-apache-kafka/) in **all** consumers and producers involved. That includes the upstream producer and downstream consumers from the perspective of you application.

   If you are using Event-driven architecture to implement inter-service communication in your system, it is likely that you will consume messages you don't control, or own. Kafka topic is just your asynchronous API you are a consumer of. The topic and the producer can be owned by another team or a 3rd party. Similarly, you may not control downstream consumers. 

    To add to the first point, outbound messages can still be written to the topic multiple times before being successfully committed, it is the responsibility of any downstream consumers to only read committed messages (i.e. be transaction aware) in order to meet the exactly-once guarantee.

- **it comes with a performance impact.**

    Exactly-once delivery comes with a performance overhead. There are simply more steps involved for a single kafka message to be processed (e.g. Kafka performs a two-phase commit to support transactions) and that results in lower throughput and increased latency.

In practice, it's often much simpler, and more common, to settle for at-least-once semantic and just de-duplicate messages on the consumer side. Especially in cases where application processing is either expensive, or more involved and consists of other actions (e.g. REST calls and DB writes). It's important to remember there is a transaction boundary gap between a DB transaction and a Kafka transaction, more on that later.

## Achieving Idempotent Processing with Kafka

This will depend on the nature of processing, and on the shape of the output. To enable idempotent processing, the trigger for the processing - whether it be a Kafka message or an HTTP request - must carry a unique identifier (i.e. an idempotency key).

### Idempotent Consumer Pattern

An [Idempotent Consumer Pattern](https://microservices.io/patterns/communication-style/idempotent-consumer.html) ensures that a Kafka consumer can handle duplicate messages correctly. Consumer can be made idempotent by recording in the database the IDs of the messages that it has processed successfully. When processing a message, a consumer can detect and discard duplicates by querying the database. To illustrate that with pseudocode:

```java
var kafkaMessage = kafkaConsumer.consume();

if (!database.isDuplicate(kafkaMessage)) {
    var result = processMessageIdempotently(kafkaMessage);
    database.updateAndRecordProcessed(result);
}

kafkaConsumer.commitOffset(kafkaMessage);
```

### Ordering of Messages

Choosing an appropriate topic key can help to ensure ordering guarantees within the same Kafka partition. For example, if messages are being processed in the context of a customer, using a customer ID as the topic key will ensure that messages for any individual customer will always be processed in the correct order.

### Retry Handling

Kafka's offset commits can be used to create a "transaction boundary" (not to be confused with Kafka transactions mentioned before) for retrying message processing in case of failure. The same message can then be consumed again until the consumer offset is committed. Retry handling is a complex topic and various strategies can be employed depending on the specific requirements of the application. Confluent has written about [Kafka Error Handling Patterns](https://www.confluent.io/en-gb/blog/error-handling-patterns-in-kafka/) that can be used to handle retries in a Kafka-based application.

### Idempotent Processing and External Side Effects

As mentioned before, there is no exactly-once guarantee for application processing. All actions occurring as part of the processing, and all external side effects, can still happen multiple times. For example, in case of REST calls to other services, calls themselves need to be idempotent, and the same idempotency key needs to be relayed over to those calls. Similarly, all database writes need to be idempotent.

## Publishing Output Messages to Kafka and Maintaining Data Consistency

When it comes to publishing messages back to Kafka after processing is complete, the complexity increases. In a Microservices architecture, services along with updating their own local data store they often need to notify other services within the organization of changes that have occurred. This is where event-driven architecture shines, allowing individual services to publish changes as events to a Kafka topic that can be consumed by other services. But how can this be achieved in a way that ensures data consistency and enables idempotent processing?

### The Simplest Solution

Consuming from Kafka has a built-in retry mechanism. If the processing is naturally idempotent, deterministic, and does not interact with other services (i.e. all its state resides in Kafka), then the solution can be relatively simple:

```java
var kafkaMessage = kafkaConsumer.consume();

var result = processMessageIdempotently(kafkaMessage);

var kafkaOutputMessage = result.toKafkaOutputMessage();

kafkaProducer.produceAndFlush(kafkaOutputMessage);
kafkaConsumer.commitOffset(kafkaMessage);
```

1) Consume the message from a Kafka topic.
2) Process the message.
3) Publish the resulting message to a Kafka topic.
4) Commit the consumer offset.

This approach ensures data consistency and enables idempotent processing. It guarantees that at least one published message is produced for every consumed message. 

To ensure at least-once delivery of published messages, it's also necessary to ensure that the message is _actually_ sent to the Kafka broker and that the Kafka producer has flushed its outgoing message queue.

### Transactional Outbox Pattern

Another approach is to utilize [Transactional Outbox Pattern](https://microservices.io/patterns/data/transactional-outbox.html) which fills the gap between the database and Kafka transaction boundary by atomically updating both within the database transaction. The reason being that it is not possible to have a single transaction that spans the application’s database as well as Kafka.

One possible implementation of this pattern is to have an “_outbox_” table and instead of publishing resulting messages directly to Kafka, the messages are written to the outbox table in a compatible format (e.g. [Avro](https://www.confluent.io/en-gb/blog/avro-kafka-data/)).

```java
var kafkaMessage = kafkaConsumer.consume();

if (!database.isDuplicate(kafkaMessage)) {
    var result = processMessageIdempotently(kafkaMessage);
    
    var transaction = database.startTransaction();
    database.updateAndRecordProcessed(result);
    database.writeOutbox(result);
    transaction.commit();
}

kafkaConsumer.commitOffset(kafkaMessage);
```

However, this pattern comes with additional complexity. The message must not only be written to the database but also published to Kafka. This can be implemented by a separate message relay service that continuously polls the database for new outbox messages, publishes them to Kafka, and marks them as processed. However, this approach has several drawbacks:

- Increased load on the database: Frequently polling the database can cause a high level of read traffic, which can lead to increased load on the database and potentially slow down other processes that are trying to access it.
- Latency: Depending on the interval at which the database is polled, there may be a significant delay between when a message is added to the outbox and when it is published to Kafka.
- Scalability: If the number of messages to be published to Kafka increases, the rate of polling will need to be increased, which can further increase the load on the database and make the system less scalable.
- Schema incompatibility issues: If the message schema is incompatible with a destination topic, application processing will succeed, but the poller could be unable to publish a message to Kafka. The risk of this can be minimized by verifying Avro schema with a schema registry before writing to the outbox table.
- Ordering of messages: Poller needs to ensure the order of messages written to the outbox tables is retained when publishing to Kafka.
- Missed messages: There is a chance that a message is not picked up by the poller and not published to Kafka.
- Lack of real-time: The messages are not published to kafka in real-time as it depends on the polling interval.

A better approach is to utilize [CDC (change data capture)](https://learn.microsoft.com/en-us/sql/relational-databases/track-changes/about-change-data-capture-sql-server?view=sql-server-ver16) if your database supports it. You can use [Debezium](https://debezium.io) and [Kafka Connect](https://docs.confluent.io/platform/current/connect/index.html) to integrate CDC with a PostgresDB for example. That way, the database and Kafka stay in sync, and you don't have to deal with the drawbacks of database polling.

### Without Transactional Outbox

However, even with the use of CDC, that will still result in another component that needs to be managed and monitored, and another possible point of failure. In certain situations it is easier to avoid the Transactional Outbox Pattern and handle writes to Kafka within the application. That can be achieved by combining the first simple solution explained above with the Idempotent Consumer pattern:

```java
var kafkaMessage = consumeKafkaMessage(kafkaClient);

if (!database.isDuplicate(kafkaMessage)) {
    result = processMessageIdempotently(kafkaMessage);
    database.updateAndRecordProcessed(result);
} else {
    result = database.readResult(kafkaMessage);
}

var kafkaOutputMessage = result.toKafkaOutputMessage();
kafkaProducer.produceAndFlush(kafkaOutputMessage);
kafkaConsumer.commitOffset(kafkaMessage);
```

1) Consume the message from a Kafka topic.
2) Consult the database to confirm the message has not been previously processed. 
If it has, read the stored result and proceed to step 5.
3) Process the message, taking care to handle any external actions in an idempotent manner.
4) Write results to the database and mark the message as successfully processed.
5) Publish the resulting message to a Kafka topic.
6) Commit the consumer offset.

The approach outlined above combines the use of the Idempotent Consumer pattern with direct publishing to Kafka, resulting in a streamlined solution for handling duplicate messages. 

Additionally, by eliminating the need for an intermediate "_outbox_" table, this approach reduces the number of components that need to be managed and monitored, resulting in a simpler overall architecture. 

Furthermore, it also benefits from reduced latency in message publishing as it avoids the added step of writing to a database before publishing to Kafka.

This approach has some downsides to consider:
- It might simplify overall architecture but it increases the complexity of processing within the application.
- The addition of a Kafka publish step can cause a performance overhead and prolong overall processing time.

## How it compares to Synchronous REST APIs

Similarly to the Idempotent Consumer Pattern, in case of a REST API, received message IDs could also be tracked in a database to handle idempotency. However, there are drawbacks to using REST call as a trigger for processing, namely:
- The retry strategy is out of the control of the application, and the caller is responsible for retrying the operation. That makes it more susceptible to failure scenarios and inconsistent states.
- There is no ordering guarantee when responding to HTTP calls, and additional care must be taken to avoid certain race conditions during processing.

Publishing output messages to Kafka in a way that maintains data consistency can be achieved by using [Transactional Outbox Pattern](https://microservices.io/patterns/data/transactional-outbox.html) to atomically update the database and publish a message to Kafka.

## Final Thoughts

Kafka is an ideal platform for implementing idempotent processing in your application, and it offers several key advantages over traditional synchronous processing methods such as REST APIs. Its built-in retry mechanism and ordering guarantees are essential for ensuring idempotence and maintaining data consistency in the presence of failures.

When it comes to message delivery guarantees, the exactly-once semantics offered by Kafka can be a powerful tool to guard against duplicate messages. However, it's important to understand the intricacies of this feature, the requirements for its implementation, and its limitations. Additionally, the performance impact and complexity of exactly-once semantics should be taken into consideration.

Achieving idempotent processing requires a thorough understanding of the triggers, actions, and outputs of the processing. Different approaches such as [Idempotent Consumer Pattern](https://microservices.io/patterns/communication-style/idempotent-consumer.html) and [Transactional Outbox Pattern](https://microservices.io/patterns/data/transactional-outbox.html) can be used to ensure that messages are processed correctly and that data consistency is maintained. It's important to weigh the complexity and potential drawbacks of each approach before deciding on the best solution for your application. As we have seen, Transactional Outbox is not always necessary.
