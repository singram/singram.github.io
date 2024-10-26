---
layout: post
title: Joy and pain with @Scheduled and @RefreshScope in SpringBoot
date: '2016-11-07 21:39:39'
tags:
- springboot
- scheduling
---

| **TLDR;** `@Scheduled` and `@RefreshScope` are powerful tools but do not work out of the box together causing dangerous inconsistencies.  Find out how to get them to play nicely and more advanced scheduling opporunities.

So I'm a fan of the [SpringBoot](https://projects.spring.io/spring-boot/) framework and it's adoption of the [Rails](http://rubyonrails.org/) "convention over configuration" as well as it's powerful and simple annotations.  All of which make a developers life more productive and allow the team to focus on delivering functional value to end users.

One of those annotations is `@Scheduled` which can be applied to any bean method.  Only two pieces of code are required;  

1\. the enabling of scheduling in the application, typically achieved with something like the following; (note the use of `@EnableScheduling` 
```language-java
@SpringBootApplication
@EnableScheduling
public class ExampleApplication {
  public static void main(String[] args) {
    SpringApplication.run(ExampleApplication.class, args);
  }
}
```
2\. And the actual method you want repeated.  For instance;
```language-java

@Slf4j
@Configuration
public class Followers {

  @Value("${follower.count:10}")
  private int followers;

  @Scheduled(fixedRate = 4000)
  public void outputFollowers() {
    log.info("===========> Followers " + followers);
  }
}
```
In this example a count of followers is autowired from the [spring property hierarchy](http://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-external-config.html) and outputted to the log every 4 seconds as per the `@Scheduled` declaration.  If you're wondering where the `log` object is defined, this is part of the magic of [Project Lombok](https://projectlombok.org/) which abstracts a lot of standard Java boiler-plating and is provided by the `@Slf4j` annotation on the class.

`@Scheduled` is a very powerful and simple annotation.  Common forms of it include;

- **Fixed rate** - Repeat every x milliseconds. `@Scheduled(fixedRate=1000)`
- **Fixed delay** - Repeat every x milliseconds between previous completion and next start. `@Scheduled(fixedDelay=1000)`
- **Crontab** - Defines a cadence using the same expressions as *nix crontab definitions.  `@Scheduled(cron="0 0 * * * *")`


A great introduction to this topic can be found [here](http://howtodoinjava.com/spring/spring-core/4-ways-to-schedule-tasks-in-spring-3-scheduled-example/).

So at this part in the article we change track over to automatic property updating.  As previously mentioned, SpringBoot has a extensive [property hierarchy](http://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-external-config.html) but no way natively to refresh properties should they change.  Enter [Spring Cloud Config](https://cloud.spring.io/spring-cloud-config/).  The main focus of this project is to establish a centralized repository of configuration for a distributed set of applications and have those applications update automatically if properties change.  A great introduction to externalized properties and centralized property management with Spring Cloud Config can be found [here](https://spring.io/blog/2015/01/13/configuring-it-all-out-or-12-factor-app-style-configuration-with-spring).

Spring Cloud Config automatically provides a JMX interface and a HTTP interface (`\refresh`) to refresh all properties in the application in classes marked with the `@RefreshScope` annotation.  Meaning if the external property source changes, all you have to do is hit `\refresh` on your application and the configuration changes are automatically pulled in.

And for the most part this works pretty nice and seamlessly as you would expect... except it doesn't work with the original `@Scheduled` code example at the start of the article.  In fact everything but `@Scheduled` annotated code will refresh causing system inconsistencies within an application.  The problem here is that the scheduled method (`outputFollowers()`) has a dependency on a property injected by the Spring framework and even when refreshed the property change is not propagated down into the scheduled code.  A discussion on this can be found in [common Spring Cloud issues](https://github.com/spring-cloud/spring-cloud-commons/issues/97) .

The solution, rather than relying on the magic of the `@Scheduled` annotation, is to specify the scheduling configuration manually.  For example;

```language-java
@SpringBootApplication
public class ExampleApplication {
  public static void main(String[] args) {
    SpringApplication.run(ExampleApplication.class, args);
  }
}
```
```language-java

@Slf4j
@RefreshScope
@Configuration
public class Followers {

  @Value("${follower.count:10}")
  private int followers;

  public void outputFollowers() {
    log.info("===========> Followers " + followers);
  }
}
```

```language-java
@Configuration
@EnableScheduling
public class SchedulingConfiguration implements SchedulingConfigurer {

  @Autowired
  Followers followers;

  @Override
  public void configureTasks(ScheduledTaskRegistrar taskRegistrar) {
    taskRegistrar.addTriggerTask(
        () -> followers.outputFollowers(),
        triggerContext -> {
          Instant nextTriggerTime = Instant.now().plus(4, SECONDS);
          return Date.from(nextTriggerTime);
        });
  }
}
```

Tackling the problem this way now yields an application that refreshes properties throughout on demand and consistently.

While initially this is somewhat of a pain, it is a solution first and foremost but it also enables more complex scheduling.

For instance you could build a trigger context that dynamically calculates the next time to run, potentially throttling an action per time period. See [here](http://stackoverflow.com/questions/14630539/scheduling-a-job-with-spring-programmatically-with-fixedrate-set-dynamically) for an example.

Scheduled code by default is limited to a single thread meaning there is an assumption that the previous call should be finished before the next execution.  When this assumption is incorrect a execution thread pool is necessary which again can be manually configured such as;

```language-java
 @Configuration
 @EnableScheduling
 public class AppConfig implements SchedulingConfigurer {

     @Override
     public void configureTasks(ScheduledTaskRegistrar taskRegistrar) {
         taskRegistrar.setScheduler(taskExecutor());
     }

     @Bean(destroyMethod="shutdown")
     public Executor taskExecutor() {
         return Executors.newScheduledThreadPool(100);
     }
 }
```
Further reading can be found on the [@EnableScheduling](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/scheduling/annotation/EnableScheduling.html) java docs.

You can of course do far more than just the simple examples above but this should be enough to firstly resolve any problems between the SpringBoot `@Scheduled` annotation and the live configuration updates you can attain by incorporating the `@RefreshScope` annotation from the Spring Cloud Config project.

Enjoy.








