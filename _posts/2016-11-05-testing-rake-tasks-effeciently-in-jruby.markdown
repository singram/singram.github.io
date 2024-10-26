---
layout: post
title: Testing rake tasks effeciently in JRuby
date: '2016-11-05 18:22:06'
tags:
- jruby
- rake
- integration-testing
---

After writing [cucumber\_characteristics](https://github.com/singram/cucumber_characteristics) to profile where a large [JRuby](http://jruby.org/) cucumber integration suite was taking it's time it soon became apparent that 30% of the time was wrapped up in testing rake tasks.

This is particularly challenging in JRuby as the usual approach to testing rake tasks is to execute them in a new shell process, capturing the system and testing any mutative effects.  Something like;
```language-ruby
When /^I run the rake task$/ do |count|
  @output = `rake some_task`
end
```
This approach is problematic specifically when using JRuby.  The above invocation needs to start up a completely new JVM per test taking several seconds each time.  In the test suite I was working with, this alone accounted for **45 minutes** on an average developer machine.

Clearly a significant problem event with Continuous Integration.

To tackle this I wrote [cucumber\_rake\_runner](https://github.com/singram/cucumber_rake_runner) which executes a rake task in the same JVM process as the integration test, negating the cost of spinning up a new JVM process per rake test.  The original invocation simply becomes.

```language-ruby
When /^I run the rake task$/ do |count|
  @output = run_rake_task('some_task')
end
```
The @output captures `STDIN`, `STDOUT` and timing information.

This was immensely useful to the project and team I was working with, reducing the time to run the full integration suite by over 30%. Hopefully this will be useful to you as well.

