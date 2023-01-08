---
title: "Avoid Tight Coupling of Tests to Implementation Details"
date: 2023-01-08
tags: ["Testing", "Microservices", "Software Architecture"]
categories: Software Engineering
---

Building backend systems today will likely involve building many small, independent services that communicate and coordinate with one another to form a distributed system. While there are many resources available discussing the pros and cons of microservices, the architecture, and when it is appropriate to use, I want to focus on the functional testing of microservices and how it differs from traditional approaches.

In my experience, the "best testing practices" have evolved with the introduction of microservices, and traditional _testing pyramids_ may not be the most effective or even potentially harmful in this context. In my work on various projects and companies, including the development of new digital banks and the migration of older systems to microservices as they scale, I have often encountered disagreements about the most appropriate testing strategies for microservices.

### Why do we have tests?

As software engineers, we rely on testing to verify that our code functions as expected. Testing should support refactoring, but it can sometimes make it more difficult. The purpose of testing is to define the intended behavior of the code, rather than the details of its implementation. In summary, tests should:

- Confirm that the code does what it should.
- Provide fast, accurate, reliable, and predictable feedback.
- Make maintenance easier, which is often overlooked when writing tests.

Effective testing is crucial for building reliable software, and it is important to keep these goals in mind when writing tests. By focusing on the intended behavior of the code and the needs of maintenance, we can write tests that give us confidence in our code and make the development process more efficient.


#### Common Mistakes 

It is not uncommon to come across codebases with a large number of tests and high test coverage percentages, only to find that the code is not truly tested and that refactoring or adding new features is difficult. In my experience, this is often due to the following pitfalls:

- Overreliance on unit tests
- Tight coupling of tests to implementation details

Avoiding these mistakes is key to writing effective tests that support the development process and ensure the reliability of the code.

### Do you need Unit Tests?

One common approach to testing is the belief that all classes, functions, and methods must be tested. This can lead to a large number of unit tests and a high test coverage percentage. However, an excess of unit tests can make it difficult to change the code without also having to modify the tests. This can undermine the confidence in the code and negate the benefits of testing if the tests must be rewritten every time the code is changed.

In the case of microservices, which are small and independent by definition, it could be argued that the microservice itself is a unit and should be tested as an isolated component through its contracts, or in a black-box fashion. In this sense, the term "unit tests" for microservices can be thought of as implementation detail tests. Instead of focusing on unit tests, it may be more effective to consider the testing of microservices at a higher level, such as through integration tests.

### Don't couple you tests to Implementation Details

{{< tweet user="NejcKorasa" id="1599395920281743361" >}}

When writing tests, it is important to avoid coupling them to implementation details. This ensures that tests serve as a reliable safety net, allowing you to refactor the internals of your microservice without having to modify the tests. 

### Focus on Integration Tests

To avoid testing implementation details we should test from the edges of microservices, by examining the inputs and outputs of the service and verifying their correctness in an isolated manner while focusing on the interaction points and making them very explicit. 

#### Define Inputs and Outputs

Look at the entrypoint of the service (e.g. a REST API, Kafka consumer) to define the inputs for your tests and find the corresponding outputs (e.g. HTTP response, published Kafka message). It may be necessary to assert multiple outputs for a single input, as processing an HTTP request could result in a database update, new kafka message, and HTTP response

#### Test the Microservice as an Isolated Component (Unit)

Spin up the microservice and all necessary infrastructure components, such as web servers and databases, and send inputs to verify the outputs. Tools like [Testcontainers for Java](https://www.testcontainers.org) can help by running the application in a short-lived test mode with dependencies, such as databases and message queues, running in Docker containers.

By setting up specific infrastructure components in a separate test setup stage, you can isolate them from the actual tests, allowing you to change the underlying infrastructure without modifying the test methods themselves (e.g. replacing the database from PostgreSQL to NoSQL).

> This approach is similar to [hexagonal architecture](https://en.wikipedia.org/wiki/Hexagonal_architecture_(software)), which decouples infrastructure and domain logic, but the testing strategy will differ.
> 
> There is a cost to it as it adds some complexity, but I have seen codebases where the benefits were worth it. Ultimately, the decision of how much complexity to add through isolation should be based on how often you anticipate changing the infrastructure of the service and whether the added complexity is justified.

#### Clear Definition of Microservice Behavior through Testing

A test suite with a focus on integration tests will likely have fewer tests overall, but they will clearly define the expected behavior of the microservice. When examining the test suite, you should be able to get a clear understanding of what the microservice is intended to do.

### There still is a place for Implementation Details Tests

There will be parts of the code that are domain specific and only contain business logic. Those naturally isolated parts have an internal complexity of their own and this is where implementation details tests should be used. Testing all variations and edge cases will be cumbersome and too heavy to test through integration tests. 


### References
- [Spotify: Testing of Microservices](https://engineering.atspotify.com/2018/01/testing-of-microservices/)
- [Testcontainers for Java](https://www.testcontainers.org)