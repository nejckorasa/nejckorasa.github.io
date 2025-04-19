---
title: "CockroachDB transaction retries with Spring Boot and Resilience4j"
date: 2023-02-04
tags: ["SQL", "Spring Boot", "CockroachDB", "Database Transactions"]
categories: Software Engineering
---

## What is CockroachDB?

CockroachDB is a distributed SQL database built on a transactional and strongly-consistent key-value store. It scales horizontally, supports strongly-consistent ACID transactions, and PostgreSQL wire protocol. It  guarantees serializable SQL transactions (the highest isolation level defined by the SQL standard), ensuring each transaction is committed to the database sequentially, maintaining data consistency. 

## Failed Transactions

Strong ACID guarantees, however, come at a cost - occasionally transactions will fail and will need to be retried. When a transaction is unable to complete due to contention with another concurrent or recent transaction attempting to write to the same data, CockroachDB will automatically attempt to retry the failed transaction without involving the client. 

If the automatic retry is not possible or fails, a transaction retry error is emitted to the client. In most cases, the correct actions to take when encountering transaction retry errors are:

- Take steps to minimize transaction retry errors in the first place, e.g. reducing transaction contention overall, and increasing the likelihood that CockroachDB can automatically retry a failed transaction.
- Update your application to support client-side retry handling.


There is a good official blog post on [what to do when a transaction fails in CockroachDB](https://www.cockroachlabs.com/blog/what-to-do-when-a-transaction-fails-in-cockroachdb/#retrying-transactions-using-application-logic), as well as the official documentation on [transaction retry error reference](https://www.cockroachlabs.com/docs/v23.1/transaction-retry-error-reference#minimize-transaction-retry-errors).

Here, I want to focus on describing how client-side transaction retries can be achieved in Spring Boot.

## Retrying Failed Transactions in Spring Boot, with Resilience4j

