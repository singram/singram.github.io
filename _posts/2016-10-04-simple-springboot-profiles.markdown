---
layout: post
title: Simple SpringBoot profiles
date: '2016-10-04 17:31:39'
tags:
- springboot
- gradle
- flyway
---

> **TLDR** SpringBoot profiles, what they are and how to use them simply with flyway example.

Many frameworks have the concept of scoping application settings together around the concept of environments, examples being; dev, test, stage & production.  Largely what you scope around is irrelevant but these examples are the most common to scope around.

So what do I mean and how is this useful?  Well for local development you probably want the local database credentials in your application properties, you may also have threads turned down or certain services disabled.  Whatever is most appropriate to facilitate and accelerate local development for you and your team.  Clearly the application settings you run against in production will be different from both performance, debugging and security perspectives and should thus be managed separately.

The [12 Factor Application](https://12factor.net) manifesto also has a great read on configuration management from a different perspective which is well worth your time, [here](https://12factor.net/config)

SpringBoot supports this concept in the form of *[profiles](http://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-profiles.html)* which are also analogous to the Ruby on Rails runtime environments (see [here](http://guides.rubyonrails.org/configuring.html#creating-rails-environments)).

**How do I use profiles?**

Very simply a SpringBoot's default application properties are specified by `src/main/resources/application.properties`.  Profile specific properties can be specified in the same file or by a separate file of the following format `src/main/sources/application-<profilename>.properties` such as `src/main/sources/application-dev.properties`.  Profile properties override the default properties in much the same way CSS does.  Profile properties do not need to specify all properties, only the ones which you wish to change from the default `application.properties` set.

Assuming you have [Actuator](http://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#production-ready) in your class path (if not, why not?!?), any `info.*` properties are exposed through the `\info` endpoint.  This makes it especially useful for exposing build & release information as well as the profile under which the application is running.

For instance in `application.properties` you may have
```language-properties
info.profile=default
spring.jackson.serialization.write-dates-as-timestamps=false
management.context-path=/actuator
```
and in `application-dev.properties` you may have
```language-properties
info.profile=dev
```
Meaning that when running your application with the `dev` profile enabled that the `\actuator\info` endpoint will yield something like 
```language-json
{
  "profile": "dev"
}

```

**So how do you pass in the desired profile to your SpringBoot application?**  Very simply;
```
$ SPRING_PROFILES_ACTIVE=dev gradle bootRun
```
Just like any SpringBoot property there's a hierarchy down which it searches (see [here](http://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-external-config.html)), so this doesn't have to be a environment variable (useful for a Containerization strategy) but could also be a property in your default `application.properties` file or specified some other way.

**Flyway example**

Recently a problem arose at work where default development data needed to be automatically loaded into all local development environments but only local development environments.  Using [Flyway](https://flywaydb.org/) to manage database migrations and data assets this became trivial to implement with the help of profiles.

Schema migrations were located in the default `resources/db/migrations` location and development environment specific migrations/data assests located in `resources/db/dev`

With the database files in place all that was needed was a `application-dev.properties` file with
```language-properties
info.environment=dev
flyway.locations=classpath:db/migration,classpath:db/dev
```
This did two things;

- publish the runtime profile to the `\info` endpoint provided by actuator
- overrode the default locations flyway examines for migrations and callback files to include both the standard schema migrations as well as any `dev` environment specific files.

One item of note is that in this particular case, the development need was for default data.  With this in mind while the data file(s) could be versioned using the standard versioned Flyway [naming schema](https://flywaydb.org/documentation/migration/sql) this requires some consideration to make sure that the versions of the dev data assets and schema migrations do not clash.  Flyway also supports callbacks which are the perfect solution to this problem (see [here](https://flywaydb.org/documentation/callbacks)).  In particular the `afterMigrate` hook.  Be aware to make your migration idempotent as it will run every time on startup, regardless of the number of migrations executed.

Simple when you know how, but sometimes the documentation isn't that transparent.  Hope this helps.  A full working example can be found on github [here](https://github.com/singram/spring-boot-profiles).