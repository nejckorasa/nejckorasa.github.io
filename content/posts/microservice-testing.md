---
title: "Don't couple you tests to implementation details"
date: 2023-01-08
tags: ["Testing", "Microservices", "Software Architecture"]
categories: Software Engineering
---

Building backend systems today will likely involve building many (micro) services that have to communicate and coordinate with each other, thus making a system distributed. There are a lot of blog posts about microservice architecture, its Dos and Don'ts, when to use it and when you might be better off building a monolith, or at least starting with one and evolve the architecture as you scale up.

I want to expand more on the functional testing of microservices and what makes it different. Why I believe that _"best testing practices"_ have fundamentally changed with the introduction of microservices and why testing _pyramids_ are generally not the best choice and could even be harmful.

In the past few years I've had an opportunity to work in a few different projects and companies, from building new digital banks to migrating older systems towards microservices as they were scaling up. I have also often found myself (and I still do) in disagreements about what testing strategies should be employed to microservices.
### Why do we have tests?

We test our code to verify that things work the way we expect them to. Testing should support refactoring but many times, it makes it harder. Tests should aim to define what is intended (behavior); not how it’s implemented (details). In summary, tests should:

- Give us confidence that the code does what it should.
- Provide fast, accurate, reliable and predictable feedback.
- Make maintenance easier, something that is commonly overlooked when writing tests.

#### Common mistakes 

I've seen many codebases with loads of tests and very high test coverage percentage that were not _actually_ tested, and made refactoring or even implementing new features a very unpleasant experience. Almost always that was due to the red flags:

- High usage of unit tests
- Coupling tests to implementation details

### Do you need Unit Tests?

The simplest (and dangerous) rule of testing is to _“test all classes, functions, and methods”_. Following that rule will get you a high testing coverage percentage and probably a high number of unit tests, one per each class. 

Having too many unit tests in microservices, which are small by definition, restricts how we can change the code without also having to change the tests. By having to change the tests we lose some confidence that the code still does what it should. And if you have to rewrite the tests every time you change the code, what was the benefit of those tests?

As microservices are small by nature you could argue that **microservice is a unit** and unit testing a microservice is treating it as an isolated component, tested through its contracts (i.e. in a black-box fashion). In that sense the term Unit Tests for microservices can be viewed as Implementation Detail Tests.

### Don't couple you tests to implementation details

{{< tweet user="NejcKorasa" id="1599395920281743361" >}}

Tests should be your safety net, when you refactor the internals of your microservice and existing tests are all green, that should be a reliable indicator that you haven't changed the expected behaviour of the microservice.

### Focus on Integration tests

To avoid testing implementation details we should test from the edges of microservices and look for clear inputs and outputs of the service. We should aim to verify the correctness of our service in a more isolated fashion while focusing on the interaction points and making them very explicit. 

#### Define inputs and outputs

Look at the entrypoint of the service (e.g. a rest API, Kafka consumer) to define the inputs for your tests and find the corresponding outputs (e.g. http response, published Kafka message). It's possible that your service will produce more than one output - processing a http request could result in a database update, new kafka message and a http response. Ideally your tests will assert that all the outputs.

#### Test the microservice as an isolated component (unit)

Spin up your microservice and all the infrastructure components (e.g. web server, database, Kafka), send in an input and verify the outputs. There are tools available, such as [Testcontainers for Java](https://www.testcontainers.org) to help run your application in a short-lived test mode with dependencies, such as databases, message queues or web servers running in Docker containers. 

If you set up specific infrastructure components in a test setup stage, you can isolate them from the actual tests. Replacing the database from PostgreSQL to NoSQL, for example, can then be achieved without having to modify the actual test methods. The only thing that would need to change is the test setup. 

> This is leaning towards [Hexagonal Architecture](https://en.wikipedia.org/wiki/Hexagonal_architecture_(software)), which is a good approach to decouple the infrastructure and domain logic. 
> 
> Testing strategy, however, differs from the approach explained here, and you can test your domain in a completely isolated way. 
> 
> There is a cost to it as it adds some complexity, but I have seen codebases where the benefits were worth it. In the end it's a compromise; how many times do you expect to change the infrastructure of your service? Is the added complexity worth it?  

Having most of your microservice covered by integration tests will also result in a test suit not having very many tests. That has a benefit of clearly defining the expected behaviour of a microservice, with tests. Looking at a test suit should give you a clear picture of what the microservice is supposed to do.

### There still is a place for implementation details tests

You will have parts of the code that are domain specific and only contain business logic. Those naturally isolated parts have an internal complexity of their own and this is where Implementation details tests should be used. Testing all variations and edge cases will be cumbersome and too heavy to test through integration tests. 


### References
- [Spotify: Testing of Microservices](https://engineering.atspotify.com/2018/01/testing-of-microservices/)
- [Testcontainers for Java](https://www.testcontainers.org)