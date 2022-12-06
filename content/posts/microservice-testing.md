---
title: "Don't couple you tests to implementation details"
date: 2022-11-01
tags: ["Testing", "Microservices", "Software Architecture"]
draft: true
categories: Software Engineering
---

Building backend systems today will likely involve building many (micro) services that have to communicate and coordinate with each other, thus making a system distributed. There are a lot of blog posts about microservice architecture, its Dos and Don'ts, when to use it and when you might be better off building a monolith, or at least starting with one and evolve the architecture as you scale up.

I want to expand more on the functional testing of microservices and what makes it different. Why I believe that _"best testing practices"_ have fundamentally changed with the introduction of microservices and why testing _pyramids_ are generally wrong and a more appropriate metaphor to be thinking of is a [honeycomb](https://engineering.atspotify.com/2018/01/testing-of-microservices/). 

In the past few years I've had an opportunity to work in a few different projects and companies, from building new digital banks to migrating older systems towards microservices as they were scaling up. I've picked up some good practices along the way, I think. But I have also often found myself (and I still do) in disagreements about what testing strategies should be employed to microservices. And so, I've tried to capture and summarise my learnings from all the discussions, read on if you want to find out :).

### Why do we have tests?

We test our code to verify that things work the way we expect them to. We also want our tests to make maintenance easier, and this is something that is commonly overlooked. Testing should support refactoring but many times, it makes it harder. Tests should aim to define what is intended (behavior); not how it’s implemented (details).

### Pyramids and Honeycombs 🍯
- reference spotify
- difference, less unit tests
- don't add any value, have to change tests when refactoring, have you actually tested anything?

### Unit tests? Ok, but what is a unit?

### Mocking or stubbing
- avoid mocks, use stubs. Difference
- 

### Integration tests

### Hexagonal architecture

### Summary

- Do you agree? 


### References