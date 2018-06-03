---
layout: post
title: On Unit Tests
keywords: software development, tests, TDD, unit tests, do you need tests, tests are useless, why to write unit tests, seva zaikov, bloomca
---

In software development it is very common to see two following extreme approaches to unit tests:

<h5>
1. Tests are not needed <em>at all</em>
</h5>

A lot of software has no tests at all, and it is not a secret. Some (sometimes quite popular) libraries, products, features, etc – it might lack tests, but still work. Why is it so?

Possible theories:

- it was written by one person
- it was written in a short timespan, while everybody was in the context. If people return to it after some time, they just rewrite it
- just by accident (no time in the beginning, later it was just working)

<h5>
2. Everything should be covered with tests on 100%
</h5>

Another extreme is to cover _everything_, so that you have 100% coverage (yes, even on getters/setters), and any smaller number means that build won't even run. Any change in the codebase requires to rewrite some chunk of tests as well, which adds cost to maintainability.

## Build Statuses

Let's talk about build statuses and what does it actually mean:

- **green build** is usually perceived as perfectly working solution, and ready to deploy to production. However, it does not mean anything – it only means that tests passed. They can have mistakes inside, some edge cases might be omitted, or maybe even some primary functionality was not covered (new feature without tests usually won't break existing tests).
- **red build** does not mean that everything is bad. Maybe some tests are incorrect, maybe they are obsolete now, maybe platform has changed, or thousands other "maybe". You also might fix some functionality, and some tests don't pass anymore!

So, build colour gives us only probability – green build is likely to work properly, and red build probably has some problems, but it is not really guaranteed: there are situations when you can decide to check something after having green build and actually deploying to production a red build.

## Business Value

Very often you might hear that tests bring value to business, that they always pay off. This is a pretty arguable statement, which generalizes all possible cases.

In reality, we need to:
- write tests (it takes time)
- maintain them (depending on granularity, you can spend a lot of time on it)
- they can't guarantee full correctness

To summarize, even 100% coverage won't give you full confidence, but will definitely require a lot of effort to write and maintain – because of that, rule "X%" coverage usually is not the best.

## What Tests Give

Tests can't prove that the program has no bugs, only that it has bugs (which also can be bugs in tests itself). They prove that _some_ contract is correct, that certain function/modules/programs with some input will output certain result. It does not guarantee it for all input, and almost always we have no way to verify for all possible inputs: we don't know what user will send us with POST request, or what we get from a REST service.

Does it mean that tests are not needed? Not really – more often they are needed than not. If there are a lot of people working on the same codebase, or you plan to do some refactoring, it'd be hard to understand that nothing has broken. Otherwise, if you write a _small_ service which will stay untouched for years, probably tests are not so crucial. Or you work on something alone and don't plan to have contributors – how often do you write unit tests for small utility functions or bash scripts?
