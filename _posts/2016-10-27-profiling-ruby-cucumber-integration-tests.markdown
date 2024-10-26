---
layout: post
title: Profiling ruby cucumber integration tests
date: '2016-10-27 18:26:38'
tags:
- ruby
- cucumber
- integration-testing
---

I've been working on a large JRuby project for a number of years now with over a thousand integration tests.  With so many tests it's important to know where time is being spend to see if there are optimizations that can be made to improve the overall performance of the test suite.  Locating highly utilized steps and dead code is also a normal part of code curation.

With this in mind, I wrote the [cucumber_characteristics](https://github.com/singram/cucumber_characteristics) gem.  The gem requires very little configuration and should work transparently with your existing setup.  Essentially it is a formatter and should drop into your existing setup transparently generating a html and/or json reports. Installation and usage instructions can be found on github [here](https://github.com/singram/cucumber_characteristics)

For each cucumber step definition executed the following is reported;

* Location of definition & regex
* Step usage location and number of times executed (background/outline etc)
* Counts for success/failure/pending/etc
* Total time taken in test run along with average, fastest, slowest times per step

For each feature test, the following is reported;

* Location and time taken to run feature
* Result and number of steps run
* Breakdown of feature by individual example run if a Scenario Outline.

There is also added support to list out all unused steps in a cucumber test run to aid step curation.   Be aware if you are only running a specific test set, for example via a TAG as you will get a larger number of unused steps than are not ‘true’ unused steps.

The gem supports ruby 1.9+ and cucumber 1.x & 2.x

Hope this is useful.  Please get in touch if there are further enhancements that would be useful or better yet submit a pull request.