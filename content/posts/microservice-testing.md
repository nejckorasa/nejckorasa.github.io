---
title: "How to (and not to) test microservices"
date: 2022-11-01
tags: ["Java", "Testing", "Microservices"]
draft: true
categories: Software Engineering
---

Building backend systems today will likely involve building many (micro) services that have to communicate and coordinate with each other, thus making a system distributed. There are a lot of blog posts about microservice architecture, its dos and dont's, when to use it and when you might be better off building a monolith, or at least starting with one and evolve the architecture as you scale up.

I want to expand more on the testing of microservices. What makes testing microservices different from testing a monolith. What are different testing strategies. And why I believe that "best testing practices" have fundamentally changed with introduction of microservices.

In the past few years I've had an opportunity to work in a few different projects and companies, from building new digital banks from scratch to migrating older systems towards microservices as they were scaling up. I've picked up some good practices along the way, I think. But I have also found myself, quite a few times, in disagreements about what testing strategies should be employed to microservices. And so, I've tried to capture and summarise my learnings from all the discussions, read on if you want to find out :). 

### Testing strategies
### Unit tests? Ok, but what is a unit?
### Component tests (Integrated tests)
### Summary
### References