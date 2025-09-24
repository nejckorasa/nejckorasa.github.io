# Kafka Backfills Done Right: A Guide to Rehydrating Services

## 1. The Inevitable Backfill: Why We Need to Rehydrate Data

Many of us have adopted **Event Sourcing** as a core pattern in our microservice architectures. Using an immutable log of events as the source of truth, with Kafka as its backbone, provides incredible flexibility and scalability. In a perfect world, we would keep this event log in Kafka forever. In the real world, however, storing an ever-growing history on high-performance broker disks is prohibitively expensive.

This leads to the inevitable compromise: **data retention policies**. We keep a few weeks or months of events in Kafka for real-time processing and offload the rest to cheaper, long-term cold storage like Amazon S3. This process becomes part of a general Data Lake sink strategy.

This works perfectly until one of these classic scenarios demands access to the full historical record:

* **Bootstrapping a New Service:** A new microservice needs to build its own materialized view of the world by processing the entire history of events, but 90% of those events are no longer on the Kafka topic.
* **Recovering from a Bug:** A subtle bug is discovered in a service, and you need to rebuild its state from a point in time months ago—long before the bug was introduced.
* **Enriching Data for New Features:** A new feature requires historical context, forcing a re-process of old events to enrich existing data models.

The core problem is the same: how do we gracefully rehydrate our services with data that now lives in cold storage?

## 2. The Two Paths to Rehydration

Here are the primary architectural choices for getting historical data from cold storage back to your services.

### Option 1: Kafka Tiered Storage

The most elegant solution is one that eliminates the need for a separate ETL process: using a Kafka distribution that supports **Tiered Storage**. This feature allows Kafka to automatically move older event segments to object storage like S3, while the topic's log remains logically intact and infinitely queryable. The data is physically in two places, but Kafka presents it as a single, seamless stream.

With Tiered Storage, a backfill becomes a trivial operational task. The process is simply:

1.  **Reset the consumer offset.** Use standard Kafka tooling to tell your service's consumer group to start reading from the beginning of the topic or a specific timestamp. Ideally, you would create a separate consumer to handle the backfill.
2.  **Let the consumer run.** Kafka automatically fetches the older data from S3 and streams it to your consumer, just as it would with live data.

### Option 2: The Pragmatic ETL Pattern (e.g., AWS Glue Trigger)

But what if you don’t have Tiered Storage? This is a very common scenario. You need a safe, reliable bridge between your S3 data lake and Kafka. Instead of writing ad-hoc scripts, you should invest in a reusable backfill platform.

The core of this pattern is a generic, on-demand ETL job (AWS Glue or Spark is a perfect fit) that is triggered by a simple API call. A service team makes a request specifying:

* The **source S3 path** of the historical data.
* A **destination Kafka topic** to publish to.

This "Backfill Publisher" job reads the data from S3, transforms it, and produces it onto the specified Kafka topic. It is completely decoupled from the service that will eventually consume the data.

[![Glue Job Diagram](/static/glue-job-diagram.png)](/static/glue-job-diagram.png)

#### Handling Schema Evolution in the Glue Job

The goal is to make the consumer's experience identical, regardless of where the data originates. With Tiered Storage, schema evolution is handled transparently; ideally, a Glue Job would replicate this.

**The Strategy: Evolve-on-Read**

The Glue job's responsibility is to pre-process the historical data so that it appears identical to live data for the consumer. Using a schema registry, the job performs a "schema-on-read" transformation:

1.  **It reads** the raw data from S3, which may use many different historical Avro schema versions.
2.  **It evolves** each record to conform to the **single, latest schema version**, automatically adding default values for new fields and ignoring removed ones.
3.  **It writes** the clean, transformed data to the backfill Kafka topic.

This means the service consumer's logic remains incredibly simple. It only needs to be aware of the latest schema and doesn't require any complex, backward-compatible code because the backfill pipeline has already handled that complexity.

## 3. Handling Rehydration in a Service: The Consumer's Responsibility

Getting the data back into Kafka is only half the challenge. The consuming service must be architected to handle rehydration safely, without disrupting live traffic.

### Idempotent Processing is Non-Negotiable

When a service re-processes historical events, it will inevitably encounter data it has already seen. The consumer logic must be **idempotent**, meaning that processing the same event multiple times produces the same result as processing it once.

### Isolate the Flood: New vs. Existing Consumers

It is almost always better to create a **new, dedicated Kafka consumer** for the backfill. This is critical for safety, because:

* **Isolation:** Isolating the high-volume, low-priority backfill traffic from the live production topic. This prevents a backfill from causing lag and impacting real-time processing.
* **Control:** Separate control of the consumer to manage its pace to avoid overwhelming the service and its database.

### Zero-Downtime Rebuilds: The Versioned Data Swap

For critical services, the safest approach is to backfill into a new, versioned database table—similar to a "blue-green" deployment for data.

**The Process**

To build the new table (`orders_v2`) without missing any data, you need two parallel streams of writes:

1.  **The Live Stream (Dual-Writing):** First, modify your existing service. The live consumer, which processes real-time events, now writes to **both** the old table (`orders_v1`) and the new one (`orders_v2`). This ensures any new data is captured in the new table from the moment you start.
2.  **The Historical Stream (Backfill Consumer):** Second, start the **new, dedicated backfill consumer**. Its job is to process all the historical data from the backfill topic and populate `orders_v2`.

With both consumers running, `orders_v2` is guaranteed to receive a complete data set.

**The Cutover**

How you switch your application to use the new table depends on your schema:

* **Simple Case (Backward-Compatible Schema):** If `orders_v2` has a schema that your *current* application code can read, you might be able to perform a simple, atomic swap.
* **Complex Case (Breaking Schema Change):** If `orders_v2` has a different structure that would break your existing code, you need a different strategy and should follow the [Parallel Change](https://martinfowler.com/bliki/ParallelChange.html) (expand and contract) pattern.

[![Zero Downtime Rebuild](/static/zero-downtime-rebuild.png)](/static/zero-downtime-rebuild.png)

## 4. Accelerating Rehydration with Snapshots

Replaying every event from the beginning of time can be slow and resource-intensive. For many use cases, you can accelerate the process by using a **snapshot**—a precomputed, materialized state of your data. There are two primary ways to implement this:

1.  **State Snapshots:** A periodically generated full snapshot of an entity's state. Rehydration then involves loading this snapshot and replaying only the events from Kafka that have occurred *since* the snapshot was created.
2.  **Kafka-Native Snapshots (Log Compaction):** For services that only need the *current state* of an entity, Kafka's **log compaction** provides a powerful, built-in solution. A compacted topic automatically retains at least the last known value for each message key. Reading this topic from the beginning provides a consumer with a full, live snapshot of the current state.